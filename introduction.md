# Event Sourcing in Data-driven Domain Modeling

*A modern approach to domain modeling using hypermedia as a state of application.*

#### Authors
Buğra Ekuklu, [[github](http://github.com/Chatatata)], Distributed Systems Developer

Copyright (c), 2017.

#### Revision history

- *17.09.25*, Buğra Ekuklu: *First draft*.

## Preface

With the advent of the **OLTP** systems, changes made to the state of the
application could be stored in a different manner, which would make
analysis over data possible if carried in to **OLAP** systems.
Database sharding, data replication, partial eventual consistency support in
immediate consistent atomic databases, built-in multi-tenancy support features
made **OLTP** solutions easier to integrate into big data applications, without
too much hassle.
Analytical systems care about the current - valid data, furthermore, they also
care about its history in order to produce time-series analytics, which cares
significance in management and execution of business.

## Implementation

Event sourcing methodology changes usage of persistence back-end found in
**OLTP** subpart of the software solution.
The following parts of this section will make a superficial view of the
methodology.

### Traditional approach

Assume that we need to build a software solution for a bank, barely making
transactions between its own accounts for the sake of simplicity.
Every user should have a balance, and they could withdraw/deposit some money.
A withdrawal operation would reflected to user's balance as a negative change,
and a deposit operation would reflected to user's balance as a positive
change.
Henceforth, we could create a table for the users, `users` like so:

| **id** | name | balance | inserted_at |
| -: | :-: | :-: | ----------- |
| 1 | Hayden Panettiere | 24.72 USD | 2017-08-28 00:22:14.960193 |
| 2 | Megan Fox | 25.14 USD | 2017-08-28 00:23:14.123512 |

When *Hayden* wants to withdraw some bucks, we could do that with following
query in *PLpgSQL*:

```sql
UPDATE "users"
SET "balance" = "balance" - ('20.00 USD')::money
  WHERE "id" = 1 AND "balance" >= ('20.00 USD')::money;
```

That would decrement *Hayden*'s balance by *20.00 USD*.
Similarly, deposit operation could be performed with a simple `UPDATE` query.

However, when it comes to transfer, following we need to perform a simple
transaction.
We would like to transfer some funds from *Hayden* to *Megan*.

```sql
BEGIN
  DECLARE
    affected_rows int DEFAULT 0;
  BEGIN
    BEGIN TRANSACTION "transfer-VEVJW9WrThixnQr9vmR274LgltOC5EQoh5CYq2";

    SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

    UPDATE "users"
    SET "balance" = "balance" - ('20.00 USD')::money
      WHERE "id" = 1 AND "balance" >= ('20.00 USD')::money;

    affected_rows := SQL%ROWCOUNT; -- Get number of affected rows

    IF affected_rows = 1 THEN
      UPDATE "users"
      SET "balance" = "balance" + ('20.00 USD')::money
        WHERE "id" = 2;

      END TRANSACTION "transfer-VEVJW9WrThixnQr9vmR274LgltOC5EQoh5CYq2";
    ELSE
      ROLLBACK;
    END IF;
  END
END
```

Of course, this is not an elegant solution.
However, it shows a canonical way of doing it correctly.

Finally, we could get the current balance of *Hayden* with:

```sql
SELECT "balance"
FROM "users"
  WHERE "id" = 1;
```

#### Advantages

- Easy, straightforward implementation.
- Performant query.

#### Disadvantages

- Change history is not stored.
- There would be no rollback action other than transaction-level rollback.

### The "Event Sourcing" approach

Traditional approach tries to store results of operations.
In *event sourcing* approach, we will store events (or artifacts) instead of
results.
With a powerful query language, like *PLpgSQL*, we could calculate results of
operations from those events.
A normalization could be performed in order to create required tables.

`users` table:

| **id** | name | inserted_at |
| -: | :-: | --- |
| 1 | Hayden Panettiere | 2017-08-28 00:22:14.960193 |
| 2 | Megan Fox | 2017-08-28 00:23:14.123512 |

`user_balance_changes` table:

| **id** | user_id | delta | inserted_at |
| -: | :-: | :-: | --- |
| 1 | 1 | + 30.00 USD | 2017-08-28 01:22:14.960193 |
| 2 | 2 | + 20.00 USD | 2017-08-28 01:23:14.123512 |
| 2 | 2 | - 5.00 USD | 2017-08-28 01:24:14.123512 |

`user_transfers` table:

| **id** | source_user_id | target_user_id | amount | inserted_at |
| -: | :-: | :-: | :-: | --- |
| 1 | 1 | 2 | + 2.50 USD | 2017-08-28 01:22:14.960193 |

Let's start from simple one, getting current balance information of a user.

Again, using `PLpgSQL`, we could calculate balance of *Hayden* using an
aggregate function.

```sql
WITH (
  SELECT sum("delta")
  FROM "user_balance_changes"
    WHERE "user_id" = 1;
    GROUP BY "user_id"
) "balance_sum_aggregate", (
  SELECT sum("amount")
  FROM "user_transfers"
    WHERE "source_user_id" = 1
    GROUP BY "source_user_id"
) "incoming_transfers_sum_aggregate", (
  SELECT sum("amount")
  FROM "user_transfers"
    WHERE "target_user_id" = 1
    GROUP BY "target_user_id"
) "outgoing_transfers_sum_aggregate"
SELECT "user_balance_changes"."sum" + "incoming_transfers_sum_aggregate"."sum" - "outgoing_transfers_sum_aggregate"."sum";
```

Much more complicated.

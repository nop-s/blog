# Let's talk about the locking clause FOR UPDATE SKIP LOCKED

A short example of how you can use `SELECT … FROM … WHERE … FOR UPDATE SKIP LOCKED`. The locking clause `FOR UPDATE SKIP LOCKED` has made the news recently with MySQL 8.0[^1] joining a cohort of real databases by finally supporting this lock strength. MariaDB has a ticket to implement this feature as well[^2]. In this example, if you follow along, I've tested the code snippets with PostgreSQL 10. The ideas presented will work for all systems that implement this particular locking clause correctly.

The locking clause FOR UPDATE SKIP LOCKED[^3] is a useful way to skip over locked records in your predicate set at the expense of an inconsistent world view. An inconsistent view of the world is of course the bane of all real database systems.

For this demostration we will use a toy example, if you're serious about needing a queue, please do the needful and checkout the available tooling to see if those can fill your needs first, one such tool is `pgq`[^4].

# Our example schema
```sql
DROP TABLE IF EXISTS sad_queue CASCADE;
CREATE TABLE sad_queue (
        internal_id UUID DEFAULT uuid_generate_v4() NOT NULL,
        external_id UUID DEFAULT uuid_generate_v4(),
        processing_host INET DEFAULT NULL,
        started_at TIMESTAMP DEFAULT NULL,
        completed_at TIMESTAMP DEFAULT NULL,
        created_at TIMESTAMP DEFAULT NOW() NOT NULL,
        PRIMARY KEY (internal_id),
        UNIQUE(external_id)

);
COMMENT ON COLUMN sad_queue.internal_id IS 'The reference identifier to uniquely identify this record internally';
COMMENT ON COLUMN sad_queue.external_id IS 'The reference identifier to uniquely identify this record externally';
COMMENT ON COLUMN sad_queue.processing_host IS 'The address of the host that is processing this record';
COMMENT ON COLUMN sad_queue.started_at IS 'The timestamp relating to when this record was selected for processing';
COMMENT ON COLUMN sad_queue.completed_at IS 'The timestamp at which the record processing finished';
COMMENT ON COLUMN sad_queue.created_at IS 'The timestamp at which the record was entered into the sad_queue';
```

A quick aside, it is a good practice when sharing identifiers to have both an external and internal identifier. In this example I use`external_id` and `internal_id`. If you're ever going to share an identifier, make it an identifier that is impossible to predict. No matter what the type of data is, you do not want someone being able to iterate through your set.

Let's add some records to be processed to our sad queue:
```sql
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
```
We do each of these outside of a transaction for brevity and additionally we want the `created_at` fields to be distinct. We could use `clock_timestamp()`[^6] instead of `now()` to generate the timestamp but we'll cover time another time.

List all your beautiful newly created records:
```sql
SELECT * FROM sad_queue;
```
```
             internal_id              |             external_id              | processing_host | started_at | completed_at |         created_at
--------------------------------------+--------------------------------------+-----------------+------------+--------------+----------------------------
 2c90c75d-1602-4ea1-b3f8-5ffac23a76d8 | 3a07369a-16c4-416a-99f4-d5694401a41f |                 |            |              | 2018-04-22 17:15:25.079527
 f000c356-e3a5-48cd-998c-9d96d13d984f | 1c3abf41-a676-4004-9290-d495064a779f |                 |            |              | 2018-04-22 17:15:25.088598
 628d3c49-f2b0-443d-8d2a-92f8081a2647 | e32eb332-269a-493c-bd2b-141c86d1195d |                 |            |              | 2018-04-22 17:15:25.090348
 80df8d44-6229-4e31-97ff-98de86db21db | 0da15373-72bd-478f-afda-cb7de63815b4 |                 |            |              | 2018-04-22 17:15:25.091879
 96036856-5f44-4d8c-ba06-098bfb6bcbe2 | a07b5c5f-f7d8-428d-ac07-71d57466c43c |                 |            |              | 2018-04-22 17:15:25.093533
(5 rows)
```

A naive implementation to retrieve a record for processing:
```sql
BEGIN;
UPDATE sad_queue SET started_at = NOW(),
                     processing_host = '192.168.0.2' -- Obviously this should be dynamically set to the actual host
WHERE 1=1
      AND started_at IS NULL
      AND completed_at IS NULL
      AND internal_id = ( SELECT internal_id
                 FROM sad_queue
                 WHERE 1=1
                       AND started_at IS NULL
                       AND completed_at IS NULL
                 ORDER BY created_at ASC
                 LIMIT 1
                 FOR UPDATE
      )
RETURNING internal_id;
COMMIT;
```

The retrieval query above will retrieve records, however due to how databases isolaton levels work the interaction of multiple workers may not be what you would expect. The sub-select will find the oldest record the satisfies the predicate `started_at` and `completed_at` with a value of NULL and return one of the qualify records advising the database system that the selected record is locked for a pending update. By advising the database of this lock, future queries that need access to that particular record will block, they need to block so that the isolation level `READ COMMITTED` is not violated because two potential updates may modify the same record. In this specific case, the database does not know that outside conditions dictate that the multiple queries are not looking for the same record but simply any available record.

This blocking behaviour is probably not an attribute we would like to preserve or that we care about should we want to distribute work as quickly as possible in our toy queue.

Since we are making an artisanal queue, blocking seems like a poor strategy in a world with more than one workers, especially if we would like to "_go fast_".

## Enter SKIP LOCKED
We can use locking clauses to inform the database of what level of consistency we are willing to accept. To do that we update our SELECT statement to inform the database that we have no interest in any locked records that fit our predicate, which is safe for the artisanal queue implementation, since we're looking for records that are available to be processed and locked records are by their very nature in an unavailable state.

We  therefore inform the database of this adding `SKIP LOCKED` to our locking clause. The updated query might look something like:
```sql
-- Now with SKIP LOCKED
BEGIN;
UPDATE sad_queue SET started_at = NOW(),
                     processing_host = '192.168.0.2' -- Obviously this should be dynamically set to the actual host
WHERE 1=1
      AND started_at IS NULL
      AND completed_at IS NULL
      AND internal_id = ( SELECT internal_id
                 FROM sad_queue
                 WHERE 1=1
                       AND started_at IS NULL
                       AND completed_at IS NULL
                 ORDER BY created_at ASC
                 LIMIT 1
                 FOR UPDATE SKIP LOCKED
      )
RETURNING internal_id;
COMMIT;
```

Resulting potential table state:
```
SELECT * FROM sad_queue ;
             internal_id              |             external_id              | processing_host |         started_at         | completed_at |         created_at
--------------------------------------+--------------------------------------+-----------------+----------------------------+--------------+----------------------------
 80df8d44-6229-4e31-97ff-98de86db21db | 0da15373-72bd-478f-afda-cb7de63815b4 |                 |                            |              | 2018-04-22 17:15:25.091879
 96036856-5f44-4d8c-ba06-098bfb6bcbe2 | a07b5c5f-f7d8-428d-ac07-71d57466c43c |                 |                            |              | 2018-04-22 17:15:25.093533
 2c90c75d-1602-4ea1-b3f8-5ffac23a76d8 | 3a07369a-16c4-416a-99f4-d5694401a41f | 192.168.0.2     | 2018-04-22 17:16:44.993914 |              | 2018-04-22 17:15:25.079527
 f000c356-e3a5-48cd-998c-9d96d13d984f | 1c3abf41-a676-4004-9290-d495064a779f | 192.168.0.2     | 2018-04-22 17:17:14.597212 |              | 2018-04-22 17:15:25.088598
 628d3c49-f2b0-443d-8d2a-92f8081a2647 | e32eb332-269a-493c-bd2b-141c86d1195d | 192.168.0.2     | 2018-04-22 17:18:18.559371 |              | 2018-04-22 17:15:25.090348
(5 rows)
```

Now every worker that uses this query to retrieve records from the table will do so in FIFO-ish ordering without blocking on any record being locked by another worker.

# What if _something_ fails?

What do we do if _something_ fails? There are a couple obvious failure cases, the in flight transaction fails, a worker fails once a record is updated, and the worker completes but does not mark the record as done.

In the case of the transacton failing the transaction would be rolled back and the next time the table is queried the record will be present and available for processing.

The worker failing during the processing of a record might leave the record in the processing state and an external decsion must be made in order remedy the record state, think garbage collection.

The worker completing the work but not updating the record so other workers know the work is completed is almost the same as the worker failing.
A particularly nasty gotcha to look out for is the completed work may have made its way downstream, not such a big problem if processing is idempotent[^5]. A property you've designed into all of your distributed systems have right?

One potential soluton to these problem is to hold the transaction open for the duration of processing, on failure the system must ensure that the transaction is aborted. However, should the database itself become unavailable, _all_ workers will need to reassemble their state into the failover database exactly as it was or throw away all unrecorded work and start again. In the former this is a particularly nasty problem as workers seeking new work will compete with workers attempting to restore state to the database. In the latter case, all work is throw away in order to restore synchronization.

Of course, as we now consider these cases we enter into depths of why you should use a specific tool for the job were someone else has already considered these details for you…


[^1]: [https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html](https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html)

[^2]: [https://jira.mariadb.org/browse/MDEV-13115](https://jira.mariadb.org/browse/MDEV-13115)

[^3]: [https://www.postgresql.org/docs/10/static/sql-select.html#SQL-FOR-UPDATE-SHARE](https://www.postgresql.org/docs/10/static/sql-select.html#SQL-FOR-UPDATE-SHARE)

[^4]: [https://wiki.postgresql.org/wiki/PGQ_Tutorial](https://wiki.postgresql.org/wiki/PGQ_Tutorial)

[^5]: [https://en.wikipedia.org/wiki/Idempotence](https://en.wikipedia.org/wiki/Idempotence)

[^6]: [https://www.postgresql.org/docs/9.1/static/functions-datetime.html](https://www.postgresql.org/docs/9.1/static/functions-datetime.html)
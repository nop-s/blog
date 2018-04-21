# Let's talk about FOR UPDATE SKIP LOCKED by making a sad queue
A short example of how to use FOR UPDATE SKIP LOCKED, SKIP LOCKED is a nice way to select an unlocked record in your predicate set, useful for selecting items to process or for making something you should really use another package like `pgq` <sup>[1](#pgq-url)</sup>

# Our schema
```SQL
DROP TABLE IF EXISTS sad_queue CASCADE;
CREATE TABLE sad_queue (
        internal_id UUID DEFAULT uuid_generate_v4() NOT NULL,
        external_id UUID DEFAULT uuid_generate_v4(),
        -- processing_host INET DEFAULT NULL, -- So you can follow up on the jobs
        started_at TIMESTAMP DEFAULT NULL,
        completed_at TIMESTAMP DEFAULT NULL,
        created_at TIMESTAMP DEFAULT NOW() NOT NULL,
        PRIMARY KEY (internal_id),
        UNIQUE(external_id)

);
```
An important aside to notice is the use of `external_id` and an `internal_id`. If you're ever going to share an identifier, make it an identifier that is hard if not impossible to predict. No matter the type of data or lack there of, you do not want someone being able to iterate through your item set.

Let's add some items to process:
```SQL
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
INSERT INTO sad_queue(internal_id) VALUES (uuid_generate_v4());
```
We do this outside of a transaction for brevity and because we want the `created_at` fields to not be same.

List all your beautiful newly created records:
```SQL
SELECT * FROM sad_queue;
```
```
             internal_id              |             external_id              | started_at | completed_at |         created_at
--------------------------------------+--------------------------------------+------------+--------------+----------------------------
 8e92d4b4-4441-4eff-ab23-178528ccbc44 | cbc720eb-f492-4d11-9f38-ffe42760c6c1 |            |              | 2018-04-21 16:30:56.155629
 fa925674-f0ca-4e7f-8e41-07f2851a556d | a56a4a27-1a8a-49f2-911c-05d1e5df2937 |            |              | 2018-04-21 16:30:56.172676
 6a0297a6-d7cd-42a3-827b-c1d81f0f4637 | 32c5823f-f789-4e0e-b8fc-f11517db8480 |            |              | 2018-04-21 16:30:56.184162
 9bc4a995-3d39-4897-a37a-36c699bca0f8 | 5490f9d5-819b-4cd4-97ca-9522cf9c6290 |            |              | 2018-04-21 16:30:56.186535
 05850458-06e2-4ca6-89d6-02a96f2bd947 | 58c64ecb-a684-4fca-a895-b74a98511c51 |            |              | 2018-04-21 16:30:56.577685
(5 rows)
```

The naive implementation to retrieve a record for processing might look something like:
```SQL
BEGIN;
UPDATE sad_queue SET started_at = NOW()
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

The retrieval query above will do the work, it will retrieve items however due to how databases work will not perform as you might expect. The sub-select will find the oldest record with `started_at` or `completed_at` set to NULL and reutrn one of those records advise the database system that the selected record is locked for an pending update, which in the worst case may never actually materialize. The database will then ensure that future transactions are advised of this and they will act accordingly, which in this default Postgresql transaction isolation level will result in blocking of all SELECTs on the table that will include that record. This behaviour is obviously not a problem with a single processor as no other process will want to access this table that can conflict with the locked record, we are free to continue INSERTing records into the table.

Naturally since we're making an artisanal queue we don't want to block on one processor looking for work or if you're intense, holding a lock on the record for the duration of the processing time.

## Enter SKIP LOCKED
We can update our SELECT statement to inform the database that we have no interest in locked records, which is safe, since we're looking for processes that are available to be processed and locked records are by their very nature in use. 

We inform the database of this by using SKIP LOCKED. It is important to repeat that we are making an explicit choice to forego a property of the database system which we have deemed to be unnecessary for our purposes. The new query might look something like:
```SQL
## Now with SKIP LOCKED
BEGIN;
UPDATE sad_queue SET started_at = NOW()
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
Now each processor that uses this query to retrieve records from the table will do so in FIFO ordering without blocking on any queue being SELECTed by another process.

# What if the SELECT query fails?

If our selection query some how fails in a transaction the SELECTed record would remain untouched, the future state that wasn't committed is wiped out, and the `started_at` and `completed_at` columns would meet the criteria to be picked up by the next processor looking for work. If the processor failed after marking the record, we could have some reaper process check on the status of records and update as necessary. However, now we enter into weeds of why you should use an application in which someone else has already considered these details for you…

<a name="#pgq-url">[1]</a>: https://wiki.postgresql.org/wiki/PGQ_Tutorial</a>
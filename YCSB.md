YCSB workloads generation
-------------------------

- Schema

    * YCSB: Single table with 10 million records (11 columns with 1 primary-key, which is the column number from 1 to 10 million, column and 10 extra columns with valid value in first column only. Details and dump of table created by DBx1000 could be found in folder /img/YCSB_table.dump)

    * **Comments from Xiangyao**:
        1. There are 10 columns but the values are NULL except the first column whose value is "hello". The schema is defined in benchmarks/YCSB_schema.txt. For the concurrency control study in our papers, *the values do not really matter so we didn't model them carefully*. - To achieve atomic operations maybe?
        2. The YCSB workload in DBx1000 is *different from the original YCSB workload*. We made these changes to make the workload more interesting to study concurrency control algorithms.
        3. The schema won't be changed. Actually, the value does not really matter. Right now for YCSB we only *model the writing behavior but always write 0 to the rows*.


- Transaction
    - Table size: config_table_size/partition_count
    - Model: Zipf (0 < theta < 1 is the skew) from paper - Quickly Generating Billion Record Synthetic Databases
    - Query: Single Key/Value query with Zipf-generated uint64 rowid and randomly generated value - rint <= 256
    - Transaction: default 16 queries per transaction. Can be altered
    - GetNextQuery: &queries[q_idx++]/run_txn - Sequential Access for query and transaction

- Storage (in memory)
  
    - Claim: 10GB database with Single table and 10 million records (1024 x 1024 x 10 rows)
    - Reality: 300\~\~500 MB database with default settings since there is no other columns excpet the first one, like key/value pair
    - Each row (tuple) has a single primary key (hash index)
    - Each row has 10 additional column of 100 byte randomly generated string data each (char) rand() % (1<<8) by the paper
    - 10GB = (1024 x 1024 x 10) x (10x100)

- Workflow
    - Warmup the DB by running # transaction(s) first
    - Choose workloads (YCSB/TPCC/TEST)
    - Import schema from benchmarks/YCSB_schema.txt
    - Populate one single table with 10 million records in which the primary key is 1, 2, 3 ... 10m
    - For each row, there is a manager object to handle rts, wts, lock & release
    - Initialize transactions with 16 YCSB queries each
    - For each transaction, there is a int64 txnid and transaction manager
    - Initialize thread list with THREAD_CNT
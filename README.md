DBx1000
=======

DBx1000 is an single node OLTP database management system (DBMS). The goal of DBx1000 is to make DBMS scalable on future 1000-core processors. We implemented all the seven classic concurrency control schemes in DBx1000. They exhibit different scalability properties under different workloads. 

The concurrency control scalability study is described in the following paper. 

    Staring into the Abyss: An Evaluation of Concurrency Control with One Thousand Cores
    Xiangyao Yu, George Bezerra, Andrew Pavlo, Srinivas Devadas, Michael Stonebraker
    http://www.vldb.org/pvldb/vol8/p209-yu.pdf
    
Build, Test & Run
------------

To build the database.

    make -j

To test the database

    python test.py
    
The DBMS can be run with 

    ./rundb


Configuration
-------------

DBMS configurations can be changed in the config.h file. Please refer to README for the meaning of each configuration. Here we only list several most important ones. 

    THREAD_CNT        : Number of worker threads running in the database.
    WORKLOAD          : Supported workloads include YCSB and TPCC
    CC_ALG            : Concurrency control algorithm. Seven algorithms are supported 
                        (DL_DETECT, NO_WAIT, HEKATON, SILO, TICTOC) 
    MAX_TXN_PER_PART  : Number of transactions to run per thread per partition.
                        
Configurations can also be specified as command argument at runtime. Run the following command for a full list of program argument. 
    
    ./rundb -h

YCSB workloads generation with Hash Index
---------------------------------------------

1. Schema

    * YCSB: Single table with 10 million records (11 columns with 1 primary-key, which is the column number from 1 to 10 million, column and 10 extra columns with valid value in first column only. Details and dump of table created by DBx1000 could be found in folder /img/YCSB_table.dump)

    * Comments from Xiangyao:
        1. There are 10 columns but the values are NULL except the first column whose value is "hello". The schema is defined in benchmarks/YCSB_schema.txt. For the concurrency control study in our papers, the values do not really matter so we didn't model them carefully.
        2. The YCSB workload in DBx1000 is different from the original YCSB workload. We made these changes to make the workload more interesting to study concurrency control algorithms.


2. Transaction
    - Table size: config_table_size/partition_count
    - Model: Zipf (0 < theta < 1 is the skew) from paper - Quickly Generating Billion Record Synthetic Databases
    - Query: Single Key/Value query with Zipf-generated uint64 rowid and randomly generated value rint <= 256
    - Transaction: default 16 queries per transaction. Can be altered
    - GetNextQuery: &queries[q_idx++]/run_txn - Sequential Access for query and transaction

3. Storage (in memory)
  
    - 10GB database with Single table and 10 million records (1024 x 1024 x 10 rows)
    - Each row (tuple) has a single primary key (hash index)
    - Each row has 10 additional column of 100 byte randomly generated string data each (char) rand() % (1<<8)
    - 10GB = (1024*1024*10) * (10x100)

4. Hash Index
    g_synth_table_size * 2 Buckets for a low probability of collision

5. Workflow
    - Warmup the DB by running # transaction(s) first
    - Choose workloads (YCSB/TPCC/TEST)
    - Import schema from benchmarks/YCSB_schema.txt
    - Populate one single table with 10 million records in which the primary key is 1, 2, 3 ... 10m
    - For each row, there is a manager object to handle rts, wts, lock & release
    - Initialize transactions with 16 YCSB queries each
    - For each transaction, there is a int64 txnid and transaction manager
    - Initialize thread list with THREAD_CNT

TPCC workloads generation with Hash Index
---------------------------------------------

1. Schema
    * TPCC: 9 tables (Warehouse, Custormer etc.): ![TPCC tables](../master/img/TPCC_tables.png?raw=true)
    * Only Payment and New Order transactions are modeled
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

Summary by YJ - Begin
=====================

1. TPCC worklaods generation: [Payment](PAYMENT.md), [New Order](NEW_ORDER.md)
2. [YCSB workloads generation](YCSB.md)
3. [Hash Index](HASH.md)
4. [Configuration](CONFIG.md)
5. [Benchmark Cassandra with Cassandra-stress & Cockroach with YCSB](http://htmlpreview.github.io/?https://github.com/shingjan/DBx1000/blob/master/Cassandra-Cockroach-Benchmark.html)
6. [Discussion with Xiangyao, author of DBx1000](https://github.com/yxymit/DBx1000/issues/17)
7. Compilation Process on Ubuntu node

Above building procedures work well with Linux Subsystem on Windows 10. Errors thrown on __libs/libjemalloc.a__ since it is not built with __-fPIC__ flag. It is a pre-compiled lib file so fix is simple: 

Change this line in Makefile from:

~~~
-LDFLAGS = -Wall -L. -L./libs -pthread -g -lrt -std=c++0x -O3 -ljemalloc
~~~

To 

~~~
-LDFLAGS = -Wall -L. -pthread -g -lrt -std=c++0x -O3 -ljemalloc
~~~

And install libjemalloc on Ubuntu via:

~~~
sudo apt-get update
sudo apt-get install libjemalloc-dev
~~~

Summary by YJ - End
===================



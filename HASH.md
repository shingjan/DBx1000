Hash Index
----------

1. "For the YCSB table, the hash index contains "g_syth_table_size * 2" buckets, thus has a low probability of collision". Hence, there are roughly 2 million buckets for this table, which means that each lookup would access, most likely, one bucket, which contains one item, and loop inside this one-item bucket once.
    
        index->init(part_cnt, tables[tname], g_synth_table_size * 2); - benchmarks/tpcc_txn.cpps
Hash Index
----------

1. "For the YCSB table, the hash index contains "g_syth_table_size * 2" buckets, thus has a low probability of collision". Hence, there are roughly 2 million buckets for this table, which means that each lookup would access, most likely, one bucket, which contains one item, and loop inside this one-item bucket once.
    
        index->init(part_cnt, tables[tname], g_synth_table_size * 2); - benchmarks/tpcc_txn.cpps

2. Read item from hash bucket without latch in **index_read**
~~~c++
void BucketHeader::read_item(idx_key_t key, itemid_t * &item, const char * tname) 
{
	BucketNode * cur_node = first_node;
	while (cur_node != NULL) {
		if (cur_node->key == key)
			break;
		cur_node = cur_node->next;
	}
M_ASSERT(cur_node->key == key, "Key does not exist!");
item = cur_node->items;
}
~~~

3. Write item to hash bucket with latch in **index_insert**
~~~c++
void BucketHeader::insert_item(idx_key_t key, 
		itemid_t * item, 
		int part_id) 
{
	BucketNode * cur_node = first_node;
	BucketNode * prev_node = NULL;
	while (cur_node != NULL) {
		if (cur_node->key == key)
			break;
		prev_node = cur_node;
		cur_node = cur_node->next;
	}
	if (cur_node == NULL) {		
		BucketNode * new_node = (BucketNode *) 
			mem_allocator.alloc(sizeof(BucketNode), part_id );
		new_node->init(key);
		new_node->items = item;
		if (prev_node != NULL) {
			new_node->next = prev_node->next;
			prev_node->next = new_node;
		} else {
			new_node->next = first_node;
			first_node = new_node;
		}
	} else {
		item->next = cur_node->items;
		cur_node->items = item;
	}
}
~~~

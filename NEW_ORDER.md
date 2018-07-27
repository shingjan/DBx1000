- **New Order Transactionn** (benchmark/tpcc_txn.cpp/tpcc_txn_man::run_new_order(tpcc_query * query))

    1.

    ~~~sql
    EXEC SQL SELECT c_discount, c_last, c_credit, w_tax
        INTO :c_discount, :c_last, :c_credit, :w_tax
        FROM customer, warehouse
        WHERE w_id = :w_id AND c_w_id = w_id AND c_d_id = :d_id AND c_id = :c_id;
    ~~~

    ~~~c++
    key = w_id;
    index = _wl->i_warehouse; 
    item = index_read(index, key, wh_to_part(w_id));
    assert(item != NULL);
    row_t * r_wh = ((row_t *)item->location);
    row_t * r_wh_local = get_row(r_wh, RD);
    if (r_wh_local == NULL) {
        return finish(Abort);
    }


    key = custKey(c_id, d_id, w_id);
    index = _wl->i_customer_id;
    item = index_read(index, key, wh_to_part(w_id));
    assert(item != NULL);
    row_t * r_cust = (row_t *) item->location;
    row_t * r_cust_local = get_row(r_cust, RD);
    if (r_cust_local == NULL) {
        return finish(Abort); 
    }    

    double w_tax;
    uint64_t c_discount;
    char * c_last;
    char * c_credit;
    r_wh_local->get_value(W_TAX, w_tax); 
    r_cust_local->get_value(C_DISCOUNT, c_discount);
    //AS THE AUTHOR CLAIMED, ONLY VALUES RETRIVED AND USED ARE ACTUALLY RETRIVED HERER
    //HENCE, c_last AND c_credit ARE COMMENTED.
    //c_last = r_cust_local->get_value(C_LAST);
    //c_credit = r_cust_local->get_value(C_CREDIT);
    ~~~

    2 & 3. 

    ~~~sql
    EXEC SQL SELECT d_next_o_id, d_tax
        INTO :d_next_o_id, :d_tax
        FROM district WHERE d_id = :d_id AND d_w_id = :w_id;
    EXEC SQL UPDATE d istrict SET d _next_o_id = :d _next_o_id + 1
        WH ERE d _id = :d_id AN D d _w _id = :w _id ;
    ~~~

    ~~~c++
    key = distKey(d_id, w_id);
    item = index_read(_wl->i_district, key, wh_to_part(w_id));
    assert(item != NULL);
    row_t * r_dist = ((row_t *)item->location);
    row_t * r_dist_local = get_row(r_dist, WR);
    if (r_dist_local == NULL) {
        return finish(Abort);
    }
    //RETRIEVAL OF d_tax ARE COMMENTED
    //double d_tax;
    int64_t o_id;
    //d_tax = *(double *) r_dist_local->get_value(D_TAX);
    o_id = *(int64_t *) r_dist_local->get_value(D_NEXT_O_ID);
    o_id ++;
    r_dist_local->set_value(D_NEXT_O_ID, o_id);
    ~~~

    4.

    ~~~sql
    EXEC SQL INSERT INTO ORDERS (o_id, o_d_id, o_w_id, o_c_id, o_entry_d, o_ol_cnt, o_all_local)
        VALUES (:o_id, :d_id, :w_id, :c_id, :datetime, :o_ol_cnt, :o_all_local);
    ~~~

    ~~~c++
    //  IMPLEMENTATION OF THIS SQL STATEMENT IS COMMENTED
    //  row_t * r_order;
    //  uint64_t row_id;
    //  _wl->t_order->get_new_row(r_order, 0, row_id);
    //  r_order->set_value(O_ID, o_id);
    //  r_order->set_value(O_C_ID, c_id);
    //  r_order->set_value(O_D_ID, d_id);
    //  r_order->set_value(O_W_ID, w_id);
    //  r_order->set_value(O_ENTRY_D, o_entry_d);
    //  r_order->set_value(O_OL_CNT, ol_cnt);
    //  int64_t all_local = (remote? 0 : 1);
    //  r_order->set_value(O_ALL_LOCAL, all_local);
    //  insert_row(r_order, _wl->t_order);
    ~~~

    5.

    ~~~SQL
    EXEC SQL INSERT INTO NEW_ORDER (no_o_id, no_d_id, no_w_id)
        VALUES (:o_id, :d_id, :w_id);
    ~~~

    ~~~C++
    //  IMPLEMENTATION OF THIS SQL STATEMENT IS COMMENTED
    //  row_t * r_no;
    //  _wl->t_neworder->get_new_row(r_no, 0, row_id);
    //  r_no->set_value(NO_O_ID, o_id);
    //  r_no->set_value(NO_D_ID, d_id);
    //  r_no->set_value(NO_W_ID, w_id);
    //  insert_row(r_no, _wl->t_neworder);  
    ~~~

    6.
    SQL statements 6 - 9 are in a for loop
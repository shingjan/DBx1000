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
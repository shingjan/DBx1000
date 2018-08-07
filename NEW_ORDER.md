TPCC workloads generation, created by YJ
-----------------------------------------

- **New Order Transactionn** 
    
    - *benchmark/tpcc_txn.cpp/tpcc_txn_man::run_new_order(tpcc_query * query)*

    Summary:

        1. 9 SQL statements in total. Statements 3 and 8 are complete.
        2. Statements 1, 2, 6 and 7 are implemented but incomplete. (Fixed in source code)
        3. Statements 4, 5 and 9 are implemented but commented. (Fixed in source code)

    - SQL statement 1 w. Patch No. 6

    ~~~sql
    EXEC SQL SELECT c_discount, c_last, c_credit, w_tax
        INTO :c_discount, :c_last, :c_credit, :w_tax
        FROM customer, warehouse
        WHERE w_id = :w_id AND c_w_id = w_id AND c_d_id = :d_id AND c_id = :c_id;
    ~~~

    - Line 259 - 286
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
    r_wh_local->get_value(W_TAX, w_tax); 
    r_cust_local->get_value(C_DISCOUNT, c_discount);
    //------------------Patch No. 6----------------------//
    //----------------Uncommented by YJ------------------//
    char * c_last;
    char * c_credit;
    c_last = r_cust_local->get_value(C_LAST);
    c_credit = r_cust_local->get_value(C_CREDIT);
    //----------------Uncommented by YJ------------------//
    ~~~

    - SQL statement 2 & 3 w. Patch No. 7

    ~~~sql
    EXEC SQL SELECT d_next_o_id, d_tax
        INTO :d_next_o_id, :d_tax
        FROM district WHERE d_id = :d_id AND d_w_id = :w_id;
    EXEC SQL UPDATE d istrict SET d_next_o_id = :d_next_o_id + 1
        WHERE d_id = :d_id AN D d _w _id = :w _id ;
    ~~~
    - Line 295 - 308
    ~~~c++
    key = distKey(d_id, w_id);
    item = index_read(_wl->i_district, key, wh_to_part(w_id));
    assert(item != NULL);
    row_t * r_dist = ((row_t *)item->location);
    row_t * r_dist_local = get_row(r_dist, WR);
    if (r_dist_local == NULL) {
        return finish(Abort);
    }

    int64_t o_id;
    o_id = *(int64_t *) r_dist_local->get_value(D_NEXT_O_ID);
    o_id ++;
    r_dist_local->set_value(D_NEXT_O_ID, o_id);
    //------------------Patch No. 7----------------------//
    //----------------Uncommented by YJ------------------//
    double d_tax;
    d_tax = *(double *) r_dist_local->get_value(D_TAX);
    //----------------Uncommented by YJ------------------//
    ~~~

    - SQL statement 4 w. Patch No. 8

    ~~~sql
    EXEC SQL INSERT INTO ORDERS (o_id, o_d_id, o_w_id, o_c_id, o_entry_d, o_ol_cnt, o_all_local)
        VALUES (:o_id, :d_id, :w_id, :c_id, :datetime, :o_ol_cnt, :o_all_local);
    ~~~
    - Line 314 - 325
    ~~~c++
    //------------------Patch No. 8----------------------//
    //----------------Uncommented by YJ------------------//
    row_t * r_order;
    uint64_t row_id;
    _wl->t_order->get_new_row(r_order, 0, row_id);
    r_order->set_value(O_ID, o_id);
    r_order->set_value(O_C_ID, c_id);
    r_order->set_value(O_D_ID, d_id);
    r_order->set_value(O_W_ID, w_id);
    int64_t date = 2013;
    r_order->set_value(O_ENTRY_D, date);
    r_order->set_value(O_OL_CNT, ol_cnt);
    int64_t all_local = (remote? 0 : 1);
    r_order->set_value(O_ALL_LOCAL, all_local);
    insert_row(r_order, _wl->t_order);
    //----------------Uncommented by YJ------------------//
    ~~~

    - SQL statement 5 w. Patch No. 9

    ~~~SQL
    EXEC SQL INSERT INTO NEW_ORDER (no_o_id, no_d_id, no_w_id)
        VALUES (:o_id, :d_id, :w_id);
    ~~~
    - Line 330 - 335
    ~~~C++
    //------------------Patch No. 9----------------------//
    //----------------Uncommented by YJ------------------//
    row_t * r_no;
    _wl->t_neworder->get_new_row(r_no, 0, row_id);
    r_no->set_value(NO_O_ID, o_id);
    r_no->set_value(NO_D_ID, d_id);
    r_no->set_value(NO_W_ID, w_id);
    insert_row(r_no, _wl->t_neworder);
    //----------------Uncommented by YJ------------------//  
    ~~~

    - SQL statement 6 w. Patch No. 10

    ~~~sql
        EXEC SQL SELECT i_price, i_name , i_data
            INTO :i_price, :i_name, :i_data
            FROM item
            WHERE i_id = :ol_i_id;
    ~~~
    - Line 347 - 361
    ~~~c++
        key = ol_i_id;
        item = index_read(_wl->i_item, key, 0);
        assert(item != NULL);
        row_t * r_item = ((row_t *)item->location);

        row_t * r_item_local = get_row(r_item, RD);
        if (r_item_local == NULL) {
            return finish(Abort);
        }
        // RETRIEVAL OF i_name AND i_data ARE COMMENTED
        int64_t i_price;
        r_item_local->get_value(I_PRICE, i_price);
        //------------------Patch No. 10----------------------//
        //----------------Uncommented by YJ------------------//
        char * i_name;
        char * i_data;
        i_name = r_item_local->get_value(I_NAME);
        i_data = r_item_local->get_value(I_DATA);
        //----------------Uncommented by YJ------------------//
    ~~~

    - SQL statement 7 w. Patch No. 11
    ~~~sql
        EXEC SQL SELECT s_quantity, s_data,
                s_dist_01, s_dist_02, s_dist_03, s_dist_04, s_dist_05,
                s_dist_06, s_dist_07, s_dist_08, s_dist_09, s_dist_10
            INTO :s_quantity, :s_data,
                :s_dist_01, :s_dist_02, :s_dist_03, :s_dist_04, :s_dist_05,
                :s_dist_06, :s_dist_07, :s_dist_08, :s_dist_09, :s_dist_10
            FROM stock
            WHERE s_i_id = :ol_i_id AND s_w_id = :ol_supply_w_id;
    ~~~
    - Line 377 - 401
    ~~~c++
        uint64_t stock_key = stockKey(ol_i_id, ol_supply_w_id);
        INDEX * stock_index = _wl->i_stock;
        itemid_t * stock_item;
        index_read(stock_index, stock_key, wh_to_part(ol_supply_w_id), stock_item);
        assert(item != NULL);
        row_t * r_stock = ((row_t *)stock_item->location);
        row_t * r_stock_local = get_row(r_stock, WR);
        if (r_stock_local == NULL) {
            return finish(Abort);
        }
        
        // XXX s_dist_xx are not retrieved.
        UInt64 s_quantity;
        int64_t s_remote_cnt;
        s_quantity = *(int64_t *)r_stock_local->get_value(S_QUANTITY);
        #if !TPCC_SMALL
        int64_t s_ytd;
        int64_t s_order_cnt;
        r_stock_local->get_value(S_YTD, s_ytd);
        r_stock_local->set_value(S_YTD, s_ytd + ol_quantity);
        r_stock_local->get_value(S_ORDER_CNT, s_order_cnt);
        r_stock_local->set_value(S_ORDER_CNT, s_order_cnt + 1);
        //------------------Patch No. 11----------------------//
        //----------------Added by YJ------------------//
        char * s_dist_01 = r_stock_local->get_value(S_DIST_01);
        char * s_dist_02 = r_stock_local->get_value(S_DIST_02);
        char * s_dist_03 = r_stock_local->get_value(S_DIST_03);
        char * s_dist_04 = r_stock_local->get_value(S_DIST_04);
        char * s_dist_05 = r_stock_local->get_value(S_DIST_05);
        char * s_dist_06 = r_stock_local->get_value(S_DIST_06);
        char * s_dist_07 = r_stock_local->get_value(S_DIST_07);
        char * s_dist_08 = r_stock_local->get_value(S_DIST_08);
        char * s_dist_09 = r_stock_local->get_value(S_DIST_09);
        char * s_dist_10 = r_stock_local->get_value(S_DIST_10);
        char * s_data = r_stock_local->get_value(S_DATA);
        //----------------Added by YJ------------------//
    ~~~

    - SQL statement 8

    ~~~sql
        EXEC SQL UPDATE stock SET s_quantity = :s_quantity
            WHERE s_i_id = :ol_i_id
            AND s_w_id = :ol_supply_w_id;
    ~~~
    - Line 407 - 413
    ~~~c++
        uint64_t quantity;
        if (s_quantity > ol_quantity + 10) {
            quantity = s_quantity - ol_quantity;
        } else {
            quantity = s_quantity - ol_quantity + 91;
        }
        r_stock_local->set_value(S_QUANTITY, &quantity);
    ~~~

    - SQL statement 9 w. Patch No. 12

    ~~~sql
        EXEC SQL INSERT
            INTO order_line(ol_o_id, ol_d_id, ol_w_id, ol_number,
                ol_i_id, ol_supply_w_id,
                ol_quantity, ol_amount, ol_dist_info)
            VALUES(:o_id, :d_id, :w_id, :ol_number,
                :ol_i_id, :ol_supply_w_id,
                :ol_quantity, :ol_amount, :ol_dist_info);
    ~~~
    - Line 425 - 440
    ~~~c++
        //------------------Patch No. 12----------------------//
        //----------------Uncommented by YJ------------------//
        // XXX district info is not inserted.
        row_t * r_ol;
        uint64_t row_id;
        _wl->t_orderline->get_new_row(r_ol, 0, row_id);
        r_ol->set_value(OL_O_ID, &o_id);
        r_ol->set_value(OL_D_ID, &d_id);
        r_ol->set_value(OL_W_ID, &w_id);
        r_ol->set_value(OL_NUMBER, &ol_number);
        r_ol->set_value(OL_I_ID, &ol_i_id);
        #if !TPCC_SMALL
        int w_tax=1, d_tax=1;
        int64_t ol_amount = ol_quantity * i_price * (1 + w_tax + d_tax) * (1 - c_discount);
        r_ol->set_value(OL_SUPPLY_W_ID, &ol_supply_w_id);
        r_ol->set_value(OL_QUANTITY, &ol_quantity);
        r_ol->set_value(OL_AMOUNT, &ol_amount);
        #endif      
        insert_row(r_ol, _wl->t_orderline);
        //----------------Uncommented by YJ------------------//
    ~~~
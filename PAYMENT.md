TPCC workloads generation, created by YJ
-----------------------------------------

- Summary
    * Schema: 9 tables 
    - item = 100 x k = 100 x 40 = 4,000 (but actually size of the item table is 400,000, all item tables are mapped into one)
    - warehouse (W) = 100
        - stock = W x 100k = 100 x 100 x 40 (k) = 400,000
        - dist = 10 x W = 1000
          - cust = W x 30k = 120,000
          - hist 
          - order 
          - new order
          - order line 
    
    ![Alt text](/img/TPCC_tables.PNG?raw=true "TPCC tables")
    
    * Only **Payment** and **New Order** transactions are modeled

- **Payment Transactionn** 

    - *benchmark/tpcc_txn.cpp/tpcc_txn_man::run_payment(tpcc_query * query)*
    
    Summary:

        - 11 SQL statements in total. Statements 1, 3, 6, and 10 are complete.
        - Statements 2, 4 and 8 are implemented but incomplete. (Fixed in source code)
        - Statements 9 and 11 are implemented but commented. (Fixed in source code) 
        - Statements 5 and 7 are not implemented.

    - SQL statement 1

    ~~~sql
        EXEC SQL UPDATE warehouse SET w_ytd = w_ytd + :h_amount 
            WHERE w_id=:w_id;
    ~~~ 

    - Line 56 - 75
        
    ~~~c++
        key = query->w_id;
        INDEX * index = _wl->i_warehouse;

        //use w_id to look up for that specific row via hash index
        item = index_read(index, key, wh_to_part(w_id));
        assert(item != NULL);
        row_t * r_wh = ((row_t *)item->location);
        row_t * r_wh_local;
        if (g_wh_update)
            r_wh_local = get_row(r_wh, WR);
        else 
            r_wh_local = get_row(r_wh, RD);
        
        if (r_wh_local == NULL) {
            return finish(Abort);
        }

        double w_ytd;
        //GET w_ytd VALUE FROM THIS ROW
        r_wh_local->get_value(W_YTD, w_ytd);
        if (g_wh_update) {
            r_wh_local->set_value(W_YTD, w_ytd + query->h_amount);
        }
    ~~~
    
    - SQL statement 2 with Patch No.1

    ~~~sql
        EXEC SQL SELECT w_street_1, w_street_2, w_city, w_state, w_zip, w_name 
            INTO :w_street_1, :w_street_2, :w_city, :w_state, :w_zip, :w_name 
            FROM warehouse 
            WHERE w_id=:w_id;
    ~~~
        
    - Line 76 - 79

    ~~~c++
        char w_name[11];
        //HERE ONLY W_NAME IS RETRIVED. OTHER VALUES SHOULD BE RETRIVED. 
        //NOT USED ALTHOUGH, ACCORDING TO SQL STATEMENT NO.2
        char * tmp_str = r_wh_local->get_value(W_NAME);
        //------------------Patch No. 1----------------------//
        //------------------ADDED BY YJ----------------------//
        char * tmp_str1 = r_wh_local->get_value(W_STREET_1);
        char * tmp_str2 = r_wh_local->get_value(W_STREET_2);
        char * tmp_str3 = r_wh_local->get_value(W_CITY);
        char * tmp_str4 = r_wh_local->get_value(W_STATE);
        char * tmp_str5 = r_wh_local->get_value(W_ZIP);
        //------------------ADDED BY YJ----------------------//
        memcpy(w_name, tmp_str, 10);
        w_name[10] = '\0';
    ~~~

    - SQL statement 3

    ~~~sql
    EXEC SQL UPDATE district SET d_ytd = d_ytd + :h_amount <br>
        WHERE d_w_id=:w_id AND d_id=:d_id;
    ~~~
      
    - Line 84 - 95  
    ~~~c++
        key = distKey(query->d_id, query->d_w_id);
        item = index_read(_wl->i_district, key, wh_to_part(w_id));
        assert(item != NULL);
        row_t * r_dist = ((row_t *)item->location);
        row_t * r_dist_local = get_row(r_dist, WR);
        if (r_dist_local == NULL) {
            return finish(Abort);
        }

        double d_ytd;
        r_dist_local->get_value(D_YTD, d_ytd);
        r_dist_local->set_value(D_YTD, d_ytd + query->h_amount);
    ~~~

    - SQL statement 4 with Patch No. 2

    ~~~sql
    EXEC SQL SELECT d_street_1, d_street_2, d_city, d_state, d_zip, d_name 
        INTO :d_street_1, :d_street_2, :d_city, :d_state, :d_zip, :d_name
        FROM district 
        WHERE d_w_id=:w_id AND d_id=:d_id;
    ~~~
    
    - Line 96 - 99
    ~~~c++
        char d_name[11];
        //HERE ONLY D_NAME IS RETRIVED. OTHER VALUES SHOULD BE RETRIVED. 
        //NOT USED ALTHOUGH, ACCORDING TO SQL STATEMENT NO.4
        tmp_str = r_dist_local->get_value(D_NAME);
        //------------------Patch No. 2----------------------//
        //------------------ADDED BY YJ----------------------//
        tmp_str1 = r_dist_local->get_value(D_STREET_1);
        tmp_str2 = r_dist_local->get_value(D_STREET_2);
        tmp_str3 = r_dist_local->get_value(D_CITY);
        tmp_str4 = r_dist_local->get_value(D_STATE);
        tmp_str5 = r_dist_local->get_value(D_ZIP);
        //------------------ADDED BY YJ----------------------//
        memcpy(d_name, tmp_str, 10);
        d_name[10] = '\0';
    ~~~

    - SQL statement 5, not implemented

    ~~~sql
    if(byname)
        -- This SQL statement is not implemented in DBx1000
        EXEC SQL SELECT count(c_id) INTO :namecnt
            FROM customer
            WHERE c_last=:c_last AND c_d_id=:c_d_id AND c_w_id=:c_w_id;
    ~~~

    - SQL statement 6

    ~~~sql
        EXEC SQL DECLARE c_byname CURSOR FOR
            SELECT c_first, c_middle, c_id, c_street_1, c_street_2, c_city, c_state,
            c_zip, c_phone, c_credit, c_credit_lim, c_discount, c_balance, c_since
            FROM customer
            WHERE c_w_id=:c_w_id AND c_d_id=:c_d_id AND c_last=:c_last
            ORDER BY c_first;
            EXEC SQL OPEN c_byname;
    ~~~

    - Line 125 - 141
    ~~~c++
            uint64_t key = custNPKey(query->c_last, query->c_d_id, query->c_w_id);
            // XXX: the list is not sorted. But let's assume it's sorted... 
            // The performance won't be much different.
            INDEX * index = _wl->i_customer_last;
            item = index_read(index, key, wh_to_part(c_w_id));
            assert(item != NULL);
            
            int cnt = 0;
            itemid_t * it = item;
            itemid_t * mid = item;
            while (it != NULL) {
                cnt ++;
                it = it->next;
                if (cnt % 2 == 0)
                    mid = mid->next;
            }
            r_cust = ((row_t *)mid->location);       
    ~~~

    - SQL statement 7, not implemented

    ~~~~sql
        -- THIS SQL STATEMMENT IS NOT IMPLEMENTED IN DBx1000
        for (n=0; n<namecnt/2; n++) {
            EXEC SQL FETCH c_byname
            INTO :c_first, :c_middle, :c_id,
                :c_street_1, :c_street_2, :c_city, :c_state, :c_zip,
                :c_phone, :c_credit, :c_credit_lim, :c_discount, :c_balance, :c_since;
        }
        EXEC SQL CLOSE c_byname;
    ~~~~

    - SQL statement 8 with Patch No. 3
    ~~~~sql
        EXEC SQL SELECT c_first, c_middle, c_last, c_street_1, c_street_2,
            c_city, c_state, c_zip, c_phone, c_credit, c_credit_lim,
            c_discount, c_balance, c_since
            INTO :c_first, :c_middle, :c_last, :c_street_1, :c_street_2,
            :c_city, :c_state, :c_zip, :c_phone, :c_credit, :c_credit_lim,
            :c_discount, :c_balance, :c_since
            FROM customer
            WHERE c_w_id=:c_w_id AND c_d_id=:c_d_id AND c_id=:c_id;
    ~~~~

    - Line 165 - 169
    ~~~c++
        key = custKey(query->c_id, query->c_d_id, query->c_w_id);
        INDEX * index = _wl->i_customer_id;
        item = index_read(index, key, wh_to_part(c_w_id));
        assert(item != NULL);
        r_cust = (row_t *) item->location;
        //------------------Patch No. 3----------------------//
        //------------------ADDED BY YJ----------------------//
        r_cust_local = get_row(r_cust, WR);
        if (r_cust_local == NULL) {
            return finish(Abort);
        }
        tmp_str1 = r_cust_local->get_value(C_FIRST);
        tmp_str2 = r_cust_local->get_value(C_MIDDLE);
        tmp_str3 = r_cust_local->get_value(C_LAST);
        tmp_str4 = r_cust_local->get_value(C_STREET_1);
        tmp_str5 = r_cust_local->get_value(C_STREET_2);
        char * tmp_c_city = r_cust_local->get_value(C_CITY);
        char * tmp_c_state = r_cust_local->get_value(C_STATE);
        char * tmp_c_zip = r_cust_local->get_value(C_ZIP);
        char * tmp_c_phone = r_cust_local->get_value(C_PHONE);
        char * tmp_c_credit = r_cust_local->get_value(C_CREDIT);
        char * tmp_credit_lim = r_cust_local->get_value(C_CREDIT_LIM);
        char * tmp_c_discount = r_cust_local->get_value(C_DISCOUNT);
        char * tmp_c_balance = r_cust_local->get_value(C_BALANCE);
        char * tmp_c_since = r_cust_local->get_value(C_SINCE);
        //------------------ADDED BY YJ----------------------//
    ~~~

    - SQL statement 9 with Patch No. 4
    
    ~~~sql
    if ( strstr(c_credit, "BC") ) 
            EXEC SQL SELECT c_data
            INTO :c_data
            FROM customer
            WHERE c_w_id=:c_w_id AND c_d_id=:c_d_id AND c_id=:c_id;
    ~~~

    - Line 201 - 206
    ~~~c++
        //------------------Patch No. 4----------------------//
        //----------------Uncommented by YJ------------------//
        char c_new_data[501];
        //sprintf(c_new_data,"| %4d %2d %4d %2d %4d $%7.2f",
        //      c_id, c_d_id, c_w_id, d_id, w_id, query->h_amount);
        char * c_data = r_cust->get_value("C_DATA");
        strncat(c_new_data, c_data, 500 - strlen(c_new_data));
        r_cust->set_value("C_DATA", c_new_data);
        //----------------Uncommented by YJ------------------//
            
    ~~~

    - SQL Statement 10

    ~~~sql
        EXEC SQL UPDATE customer SET c_balance = :c_balance, c_data = :c_new_data
        WHERE c_w_id = :c_w_id AND c_d_id = :c_d_id AND c_id = :c_id;
    ~~~

    - Line 176 - 189
    ~~~c++
    row_t * r_cust_local = get_row(r_cust, WR);
    if (r_cust_local == NULL) {
        return finish(Abort);
    }
    double c_balance;
    double c_ytd_payment;
    double c_payment_cnt;

    r_cust_local->get_value(C_BALANCE, c_balance);
    r_cust_local->set_value(C_BALANCE, c_balance - query->h_amount);
    r_cust_local->get_value(C_YTD_PAYMENT, c_ytd_payment);
    r_cust_local->set_value(C_YTD_PAYMENT, c_ytd_payment + query->h_amount);
    r_cust_local->get_value(C_PAYMENT_CNT, c_payment_cnt);
    r_cust_local->set_value(C_PAYMENT_CNT, c_payment_cnt + 1);
    ~~~

    - SQL statement 11 with Patch No. 5

    ~~~sql
      EXEC SQL INSERT INTO
      history (h_c_d_id, h_c_w_id, h_c_id, h_d_id, h_w_id, h_date, h_amount, h_data)
      VALUES (:c_d_id, :c_w_id, :c_id, :d_id, :w_id, :datetime, :h_amount, :h_data);
    ~~~

    - Line 222 - 236
    ~~~c++
    //------------------Patch No. 5----------------------//
    //----------------Uncommented by YJ------------------//
    row_t * r_hist;
    uint64_t row_id;
    _wl->t_history->get_new_row(r_hist, 0, row_id);
    r_hist->set_value(H_C_ID, query->c_id);
    r_hist->set_value(H_C_D_ID, query->c_d_id);
    r_hist->set_value(H_C_W_ID, c_w_id);
    r_hist->set_value(H_D_ID, query->d_id);
    r_hist->set_value(H_W_ID, w_id);
    int64_t date = 2013;        
    r_hist->set_value(H_DATE, date);
    r_hist->set_value(H_AMOUNT, query->h_amount);
    #if !TPCC_SMALL
    r_hist->set_value(H_DATA, h_data);
    #endif
    insert_row(r_hist, _wl->t_history);
    //----------------Uncommented by YJ------------------//
    ~~~
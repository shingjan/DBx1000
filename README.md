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

Summary by YJ
===============

1. [YCSB worklaods generation](TPCC.md)
2. [TPCC workloads generation](YCSB.md)
3. [Hash Index](HASH.md)
4. [Configuration](CONFIG.md)




TPCC workloads generation with Hash Index
-----------------------------------------

- Schema
    * TPCC: 9 tables (Warehouse, Custormer etc.): 
    
    ![Alt text](/img/TPCC_tables.PNG?raw=true "TPCC tables")
    
    * Only **Payment** and **New Order** transactions are modeled

- **Payment Transactionn** (benchmark/tpcc_txn.cpp/tpcc_txn_man::run_payment(tpcc_query * query))
    
    1.

    ~~~sql
    EXEC SQL UPDATE warehouse SET w_ytd = w_ytd + :h_amount <br>
            WHERE w_id=:w_id;
    ~~~ 
        
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
    
    2.

    ~~~sql
    EXEC SQL SELECT w_street_1, w_street_2, w_city, w_state, w_zip, w_name <br>
        INTO :w_street_1, :w_street_2, :w_city, :w_state, :w_zip, :w_name <br>
        FROM warehouse <br>
        WHERE w_id=:w_id;
    ~~~
        
    ~~~c++
        char w_name[11];
        //HERE ONLY W_NAME IS RETRIVED. OTHER VALUES SHOULD BE RETRIVED. 
        //NOT USED ALTHOUGH, ACCORDING TO SQL STATEMENT NO.2
        char * tmp_str = r_wh_local->get_value(W_NAME);
        //------------------ADDED BY YJ----------------------//
        //char * tmp_str1 = r_wh_local->get_value(W_STREET_1);
        //char * tmp_str2 = r_wh_local->get_value(W_STREET_2);
        //char * tmp_str3 = r_wh_local->get_value(W_CITY);
        //char * tmp_str4 = r_wh_local->get_value(W_STATE);
        //char * tmp_str5 = r_wh_local->get_value(W_ZIP);
        //------------------ADDED BY YJ----------------------//
        memcpy(w_name, tmp_str, 10);
        w_name[10] = '\0';
    ~~~

    3.

    ~~~sql
    EXEC SQL UPDATE district SET d_ytd = d_ytd + :h_amount <br>
        WHERE d_w_id=:w_id AND d_id=:d_id;
    ~~~
        
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

    4.

    ~~~sql
    EXEC SQL SELECT d_street_1, d_street_2, d_city, d_state, d_zip, d_name 
        INTO :d_street_1, :d_street_2, :d_city, :d_state, :d_zip, :d_name
        FROM district 
        WHERE d_w_id=:w_id AND d_id=:d_id;
    ~~~

    ~~~c++
        double d_ytd;
        r_dist_local->get_value(D_YTD, d_ytd);
        r_dist_local->set_value(D_YTD, d_ytd + query->h_amount);
        char d_name[11];
        //HERE ONLY D_NAME IS RETRIVED. OTHER VALUES SHOULD BE RETRIVED. 
        //NOT USED ALTHOUGH, ACCORDING TO SQL STATEMENT NO.4
        tmp_str = r_dist_local->get_value(D_NAME);
        //------------------ADDED BY YJ----------------------//
        //char * tmp_str1 = r_dist_local->get_value(D_STREET_1);
        //char * tmp_str2 = r_dist_local->get_value(D_STREET_2);
        //char * tmp_str3 = r_dist_local->get_value(D_CITY);
        //char * tmp_str4 = r_dist_local->get_value(D_STATE);
        //char * tmp_str5 = r_dist_local->get_value(D_ZIP);
        //------------------ADDED BY YJ----------------------//
        memcpy(d_name, tmp_str, 10);
        d_name[10] = '\0';
    ~~~

    5. if(byname) 
    ~~~sql
        -- This SQL statement is not implemented in DBx1000
        EXEC SQL SELECT count(c_id) INTO :namecnt
            FROM customer
            WHERE c_last=:c_last AND c_d_id=:c_d_id AND c_w_id=:c_w_id;
    ~~~

    6.

    ~~~sql
        EXEC SQL DECLARE c_byname CURSOR FOR
            SELECT c_first, c_middle, c_id, c_street_1, c_street_2, c_city, c_state,
            c_zip, c_phone, c_credit, c_credit_lim, c_discount, c_balance, c_since
            FROM customer
            WHERE c_w_id=:c_w_id AND c_d_id=:c_d_id AND c_last=:c_last
            ORDER BY c_first;
            EXEC SQL OPEN c_byname;
    ~~~

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

    7.

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

    8. else
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

    ~~~c++
        key = custKey(query->c_id, query->c_d_id, query->c_w_id);
        INDEX * index = _wl->i_customer_id;
        item = index_read(index, key, wh_to_part(c_w_id));
        assert(item != NULL);
        r_cust = (row_t *) item->location;
        //ONLY THE POINTER OF THIS ROW IS RETRIVED.
        //NO VALID VALUE IS RETRIVED 
        //------------------ADDED BY YJ----------------------//
        //row_t * r_cust_local = get_row(r_cust, RD);
        //char * tmp_str1 = r_cust_local->get_value(C_FIRST);
        //char * tmp_str2 = r_cust_local->get_value(C_MIDDLE);
        //char * tmp_str3 = r_cust_local->get_value(C_LAST);
        //char * tmp_str4 = r_cust_local->get_value(C_STREET_1);
        //char * tmp_str5 = r_cust_local->get_value(C_STREET_2);
        //char * tmp_str5 = r_cust_local->get_value(C_CITY);
        //char * tmp_str5 = r_cust_local->get_value(C_STATE);
        //char * tmp_str5 = r_cust_local->get_value(C_ZIP);
        //char * tmp_str5 = r_cust_local->get_value(C_PHONE);
        //char * tmp_str5 = r_cust_local->get_value(C_CREDIT);
        //char * tmp_str5 = r_cust_local->get_value(C_CREDIT_LIM);
        //char * tmp_str5 = r_cust_local->get_value(C_DISCOUNT);
        //char * tmp_str5 = r_cust_local->get_value(C_BALANCE);
        //char * tmp_str5 = r_cust_local->get_value(C_SINCE);
        //------------------ADDED BY YJ----------------------//
    ~~~

    9. if ( strstr(c_credit, "BC") ) 
    
    ~~~sql
            EXEC SQL SELECT c_data
            INTO :c_data
            FROM customer
            WHERE c_w_id=:c_w_id AND c_d_id=:c_d_id AND c_id=:c_id;
    ~~~

    ~~~c++
    //IMPLEMENTATION OF THIS SQL STATEMENT IS COMPLETED BUT COMMENTED OUT IN DBx1000
    //      char c_new_data[501];
    //      sprintf(c_new_data,"| %4d %2d %4d %2d %4d $%7.2f",
    //          c_id, c_d_id, c_w_id, d_id, w_id, query->h_amount);
    //      char * c_data = r_cust->get_value("C_DATA");
    //      strncat(c_new_data, c_data, 500 - strlen(c_new_data));
    //      r_cust->set_value("C_DATA", c_new_data);
            
    ~~~

    10. else

    ~~~sql
        EXEC SQL UPDATE customer SET c_balance = :c_balance, c_data = :c_new_data
        WHERE c_w_id = :c_w_id AND c_d_id = :c_d_id AND c_id = :c_id;
    ~~~

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

    11. 

    ~~~sql
      EXEC SQL INSERT INTO
      history (h_c_d_id, h_c_w_id, h_c_id, h_d_id, h_w_id, h_date, h_amount, h_data)
      VALUES (:c_d_id, :c_w_id, :c_id, :d_id, :w_id, :datetime, :h_amount, :h_data);
    ~~~

    ~~~c++
    //IMPLEMENTATION OF THIS SQL STATEMENT IS COMPLETED BUT COMMENTED OUT IN DBx1000
    //  row_t * r_hist;
    //  uint64_t row_id;
    //  _wl->t_history->get_new_row(r_hist, 0, row_id);
    //  r_hist->set_value(H_C_ID, c_id);
    //  r_hist->set_value(H_C_D_ID, c_d_id);
    //  r_hist->set_value(H_C_W_ID, c_w_id);
    //  r_hist->set_value(H_D_ID, d_id);
    //  r_hist->set_value(H_W_ID, w_id);
    //  int64_t date = 2013;        
    //  r_hist->set_value(H_DATE, date);
    //  r_hist->set_value(H_AMOUNT, h_amount);
    #if !TPCC_SMALL
    //  r_hist->set_value(H_DATA, h_data);
    #endif
    //  insert_row(r_hist, _wl->t_history); 
    ~~~

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



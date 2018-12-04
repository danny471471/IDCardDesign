create or replace
PROCEDURE SP_BSNS_TERM_WRMBORG_MUL_TEST(
    V_CASCADE_ID CSCD_RQST.CSCD_RQST_ID%TYPE,
    V_MSTR_PROV_ID POA.MSTR_PROV_ID%TYPE,
    V_POA_KEY_STR VARCHAR2,
    V_RA_1_ID POA_RA_W.RA_1_ID%TYPE,
   -- V_RA_TERMNTN_DT POA.PADRS_TRMNTN_DT%TYPE,
   -- V_RA_TRMNTN_RSN_CD POA.PADRS_TRMNTN_RSN_CD%TYPE,
    V_LAST_UPDT_USER_ID POA.LAST_UPDT_USER_ID%TYPE)
IS
  w_termdate POA.PADRS_TRMNTN_DT%TYPE;
  w_effdate POA.PADRS_EFCTV_DT%TYPE;
  w_reasoncode POA.PADRS_TRMNTN_RSN_CD%TYPE;
  w_prov_id POA.MSTR_PROV_ID%TYPE;
  w_poa_ntwk_key POA_NTWK.POA_NTWK_KEY%TYPE;
  V_ERR_IND NUMBER;
  date_val_result DATE_VALIDATION_RESULT;
  v_err_message  VARCHAR2(4000);
  v_err_code     VARCHAR2(10) := '8600';
   v_src_sys_cd                     VARCHAR2(10) := 'SPWRMMUL';                  -- The Jira Number / Stored procedure number as per TDD
  v_src_prov_id                    VARCHAR2(10) := V_CASCADE_ID;                  
  v_creat_user                     VARCHAR2(10) := 'SPS';
  v_err_dtl_seq  NUMBER;
  v_aud_tab_flag NUMBER   := 0;
  v_NT_ERR_DTL NT_ERR_DTL := NT_ERR_DTL();
  v_t_err_dtl T_ERR_DTL;
  v_err_tab_index          NUMBER := 0;
  v_transaction_start_date DATE;
  v_transaction_end_date   DATE;
  v_NT_cascade_audt_dtl NT_cascade_audt_dtl := NT_cascade_audt_dtl();
  v_t_cascade_audt_dtl T_cascade_audt_dtl;
  v_cascade_audt_dtl_index NUMBER := 0;
  V_POA_KEY_ARR STRING_TO_ARRAY_PKG_NEW.t_array;
  V_POA_KEY POA_RA_W.POA_KEY%TYPE;
  V_RA_TERMNTN_DT POA.PADRS_TRMNTN_DT%TYPE;
  V_RA_TRMNTN_RSN_CD POA.PADRS_TRMNTN_RSN_CD%TYPE;
  v_JulianDate                          VARCHAR2(30) := '01-JAN-1970';
  V_NT_poa_input_array NT_poa_input_array := NT_poa_input_array();
  V_t_poa_input_array t_poa_input_array ;
 -- v_input_string_arr_index          NUMBER := 0;
  
  CURSOR c_poa_ra_w (V_POA_KEY POA_RA_W.POA_KEY%TYPE)
  IS
    SELECT POA_RA_W_KEY,
      RA_EFCTV_DT,
      RA_TRMNTN_DT,
      RA_1_ID,
      RA_SOR_CD
    FROM POA_RA_W
    WHERE POA_KEY    = V_POA_KEY
    --AG10016
    AND RA_TRMNTN_DT >= SYSDATE;
  CURSOR c_rltd_padrs (V_MSTR_PROV_ID RLTD_PADRS.MSTR_PROV_ID%TYPE, V_POA_KEY RLTD_PADRS.POA_KEY%TYPE)
  IS
    SELECT RLTD_PADRS_KEY,
      RLTD_PADRS_EFCTV_DT,
      RLTD_PADRS_TRMNTN_DT,
      RLTD_MSTR_PROV_ID
    FROM RLTD_PADRS
    WHERE MSTR_PROV_ID       = V_MSTR_PROV_ID
    AND POA_KEY              = V_POA_KEY
    AND RLTD_PADRS_TRMNTN_DT >= SYSDATE;
  CURSOR c_rltd_padrs_ra_w (V_RLTD_PADRS_KEY RLTD_PADRS_RA_W.RLTD_PADRS_KEY%TYPE)
  IS
    SELECT RLTD_PADRS_RA_W_KEY,
      RA_EFCTV_DT,
      RA_TRMNTN_DT,
      RLTD_PADRS_KEY,
      RA_1_ID
    FROM RLTD_PADRS_RA_W
    WHERE RLTD_PADRS_KEY = V_RLTD_PADRS_KEY
    AND RA_TRMNTN_DT     >= SYSDATE;
BEGIN
  SET transaction ISOLATION level serializable;

  V_ERR_IND     := 0;
    
  V_POA_KEY_ARR := STRING_TO_ARRAY_PKG_NEW.SPLIT_STRING(V_POA_KEY_STR,'|');

  FOR i IN 1..V_POA_KEY_ARR.COUNT LOOP

  V_NT_poa_input_array.EXTEND(1);
 -- v_input_string_arr_index := v_input_string_arr_index + 1;
  V_t_poa_input_array := STRING_TO_ARRAY_PKG_NEW.SPLIT_STRING_TO_ARRAY(V_POA_KEY_ARR(i),'#');
  V_NT_poa_input_array(i) := V_t_poa_input_array;

  END LOOP;

  -- Update cascade table that activity has just begun
  UPDATE CSCD_RQST
  SET CSCD_OPRTN_STTS_CD= 'STARTED',
    SPROC_PRCSG_STRT_TM    = SYSDATE,
    LAST_UPDT_DT      = SYSTIMESTAMP,
    LAST_UPDT_USER_ID = NVL(V_LAST_UPDT_USER_ID,'TERM_WRM')
  WHERE CSCD_RQST_ID  = V_CASCADE_ID;
  IF (SQL%ROWCOUNT    = 0) THEN
    v_err_message    := 'Invalid CASCADE_ID or there is no record in CSCD_RQST with cascade ID ' || V_CASCADE_ID;
    -- v_err_code := SQLCODE;
--    SELECT ERR_DTL_SQNC.nextval
--    INTO v_err_dtl_seq
--    FROM dual;
--    -- Insert descriptive message in Log table
--    INSERT
--    INTO ERR_DTL
--      (
--        ERR_RCRD_ID,
--        ERR_CD,
--        SRC_SYS_CD,
--        SRC_PROV_ID,
--        CMPNT_NM,
--        MSTR_PROV_ID,
--        IT_ERR_DESC,
--        CREAT_DT,
--        CREAT_USER_ID,
--        SRC_SRGT_KEY
--      )
--      VALUES
--      (
--        v_err_dtl_seq,
--        v_err_code ,
--        'SPDS-311',
--        V_CASCADE_ID,
--        'SP_BSNS_CSCD_TERM_WRMBORG_MUL',
--        V_MSTR_PROV_ID,
--        v_err_message ,
--        SYSDATE,
--        'SPS',
--        to_number(SYSDATE - to_date(v_JulianDate, 'DD-MON-YYYY')) * (24 * 60 * 60 * 1000)
--      ) ;
--    COMMIT;
    RAISE_APPLICATION_ERROR(-20000,'There is no record in CSCD_RQST with CASCADE_ID = ' || V_CASCADE_ID);
  END IF;
  COMMIT;
  -- Below block is for updating Child1 table - POA_RA_W
  v_aud_tab_flag := 0;
  FOR i IN 1..V_NT_poa_input_array.COUNT
  LOOP
    V_POA_KEY := V_NT_poa_input_array(i).POA_KEY;
    V_RA_TERMNTN_DT := V_NT_poa_input_array(i).TERMINATION_DATE;
    V_RA_TRMNTN_RSN_CD := V_NT_poa_input_array(i).REASON_CODE;

    FOR V_POA_RA_W IN c_POA_RA_W(V_POA_KEY)  LOOP
      BEGIN
        date_val_result                 := VALIDATE_DATES_REUIM_TEST(V_POA_RA_W.RA_EFCTV_DT, V_POA_RA_W.RA_TRMNTN_DT, V_RA_TRMNTN_RSN_CD, V_RA_TERMNTN_DT);
        IF date_val_result.IS_UPDATE_REQ = 'Y' THEN
          SELECT SYSDATE INTO v_transaction_start_date FROM DUAL;
          UPDATE POA_RA_W
          SET RA_TRMNTN_DT           = date_val_result.TERM_DATE,
            RA_TRMNTN_RSN_CD         = date_val_result.REASON_CODE,
            PROV_WTHLD_TRMNTN_DT     = date_val_result.TERM_DATE,
            PROV_WTHLD_TRMNTN_RSN_CD = date_val_result.REASON_CODE,
            LAST_UPDT_DT             = SYSTIMESTAMP,
            LAST_UPDT_USER_ID        = NVL(V_LAST_UPDT_USER_ID,'TERM_WRM')
          WHERE POA_RA_W_KEY         = V_POA_RA_W.POA_RA_W_KEY
          AND RA_1_ID                = V_RA_1_ID;
          SELECT SYSDATE INTO v_transaction_end_date FROM dual;
          IF v_aud_tab_flag = 0 THEN
            -- Update AUDIT table
            AUDIT_TAB_UPDATE('POA_RA_W',SYSTIMESTAMP,'UPDATE',V_POA_RA_W.RA_1_ID);
            v_aud_tab_flag := 1;
          END IF;
          -- Inserting record into CASCADE_AUDT_DTL nested table
          v_NT_CASCADE_AUDT_DTL.EXTEND(1);
          v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
          v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_POA_RA_W.POA_RA_W_KEY,'POA_RA_W','SUCCESS',v_transaction_start_date,v_transaction_end_date,'','',V_LAST_UPDT_USER_ID);
          v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
        END IF;
      EXCEPTION
      WHEN NO_DATA_FOUND THEN
        -- ROLLBACK;
        SELECT SYSDATE INTO v_transaction_end_date FROM dual;
        V_ERR_IND     := 1;
        v_err_message := 'Update failed in POA_RA_W due to NO_DATA_FOUND. Oracle error message is : ' || SQLERRM || '. Business keys are POA_KEY = ' || V_POA_KEY || ', RA_1_ID = ' || V_POA_RA_W.RA_1_ID || ', RA_SOR_CD = ' || V_POA_RA_W.RA_SOR_CD || ', RA_EFCTV_DT = ' || V_POA_RA_W.RA_EFCTV_DT;
        -- v_err_code := SQLCODE;
        --  SELECT ERR_DTL_SQNC.nextval INTO v_err_dtl_seq from dual;
        v_NT_ERR_DTL.EXTEND(1);
        v_err_tab_index               := v_err_tab_index + 1;
        v_t_err_dtl                   := t_err_dtl(v_err_code, v_src_sys_cd, v_src_prov_id, 'BSNS_CSCD_SP_' || to_char(current_timestamp, 'yyyy/mm/dd hh24:mi:ss.SS'), V_MSTR_PROV_ID, v_err_message, NULL, v_creat_user);
        v_NT_ERR_DTL(v_err_tab_index) := v_t_err_dtl;
        -- Inserting record into CASCADE_AUDT_DTL nested table
        v_NT_CASCADE_AUDT_DTL.EXTEND(1);
        v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
        v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_POA_RA_W.POA_RA_W_KEY,'POA_RA_W','FAIL',v_transaction_start_date,v_transaction_end_date,'DATA_ERROR','UPDATE in POA_RA_W failed',V_LAST_UPDT_USER_ID);
        v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
      WHEN TOO_MANY_ROWS THEN
        -- ROLLBACK;
        SELECT SYSDATE INTO v_transaction_end_date FROM dual;
        V_ERR_IND     := 1;
        v_err_message := 'Update failed in POA_RA_W due to TOO_MANY_ROWS. Oracle error message is : ' || SQLERRM || '. Business keys are POA_KEY = ' || V_POA_KEY || ', RA_1_ID = ' || V_POA_RA_W.RA_1_ID || ', RA_SOR_CD = ' || V_POA_RA_W.RA_SOR_CD || ', RA_EFCTV_DT = ' || V_POA_RA_W.RA_EFCTV_DT;
        -- v_err_code := SQLCODE;
        --  SELECT ERR_DTL_SQNC.nextval INTO v_err_dtl_seq from dual;
        v_NT_ERR_DTL.EXTEND(1);
        v_err_tab_index               := v_err_tab_index + 1;
        v_t_err_dtl                   := t_err_dtl(v_err_code, v_src_sys_cd, v_src_prov_id, 'BSNS_CSCD_SP_' || to_char(current_timestamp, 'yyyy/mm/dd hh24:mi:ss.SS'), V_MSTR_PROV_ID, v_err_message, NULL, v_creat_user);
        v_NT_ERR_DTL(v_err_tab_index) := v_t_err_dtl;
        -- Inserting record into CASCADE_AUDT_DTL nested table
        v_NT_CASCADE_AUDT_DTL.EXTEND(1);
        v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
        v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_POA_RA_W.POA_RA_W_KEY,'POA_RA_W','FAIL',v_transaction_start_date,v_transaction_end_date,'DATA_ERROR','UPDATE in POA_RA_W failed',V_LAST_UPDT_USER_ID);
        v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
      WHEN INVALID_NUMBER THEN
        -- ROLLBACK;
        SELECT SYSDATE INTO v_transaction_end_date FROM dual;
        V_ERR_IND     := 1;
        v_err_message := 'Update failed in POA_RA_W due to INVALID_NUMBER. Oracle error message is : ' || SQLERRM || '. Business keys are POA_KEY = ' || V_POA_KEY || ', RA_1_ID = ' || V_POA_RA_W.RA_1_ID || ', RA_SOR_CD = ' || V_POA_RA_W.RA_SOR_CD || ', RA_EFCTV_DT = ' || V_POA_RA_W.RA_EFCTV_DT;
        -- v_err_code := SQLCODE;
        --  SELECT ERR_DTL_SQNC.nextval INTO v_err_dtl_seq from dual;
        v_NT_ERR_DTL.EXTEND(1);
        v_err_tab_index               := v_err_tab_index + 1;
        v_t_err_dtl                   := t_err_dtl(v_err_code, v_src_sys_cd, v_src_prov_id, 'BSNS_CSCD_SP_' || to_char(current_timestamp, 'yyyy/mm/dd hh24:mi:ss.SS'), V_MSTR_PROV_ID, v_err_message, NULL, v_creat_user);
        v_NT_ERR_DTL(v_err_tab_index) := v_t_err_dtl;
        -- Inserting record into CASCADE_AUDT_DTL nested table
        v_NT_CASCADE_AUDT_DTL.EXTEND(1);
        v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
        v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_POA_RA_W.POA_RA_W_KEY,'POA_RA_W','FAIL',v_transaction_start_date,v_transaction_end_date,'DATA_ERROR','UPDATE in POA_RA_W failed',V_LAST_UPDT_USER_ID);
        v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
      WHEN OTHERS THEN
        -- ROLLBACK;
        SELECT SYSDATE INTO v_transaction_end_date FROM dual;
        V_ERR_IND     := 1;
        v_err_message := 'Update failed in POA_RA_W. Oracle error message is : ' || SQLERRM || '. Business keys are POA_KEY = ' || V_POA_KEY || ', RA_1_ID = ' || V_POA_RA_W.RA_1_ID || ', RA_SOR_CD = ' || V_POA_RA_W.RA_SOR_CD || ', RA_EFCTV_DT = ' || V_POA_RA_W.RA_EFCTV_DT;
        -- v_err_code := SQLCODE;
        --  SELECT ERR_DTL_SQNC.nextval INTO v_err_dtl_seq from dual;
        v_NT_ERR_DTL.EXTEND(1);
        v_err_tab_index               := v_err_tab_index + 1;
        v_t_err_dtl                   := t_err_dtl(v_err_code, v_src_sys_cd, v_src_prov_id, 'BSNS_CSCD_SP_' || to_char(current_timestamp, 'yyyy/mm/dd hh24:mi:ss.SS'), V_MSTR_PROV_ID, v_err_message, NULL, v_creat_user);
        v_NT_ERR_DTL(v_err_tab_index) := v_t_err_dtl;
        -- Inserting record into CASCADE_AUDT_DTL nested table
        v_NT_CASCADE_AUDT_DTL.EXTEND(1);
        v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
        v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_POA_RA_W.POA_RA_W_KEY,'POA_RA_W','FAIL',v_transaction_start_date,v_transaction_end_date,'DATA_ERROR','UPDATE in POA_RA_W failed',V_LAST_UPDT_USER_ID);
        v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
      END;
    END LOOP;
  END LOOP;
  -- Below block is for updating Child2 table - RLTD_PADRS
  v_aud_tab_flag := 0;
  FOR i IN 1..V_NT_poa_input_array.COUNT
  LOOP
    V_POA_KEY := V_NT_poa_input_array(i).POA_KEY;
    V_RA_TERMNTN_DT := V_NT_poa_input_array(i).TERMINATION_DATE;
    V_RA_TRMNTN_RSN_CD := V_NT_poa_input_array(i).REASON_CODE;

    FOR V_RLTD_PADRS IN c_rltd_padrs(V_MSTR_PROV_ID,V_POA_KEY)
    LOOP
      FOR V_RLTD_PADRS_RA_W IN c_RLTD_PADRS_RA_W(V_RLTD_PADRS.RLTD_PADRS_KEY)
      LOOP
        BEGIN
          date_val_result                 := VALIDATE_DATES_REUIM_TEST(V_RLTD_PADRS_RA_W.RA_EFCTV_DT, V_RLTD_PADRS_RA_W.RA_TRMNTN_DT, V_RA_TRMNTN_RSN_CD, V_RA_TERMNTN_DT);
          IF date_val_result.IS_UPDATE_REQ = 'Y' THEN
            SELECT SYSDATE INTO v_transaction_start_date FROM DUAL;
            UPDATE RLTD_PADRS_RA_W
            SET RA_TRMNTN_DT           = date_val_result.TERM_DATE,
              PROV_WTHLD_TRMNTN_DT     = date_val_result.TERM_DATE,
              RA_TRMNTN_RSN_CD         = date_val_result.REASON_CODE,
              PROV_WTHLD_TRMNTN_RSN_CD = date_val_result.REASON_CODE,
              LAST_UPDT_DT             = SYSTIMESTAMP,
              LAST_UPDT_USER_ID        = NVL(V_LAST_UPDT_USER_ID,'TERM_WRM')
            WHERE RLTD_PADRS_RA_W_KEY  = V_RLTD_PADRS_RA_W.RLTD_PADRS_RA_W_KEY
            AND RA_1_ID                = V_RA_1_ID;
            SELECT SYSDATE INTO v_transaction_end_date FROM dual;
            IF v_aud_tab_flag          = 0 THEN
              -- Update AUDIT table
              AUDIT_TAB_UPDATE('RLTD_PADRS_RA_W',SYSTIMESTAMP,'UPDATE',V_RLTD_PADRS.RLTD_MSTR_PROV_ID);
              v_aud_tab_flag := 1;
            END IF;
            -- Inserting record into CASCADE_AUDT_DTL nested table
            v_NT_CASCADE_AUDT_DTL.EXTEND(1);
            v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
            v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_RLTD_PADRS_RA_W.RLTD_PADRS_RA_W_KEY,'RLTD_PADRS_RA_W','SUCCESS',v_transaction_start_date,v_transaction_end_date,'','',V_LAST_UPDT_USER_ID);
            v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
          END IF;
        EXCEPTION
        WHEN NO_DATA_FOUND THEN
          -- ROLLBACK;
          SELECT SYSDATE INTO v_transaction_end_date FROM dual;
          V_ERR_IND     := 1;
          v_err_message := 'Update failed in RLTD_PADRS_RA_W due to NO_DATA_FOUND. Oracle error message is : ' || SQLERRM || '. Business keys are RLTD_PADRS_KEY = ' || V_RLTD_PADRS_RA_W.RLTD_PADRS_KEY || ', RA_1_ID = ' || V_RLTD_PADRS_RA_W.RA_1_ID || ', RA_EFCTV_DT = ' || V_RLTD_PADRS_RA_W.RA_EFCTV_DT;
          -- v_err_code := SQLCODE;
          --  SELECT ERR_DTL_SQNC.nextval INTO v_err_dtl_seq from dual;
          v_NT_ERR_DTL.EXTEND(1);
          v_err_tab_index               := v_err_tab_index + 1;
          v_t_err_dtl                   := t_err_dtl(v_err_code, v_src_sys_cd, v_src_prov_id, 'BSNS_CSCD_SP_' || to_char(current_timestamp, 'yyyy/mm/dd hh24:mi:ss.SS'), V_MSTR_PROV_ID, v_err_message, NULL, v_creat_user);
          v_NT_ERR_DTL(v_err_tab_index) := v_t_err_dtl;
          -- Inserting record into CASCADE_AUDT_DTL nested table
          v_NT_CASCADE_AUDT_DTL.EXTEND(1);
          v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
          v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_RLTD_PADRS_RA_W.RLTD_PADRS_RA_W_KEY,'RLTD_PADRS_RA_W','FAIL',v_transaction_start_date,v_transaction_end_date,'DATA_ERROR','UPDATE in RLTD_PADRS_RA_W failed',V_LAST_UPDT_USER_ID);
          v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
        WHEN TOO_MANY_ROWS THEN
          -- ROLLBACK;
          SELECT SYSDATE INTO v_transaction_end_date FROM dual;
          V_ERR_IND     := 1;
          v_err_message := 'Update failed in RLTD_PADRS_RA_W due to TOO_MANY_ROWS. Oracle error message is : ' || SQLERRM || '. Business keys are RLTD_PADRS_KEY = ' || V_RLTD_PADRS_RA_W.RLTD_PADRS_KEY || ', RA_1_ID = ' || V_RLTD_PADRS_RA_W.RA_1_ID || ', RA_EFCTV_DT = ' || V_RLTD_PADRS_RA_W.RA_EFCTV_DT;
          -- v_err_code := SQLCODE;
          --  SELECT ERR_DTL_SQNC.nextval INTO v_err_dtl_seq from dual;
          v_NT_ERR_DTL.EXTEND(1);
          v_err_tab_index               := v_err_tab_index + 1;
          v_t_err_dtl                   := t_err_dtl(v_err_code, v_src_sys_cd, v_src_prov_id, 'BSNS_CSCD_SP_' || to_char(current_timestamp, 'yyyy/mm/dd hh24:mi:ss.SS'), V_MSTR_PROV_ID, v_err_message, NULL, v_creat_user);
          v_NT_ERR_DTL(v_err_tab_index) := v_t_err_dtl;
          -- Inserting record into CASCADE_AUDT_DTL nested table
          v_NT_CASCADE_AUDT_DTL.EXTEND(1);
          v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
          v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_RLTD_PADRS_RA_W.RLTD_PADRS_RA_W_KEY,'RLTD_PADRS_RA_W','FAIL',v_transaction_start_date,v_transaction_end_date,'DATA_ERROR','UPDATE in RLTD_PADRS_RA_W failed',V_LAST_UPDT_USER_ID);
          v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
        WHEN INVALID_NUMBER THEN
          -- ROLLBACK;
          SELECT SYSDATE INTO v_transaction_end_date FROM dual;
          V_ERR_IND     := 1;
          v_err_message := 'Update failed in RLTD_PADRS_RA_W due to INVALID_NUMBER. Oracle error message is : ' || SQLERRM || '. Business keys are RLTD_PADRS_KEY = ' || V_RLTD_PADRS_RA_W.RLTD_PADRS_KEY || ', RA_1_ID = ' || V_RLTD_PADRS_RA_W.RA_1_ID || ', RA_EFCTV_DT = ' || V_RLTD_PADRS_RA_W.RA_EFCTV_DT;
          -- v_err_code := SQLCODE;
          --  SELECT ERR_DTL_SQNC.nextval INTO v_err_dtl_seq from dual;
          v_NT_ERR_DTL.EXTEND(1);
          v_err_tab_index               := v_err_tab_index + 1;
          v_t_err_dtl                   := t_err_dtl(v_err_code, v_src_sys_cd, v_src_prov_id, 'BSNS_CSCD_SP_' || to_char(current_timestamp, 'yyyy/mm/dd hh24:mi:ss.SS'), V_MSTR_PROV_ID, v_err_message, NULL, v_creat_user);
          v_NT_ERR_DTL(v_err_tab_index) := v_t_err_dtl;
          -- Inserting record into CASCADE_AUDT_DTL nested table
          v_NT_CASCADE_AUDT_DTL.EXTEND(1);
          v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
          v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_RLTD_PADRS_RA_W.RLTD_PADRS_RA_W_KEY,'RLTD_PADRS_RA_W','FAIL',v_transaction_start_date,v_transaction_end_date,'DATA_ERROR','UPDATE in RLTD_PADRS_RA_W failed',V_LAST_UPDT_USER_ID);
          v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
        WHEN OTHERS THEN
          -- ROLLBACK;
          SELECT SYSDATE INTO v_transaction_end_date FROM dual;
          V_ERR_IND     := 1;
          v_err_message := 'Update failed in RLTD_PADRS_RA_W. Oracle error message is : ' || SQLERRM || '. Business keys are RLTD_PADRS_KEY = ' || V_RLTD_PADRS_RA_W.RLTD_PADRS_KEY || ', RA_1_ID = ' || V_RLTD_PADRS_RA_W.RA_1_ID || ', RA_EFCTV_DT = ' || V_RLTD_PADRS_RA_W.RA_EFCTV_DT;
          -- v_err_code := SQLCODE;
          --  SELECT ERR_DTL_SQNC.nextval INTO v_err_dtl_seq from dual;
          v_NT_ERR_DTL.EXTEND(1);
          v_err_tab_index               := v_err_tab_index + 1;
          v_t_err_dtl                   := t_err_dtl(v_err_code, v_src_sys_cd, v_src_prov_id, 'BSNS_CSCD_SP_' || to_char(current_timestamp, 'yyyy/mm/dd hh24:mi:ss.SS'), V_MSTR_PROV_ID, v_err_message, NULL, v_creat_user);
          v_NT_ERR_DTL(v_err_tab_index) := v_t_err_dtl;
          -- Inserting record into CASCADE_AUDT_DTL nested table
          v_NT_CASCADE_AUDT_DTL.EXTEND(1);
          v_CASCADE_AUDT_DTL_index                        := v_CASCADE_AUDT_DTL_index + 1;
          v_t_CASCADE_AUDT_DTL                            := t_CASCADE_AUDT_DTL(V_CASCADE_ID,V_RLTD_PADRS_RA_W.RLTD_PADRS_RA_W_KEY,'RLTD_PADRS_RA_W','FAIL',v_transaction_start_date,v_transaction_end_date,'DATA_ERROR','UPDATE in RLTD_PADRS_RA_W failed',V_LAST_UPDT_USER_ID);
          v_NT_CASCADE_AUDT_DTL(v_CASCADE_AUDT_DTL_index) := v_t_CASCADE_AUDT_DTL;
        END;
      END LOOP;
    END LOOP;
  END LOOP;
  
  -- If all transactions are successfull, commit the changes in parent and child table and UPDATE CASCADE_REQUEST table
  IF V_ERR_IND = 0 THEN
    COMMIT;
    
    UPDATE CSCD_RQST
    SET CSCD_OPRTN_STTS_CD   = 'SUCCESS',
      SPROC_PRCSG_END_TM       = SYSDATE,
      LAST_UPDT_DT      = SYSTIMESTAMP,
      LAST_UPDT_USER_ID = NVL(V_LAST_UPDT_USER_ID,'TERM_WRM')
    WHERE CSCD_RQST_ID    = V_CASCADE_ID;
    COMMIT;
  ELSE
    ROLLBACK;
    UPDATE CSCD_RQST
    SET CSCD_OPRTN_STTS_CD = 'FAILED',
      SPROC_PRCSG_END_TM       = SYSDATE,
      LAST_UPDT_DT      = SYSTIMESTAMP,
      LAST_UPDT_USER_ID = NVL(V_LAST_UPDT_USER_ID,'TERM_WRM')
    WHERE CSCD_RQST_ID    = V_CASCADE_ID;
    COMMIT;
  END IF;
  
  -- Call procedure to insert into CASCADE_AUDIT_DTL table
  IF v_NT_CASCADE_AUDT_DTL.COUNT > 0 THEN
  SP_INS_CASCADE_AUDT_DTL(v_NT_CASCADE_AUDT_DTL);
  END IF;
  -- Call procedure to insert all error records in ERR_DTL table
  IF v_NT_ERR_DTL.COUNT > 0 THEN
    SP_INS_ERR_DTL(v_NT_ERR_DTL);
  END IF;
  
EXCEPTION
WHEN OTHERS THEN -- Write in ERR LOGGING table & STANDARD CATCH ERRORS AND PUT IT IN ERR_DTL & Initiate Rollback & set flag as 'N'
  ROLLBACK;
  v_err_message := 'Stored proc SP_BSNS_CSCD_TERM_WRMBORG_MUL failed. Oracle error message is ' || SQLERRM;
  -- v_err_code := SQLCODE;
  SELECT ERR_DTL_SQNC.nextval
  INTO v_err_dtl_seq
  FROM dual;
  -- Insert descriptive message in Log table
  INSERT
  INTO ERR_DTL
    (
      ERR_RCRD_ID,
      ERR_CD,
      SRC_SYS_CD,
      SRC_PROV_ID,
      CMPNT_NM,
      MSTR_PROV_ID,
      IT_ERR_DESC,
      CREAT_DT,
      CREAT_USER_ID,
      SRC_SRGT_KEY
    )
    VALUES
    (
       v_err_dtl_seq,
      v_err_code,
      v_src_sys_cd,
      V_CASCADE_ID,
      'BSNS_CSCD_SP_' || to_char(current_timestamp, 'yyyy/mm/dd hh24:mi:ss.SS'),
      V_MSTR_PROV_ID,
      v_err_message ,
      SYSTIMESTAMP,
      v_creat_user,
      NULL
    ) ;
    
    UPDATE CSCD_RQST
    SET CSCD_OPRTN_STTS_CD = 'FAILED',
      SPROC_PRCSG_END_TM       = SYSDATE,
      LAST_UPDT_DT      = SYSTIMESTAMP,
      LAST_UPDT_USER_ID = NVL(V_LAST_UPDT_USER_ID,'TERM_WRM')
    WHERE CSCD_RQST_ID    = V_CASCADE_ID;
    
  COMMIT;
END SP_BSNS_TERM_WRMBORG_MUL_TEST;

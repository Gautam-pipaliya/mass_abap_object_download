*&---------------------------------------------------------------------*
*& Report ZDTP_MASSDOWNLOAD
*&---------------------------------------------------------------------*
*& The Ultimate RICEFW SAP Extractor
*& PROG (Reports & Module Pools), CLAS/INTF, FUGR, ENHO, TRAN,
*& IDOC, ODATA, DDLS (CDS), DDIC + Table Data (CSV),
*& Forms (SmartForms, Adobe Forms, SAPScripts) exported as XML.
*&---------------------------------------------------------------------*
REPORT zdtp_massdownload.

TYPE-POOLS: icon.
TABLES: tadir.

*----------------------------------------------------------------------*
* SELECTION SCREEN
*----------------------------------------------------------------------*
SELECTION-SCREEN BEGIN OF BLOCK b_code WITH FRAME TITLE tit1.
  PARAMETERS: p_prog AS CHECKBOX DEFAULT 'X', " Reports & Module Pools
              p_clas AS CHECKBOX DEFAULT 'X', " Classes & Interfaces
              p_fugr AS CHECKBOX DEFAULT 'X', " Function Groups / FMs
              p_enho AS CHECKBOX DEFAULT 'X'. " Enhancements (ENHO)
SELECTION-SCREEN END OF BLOCK b_code.

SELECTION-SCREEN BEGIN OF BLOCK b_rice WITH FRAME TITLE tit2.
  PARAMETERS: p_tran AS CHECKBOX DEFAULT 'X', " Transactions (TRAN)
              p_form AS CHECKBOX DEFAULT 'X', " Forms to XML (SFP, SmartForms)
              p_idoc AS CHECKBOX DEFAULT 'X', " IDocs (Basic Types)
              p_odat AS CHECKBOX DEFAULT 'X', " OData / API Services
              p_cds  AS CHECKBOX DEFAULT 'X'. " CDS Views (DDLS)
SELECTION-SCREEN END OF BLOCK b_rice.

SELECTION-SCREEN BEGIN OF BLOCK b_ddic WITH FRAME TITLE tit3.
  PARAMETERS: p_ddic  AS CHECKBOX DEFAULT 'X', " DDIC (TABL, TTYP, DTEL, DOMA)
              p_tdata AS CHECKBOX,             " Download Table Data to CSV
              p_tmax  TYPE i DEFAULT 100.      " Max Rows per table data
SELECTION-SCREEN END OF BLOCK b_ddic.

SELECTION-SCREEN BEGIN OF BLOCK b_filt WITH FRAME TITLE tit4.
  SELECT-OPTIONS: s_obj  FOR tadir-obj_name,
                  s_devc FOR tadir-devclass.
  SELECTION-SCREEN SKIP.
  PARAMETERS: p_cstchk AS CHECKBOX DEFAULT 'X', " Custom Objects (Z*/Y*)
              p_stdchk AS CHECKBOX.             " Standard Objects (SAP)
SELECTION-SCREEN END OF BLOCK b_filt.

SELECTION-SCREEN BEGIN OF BLOCK b_out WITH FRAME TITLE tit5.
  PARAMETERS: p_limit TYPE i DEFAULT 10000.     " Max Objects Guard
  PARAMETERS: p_incl  AS CHECKBOX DEFAULT 'X',  " Include PROG includes
              p_linc  AS CHECKBOX DEFAULT 'X'.  " Include CLAS includes
  SELECTION-SCREEN SKIP.
  PARAMETERS: p_path TYPE string LOWER CASE OBLIGATORY.
  PARAMETERS: p_zip  RADIOBUTTON GROUP r2 DEFAULT 'X',
              p_file RADIOBUTTON GROUP r2.
SELECTION-SCREEN END OF BLOCK b_out.

*----------------------------------------------------------------------*
* INITIALIZATION
*----------------------------------------------------------------------*
INITIALIZATION.
  tit1 = '1. RICEFW: Code & Module Pools'.
  tit2 = '2. RICEFW: Forms (XML), APIs & Views'.
  tit3 = '3. Data Dictionary & Table Extraction'.
  tit4 = '4. Scope & Filters'.
  tit5 = '5. Output Settings'.

AT SELECTION-SCREEN ON VALUE-REQUEST FOR p_path.
  DATA: lv_sel_folder TYPE string.
  cl_gui_frontend_services=>directory_browse(
    EXPORTING initial_folder  = p_path
    CHANGING  selected_folder = lv_sel_folder
    EXCEPTIONS OTHERS         = 4 ).
  IF sy-subrc = 0 AND lv_sel_folder IS NOT INITIAL.
    p_path = lv_sel_folder.
  ENDIF.

*----------------------------------------------------------------------*
* LOCAL CLASS DEFINITION
*----------------------------------------------------------------------*
CLASS lcl_mass_downloader DEFINITION FINAL.
  PUBLIC SECTION.
    METHODS execute.

  PRIVATE SECTION.
    TYPES: BEGIN OF ty_object,
             obj_type TYPE tadir-object,
             obj_name TYPE tadir-obj_name,
             devclass TYPE tadir-devclass,
             author   TYPE tadir-author,
           END OF ty_object.

    TYPES: BEGIN OF ty_log,
             icon     TYPE icon_d,
             obj_type TYPE tadir-object,
             obj_name TYPE tadir-obj_name,
             message  TYPE string,
           END OF ty_log.

    DATA: mt_objects TYPE STANDARD TABLE OF ty_object,
          mt_log     TYPE STANDARD TABLE OF ty_log,
          mv_success TYPE i,
          mv_error   TYPE i.

    METHODS fetch_objects.
    METHODS process_downloads.

    " Source & Data Extractors
    METHODS get_prog_source  IMPORTING iv_name TYPE tadir-obj_name RETURNING VALUE(rt_src) TYPE string_table.
    METHODS get_clas_source  IMPORTING iv_name TYPE tadir-obj_name RETURNING VALUE(rt_src) TYPE string_table.
    METHODS get_fugr_source  IMPORTING iv_name TYPE tadir-obj_name RETURNING VALUE(rt_src) TYPE string_table.
    METHODS get_enho_source  IMPORTING iv_name TYPE tadir-obj_name RETURNING VALUE(rt_src) TYPE string_table.
    METHODS get_cds_source   IMPORTING iv_name TYPE tadir-obj_name RETURNING VALUE(rt_src) TYPE string_table.
    METHODS get_ddic_source  IMPORTING iv_name TYPE tadir-obj_name iv_type TYPE tadir-object RETURNING VALUE(rt_src) TYPE string_table.
    METHODS get_table_data   IMPORTING iv_name TYPE tadir-obj_name RETURNING VALUE(rt_src) TYPE string_table.
    METHODS get_dynpro_flow  IMPORTING iv_name TYPE tadir-obj_name RETURNING VALUE(rt_src) TYPE string_table.

    " Metadata Extractors
    METHODS get_tran_meta    IMPORTING iv_name TYPE tadir-obj_name RETURNING VALUE(rt_src) TYPE string_table.
    METHODS get_idoc_meta    IMPORTING iv_name TYPE tadir-obj_name RETURNING VALUE(rt_src) TYPE string_table.
    METHODS get_odata_meta   IMPORTING iv_name TYPE tadir-obj_name iv_type TYPE tadir-object RETURNING VALUE(rt_src) TYPE string_table.

    " XML Form Extractor
    METHODS get_form_xml     IMPORTING iv_name TYPE tadir-obj_name iv_type TYPE tadir-object RETURNING VALUE(rt_xstr) TYPE xstring.

    METHODS build_header     IMPORTING is_obj TYPE ty_object RETURNING VALUE(rt_src) TYPE string_table.
    METHODS display_log.
    METHODS add_log          IMPORTING iv_icon TYPE icon_d iv_type TYPE tadir-object iv_name TYPE tadir-obj_name iv_msg TYPE string.
ENDCLASS.

*----------------------------------------------------------------------*
* LOCAL CLASS IMPLEMENTATION
*----------------------------------------------------------------------*
CLASS lcl_mass_downloader IMPLEMENTATION.

  METHOD execute.
    IF p_cstchk = abap_false AND p_stdchk = abap_false.
      MESSAGE 'Please select Custom or Standard objects.' TYPE 'E'.
    ENDIF.

    fetch_objects( ).

    IF mt_objects IS INITIAL.
      MESSAGE 'No objects found matching criteria.' TYPE 'S' DISPLAY LIKE 'E'.
      RETURN.
    ENDIF.

    process_downloads( ).
    display_log( ).
  ENDMETHOD.

  METHOD fetch_objects.
    DATA: lr_type TYPE RANGE OF tadir-object,
          lr_name TYPE RANGE OF tadir-obj_name.

    IF p_prog = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'PROG' ) TO lr_type.
    ENDIF.

    IF p_clas = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'CLAS' ) TO lr_type.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'INTF' ) TO lr_type.
    ENDIF.

    IF p_fugr = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'FUGR' ) TO lr_type.
    ENDIF.

    IF p_enho = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'ENHO' ) TO lr_type.
    ENDIF.

    IF p_cds = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'DDLS' ) TO lr_type.
    ENDIF.

    IF p_tran = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'TRAN' ) TO lr_type.
    ENDIF.

    IF p_idoc = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'IDOC' ) TO lr_type.
    ENDIF.

    IF p_form = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'SSFO' ) TO lr_type. " SmartForms
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'SFPF' ) TO lr_type. " Adobe Forms
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'FORM' ) TO lr_type. " SAPScripts
    ENDIF.

    IF p_odat = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'IWSV' ) TO lr_type.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'IWMO' ) TO lr_type.
    ENDIF.

    IF p_ddic = abap_true.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'TABL' ) TO lr_type.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'TTYP' ) TO lr_type.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'DTEL' ) TO lr_type.
      APPEND VALUE #( sign = 'I' option = 'EQ' low = 'DOMA' ) TO lr_type.
    ENDIF.

    IF lr_type IS INITIAL.
      RETURN.
    ENDIF.

    IF p_cstchk = abap_true AND p_stdchk = abap_false.
      APPEND VALUE #( sign = 'I' option = 'CP' low = 'Z*' ) TO lr_name.
      APPEND VALUE #( sign = 'I' option = 'CP' low = 'Y*' ) TO lr_name.
    ELSEIF p_stdchk = abap_true AND p_cstchk = abap_false.
      APPEND VALUE #( sign = 'E' option = 'CP' low = 'Z*' ) TO lr_name.
      APPEND VALUE #( sign = 'E' option = 'CP' low = 'Y*' ) TO lr_name.
    ENDIF.

    SELECT object, obj_name, devclass, author
      FROM tadir
      INTO TABLE @mt_objects
      UP TO @p_limit ROWS
      WHERE pgmid    = 'R3TR'
        AND object   IN @lr_type
        AND obj_name IN @s_obj
        AND obj_name IN @lr_name
        AND devclass IN @s_devc.
  ENDMETHOD.

  METHOD process_downloads.
    DATA: lo_zip    TYPE REF TO cl_abap_zip,
          lt_source TYPE string_table,
          lv_str    TYPE string,
          lv_xstr   TYPE xstring,
          lv_total  TYPE i,
          lv_curr   TYPE i,
          lv_pct    TYPE i,
          lv_msg    TYPE char70,
          lv_fname  TYPE string,
          lv_is_xml TYPE abap_bool.

    lv_total = lines( mt_objects ).
    IF p_zip = abap_true.
      CREATE OBJECT lo_zip.
    ENDIF.

    LOOP AT mt_objects INTO DATA(ls_obj).
      lv_curr = lv_curr + 1.
      lv_pct  = ( lv_curr * 100 ) / lv_total.
      lv_msg  = |Processing { ls_obj-obj_type } { ls_obj-obj_name } ({ lv_curr }/{ lv_total })|.

      CALL FUNCTION 'SAPGUI_PROGRESS_INDICATOR'
        EXPORTING
          percentage = lv_pct
          text       = lv_msg.

      CLEAR: lt_source, lv_xstr, lv_is_xml.
      lt_source = build_header( ls_obj ).

      CASE ls_obj-obj_type.
        WHEN 'SSFO' OR 'SFPF' OR 'FORM'.
          lv_xstr   = get_form_xml( iv_name = ls_obj-obj_name iv_type = ls_obj-obj_type ).
          lv_fname  = |{ ls_obj-obj_name }.{ ls_obj-obj_type }.xml|.
          lv_is_xml = abap_true.

        WHEN 'PROG'.
          APPEND LINES OF get_prog_source( ls_obj-obj_name ) TO lt_source.
          lv_fname = |{ ls_obj-obj_name }.prog.abap|.

        WHEN 'CLAS' OR 'INTF'.
          APPEND LINES OF get_clas_source( ls_obj-obj_name ) TO lt_source.
          lv_fname = COND #( WHEN ls_obj-obj_type = 'CLAS'
                             THEN |{ ls_obj-obj_name }.clas.abap|
                             ELSE |{ ls_obj-obj_name }.intf.abap| ).

        WHEN 'FUGR'.
          APPEND LINES OF get_fugr_source( ls_obj-obj_name ) TO lt_source.
          lv_fname = |{ ls_obj-obj_name }.fugr.abap|.

        WHEN 'ENHO'.
          APPEND LINES OF get_enho_source( ls_obj-obj_name ) TO lt_source.
          lv_fname = |{ ls_obj-obj_name }.enho.abap|.

        WHEN 'DDLS'.
          APPEND LINES OF get_cds_source( ls_obj-obj_name ) TO lt_source.
          lv_fname = |{ ls_obj-obj_name }.ddls.asddls|.

        WHEN 'TABL' OR 'TTYP' OR 'DTEL' OR 'DOMA'.
          APPEND LINES OF get_ddic_source( iv_name = ls_obj-obj_name iv_type = ls_obj-obj_type ) TO lt_source.
          lv_fname = |{ ls_obj-obj_name }.{ ls_obj-obj_type }.txt|.

        WHEN 'TRAN'.
          APPEND LINES OF get_tran_meta( ls_obj-obj_name ) TO lt_source.
          lv_fname = |{ ls_obj-obj_name }.tran.txt|.

        WHEN 'IDOC'.
          APPEND LINES OF get_idoc_meta( ls_obj-obj_name ) TO lt_source.
          lv_fname = |{ ls_obj-obj_name }.idoc.txt|.

        WHEN 'IWSV' OR 'IWMO'.
          APPEND LINES OF get_odata_meta( iv_name = ls_obj-obj_name iv_type = ls_obj-obj_type ) TO lt_source.
          lv_fname = |{ ls_obj-obj_name }.odata.txt|.
      ENDCASE.

      " Convert String Table for non-XML outputs
      IF lv_is_xml = abap_false AND lines( lt_source ) > 5.
        lv_str  = concat_lines_of( table = lt_source sep = cl_abap_char_utilities=>cr_lf ).
        lv_xstr = cl_abap_codepage=>convert_to( source = lv_str ).
      ENDIF.

      " Save Logic (XML or Text)
      IF xstrlen( lv_xstr ) > 0.
        IF p_zip = abap_true.
          lo_zip->add( name = lv_fname content = lv_xstr ).
          add_log( iv_icon = icon_green_light
                   iv_type = ls_obj-obj_type
                   iv_name = ls_obj-obj_name
                   iv_msg  = 'Extracted' ).
          mv_success = mv_success + 1.
        ELSE.
          DATA(lv_full_path) = |{ p_path }\\{ lv_fname }|.
          DATA(lt_bin)       = cl_bcs_convert=>xstring_to_solix( lv_xstr ).
          cl_gui_frontend_services=>gui_download(
            EXPORTING filename     = lv_full_path
                      filetype     = 'BIN'
                      bin_filesize = xstrlen( lv_xstr )
            CHANGING  data_tab     = lt_bin
            EXCEPTIONS OTHERS      = 1 ).
          IF sy-subrc = 0.
            add_log( iv_icon = icon_green_light
                     iv_type = ls_obj-obj_type
                     iv_name = ls_obj-obj_name
                     iv_msg  = 'File saved' ).
            mv_success = mv_success + 1.
          ELSE.
            add_log( iv_icon = icon_red_light
                     iv_type = ls_obj-obj_type
                     iv_name = ls_obj-obj_name
                     iv_msg  = 'GUI Download failed' ).
            mv_error = mv_error + 1.
          ENDIF.
        ENDIF.
      ELSE.
        add_log( iv_icon = icon_red_light
                 iv_type = ls_obj-obj_type
                 iv_name = ls_obj-obj_name
                 iv_msg  = 'Empty / Not Found' ).
        mv_error = mv_error + 1.
      ENDIF.

      " Extract Table Data to CSV
      IF ls_obj-obj_type = 'TABL' AND p_tdata = abap_true.
        DATA(lt_csv) = get_table_data( ls_obj-obj_name ).
        IF lines( lt_csv ) > 1.
          DATA(lv_csv_str)   = concat_lines_of( table = lt_csv sep = cl_abap_char_utilities=>cr_lf ).
          DATA(lv_csv_xstr)  = cl_abap_codepage=>convert_to( source = lv_csv_str ).
          DATA(lv_csv_fname) = |{ ls_obj-obj_name }_DATA.csv|.

          IF p_zip = abap_true.
            lo_zip->add( name = lv_csv_fname content = lv_csv_xstr ).
            add_log( iv_icon = icon_xls
                     iv_type = 'DATA'
                     iv_name = ls_obj-obj_name
                     iv_msg  = 'Data CSV Extracted' ).
          ELSE.
            DATA(lv_csv_path) = |{ p_path }\\{ lv_csv_fname }|.
            DATA(lt_csv_bin)  = cl_bcs_convert=>xstring_to_solix( lv_csv_xstr ).
            cl_gui_frontend_services=>gui_download(
              EXPORTING filename     = lv_csv_path
                        filetype     = 'BIN'
                        bin_filesize = xstrlen( lv_csv_xstr )
              CHANGING  data_tab     = lt_csv_bin
              EXCEPTIONS OTHERS      = 1 ).
          ENDIF.
        ENDIF.
      ENDIF.
    ENDLOOP.

    IF p_zip = abap_true AND mv_success > 0.
      DATA(lv_zip_xstr) = lo_zip->save( ).
      DATA(lv_zip_path) = |{ p_path }\\ZDTP_MASSDOWNLOAD_{ sy-datum }_{ sy-uzeit }.zip|.
      DATA(lt_zip_bin)  = cl_bcs_convert=>xstring_to_solix( lv_zip_xstr ).

      cl_gui_frontend_services=>gui_download(
        EXPORTING filename     = lv_zip_path
                  filetype     = 'BIN'
                  bin_filesize = xstrlen( lv_zip_xstr )
        CHANGING  data_tab     = lt_zip_bin
        EXCEPTIONS OTHERS      = 1 ).
      IF sy-subrc <> 0.
        MESSAGE 'Failed to save ZIP file to PC.' TYPE 'E'.
      ENDIF.
    ENDIF.
  ENDMETHOD.

  " ==========================================
  " XML FORMS EXTRACTOR (SMARTFORMS & SFP)
  " ==========================================
  METHOD get_form_xml.
    DATA: lt_adm  TYPE TABLE OF stxfadm,
          lt_txt  TYPE TABLE OF stxftxt,
          lt_stxh TYPE TABLE OF stxh.

    " Dynamic references for Adobe Forms (prevents syntax errors if ADS is absent)
    DATA: dref_fpdir TYPE REF TO data,
          dref_fpctx TYPE REF TO data,
          dref_fplay TYPE REF TO data.
    FIELD-SYMBOLS: <lt_fpdir> TYPE ANY TABLE,
                   <lt_fpctx> TYPE ANY TABLE,
                   <lt_fplay> TYPE ANY TABLE.

    CASE iv_type.
      WHEN 'SSFO'. " SmartForms
        SELECT * FROM stxfadm INTO TABLE @lt_adm WHERE formname = @iv_name.
        SELECT * FROM stxftxt INTO TABLE @lt_txt WHERE formname = @iv_name.
        IF lt_adm IS NOT INITIAL OR lt_txt IS NOT INITIAL.
          CALL TRANSFORMATION id SOURCE adm = lt_adm txt = lt_txt RESULT XML rt_xstr.
        ENDIF.

      WHEN 'SFPF'. " Adobe Forms (Dynamic Database Fetch)
        TRY.
            DATA(lv_tab_dir) = 'FPDIR'.
            DATA(lv_tab_ctx) = 'FPCONTEXT'.
            DATA(lv_tab_lay) = 'FPLAYOUTT'.

            CREATE DATA dref_fpdir TYPE TABLE OF (lv_tab_dir).
            CREATE DATA dref_fpctx TYPE TABLE OF (lv_tab_ctx).
            CREATE DATA dref_fplay TYPE TABLE OF (lv_tab_lay).

            ASSIGN dref_fpdir->* TO <lt_fpdir>.
            ASSIGN dref_fpctx->* TO <lt_fpctx>.
            ASSIGN dref_fplay->* TO <lt_fplay>.

            SELECT * FROM (lv_tab_dir) INTO TABLE @<lt_fpdir> WHERE name = @iv_name.
            SELECT * FROM (lv_tab_ctx) INTO TABLE @<lt_fpctx> WHERE name = @iv_name.
            SELECT * FROM (lv_tab_lay) INTO TABLE @<lt_fplay> WHERE name = @iv_name.

            IF <lt_fpdir> IS ASSIGNED AND <lt_fpdir> IS NOT INITIAL.
              CALL TRANSFORMATION id
                SOURCE dir = <lt_fpdir> ctx = <lt_fpctx> lay = <lt_fplay>
                RESULT XML rt_xstr.
            ENDIF.
          CATCH cx_root.
            " Safely ignore if Adobe Forms tables are not installed on this system
        ENDTRY.

      WHEN 'FORM'. " SAPScripts
        SELECT * FROM stxh INTO TABLE @lt_stxh WHERE tdobject = 'FORM' AND tdname = @iv_name.
        IF sy-subrc = 0.
          CALL TRANSFORMATION id SOURCE stxh = lt_stxh RESULT XML rt_xstr.
        ENDIF.
    ENDCASE.
  ENDMETHOD.

  " ==========================================
  " MODULE POOL DYNPRO SCREEN LOGIC
  " ==========================================
  METHOD get_dynpro_flow.
    " D022S is a structure in modern SAP systems, not a transparent table.
    APPEND |* ====================================================| TO rt_src.
    APPEND |* MODULE POOL SCREEN FLOW LOGIC (DYNP)| TO rt_src.
    APPEND |* (Extraction bypassed: Native DB read restricted by SAP)| TO rt_src.
    APPEND |* ====================================================| TO rt_src.
  ENDMETHOD.

  " ==========================================
  " RICEFW METADATA EXTRACTORS
  " ==========================================
  METHOD get_tran_meta.
    SELECT SINGLE * FROM tstc INTO @DATA(ls_tstc) WHERE tcode = @iv_name.
    IF sy-subrc = 0.
      APPEND |TRANSACTION CODE : { iv_name }| TO rt_src.
      APPEND |PROGRAM          : { ls_tstc-pgmna }| TO rt_src.
      APPEND |SCREEN NUMBER    : { ls_tstc-dypno }| TO rt_src.
    ENDIF.
  ENDMETHOD.

  METHOD get_idoc_meta.
    APPEND |IDOC BASIC TYPE  : { iv_name }| TO rt_src.
    SELECT segtyp FROM edisyn INTO TABLE @DATA(lt_seg) WHERE idoctyp = @iv_name.
    IF sy-subrc = 0.
      APPEND |----------------------------------------------------| TO rt_src.
      APPEND |ASSOCIATED SEGMENTS:| TO rt_src.
      LOOP AT lt_seg INTO DATA(ls_seg).
        APPEND |- { ls_seg-segtyp }| TO rt_src.
      ENDLOOP.
    ENDIF.
  ENDMETHOD.

  METHOD get_odata_meta.
    IF iv_type = 'IWSV'.
      APPEND |ODATA SERVICE    : { iv_name }| TO rt_src.
    ELSE.
      APPEND |ODATA MODEL      : { iv_name }| TO rt_src.
    ENDIF.
  ENDMETHOD.

  " ==========================================
  " STANDARD SOURCE EXTRACTORS
  " ==========================================
  METHOD get_enho_source.
    DATA: lt_raw TYPE TABLE OF string.
    SELECT enhinclude FROM enhincinx INTO TABLE @DATA(lt_enh) WHERE enhname = @iv_name.
    IF sy-subrc = 0.
      APPEND |* ----------------------------------------------------| TO rt_src.
      APPEND |* ENHANCEMENT IMPLEMENTATION: { iv_name }| TO rt_src.
      APPEND |* ----------------------------------------------------| TO rt_src.
      LOOP AT lt_enh INTO DATA(ls_enh).
        CLEAR lt_raw.
        READ REPORT ls_enh-enhinclude INTO lt_raw STATE 'A'.
        IF sy-subrc = 0 AND lines( lt_raw ) > 0.
          APPEND |* >>> ENHANCEMENT INCLUDE: { ls_enh-enhinclude }| TO rt_src.
          APPEND LINES OF lt_raw TO rt_src.
        ENDIF.
      ENDLOOP.
    ENDIF.
  ENDMETHOD.

  METHOD get_cds_source.
    DATA: lv_source TYPE string,
          lt_raw    TYPE TABLE OF string.
    SELECT SINGLE source FROM ddddlsrc INTO @lv_source WHERE ddlname = @iv_name.
    IF sy-subrc = 0.
      SPLIT lv_source AT cl_abap_char_utilities=>newline INTO TABLE lt_raw.
      APPEND LINES OF lt_raw TO rt_src.
    ENDIF.
  ENDMETHOD.

  METHOD get_table_data.
    DATA: dref_tab TYPE REF TO data,
          lv_line  TYPE string,
          lv_val   TYPE string.
    FIELD-SYMBOLS: <fs_tab>   TYPE ANY TABLE,
                   <fs_row>   TYPE any,
                   <fs_field> TYPE any.

    SELECT SINGLE tabclass FROM dd02l INTO @DATA(lv_class) WHERE tabname = @iv_name.
    IF lv_class <> 'TRANSP' AND lv_class <> 'POOL' AND lv_class <> 'CLUSTER'.
      RETURN.
    ENDIF.

    TRY.
        CREATE DATA dref_tab TYPE TABLE OF (iv_name).
        ASSIGN dref_tab->* TO <fs_tab>.
        SELECT * FROM (iv_name) INTO TABLE @<fs_tab> UP TO @p_tmax ROWS.
      CATCH cx_root.
        RETURN.
    ENDTRY.

    IF lines( <fs_tab> ) = 0.
      RETURN.
    ENDIF.

    SELECT fieldname FROM dd03l INTO TABLE @DATA(lt_fields)
      WHERE tabname = @iv_name AND as4local = 'A'
      ORDER BY position.

    LOOP AT lt_fields INTO DATA(ls_field).
      IF sy-tabix = 1.
        lv_line = ls_field-fieldname.
      ELSE.
        lv_line = |{ lv_line },{ ls_field-fieldname }|.
      ENDIF.
    ENDLOOP.
    APPEND lv_line TO rt_src.

    LOOP AT <fs_tab> ASSIGNING <fs_row>.
      CLEAR lv_line.
      DO.
        ASSIGN COMPONENT sy-index OF STRUCTURE <fs_row> TO <fs_field>.
        IF sy-subrc <> 0.
          EXIT.
        ENDIF.
        lv_val = |{ <fs_field> }|.
        REPLACE ALL OCCURRENCES OF '"' IN lv_val WITH ''''.
        IF lv_val CS ','.
          lv_val = |"{ lv_val }"|.
        ENDIF.
        IF sy-index = 1.
          lv_line = lv_val.
        ELSE.
          lv_line = |{ lv_line },{ lv_val }|.
        ENDIF.
      ENDDO.
      APPEND lv_line TO rt_src.
    ENDLOOP.
  ENDMETHOD.

  METHOD get_prog_source.
    DATA: lt_raw TYPE TABLE OF string,
          lt_inc TYPE TABLE OF progname.

    READ REPORT iv_name INTO lt_raw STATE 'A'.
    IF sy-subrc = 0.
      APPEND LINES OF lt_raw TO rt_src.
    ENDIF.

    IF p_incl = abap_true.
      CALL FUNCTION 'GET_INCLUDETAB'
        EXPORTING progname = iv_name
        TABLES    incltab  = lt_inc
        EXCEPTIONS OTHERS  = 1.

      LOOP AT lt_inc INTO DATA(lv_inc).
        CLEAR lt_raw.
        READ REPORT lv_inc INTO lt_raw STATE 'A'.
        IF sy-subrc = 0.
          APPEND |* >>> INCLUDE { lv_inc }| TO rt_src.
          APPEND LINES OF lt_raw TO rt_src.
        ENDIF.
      ENDLOOP.
    ENDIF.

    APPEND LINES OF get_dynpro_flow( iv_name ) TO rt_src.
  ENDMETHOD.

  METHOD get_clas_source.
    DATA: lt_inc   TYPE TABLE OF progname,
          lt_raw   TYPE TABLE OF string,
          lv_class TYPE seoclsname.

    lv_class = iv_name.

    TRY.
        cl_oo_classname_service=>get_all_class_includes(
          EXPORTING  class_name = lv_class
          RECEIVING  result     = lt_inc
          EXCEPTIONS OTHERS     = 1 ).
      CATCH cx_root.
        DATA(lv_search) = |{ iv_name }%IU|.
        SELECT SINGLE name FROM trdir INTO @DATA(lv_intf_prog) WHERE name LIKE @lv_search.
        IF sy-subrc = 0.
          APPEND lv_intf_prog TO lt_inc.
        ENDIF.
    ENDTRY.

    LOOP AT lt_inc INTO DATA(lv_inc).
      CLEAR lt_raw.
      READ REPORT lv_inc INTO lt_raw STATE 'A'.
      IF sy-subrc = 0 AND lines( lt_raw ) > 0 AND p_linc = abap_true.
        APPEND |* >>> INCLUDE { lv_inc }| TO rt_src.
        APPEND LINES OF lt_raw TO rt_src.
      ENDIF.
    ENDLOOP.
  ENDMETHOD.

  METHOD get_fugr_source.
    DATA: lt_raw TYPE TABLE OF string,
          lt_inc TYPE TABLE OF progname.

    DATA(lv_prog) = CONV progname( |SAPL{ iv_name }| ).
    READ REPORT lv_prog INTO lt_raw STATE 'A'.
    IF sy-subrc = 0.
      APPEND |* MAIN PROGRAM: { lv_prog }| TO rt_src.
      APPEND LINES OF lt_raw TO rt_src.
    ENDIF.

    CALL FUNCTION 'GET_INCLUDETAB'
      EXPORTING progname = lv_prog
      TABLES    incltab  = lt_inc
      EXCEPTIONS OTHERS  = 1.

    LOOP AT lt_inc INTO DATA(lv_inc).
      CLEAR lt_raw.
      READ REPORT lv_inc INTO lt_raw STATE 'A'.
      IF sy-subrc = 0.
        APPEND |* >>> INCLUDE: { lv_inc }| TO rt_src.
        APPEND LINES OF lt_raw TO rt_src.
      ENDIF.
    ENDLOOP.

    APPEND LINES OF get_dynpro_flow( lv_prog ) TO rt_src.
  ENDMETHOD.

  METHOD get_ddic_source.
    TYPES: BEGIN OF ty_dd03,
             fieldname TYPE dd03l-fieldname,
             rollname  TYPE dd03l-rollname,
             datatype  TYPE dd03l-datatype,
             leng      TYPE dd03l-leng,
           END OF ty_dd03.
    DATA: lt_fields TYPE TABLE OF ty_dd03.

    CASE iv_type.
      WHEN 'TABL'.
        SELECT fieldname, rollname, datatype, leng
          FROM dd03l
          INTO TABLE @lt_fields
          WHERE tabname  = @iv_name
            AND as4local = 'A'
          ORDER BY position.

        APPEND |{ 'FIELD' WIDTH = 30 } \| { 'DATA ELEMENT' WIDTH = 20 } \| { 'TYPE' WIDTH = 10 } \| { 'LEN' WIDTH = 5 }| TO rt_src.
        APPEND |-----------------------------------------------------------------------| TO rt_src.
        LOOP AT lt_fields INTO DATA(ls_field).
          APPEND |{ ls_field-fieldname WIDTH = 30 } \| { ls_field-rollname WIDTH = 20 } \| { ls_field-datatype WIDTH = 10 } \| { ls_field-leng WIDTH = 5 }| TO rt_src.
        ENDLOOP.

      WHEN 'TTYP'.
        SELECT SINGLE rowtype FROM dd40l INTO @DATA(lv_rowtype) WHERE typename = @iv_name.
        APPEND |TABLE TYPE: { iv_name }| TO rt_src.
        APPEND |LINE TYPE : { lv_rowtype }| TO rt_src.

      WHEN 'DTEL'.
        SELECT SINGLE domname, datatype, leng FROM dd04l INTO @DATA(ls_dtel) WHERE rollname = @iv_name.
        APPEND |DATA ELEMENT: { iv_name }| TO rt_src.
        APPEND |DOMAIN      : { ls_dtel-domname }| TO rt_src.
        APPEND |DATATYPE    : { ls_dtel-datatype } ({ ls_dtel-leng })| TO rt_src.

      WHEN 'DOMA'.
        SELECT SINGLE datatype, leng, outputlen FROM dd01l INTO @DATA(ls_doma) WHERE domname = @iv_name.
        APPEND |DOMAIN    : { iv_name }| TO rt_src.
        APPEND |DATATYPE  : { ls_doma-datatype }| TO rt_src.
        APPEND |LENGTH    : { ls_doma-leng } (Output: { ls_doma-outputlen })| TO rt_src.
    ENDCASE.
  ENDMETHOD.

  METHOD build_header.
    APPEND |* ----------------------------------------------------| TO rt_src.
    APPEND |* OBJECT : { is_obj-obj_name }| TO rt_src.
    APPEND |* TYPE   : { is_obj-obj_type }| TO rt_src.
    APPEND |* PACKAGE: { is_obj-devclass }| TO rt_src.
    APPEND |* ----------------------------------------------------| TO rt_src.
  ENDMETHOD.

  METHOD add_log.
    APPEND VALUE #( icon     = iv_icon
                    obj_type = iv_type
                    obj_name = iv_name
                    message  = iv_msg ) TO mt_log.
  ENDMETHOD.

  METHOD display_log.
    DATA: lo_alv TYPE REF TO cl_salv_table.
    TRY.
        cl_salv_table=>factory( IMPORTING r_salv_table = lo_alv
                                CHANGING  t_table      = mt_log ).
        lo_alv->get_functions( )->set_all( abap_true ).
        lo_alv->get_columns( )->get_column( 'ICON' )->set_short_text( 'Status' ).
        lo_alv->get_display_settings( )->set_striped_pattern( abap_true ).
        lo_alv->get_display_settings( )->set_list_header( |Success: { mv_success } - Error: { mv_error }| ).
        lo_alv->display( ).
      CATCH cx_root.
        MESSAGE |Extraction complete. Success: { mv_success }, Errors: { mv_error }| TYPE 'S'.
    ENDTRY.
  ENDMETHOD.

ENDCLASS.

*----------------------------------------------------------------------*
* EXECUTION START
*----------------------------------------------------------------------*
START-OF-SELECTION.
  DATA(lo_downloader) = NEW lcl_mass_downloader( ).
  lo_downloader->execute( ).

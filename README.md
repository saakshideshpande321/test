# RAP - Print formEmail Sending
RAP - Print formEmail Sending
[RAP - Print formEmail Sending.txt](https://github.com/user-attachments/files/26487256/RAP.-.Print.formEmail.Sending.txt)
*******************************************************************************************
"GET Stream - for PDF Content

class ZCL_ZPRINT_DPC_EXT definition
  public
  inheriting from ZCL_ZPRINT_DPC
  create public .

public section.

  methods /IWBEP/IF_MGW_APPL_SRV_RUNTIME~GET_STREAM
    redefinition .
protected section.
private section.
ENDCLASS.



CLASS ZCL_ZPRINT_DPC_EXT IMPLEMENTATION.


  METHOD /iwbep/if_mgw_appl_srv_runtime~get_stream.
    DATA: schedulingrecords TYPE ztschedulinglines,
          docparams         TYPE sfpdocparams,
          formoutput        TYPE fpformoutput,
          function_name     TYPE funcname,
          stream            TYPE ty_s_media_resource,
          header            TYPE ihttpnvp,
          vendor            TYPE lifnr,
          po                TYPE ebeln.

    DATA(purchasedoc) =  VALUE #( it_key_tab[ name = 'Purchasingdocument' ]-value OPTIONAL ).
    DATA(purchasingitem) =  VALUE #( it_key_tab[ name = 'PurchasingDocumentItem' ]-value OPTIONAL ).
    DATA(scheduleline) =  VALUE #( it_key_tab[ name = 'Scheduleine' ]-value OPTIONAL ).
    DATA(month) = VALUE #( it_key_tab[ name = 'CalenderMonth' ]-value OPTIONAL ) .
    DATA(year) = VALUE #( it_key_tab[ name = 'CalenderYear' ]-value OPTIONAL ).
    vendor =  VALUE #( it_key_tab[ name = 'Supplier' ]-value  OPTIONAL  ).
    vendor = |{ vendor ALPHA = IN }|.

    SPLIT purchasedoc AT ',' INTO TABLE DATA(purchasedocs).
    SPLIT purchasingitem AT ',' INTO TABLE DATA(purchasingitems).
    SPLIT scheduleline AT ',' INTO TABLE DATA(schedulelines).

    po = VALUE #( purchasedocs[ 1 ] OPTIONAL ).
    SELECT SINGLE plant FROM i_purchasingdocumentitem INTO @DATA(plant)
                                                      WHERE purchasingdocument = @po.


    DO lines( purchasedocs ) TIMES.
      APPEND VALUE #( purchasingdocument     = VALUE #( purchasedocs[ sy-index ] OPTIONAL )
                      purchasingdocumentitem = VALUE #( purchasingitems[ sy-index ] OPTIONAL )
                      scheduleline           = VALUE #( schedulelines[ sy-index ] OPTIONAL )
                      supplier               = vendor "VALUE #( it_key_tab[ name = 'Supplier' ]-value  OPTIONAL  )
                      calendarmonth          = VALUE #( it_key_tab[ name = 'CalenderMonth' ]-value OPTIONAL )
                      calendaryear           = VALUE #( it_key_tab[ name = 'CalenderYear' ]-value OPTIONAL )
                      plant                  = plant ) TO schedulingrecords.
    ENDDO.


    yglobalutility=>jobopen( skip = abap_true ).
    " Get form name
    TRY.
        CALL FUNCTION 'FP_FUNCTION_MODULE_NAME' " DESTINATION 'NONE'
          EXPORTING
            i_name     = 'ZAF_FORECASTSCHEDULE'
          IMPORTING
            e_funcname = function_name.
      CATCH cx_fp_api_usage cx_fp_api_repository cx_fp_api_internal.
    ENDTRY.

    IF function_name IS NOT INITIAL.

      CALL FUNCTION function_name
        EXPORTING
          /1bcdwb/docparams     = docparams
          schedulingitemrecords = schedulingrecords
        IMPORTING
          /1bcdwb/formoutput    = formoutput
        EXCEPTIONS
          usage_error           = 1
          system_error          = 2
          internal_error        = 3
          OTHERS                = 4.
    ENDIF.

    yglobalutility=>jobclose(  ).


*    APPEND VALUE #( value     = formoutput-pdf
*                    mime_type = 'application/pdf' ) TO stream.

    stream-value     = formoutput-pdf.
    stream-mime_type     = 'application/pdf'.

    copy_data_to_ref(
      EXPORTING
        is_data = stream
      CHANGING
        cr_data = er_stream ).

    CONCATENATE 'Scheduling Print- ' year month  INTO DATA(pdfname).
*    lwa_header-name = 'content-disposition'.
    CONCATENATE pdfname '.pdf' INTO pdfname.
    CONCATENATE 'outline; filename=' pdfname INTO header-value.
    set_header( EXPORTING is_header = header ).


  ENDMETHOD.
ENDCLASS.

*******************************************************************************************

class:
CLASS zmailwithschedulprint DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

  PUBLIC SECTION.

    INTERFACES zif_mailwithschedulprint .

    ALIASES create_instant
      FOR zif_mailwithschedulprint~create_instantce .
    ALIASES send_email_notification
      FOR zif_mailwithschedulprint~send_email_notification .

    TYPES:
      BEGIN OF mail_body_and_subject,
        mail_body TYPE bcsy_text,
        subject   TYPE so_obj_des,
      END OF mail_body_and_subject .

    CLASS-METHODS getpdf
      IMPORTING
        !schedulingdtls_records TYPE zif_mailwithschedulprint=>schedulingdetails
      RETURNING
        VALUE(result)           TYPE fpformoutput .
    METHODS prepare_mail_body_and_subject
      IMPORTING
        !schedulingdtls_records TYPE ztschedulinglines
      RETURNING
        VALUE(result)           TYPE mail_body_and_subject .
  PROTECTED SECTION.
  PRIVATE SECTION.

    TYPES:
      calendermonth_text TYPE c LENGTH 3 .

    DATA schedulingdetails_records TYPE zif_mailwithschedulprint=>schedulingdetails .

    METHODS prepare_pdf
      IMPORTING
        !schedulingdtls_records TYPE zif_mailwithschedulprint=>schedulingdetails
      EXPORTING
        VALUE(return)           TYPE char1
      RETURNING
        VALUE(result)           TYPE fpformoutput .
    METHODS read_month_text
      IMPORTING
        !month        TYPE zi_schedulingagreementline-calendarmonth
      RETURNING
        VALUE(result) TYPE calendermonth_text .
ENDCLASS.



CLASS ZMAILWITHSCHEDULPRINT IMPLEMENTATION.


  METHOD create_instant.
    DATA(instantce) = NEW zmailwithschedulprint(  ).
    instantce->schedulingdetails_records = schedulingdetails_records.
    result = instantce.
  ENDMETHOD.


  METHOD send_email_notification.
    "Get PDF Data
*    DATA(pdfdata) =
    prepare_pdf( EXPORTING schedulingdtls_records = schedulingdetails_records
                 IMPORTING return                 = flag
                 RECEIVING result                 = DATA(pdfdata) ).

  ENDMETHOD.


  METHOD  prepare_pdf.
*    DATA:return TYPE char1.

    CALL FUNCTION 'ZFORECASTSCHEDULELINE' DESTINATION 'NONE'
      EXPORTING
        schedulingagreementlinerecords = schedulingdetails_records
      IMPORTING
        return                         = return.
  ENDMETHOD.


  METHOD prepare_mail_body_and_subject.
    CONSTANTS: linebreak TYPE c LENGTH 8 VALUE '<br><br>'.

    DATA(calender_month) = VALUE #( schedulingdtls_records[ 1 ]-calendarmonth OPTIONAL ).
    DATA(calender_year) = VALUE #( schedulingdtls_records[ 1 ]-calendaryear OPTIONAL ).


    DATA(calender_month_text) = read_month_text( calender_month ).
    result-subject = |{ TEXT-001 }{ | | }{ calender_month_text }{ '-' }{ calender_year }|.
    " Get system user name
    SELECT SINGLE  nametext FROM zi_userdetails INTO @DATA(systemusername)
                                                WHERE userid = @sy-uname.
    IF sy-subrc <> 0.
      CLEAR:systemusername.
    ENDIF.
    result-mail_body = VALUE #( BASE result-mail_body
                                ( line = TEXT-002 )
                                ( line = linebreak )
                                ( line = TEXT-003 )
                                ( line = linebreak )
                                ( line = |{ TEXT-004 }{ | | }{ calender_month_text }{ '-' }{ calender_year }| )
                                ( line = linebreak )
                                ( line = TEXT-005 )
                                ( line = linebreak )
                                ( line = systemusername ) ).

  ENDMETHOD.


  METHOD read_month_text.

    SELECT SINGLE FROM i_months_f2200 WITH PRIVILEGED ACCESS
              FIELDS calendarmonthname WHERE calendarmonth = @month
                                       AND language = @sy-langu
                                       INTO @result.
    IF sy-subrc <> 0.
      CLEAR : result.
    ENDIF.

  ENDMETHOD.


  METHOD getpdf.

    DATA:function_name     TYPE funcname,
         docparams         TYPE sfpdocparams,
         schedulingrecords TYPE ztschedulinglines.
    yglobalutility=>jobopen( skip = abap_true ).
    " Get form name
    TRY.
        CALL FUNCTION 'FP_FUNCTION_MODULE_NAME' " DESTINATION 'NONE'
          EXPORTING
            i_name     = 'ZAF_FORECASTSCHEDULE'
          IMPORTING
            e_funcname = function_name.

      CATCH cx_fp_api_usage cx_fp_api_repository cx_fp_api_internal.
    ENDTRY.

    WAIT UP TO 1 / 2 SECONDS.
    IF function_name IS NOT INITIAL.
      DATA(lv_flag) = abap_true.
      schedulingrecords = CORRESPONDING #( schedulingdtls_records ).
      CALL FUNCTION function_name
        EXPORTING
          /1bcdwb/docparams     = docparams
          schedulingitemrecords = schedulingrecords
        IMPORTING
          /1bcdwb/formoutput    = result
        EXCEPTIONS
          usage_error           = 1
          system_error          = 2
          internal_error        = 3
          OTHERS                = 4.
      IF sy-subrc <> 0.
* Implement suitable error handling here
      ENDIF.
      yglobalutility=>jobclose(  ).
    ENDIF.
  ENDMETHOD.


  METHOD zif_mailwithschedulprint~final.
    DATA: schedulingrecords TYPE ztschedulinglines.
    CALL FUNCTION 'Z_SCHEDULING_SENDMAIL' DESTINATION 'NONE'
      EXPORTING
        schedulingdetails_records = schedulingdetails_records
      IMPORTING
        flag                      = flag.
  ENDMETHOD.
ENDCLASS.



************************************************************************************************
Behaviour
CLASS lhc_zi_schedulingagreementline DEFINITION INHERITING FROM cl_abap_behavior_handler.
  PRIVATE SECTION.

    TYPES:BEGIN OF schedulinglines,
            purchasingdocument     TYPE  char10,
            purchasingdocumentitem TYPE  n LENGTH 5,
            scheduleline           TYPE  numc4,
            supplier               TYPE  char10,
            calendarmonth          TYPE  numc2,
            calendaryear           TYPE  numc4,
            plant                  TYPE  char4,
            material               type  char40,
            materialName           type  char40,
          END OF schedulinglines.
    DATA:schdlgaggrementlinerecords TYPE TABLE OF schedulinglines.


    METHODS get_instance_authorizations FOR INSTANCE AUTHORIZATION
      IMPORTING keys REQUEST requested_authorizations FOR zi_schedulingagreementline_m RESULT result.

    METHODS create FOR MODIFY
      IMPORTING entities FOR CREATE zi_schedulingagreementline_m.

    METHODS update FOR MODIFY
      IMPORTING entities FOR UPDATE zi_schedulingagreementline_m.

    METHODS delete FOR MODIFY
      IMPORTING keys FOR DELETE zi_schedulingagreementline_m.

    METHODS read FOR READ
      IMPORTING keys FOR READ zi_schedulingagreementline_m RESULT result.

    METHODS lock FOR LOCK
      IMPORTING keys FOR LOCK zi_schedulingagreementline_m.

    METHODS downloadform FOR MODIFY
      IMPORTING keys FOR ACTION zi_schedulingagreementline_m~downloadform.

    METHODS sendemail FOR MODIFY
      IMPORTING keys FOR ACTION zi_schedulingagreementline_m~sendemail.

ENDCLASS.

CLASS lhc_zi_schedulingagreementline IMPLEMENTATION.

  METHOD get_instance_authorizations.
  ENDMETHOD.

  METHOD create.
  ENDMETHOD.

  METHOD update.
  ENDMETHOD.

  METHOD delete.
  ENDMETHOD.

  METHOD read.
    SELECT FROM zi_schedulingagreementline AS _schedulingagreementline
              INNER JOIN @keys AS _keys
              ON _schedulingagreementline~purchasingdocument = _keys~purchasingdocument
              AND _schedulingagreementline~purchasingdocumentitem = _keys~purchasingdocumentitem
              AND _schedulingagreementline~scheduleline = _keys~scheduleline
              FIELDS _schedulingagreementline~*
              INTO TABLE @DATA(schedulingagreementlinerecords).
    IF sy-subrc = 0.
      result = CORRESPONDING  #( schedulingagreementlinerecords ).
    ENDIF.
  ENDMETHOD.

  METHOD lock.
  ENDMETHOD.

  METHOD downloadform.



  ENDMETHOD.

  METHOD sendemail.

    SELECT FROM zi_schedulingagreementline AS _schedulingagreementline
                  INNER JOIN @keys AS _keys
                  ON _schedulingagreementline~purchasingdocument = _keys~purchasingdocument
                  AND _schedulingagreementline~purchasingdocumentitem = _keys~purchasingdocumentitem
                  AND _schedulingagreementline~scheduleline = _keys~scheduleline
                  FIELDS _schedulingagreementline~*
                  INTO TABLE @DATA(shdllinerecords).
    IF sy-subrc = 0 .
      schdlgaggrementlinerecords = CORRESPONDING #( shdllinerecords ).
    ENDIF.
    DATA(result) = zmailwithschedulprint=>create_instant( schdlgaggrementlinerecords ).
    result->final( RECEIVING flag = DATA(retflag) ).

    IF retflag IS NOT INITIAL.
      INSERT VALUE #(
                   %action = VALUE #(  sendemail = if_abap_behv=>mk-on )
                   %msg    = new_message_with_text(
                                      severity  = if_abap_behv_message=>severity-success
                                      text      = 'Mail Sent Successfully' )
      ) INTO TABLE reported-zi_schedulingagreementline_m.
    ENDIF.

*INSERT VALUE #( %action-timesheetrecordssendtoemployee = if_abap_behv=>mk-on ) INTO TABLE failed-timesheetpreapprovalcheck.

  ENDMETHOD.

ENDCLASS.

CLASS lsc_zi_schedulingagreementline DEFINITION INHERITING FROM cl_abap_behavior_saver.
  PROTECTED SECTION.

    METHODS finalize REDEFINITION.

    METHODS check_before_save REDEFINITION.

    METHODS save REDEFINITION.

    METHODS cleanup REDEFINITION.

    METHODS cleanup_finalize REDEFINITION.

ENDCLASS.

CLASS lsc_zi_schedulingagreementline IMPLEMENTATION.

  METHOD finalize.
  ENDMETHOD.

  METHOD check_before_save.
  ENDMETHOD.

  METHOD save.
  ENDMETHOD.

  METHOD cleanup.
  ENDMETHOD.

  METHOD cleanup_finalize.
  ENDMETHOD.

ENDCLASS.

*********************************************************************************************

FUNCTION zforecastscheduleline
  IMPORTING
    VALUE(schedulingagreementlinerecords) TYPE ztschedulinglines OPTIONAL
  EXPORTING
    VALUE(return) TYPE char1.




  DATA:ccmail_ids TYPE yglobalutility=>tt_email_ids,
       tomail_ids TYPE yglobalutility=>tt_email_ids.

  DATA(suppliernumber) = VALUE #( schedulingagreementlinerecords[ 1 ]-supplier OPTIONAL ).
  DATA(purchaseorder) = VALUE #( schedulingagreementlinerecords[ 1 ]-purchasingdocument OPTIONAL ).

  "Get Current user Email ID
  SELECT address~emailaddress FROM i_user  WITH PRIVILEGED ACCESS AS user
                              INNER JOIN i_addressemailaddress  WITH PRIVILEGED ACCESS
                              AS address ON address~addressid = user~addressid
                              AND address~person = user~addresspersonid
         WHERE user~userid = @sy-uname INTO TABLE @ccmail_ids.
  IF sy-subrc = 0.
  ENDIF.
  "Get Vendor Mail Id(to mail id)
  SELECT  address~emailaddress FROM i_supplier AS supplier
                              INNER JOIN i_addressemailaddress  WITH PRIVILEGED ACCESS
                              AS address ON address~addressid = supplier~addressid
         WHERE supplier~supplier = @suppliernumber into TABLE @tomail_ids.
  IF sy-subrc = 0.
  ENDIF.

  "Get CC Mail Id's
  SELECT ccmail~ccemail  FROM zi_mmmailmaintain AS ccmail
                        INNER JOIN i_purchasingdocumentitem AS  poitem
                                                            ON  poitem~plant = ccmail~plant
                                                            AND poitem~purchasingdocument = @purchaseorder
                        APPENDING TABLE @ccmail_ids."DATA(ccmailids).
  IF sy-subrc = 0.
  ENDIF.

  DATA(formoutput) = zmailwithschedulprint=>getpdf( schedulingagreementlinerecords ).

*  yglobalutility=>jobclose(  ).
  DATA(lo_mailbody) = NEW zmailwithschedulprint( ).
  "Get mail body and subject
  DATA(mailbody_and_sub) = lo_mailbody->prepare_mail_body_and_subject( EXPORTING schedulingdtls_records = schedulingagreementlinerecords ).

  TRY.
      yglobalutility=>sendemail_with_attchment(
        EXPORTING
          mailinfo     = VALUE #( message_body = mailbody_and_sub-mail_body
                                  pdfname      = TEXT-007
                                  sender_mail  = 'SAP.NOTIFICATION@RSBGLOBAL.COM' "systememailid
                                  subject      = mailbody_and_sub-subject
                                  formoutput   = formoutput )
          toemail_ids  = tomail_ids
          ccemail_ids  = ccmail_ids
        IMPORTING
          success_flag = DATA(flag)
          message      = DATA(message) ).
    CATCH cx_root.
  ENDTRY.
  IF flag =  /isdfps/cl_const_abc_123=>gc_s.
    return = abap_true.
  ENDIF.

ENDFUNCTION.

***********************************************************************************************
FUNCTION z_scheduling_sendmail
  IMPORTING
    VALUE(schedulingdetails_records) TYPE ztschedulinglines
  EXPORTING
    VALUE(flag) TYPE char1.




  DATA(result) = zmailwithschedulprint=>create_instant( CORRESPONDING #( schedulingdetails_records ) ).
  result->send_email_notification( RECEIVING flag = flag ).




ENDFUNCTION.

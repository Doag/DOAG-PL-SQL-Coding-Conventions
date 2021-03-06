# Code Practises

## Creating a package
All procedures and functions should reside in a package. Here is how a package is typically created: 
```
CREATE OR REPLACE PACKAGE str_pkg
IS
   /* Description 
   *  Author
   *  Parameter
   */
   PROCEDURE between
   ( pi_str_in   IN  VARCHAR2
   , pi_start_in IN  PLS_INTEGER
   , pi_end_in   IN  PLS_INTEGER
   , po_str_out  OUT VARCHAR2
   );
END str_pkg;
```
Next, I add only the following code to the package body:
```
CREATE OR REPLACE PACKAGE BODY str_pkg
IS
   PROCEDURE between
   ( pi_str_in   IN  VARCHAR2
   , pi_start_in IN  PLS_INTEGER
   , pi_end_in   IN  PLS_INTEGER
   , po_str_out  OUT VARCHAR2
   )
   IS
   BEGIN
     # Start your work here and comment it
     null;
   
   EXCEPTION
     WHEN OTHERS
     THEN
       # use your favorite logger tool here before raising an exception for system errors
       raise;
   END between;
END str_pkg;
```

## Avoid hardcoding of system variables, IDs and VARCHAR2 length
- System variables that depend on your environment should never be hardcoded in PL/SQL
- Columns that end with _ID usually contain technical IDs. Don't hardcode these in your PL/SQL, but use a procedure to get them
- Make use of %type and %rowtype. (avoid hardcoding the length of a varchar2 column)

## When writing dynamic queries, making use of "q" will lead to much more maintainable code
```
CREATE OR REPLACE PACKAGE BODY query_builder_pkg
IS

PROCEDURE dynamic_query
IS

  v_sql  VARCHAR2(1024);
  v_cnt  PLS_INTEGER;

BEGIN
  v_sql := q'[SELECT COUNT(*) FROM user_objects WHERE object_type = 'TABLE']';
  EXECUTE IMMEDIATE v_sql INTO v_cnt;
  DBMS_OUTPUT.PUT_LINE(TO_CHAR(v_cnt) || ' tables in USER_OBJECTS.');

EXCEPTION
  WHEN OTHERS
  THEN
    # use your favorite logger tool here before raising an exception for system errors
    raise;
END dynamic_query;

END query_builder_pkg;
```

## Keep in mind: bulk operations are a LOT quicker then row-by-row processing
```
declare
  cursor get_activity
  is
    select *
    from   activity
    where  id between 1000000 and 1999999; -- 1 million records

  type activity_array is table of get_activity%rowtype;

  la_activity_rec activity_array:= activity_array();
  ts1             timestamp;

begin
  ts1:= systimestamp;
  dbms_output.put_Line ('started at '||ts1); open get_activity;
 
  loop
    fetch get_activity bulk collect into la_activity_rec limit 500;
    forall i in indices of la_activity_rec
      insert into activity2 values la_activity_rec(i);
    exit when get_activity%notfound;
  end loop;

  close get_activity;

  dbms_output.put_Line ('fin at '||systimestamp);
  commit;
end;
```

## If you need error handling during the bulk operation, here is an example:
```
loop
  fetch get_activity bulk collect into la_activity_rec limit 500;

  begin
    forall i in indices of la_activity_rec save exceptions insert into activity3 values la_activity_rec(i);

  exception
    when others
    then
      if sqlcode=-24381
      then
        for errno in 1..sql%bulk_exceptions.count
        loop
          dbms_output.put_line('Error in record ‘ ||sql%bulk_exceptions(errno).error_index||
        end loop;
      else
        raise;
      end if;
  end;

  exit when get_activity%notfound;
end loop;
close get_activity;
```

## Getting only a few rows using SQL
```
select   prod_id
,        prod_list_price
from     sh.products
order by prod_list_price
fetch first 5 rows only
;
```

## When using CASE (and you should use it instead of IF!), make sure that there is a ELSE clause
```
FUNCTION fruit_translator
( pi_letter_in IN VARCHAR2
) RETURN VARCHAR2
IS
  retval VARCHAR2(100);
BEGIN
  CASE
    WHEN pi_letter_in = 'A' THEN retval := 'Apple';
    WHEN pi_letter_in = 'B' THEN retval := 'Banana';
    ELSE retval:= '';
  END CASE;

  RETURN retval;
END fruit_translator;
```

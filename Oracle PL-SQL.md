#### Get Oracle version information

```sql
SELECT * FROM v$version;
```

#### See tables you have been granted some level of access to, whether directly or through a role

```sql
SELECT DISTINCT OWNER, OBJECT_NAME, OBJECT_TYPE
FROM   ALL_OBJECTS
WHERE  OBJECT_TYPE = 'TABLE'
AND    OWNER = '[some other schema]'
```

#### See database objects

```sql
SELECT * FROM ALL_OBJECTS                   --All 
SELECT * FROM ALL_TABLES                    --Tables
SELECT * FROM ALL_TAB_COLUMNS               --Columns
SELECT * FROM ALL_VIEWS                     --Views
SELECT * FROM ALL_PROCEDURES                --Stored Procedures
```

#### Query Data Dictionary for your own tables
> Rights to one's own tables cannot be revoked as of Oracle 10g

*- Method 1*

```sql
SELECT DISTINCT OBJECT_NAME, OBJECT_TYPE
FROM   USER_OBJECTS
WHERE  OBJECT_TYPE = 'TABLE'
```

*- Method 2*

```sql
SELECT TABLE_NAME 
FROM   ALL_TABLES 
WHERE  OWNER = 'YOURSCHEMA';
```

*- Method 3*

```sql
SELECT TABLE_NAME FROM USER_TABLES
```

#### Enable / disable `dbms_output`

```sql
dbms_output.enable();
dbms_output.disable();
```

#### Anonymous block

```sql
DECLARE
	x INT := 2;
BEGIN
	dbms_output.put_line('The value of x is ' || x);
END;
```

#### Derived SQL
```sql
SELECT a.today, a.custdate, 
       fn_date_difference(a.today, a.custdate, 'DAYS') days_diff
FROM   (
        SELECT sysdate today, 
               CAST('12-May-1977' AS DATE) custdate
        FROM   dual
       ) a;
```

#### Common Table Expression (CTE) - Temporary tables not recommended; table variables do not exist

```sql
WITH
  cte1 AS (SELECT * FROM deep_test WHERE Id > 1),
  cte2 AS (SELECT * FROM deep_test WHERE Id > 2)
  
SELECT * FROM cte1
UNION
SELECT * FROM cte2;	  
```

#### `sys_refcursor` - Returns resultset of a query without using a cursor (like T-SQL)

```sql
DECLARE
    rc sys_refcursor;
BEGIN
    OPEN rc FOR SELECT * FROM PS_DEPT_TBL;
    dbms_sql.return_result(rc);
END;
```
 
 #### Basic loop (This is the PL/SQL Until loop.)

```sql
DECLARE
    iCnt INT := 0;    
BEGIN
    LOOP
        dbms_output.put_line('Counter = ' || icnt);
        iCnt := iCnt + 1;
        EXIT WHEN iCnt >= 10;
    END LOOP;
END;
```

#### FOR loop

```sql
DECLARE
    iCnt INT := 0;  
    iMax INT := 10;
BEGIN
    FOR iCnt in 1..iMax
        LOOP
            dbms_output.put_line('Counter = ' || icnt);
        END LOOP;
END;
```

#### WHILE Loop

```sql
DECLARE
    iCnt INT := 0;    
BEGIN
    WHILE iCnt < 10
        LOOP
            dbms_output.put_line('Counter = ' || icnt);
            iCnt := iCnt + 1;
        END LOOP;
END;
```

#### Basic loop (using a cursor)

```sql
DECLARE
    cur_id deep_test.id%type;
    cur_descr deep_test.descr%type;
    CURSOR myCursor IS
        SELECT * FROM deep_test ORDER BY id;
BEGIN
    OPEN myCursor;
    
    LOOP
        FETCH myCursor INTO cur_id, cur_descr;
        EXIT WHEN myCursor%NOTFOUND;
        dbms_output.put_line(cur_Id || '  ' || cur_descr);
    END LOOP;
    
    CLOSE myCursor;
END;  
```

#### FOR loop (using a cursor)

```sql
DECLARE
    cur_id deep_test.id%type;
    cur_descr deep_test.descr%type;
    CURSOR myCursor IS
        SELECT * FROM deep_test ORDER BY id;
    iCnt INT;
    iMax INT;
BEGIN
    OPEN myCursor;
    
    SELECT   COUNT(*) 
    INTO     iMax 
    FROM     deep_test 
    ORDER BY id;
    
    FOR iCnt in 1..iMax
        LOOP
            FETCH myCursor INTO cur_id, cur_descr;
            dbms_output.put_line('Counter = ' || icnt ||'  ' || cur_Id || '  ' || cur_descr);
        END LOOP;
    
    CLOSE myCursor;
END;
```

#### WHILE loop (using a cursor)

```sql
DECLARE
    cur_id deep_test.id%type;
    cur_descr deep_test.descr%type;
    CURSOR myCursor IS
        SELECT * FROM deep_test ORDER BY id;
    iCnt INT;
    iMax INT;
BEGIN
    OPEN myCursor;
    
    SELECT   COUNT(*) 
    INTO     iMax 
    FROM     deep_test 
    ORDER BY id;
    
    WHILE iCnt <= iMax
        LOOP
            FETCH myCursor INTO cur_id, cur_descr;
            dbms_output.put_line('Counter = ' || icnt ||'  ' || cur_Id || '  ' || cur_descr);
        END LOOP;
    
    CLOSE myCursor;
END;
```

#### Cursor-FOR-Loop (automatically opens/closes the Cursor and exits after last record)

```sql
DECLARE
    CURSOR myCursor IS
        SELECT * FROM deep_test ORDER BY id;
    iTot INT;    
BEGIN
    iTot := 0;
    
    For myRec IN myCursor
        LOOP
            iTot := iTot + myRec.id;
            dbms_output.put_line('Total = ' || iTot);
        END LOOP;
END; 
```

#### `IF` and `CASE`

```sql
SELECT id, 
       CASE 
          WHEN descr = 'one' Then 'Uno'
          WHEN descr = 'two' Then 'Dos'
          ELSE 'Tres o Mas'
       END "DESCRIPTION"
FROM   deep_test

DECLARE 
    x INT := 2;   
BEGIN
    IF x > 0
        THEN
            dbms_output.put_line('The value of x is positive ' || x);
        ELSE
            dbms_output.put_line('The value of x is negative ' || x);
    END IF;
    
    CASE 
        WHEN x > 0 THEN dbms_output.put_line('The value of x is positive ' || x);
        WHEN x < 0 THEN dbms_output.put_line('The value of x is negative ' || x);
    END CASE;
END;
```

#### Read from a file

```sql
CREATE OR REPLACE DIRECTORY dirUser AS '/home/oracle'; 
GRANT READ ON DIRECTORY dirUser TO PUBLIC;

DECLARE
    varTextFileLine VARCHAR(100);
    filInputTextFile UTL_FILE.FILE_TYPE;
BEGIN
    
    filInputTextFile := UTL_FILE.FOPEN('dirUser', 'temp.txt', 'R');
    --R = read | W = write | A = append 
    
    LOOP
        BEGIN
            UTL_FILE.GET_LINE(filInputTextFile, varTextFileLine);
            dbms_output.put_line(varTextFileLine);
            EXCEPTION WHEN NO_DATA_FOUND THEN EXIT;
        END;
    END LOOP;

    UTL_FILE.FCLOSE(filInputTextFile);
    
END;
```

#### Write to a file

```sql
CREATE OR REPLACE DIRECTORY dirUser AS 'D:\test'; 
GRANT READ ON DIRECTORY dirUser TO PUBLIC;

DECLARE
    filOutputTextFile UTL_FILE.FILE_TYPE;
BEGIN
    
    filOutputTextFile := UTL_FILE.FOPEN('dirUser', 'temp.txt', 'W');
    --R = read | W = write | A = append 
    
    UTL_FILE.PUT_LINE(filOutputTextFile, 'Hello World!');

    UTL_FILE.FCLOSE(filOutputTextFile);
    
END;
```

#### `EXTRACT` for getting parts of a date

```sql
SELECT birthdate, 
       CAST(EXTRACT(YEAR FROM birthdate) AS VARCHAR(4)) "Year"      --2015
       CAST(EXTRACT(MONTH FROM birthdate) AS VARCHAR(4)) "Month"    --7
       CAST(EXTRACT(DAY FROM birthdate) AS VARCHAR(4)) "Date"       --29
       CAST(EXTRACT(HOUR FROM birthdate) AS VARCHAR(4)) "Hour"      --12
       CAST(EXTRACT(MINUTE FROM birthdate) AS VARCHAR(4)) "Minute"  --30
       CAST(EXTRACT(SECOND FROM birthdate) AS VARCHAR(4)) "Second"  --45
FROM   family_birthdays     
```

#### `LEAD` and `LAG`

```sql
SELECT last_name, hire_date, 
       LEAD(hire_date, 1) OVER (ORDER BY hire_date) "NextHired" 
FROM   employees 
WHERE  department_id = 30;

SELECT last_name, hire_date, 
       LAG(hire_date, 1) OVER (ORDER BY hire_date) "PreviousHired" 
FROM   employees 
WHERE  department_id = 30;
``

#### `ROWNUMBER OVER [PARTITION]`

```sql
SELECT department_id, last_name, employee_id, 
       ROW_NUMBER() OVER (ORDER BY employee_id) AS emp_id
FROM   employees;

SELECT department_id, last_name, employee_id, 
       ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY employee_id) AS emp_id
FROM   employees;
```

#### Place values from rows into columns

```sql
SELECT     SETID, CNTRCT_ID, VERSION_NBR,
           LTRIM(MAX(SYS_CONNECT_BY_PATH(OPRID, ', ')) KEEP (DENSE_RANK LAST ORDER BY curr), ',') AS OPRID
FROM       (
            SELECT   SETID, CNTRCT_ID, VERSION_NBR, OPRID,
                     ROW_NUMBER() OVER (PARTITION BY SETID, CNTRCT_ID, VERSION_NBR ORDER BY OPRID) AS curr,
                     ROW_NUMBER() OVER (PARTITION BY SETID, CNTRCT_ID, VERSION_NBR ORDER BY OPRID) -1 AS prev
            FROM     PS_CNTRCT_NTFYUSER
            GROUP BY SETID, CNTRCT_ID, VERSION_NBR, OPRID
           )
GROUP BY   SETID, CNTRCT_ID, VERSION_NBR
CONNECT BY prev = PRIOR curr 
AND        CNTRCT_ID = PRIOR CNTRCT_ID
START WITH curr = 1     
```

#### Create a table from another table

```sql
CREATE TABLE deep_new AS
    (SELECT * FROM deep_test);
```    
    
#### Query against date / time / timestamp fields

> [https://www.techonthenet.com/oracle/functions/to_date.php](https://www.techonthenet.com/oracle/functions/to_date.php) <br/>
> [https://www.techonthenet.com/oracle/functions/to_char.php](https://www.techonthenet.com/oracle/functions/to_char.php)

```sql
SELECT *
FROM   family_birthdays
WHERE  birthdate > TO_DATE('2015-01-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS')   

SELECT TO_CHAR(birthdate, 'MM/DD/YYYY') AS Birthdate
FROM   family_birthdays
WHERE  birthdate > TO_DATE('2015-01-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS') 
```
	
#### String functions
```sql
SELECT LENGTH('abc') FROM dual;
> 3

SELECT UPPER('abc') FROM dual;
> ABC

SELECT LOWER('ABC') FROM dual;
> abc

SELECT REVERSE('abc') FROM dual;
> cba

SELECT REPLACE('xyz', 'y', 's') FROM DUAL;
> xsz

SELECT SUBSTR('xyz', 1, 2) FROM DUAL;
> xy

SELECT SUBSTR('xyz', -2, 2) FROM DUAL;
> yz

SELECT INSTR('abcdefgabcdefg', 'g') FROM dual;
> 7

SELECT INSTR('abcdefgabcdefg', 'g', 1, 2) FROM dual;  --(string, search string, position, occurrence)
> 14

SELECT CONCAT('a', 'b') FROM dual;
> ab

SELECT LTRIM('   abc   ') FROM dual;
> abc___   

SELECT RTRIM('   abc   ') FROM dual;
>    abc

SELECT TRIM('   abc   ') FROM dual;
> abc

SELECT LPAD('abc', 5, '*') FROM dual;
> **abc

SELECT RPAD('abc', 10, '*') FROM dual;
> abc*******
```

#### Note:

`VARCHAR` is reserved by Oracle to support distinction between `NULL` and empty string in future, as ANSI standard prescribes.
`VARCHAR2` does not distinguish between a `NULL` and empty string, and never will.

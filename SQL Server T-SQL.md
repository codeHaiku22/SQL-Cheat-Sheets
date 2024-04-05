
#### Get SQL Server version information

```sql
SELECT CASE 
	       WHEN CONVERT(VARCHAR(128), SERVERPROPERTY ('productversion')) LIKE '8%'    THEN 'SQL2000'
           WHEN CONVERT(VARCHAR(128), SERVERPROPERTY ('productversion')) LIKE '9%'    THEN 'SQL2005'
           WHEN CONVERT(VARCHAR(128), SERVERPROPERTY ('productversion')) LIKE '10.0%' THEN 'SQL2008'
           WHEN CONVERT(VARCHAR(128), SERVERPROPERTY ('productversion')) LIKE '10.5%' THEN 'SQL2008 R2'
           WHEN CONVERT(VARCHAR(128), SERVERPROPERTY ('productversion')) LIKE '11%'   THEN 'SQL2012'
           WHEN CONVERT(VARCHAR(128), SERVERPROPERTY ('productversion')) LIKE '12%'   THEN 'SQL2014'
           WHEN CONVERT(VARCHAR(128), SERVERPROPERTY ('productversion')) LIKE '13%'   THEN 'SQL2016'     
           WHEN CONVERT(VARCHAR(128), SERVERPROPERTY ('productversion')) LIKE '14%'   THEN 'SQL2017' 
           WHEN CONVERT(VARCHAR(128), SERVERPROPERTY ('productversion')) LIKE '15%'   THEN 'SQL2019' 
           ELSE 'Unknown'
       END 'MajorVersion',
       SERVERPROPERTY('ProductLevel') 'ProductLevel',
       SERVERPROPERTY('Edition') 'Edition',
       SERVERPROPERTY('ProductVersion') 'ProductVersion'
```

#### See database objects

```sql
SELECT * FROM SYS.ALL_OBJECTS                   --All 
SELECT * FROM SYS.TABLES                        --Tables
SELECT * FROM SYS.VIEWS                         --Views
SELECT * FROM SYS.PROCEDURES                    --Stored Procedures
SELECT * FROM SYS.DATABASES                     --Databases
SELECT * FROM SYS.MASTER_FILES                  --Database Files

SELECT * FROM INFORMATION_SCHEMA.TABLES         --Tables and Views
SELECT * FROM INFORMATION_SCHEMA.VIEWS          --Views
SELECT * FROM INFORMATION_SCHEMA.ROUTINES       --Functions and Stored Procedures
```

#### See all database sizes

```sql
SELECT   d.[name], 
         ROUND(SUM(CAST(mf.size AS bigint)) * 8 / 1024, 0) 'Size_MB', 
         (SUM(CAST(mf.size AS bigint)) * 8 / 1024) / 1024 'Size_GB'
FROM     sys.master_files mf
JOIN     sys.databases d 
ON       d.database_id = mf.database_id
--WHERE  d.database_id > 4 -- Skip system databases
GROUP BY d.[name]
ORDER BY d.[name]
```

#### See all .mdf and .ldf file sizes

```sql
SELECT d.[name] 'Database_Name', 
       CASE d.[compatibility_level]
           WHEN 80  THEN CAST(d.[compatibility_level] AS VARCHAR(3)) + ' - SQL Server 2000'
           WHEN 90  THEN CAST(d.[compatibility_level] AS VARCHAR(3)) + ' - SQL Server 2005'
           WHEN 100 THEN CAST(d.[compatibility_level] AS VARCHAR(3)) + ' - SQL Server 2008'
           WHEN 110 THEN CAST(d.[compatibility_level] AS VARCHAR(3)) + ' - SQL Server 2012'
           WHEN 120 THEN CAST(d.[compatibility_level] AS VARCHAR(3)) + ' - SQL Server 2014'
           WHEN 130 THEN CAST(d.[compatibility_level] AS VARCHAR(3)) + ' - SQL Server 2016'
           WHEN 140 THEN CAST(d.[compatibility_level] AS VARCHAR(3)) + ' - SQL Server 2017'
           WHEN 150 THEN CAST(d.[compatibility_level] AS VARCHAR(3)) + ' - SQL Server 2019'
           ELSE 'Unknown'
       END 'compatibility_level',
       d.create_date, d.state_desc, d.recovery_model_desc,
       mf.[type_desc], mf.[name] 'File_Name', 
       mf.physical_name,   
       ROUND(CAST(mf.size AS BIGINT) * 8 / 1024, 0) 'Size_MB',
       (CAST(mf.size AS BIGINT) * 8 / 1024) / 1024 'Size_GB',
       ROUND(CAST(mf.max_size AS BIGINT) * 8 / 1024, 0) 'Max_Size_MB',
       (CAST(mf.max_size AS BIGINT) * 8 / 1024) / 1024 'Max_Size_GB',
       CAST(ROUND(CAST(mf.growth AS BIGINT) * 8 / 1024, 0) AS VARCHAR(15)) + ' MB' 'Grow_By'
FROM   sys.master_files mf
JOIN   sys.databases d
ON     mf.database_id = d.database_id
```

#### See table and index size / disk usage

```sql
WITH cte1 AS (
    SELECT t.[name] 'TableName',
           SUM (s.used_page_count) 'Used_Pages_Count',
           SUM (
                CASE
                    WHEN (i.index_id < 2) THEN (in_row_data_page_count + lob_used_page_count + row_overflow_used_page_count)
                    ELSE lob_used_page_count + row_overflow_used_page_count
                END
               ) 'Pages'
    FROM   sys.dm_db_partition_stats s 
    JOIN   sys.tables t 
    ON     s.[object_id] = t.[object_id]
    JOIN   sys.indexes i 
    ON     i.[object_id] = t.[object_id] 
    AND    s.index_id = i.index_id
    GROUP BY t.name
    ),

cte2 AS (
    SELECT cte1.TableName, 
           CAST((cte1.pages * 8) AS DECIMAL(10,0)) 'TableSizeInKB', 
           CAST(((CASE WHEN cte1.used_pages_count > cte1.pages THEN cte1.used_pages_count - cte1.pages ELSE 0 END) * 8) AS DECIMAL(10,0)) 'IndexSizeInKB'
    FROM   cte1
    )

SELECT   TableName,TableSizeInKB,IndexSizeInKB,
         CASE 
             WHEN (TableSizeInKB + IndexSizeInKB) > (1024 * 1024) THEN CAST(CAST((TableSizeInKB + IndexSizeInKB) / 1024 / 1024 AS DECIMAL (6,2)) AS VARCHAR) + 'GB'
             WHEN (TableSizeInKB + IndexSizeInKB) > 1024 THEN CAST(CAST((TableSizeInKB + IndexSizeInKB) / 1024 AS DECIMAL (6,2)) AS VARCHAR) + 'MB'
             ELSE CAST((TableSizeInKB + IndexSizeInKB) AS VARCHAR) + 'KB' 
         END 'TableSizeIn+IndexSizeIn'
FROM     cte2
ORDER BY 2 DESC
```

#### See table create date, modify date, row count

```sql
SELECT   t.[object_id], t.[name], t.create_date, t.modify_date, r.row_count 
FROM     sys.tables t
JOIN     (
          SELECT   t.[object_id], 
                   SUM(p.[rows]) 'row_count'
          FROM     sys.tables t
          JOIN     sys.partitions p
          ON       t.[object_id] = p.[object_id]
          JOIN     sys.indexes i
          ON       p.index_id = i.index_id
          AND      p.[object_id] = i.[object_id]
          WHERE    i.index_id < 2
          GROUP BY t.[object_id], i.[object_id]
         ) r
ON       t.object_id = r.object_id
ORDER BY 2
```

#### See table activity from all users and system

```sql
SELECT OBJECT_NAME([object_id]) 'object_name', *
FROM   sys.dm_db_index_usage_stats
WHERE  database_id = DB_ID('mtu')
```

#### See table information

```
EXEC sp_help arrest

EXEC sp_columns arrest

SELECT   * 
FROM     INFORMATION_SCHEMA.COLUMNS
WHERE    TABLE_NAME = <name_of_table>
ORDER BY ORDINAL_POSITION
```

#### Derived SQL

```sql
SELECT a.today, a.custdate,
       DATEDIFF(DAY, a.custdate, a.today) 'days_diff'
FROM   (
		SELECT CURRENT_TIMESTAMP 'today',
			   CAST('12-May-1977' AS DATE) 'custdate'
	   ) a
```

#### Temporary tables

```sql
CREATE TABLE #temp0 (
	id INT,
	descr VARCHAR(30)
    )
	
INSERT INTO #temp0
	SELECT 1, 'one'
	UNION
	SELECT 2, 'two'
	UNION
	SELECT 3, 'three'
    
SELECT * FROM #temp0

DROP TABLE #temp0   

SELECT *
INTO   #temp1
FROM   realTable 

DROP TABLE #temp1
```

#### Table variable

```sql
DECLARE @temp TABLE (
	id INT,
	descr VARCHAR(30)
)
	
INSERT INTO @temp
	SELECT 1, 'one'
	UNION
	SELECT 2, 'two'
	UNION
	SELECT 3, 'three'

SELECT * FROM @temp
```

#### Common Table Expressions (CTE)

```sql
WITH
  cte1 AS (SELECT * FROM deep_test WHERE Id > 1),
  cte2 AS (SELECT * FROM deep_test WHERE Id > 2)
  
SELECT * FROM cte1
UNION
SELECT * FROM cte2
```

#### WHILE loop (only loop in T-SQL)

```sql
DECLARE @intCnt INTEGER = 0

WHILE @intCnt < 5
    BEGIN
        SET @intCnt = @intCnt + 1
        SELECT @intCnt        
    END
```

#### WHILE loop with `BREAK`

```sql
DECLARE @intCnt INTEGER = 0

WHILE @intCnt < 5
    BEGIN
        SET @intCnt = @intCnt + 1
        IF @intCnt = 3
            BREAK
        SELECT @intCnt
    END
```

#### WHILE loop with `CONTINUE`

```sql
DECLARE @intCnt INTEGER = 0

WHILE @intCnt < 5
    BEGIN
        SET @intCnt = @intCnt + 1
        IF @intCnt = 3
            CONTINUE
        SELECT @intCnt
    END    
 ```

#### Cursor

```sql
DECLARE @intSomeInteger INTEGER

SET @curJustAnotherCursor = CURSOR SCROLL KEYSET FOR
    SELECT *
    FROM   someTable
    WHERE  someColumn > 0

OPEN @curJustAnotherCursor
    FETCH NEXT FROM @curJustAnotherCursor INTO @intSomeInteger
    WHILE @@FETCH_STATUS = 0
        BEGIN
            EXEC sp_SomeStoredProcedure CURRENT_TIMESTAMP, @intSomeInteger
            FETCH NEXT FROM @curJustAnotherCursor INTO @intSomeInteger
        END

CLOSE @curJustAnotherCursor                
    
DEALLOCATE @curJustAnotherCursor 
```
 
#### `IF` statement

```sql
DECLARE @x INT = 2  

IF @x > 0
    BEGIN
        SELECT(CONCAT('The value of x is positive ', @x))
        SET @x = @x * -1
    END
ELSE
    BEGIN
        SELECT(CONCAT('The value of x is negative ', @x))
        SET @x = @x * -1
    END
```

#### `IF NOT EXISTS`

```sql
IF NOT EXISTS(SELECT * FROM SYS.ALL_OBJECTS WHERE NAME = 'ThisTable' AND TYPE = 'U')
    BEGIN
        SELECT * 
        INTO   ThisTable 
        FROM   SomeTable
    END
ELSE
    BEGIN
        SET IDENTITY_INSERT ThisTable ON        --Allows explicit values to be inserted into the identity column of a table.
        INSERT ThisTable(Id, Note, Code, CreatedOnDate)
            SELECT * 
            FROM   SomeTable
            WHERE  ABS(DATEDIFF(DAY, CreatedOnDate, CURRENT_TIMESTAMP)) > 1
        SET IDENTITY_INSERT ThisTable OFF
    END    
```

#### `CASE` statement

```sql
DECLARE @desc VARCHAR(5) = 'one'

SELECT CASE 
           WHEN @desc = 'one' Then 'Uno'
           WHEN @desc = 'two' Then 'Dos'
           ELSE 'Tres o Mas'
       END "DESCRIPTION"
```

#### `DATEPART` for parsing dates

```sql
DECLARE @datmCurrent DATETIME = CURRENT_TIMESTAMP

SELECT DATEPART(YEAR, @datmCurrent) 'Year',     --2021
       DATEPART(MONTH, @datmCurrent) 'Month',   --2
	   DATEPART(DAY, @datmCurrent) 'Day',       --3 
	   DATEPART(HOUR, @datmCurrent) 'Hour',     --16
	   DATEPART(MINUTE, @datmCurrent) 'Minute', --36
	   DATEPART(SECOND, @datmCurrent) 'Second'  --29
```

#### `DATEADD` for incrementing and decrementing dates

```sql
DECLARE @datmCurrent DATETIME = CURRENT_TIMESTAMP

SELECT DATEADD(YEAR, 1, @datmCurrent) 'Added 1 Year',
       DATEADD(MONTH, 4, @datmCurrent) 'Added 4 Months',
	   DATEADD(DAY, 9, @datmCurrent) 'Added 9 Days',
	   DATEADD(HOUR, 10, @datmCurrent) 'Added 10 Hours',
	   DATEADD(MINUTE, 57, @datmCurrent) 'Added 57 Minutes',
	   DATEADD(SECOND, -100, @datmCurrent) 'Subtracted 100 Seconds'       
```

#### `DATEDIFF` for getting differences between dates

```sql
DECLARE @datmCurrent DATETIME = CURRENT_TIMESTAMP
DECLARE @datmFirstDay DATETIME = CAST('5/11/2020 08:00:00' AS DATETIME)

SELECT DATEDIFF(YEAR, @datmFirstDay, @datmCurrent) 'Total Years',
       DATEDIFF(MONTH, @datmFirstDay, @datmCurrent) 'Total Months',
       DATEDIFF(DAY, @datmFirstDay, @datmCurrent) 'Total Days',
       DATEDIFF(HOUR, @datmFirstDay, @datmCurrent) 'Total Hours',
       DATEDIFF(MINUTE, @datmFirstDay, @datmCurrent) 'Total Minutes',
       DATEDIFF(SECOND, @datmFirstDay, @datmCurrent) 'Total Seconds'      
```

#### Get current date time

```sql
SELECT GETDATE()
> 2021-02-02 16:29:29.943

SELECT CURRENT_TIMESTAMP
> 2021-02-02 16:29:29.943

SELECT SYSDATETIME()
> 2021-02-02 16:29:29.9453914

SELECT CONVERT(VARCHAR(10), SYSDATETIME(), 101)
> 02/02/2021

DECLARE @datDate DATE 
DECLARE @timTime TIME

SET @datDate = (SELECT CAST(CURRENT_TIMESTAMP AS DATE))
SET @timTime = (SELECT CAST(CURRENT_TIMESTAMP AS TIME))
```

#### Query against date / time / timestamp fields

```sql
SELECT *
FROM   family_birthdays
WHERE  birthdate > '1/1/2015'

SELECT *
FROM   family_birthdays
WHERE  birthdate > CAST('1/1/2015' AS DATE)

SELECT *
FROM   family_birthdays
WHERE  birthdate > '1/1/2015 01:23:45'

SELECT *
FROM   family_birthdays
WHERE  birthdate > CAST('1/1/2015 01:23:45' AS DATETIME)
```

#### `LEAD` and `LAG`

```sql
CREATE TABLE #months (ordinal INT, abbrev VARCHAR(3), name VARCHAR(20))

INSERT INTO #months VALUES
    (1, 'JAN', 'January'),
    (2, 'FEB', 'February'),
    (3, 'MAR', 'March'),
    (4, 'APR', 'April'),
    (5, 'MAY', 'May'),
    (6, 'JUN', 'June'),
    (7, 'JUL', 'July'),
    (8, 'AUG', 'August'),
    (9, 'SEP', 'September'),
    (10, 'OCT', 'October'),
    (11, 'NOV', 'November'),
    (12, 'DEC', 'December')

SELECT m.ordinal, m.abbrev, m.name,
       LAG(m.name, 1) OVER (ORDER BY m.ordinal) 'Last Month',
       LEAD(m.name, 1) OVER (ORDER BY m.ordinal) 'Next Month'
FROM   #months m

DROP TABLE #months
```

#### `ROWNUMBER OVER [PARTITION]`

```sql
SELECT department_id, last_name, employee_id, 
       ROW_NUMBER() OVER (ORDER BY employee_id) AS emp_id
FROM   employees

SELECT department_id, last_name, employee_id, 
       ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY employee_id) AS emp_id
FROM   employees;
```

#### `PIVOT` - place values from rows into columns

```sql
SELECT cmcs.Route, cmcs.ConsistId, cmcs.Car1, cmcs.Car2, cmcs.Car3, cmcs.Car4, cmcs.Car5
FROM   (
       SELECT    Route, ConsistID, 
                 MAX(ISNULL([1], 0)) 'Car1', 
                 MAX(ISNULL([2], 0)) 'Car2', 
                 MAX(ISNULL([3], 0)) 'Car3', 
                 MAX(ISNULL([4], 0)) 'Car4',
                 MAX(ISNULL([5], 0)) 'Car5'
        FROM     view_CM_CurrentStatus
        PIVOT    (MAX(CarId) FOR SeqNumber IN ([1],[2],[3],[4],[5])) pvt 
        WHERE    Route > 0
        GROUP BY Route, ConsistID
      ) cmcs   
```

#### Create new table from an existing table (automatic schema inheritance)

```sql
SELECT *
INTO   #tempTable
FROM   realTable
```

#### Insert records into existing table from another table

```sql
INSERT INTO #tempTable
    SELECT *
    FROM   realTable
    
INSERT INTO statesCapitols(id, stateabbrev, statename, capitolcity)
VALUES
    (1, 'CA', 'California', 'Sacramento'),
    (2, 'NV', 'Nevada', 'Carson City'),
    (3, 'OR', 'Oregon', 'Salem'),
    (4, 'WA', 'Washington', 'Olympia'),
    (5, 'AZ', 'Arizona', 'Phoenix')    
```

#### Insert records into table from stored procedure

```sql
DECLARE @stateAbbreviation VARCHAR(2) = 'CA'

CREATE TABLE #CA_Cities(
                        CityId,
                        CityName
)

INSERT #CA_Cities
    EXEC dbo.sp_Get_Cities_By_State @stateAbbreviation    
```

#### String functions

```sql
SELECT LEN('abc') 
> 3

SELECT UPPER('abc') 
> ABC

SELECT LOWER('ABC') 
> abc

SELECT REVERSE('abc') 
> cba

SELECT REPLACE('xyz', 'y', 's') 
> xsz

SELECT SUBSTRING('xyz', 1, 2) 
> xy

SELECT CHARINDEX('g', 'abcdefgabcdefg', 0)
> 7

SELECT CONCAT('a', 'b') 
> ab

SELECT LTRIM('   abc   ') 
> abc___   

SELECT RTRIM('   abc   ') 
>    abc
```

#### Transaction control

```sql
SET NOCOUNT ON;

DECLARE @intError INTEGER

BEGIN TRANSACTION

    UPDATE someTable
    SET    someColumn = 0
    WHERE  someOtherColumn = 'a'

SET @intError = @@ERROR

IF @intError > 0
    ROLLBACK TRANSACTION
ELSE
    COMMIT TRANSACTION

--For Stored Procedures/Functions:
--RETURN @intError     
```

#### Query statistics

```sql
SET STATISTICS TIME, IO ON

SELECT * 
FROM   realTable

SET STATISTICS TIME, IO OFF
```

anatomy of every DBA query

```sql
SELECT   what_you_want_to_see   -- columns
FROM     where_the_data_lives   -- table or view
WHERE    filter_condition       -- optional: narrow the rows
ORDER BY how_to_sort_the_result -- optional: sort
```

```sql
SELECT name, db_unique_name, open_mode, log_mode   -- 4  columns 
FROM v$database                                    -- oracle's internal view

```

where clause 
```sql
-- without where returns all databfiles 
SELECT file#, name, bytes/1024/1024 as MB
FROM v$datafile;

-- with where: returns only ORCLPDB1 datafiles
SELECT file#, name, bytes/1024/1024 as MB
FROM v$datafile
WHERE name LIKE '%ORACLPDB1%'

-- with where + and : only ORCLPDB1 files bigger than 200MB
SELECT file#, name, bytes/1024/1024 as MB
FROM v$datafile
WHERE name LIKE '%ORACLPDB1%'
AND bytes/1024/1024 > 200;
```

% matches the wildcard before and after , anywhere 

expressions in select (doing the math inline)

```sql
-- bytes is stores in oracle as raw bytes(big number)
-- we divide inline to make it human readable

SELECT name, 
       bytes                        raw_bytes,   -- original
       bytes/1024/1024              KB,          -- kilobytes
       bytes/1024/1024              MB,          -- megabytes 
       bytes/1024/1024/1024         GB           -- gigabytes
FROM v$datafile;
```
raw_bytes, kb word after the expression is called alias - to label the column in output

groupby (summarising data)

```sql
SELECT pool,
       sum(bytes)/1024/1024 MB   -- add up all bytes in each pool, convert to MB
FROM   v$sgastat
GROUP BY pool                    -- one output row per unique pool name
ORDER BY MB DESC;                -- biggest pool first
```

`GROUP BY` says: _instead of one row per data row, give me one row per unique value of this column, with the math applied across the whole group._

Inline / subquery
```sql
SELECT df.tablespace_name,
       round(df.total_mb)                    total_mb,
       round(df.total_mb - fs.free_mb)       used_mb,
       round(fs.free_mb)                     free_mb,
       round((fs.free_mb / df.total_mb)*100) pct_free
FROM
  (SELECT tablespace_name, sum(bytes)/1024/1024 total_mb
   FROM   dba_data_files
   GROUP BY tablespace_name) df,
  (SELECT tablespace_name, sum(bytes)/1024/1024 free_mb
   FROM   dba_free_space
   GROUP BY tablespace_name) fs
WHERE  df.tablespace_name = fs.tablespace_name
ORDER BY pct_free ASC;
```

```sql
SELECT tablespace_name, sum(bytes)/1024/1024 total_mb
   FROM   dba_data_files
   GROUP BY tablespace_name
   
TABLESPACE_NAME 		 TOTAL_MB
------------------------------ ----------
USERS					7
UNDOTBS1			       40
SYSTEM				     1120
SYSAUX				      820

```

```sql
SELECT tablespace_name, sum(bytes)/1024/1024 free_mb FROM dba_free_space GROUP BY tablespace_name

TABLESPACE_NAME 		  FREE_MB
------------------------------ ----------
SYSTEM				     5.25
SYSAUX				   44.625
UNDOTBS1			   11.875
USERS				    .9375
```

```sql
Step 1 → Run inner query df → produces: tablespace_name, total_mb → one row per tablespace 
Step 2 → Run innerquery fs → produces: tablespace_name, free_mb → one row per tablespace 
Step 3 → JOIN df and fs on tablespace_name→ combine total and free into one row → calculate pct_free
```

```sql
TABLESPACE_NAME 		 TOTAL_MB    USED_MB	FREE_MB   PCT_FREE
------------------------------ ---------- ---------- ---------- ----------
SYSTEM				     1120	1115	      5 	 0
SYSAUX				      820	 776	     44 	 5
USERS					7	   6	      1 	13
UNDOTBS1			       40	  29	     11 	27
```

```sql
TABLESPACE    TOTAL    USED    FREE    PCT_FREE
──────────    ─────    ────    ────    ────────
SYSTEM         1120    1115       5        0%   ← 🔴 CRITICAL
SYSAUX          820     775      45        5%   ← 🟡 WARNING
USERS             7       6       1       13%   ← 🟡 LOW
UNDOTBS1         40      28      12       30%   ← 🟢 OK
```

check the actual usage on system

```sql
SELECT file_name,
       autoextensible,
       round(bytes/1024/1024)     current_mb,
       round(maxbytes/1024/1024)  max_mb
FROM   dba_data_files
WHERE  tablespace_name = 'SYSTEM';

FILE_NAME				      AUT CURRENT_MB	 MAX_MB
--------------------------------------------- --- ---------- ----------
/opt/oracle/oradata/ORCLCDB/system01.dbf      YES	1120   33554432
```

```
AUTOEXTENSIBLE = YES
```

This means when SYSTEM tablespace fills up — Oracle automatically extends `system01.dbf` on disk. It does not stop and wait for a DBA. It just grows.

```
File          system01.dbf
Current size  1120 MB      ← where it is now
Max size      33,554,432 MB ← where it can grow to
AUTOEXTEND    YES
```

```
33,554,432 MB
÷ 1024 = 32,768 GB
÷ 1024 = 32 TB
```

Your SYSTEM datafile can grow to **32 TB** automatically. It is currently at 1120 MB. You have effectively unlimited headroom.

This is Oracle's default MAXSIZE when you enable AUTOEXTEND without specifying a limit — it sets the theoretical maximum for the filesystem/OS. In practice your disk will run out long before 32 TB.

```sql
col file_name for a45
SELECT tablespace_name,
       autoextensible,
       round(bytes/1024/1024)      current_mb,
       round(maxbytes/1024/1024)   max_mb,
       round(user_bytes/1024/1024) usable_mb
FROM   dba_data_files
ORDER BY tablespace_name;

TABLESPACE_NAME 	       AUT CURRENT_MB	  MAX_MB  USABLE_MB
------------------------------ --- ---------- ---------- ----------
SYSAUX			       YES	  820	33554432	752
SYSTEM			       YES	 1120	33554432       1119
UNDOTBS1		       YES	   40	33554432	 39
USERS			       YES	    7	33554432	  1
```

```
AUTOEXTEND OFF database          AUTOEXTEND ON database
────────────────────────         ──────────────────────
Monitor PCT_FREE closely         Monitor DISK space closely
Tablespace full = immediate stop Tablespace grows automatically
DBA must manually extend         DBA must watch OS disk usage
```

```bash
[oracle@ol9ai ~]$ df -h /opt/oracle/oradata/
Filesystem          Size  Used Avail Use% Mounted on
/dev/mapper/ol-u01   53G   11G   43G  21% /opt/oracle
[oracle@ol9ai ~]$
```

```
SYSTEM      current 1120 MB  →  cap at 2048 MB   (2 GB)
SYSAUX      current  820 MB  →  cap at 2048 MB   (2 GB)
UNDOTBS1    current   40 MB  →  cap at 1024 MB   (1 GB)
USERS       current    7 MB  →  cap at 1024 MB   (1 GB)
─────────────────────────────────────────────────────
Total maximum footprint       →  ~6 GB
Leaves                        →  ~47 GB free
```

## Step 2 — Tablespace and Datafile Management

Now we do real work. Four tasks:

```
Task 1 → Set sensible MAXSIZE on existing datafiles
Task 2 → Create a new tablespace from scratch
Task 3 → Add a datafile to an existing tablespace
Task 4 → Resize a datafile manually
```
Task 1 — Set MAXSIZE on Existing Datafiles
```sql
SQL> col file_name for a45
SELECT tablespace_name,
       round(bytes/1024/1024)     current_mb,
       round(maxbytes/1024/1024)  max_mb,
       autoextensible
FROM   dba_data_files
ORDER BY tablespace_name;SQL>   2    3    4    5    6

TABLESPACE_NAME 	       CURRENT_MB     MAX_MB AUT
------------------------------ ---------- ---------- ---
SYSAUX				      820   33554432 YES
SYSTEM				     1120   33554432 YES
UNDOTBS1			       40   33554432 YES
USERS

-- Cap SYSTEM at 2GB
ALTER DATABASE DATAFILE
'/opt/oracle/oradata/ORCLCDB/system01.dbf'
AUTOEXTEND ON NEXT 100M MAXSIZE 2048M;

-- Cap SYSAUX at 2GB
ALTER DATABASE DATAFILE
'/opt/oracle/oradata/ORCLCDB/sysaux01.dbf'
AUTOEXTEND ON NEXT 100M MAXSIZE 2048M;

-- Cap UNDOTBS1 at 1GB
ALTER DATABASE DATAFILE
'/opt/oracle/oradata/ORCLCDB/undotbs01.dbf'
AUTOEXTEND ON NEXT 64M MAXSIZE 1024M;

-- Cap USERS at 1GB
ALTER DATABASE DATAFILE
'/opt/oracle/oradata/ORCLCDB/users01.dbf'
AUTOEXTEND ON NEXT 32M MAXSIZE 1024M;

TABLESPACE_NAME 	       CURRENT_MB     MAX_MB AUT
------------------------------ ---------- ---------- ---
SYSAUX				      820	2048 YES
SYSTEM				     1120	2048 YES
UNDOTBS1			       40	1024 YES
USERS					7	1024 YES

SQL>
```

## Task 2 — Create a New Tablespace From Scratch

This is one of the most common DBA tasks. A new application arrives — it needs its own tablespace.

```sql
CREATE TABLESPACE training_tbs
DATAFILE '/opt/oracle/oradata/ORCLCDB/training01.dbf'
SIZE 100M
AUTOEXTEND ON NEXT 50M MAXSIZE 500M
EXTENT MANAGEMENT LOCAL
SEGMENT SPACE MANAGEMENT AUTO;
```

Every clause has a reason:
```sql
SIZE 100M              →  start at 100MB
AUTOEXTEND ON          →  grow automatically
NEXT 50M               →  grow 50MB at a time
MAXSIZE 500M           →  never exceed 500MB
EXTENT MANAGEMENT LOCAL →  modern method, bitmap tracking
SEGMENT SPACE MANAGEMENT AUTO → Oracle manages free space
```

```sql
SQL> CREATE TABLESPACE training_tbs
DATAFILE '/opt/oracle/oradata/ORCLCDB/training01.dbf'
SIZE 100M
AUTOEXTEND ON NEXT 50M MAXSIZE 500M
EXTENT MANAGEMENT LOCAL
SEGMENT SPACE MANAGEMENT AUTO;  2    3    4    5    6

Tablespace created.

SQL> SELECT tablespace_name, status, contents,
       extent_management, segment_space_management
FROM   dba_tablespaces
WHERE  tablespace_name = 'TRAINING_TBS';  2    3    4

TABLESPACE_NAME 	       STATUS	 CONTENTS	       EXTENT_MAN SEGMEN
------------------------------ --------- --------------------- ---------- ------
TRAINING_TBS		       ONLINE	 PERMANENT	       LOCAL	  AUTO

SQL>
```


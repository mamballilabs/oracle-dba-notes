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

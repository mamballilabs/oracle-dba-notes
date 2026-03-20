```sql
SQL> select name, value/1024/1024 MB from v$sga;

NAME			     MB
-------------------- ----------
Fixed Size	     4.79325104
Variable Size		   1200
Database Buffers	   5856
Redo Buffers	      8.4453125

SQL> 

```

SGA is a shared memory area allocated from RAM when the instance starts. It contains the database buffer cache which holds data blocks read from disk, the redo log buffer which temporarily holds changes before they are hardened to redo log files, variable size contains the shared pool which caches SQL statements and execution plans.

Variable size:
```sql
SQL> select pool, sum(bytes)/1024/1024 MB 
  2  from v$sgastat
  3  where pool is not null
  4  group by pool
  5  order by MB desc;

POOL		       MB
-------------- ----------
shared pool	     1136
large pool	       48

SQL> 
```

Large pool:
```
RMAN          → backup and restore operations
Parallel Query → when Oracle uses multiple CPUs for one query
Shared Server  → when many thin clients share server processes
```

shared pool:
this is where the sql lives 

1. library cahce
every sql statement you run gets stored here after oracle parses it
This is oracle's sql memory. same query, same plan, no wasted work
```
First time you run a query:
SELECT * FROM employees WHERE dept = 10

Oracle reads it → checks syntax → builds execution plan
→ stores SQL + plan in Library Cache
Time taken: relatively slow (hard parse)

Second time same query runs:
Oracle finds it in Library Cache → reuses the plan
Time taken: very fast (soft parse)
```

2. dictionary cahce
stores metadata about the database objects - table names, column names, user privileges
every time oracle parses sql it needs to check 
- does this table exist
- does this user have permissions
- what columns does this table have
without this oracle should read this from system datafile on disk every single time, slow

```
SGA
│
├── Database Buffers  5856 MB  ← data blocks from disk
│
├── Variable Size     1200 MB
│   ├── Shared Pool   1136 MB
│   │   ├── Library Cache      ← SQL + execution plans
│   │   └── Dictionary Cache   ← table/user/privilege metadata
│   │
│   └── Large Pool      48 MB  ← RMAN, parallel query
│
├── Redo Buffers         8 MB  ← changes before hardening
│
└── Fixed Size           5 MB  ← Oracle internals
```

what happens when we run the same sql again:
first run
```
First execution:
─────────────────
SQL arrives → Library Cache checked → NOT FOUND
                                           │
                                           ▼
                                    Hard Parse begins
                                           │
                                           ▼
                                 Dictionary Cache consulted
                                 "does this table exist?"
                                 "what columns does it have?"
                                 "does this user have access?"
                                           │
                                           ▼
                                 Execution plan built
                                           │
                                           ▼
                                 SQL + plan stored in Library Cache
```

second run
```
2nd through 100th execution:
──────────────────────────────
SQL arrives → Library Cache checked → FOUND
                                           │
                                           ▼
                                    Soft Parse
                                    reuse existing plan
                                    Dictionary Cache NOT needed
                                    execution plan NOT rebuilt
                                    extremely fast
```

when oracle finds the sql in library cache and reuses the plan - cursor sharing 
the stored sql + plan in the library cache is called cursor 
reusing it is cursor sharing

```sql
SELECT component,
       current_size/1024/1024 current_MB
FROM   v$sga_dynamic_components
WHERE  current_size > 0
ORDER BY current_MB DESC;
COMPONENT							 CURRENT_MB
---------------------------------------------------------------- ----------
DEFAULT buffer cache						       5728
shared pool							       1136
Shared IO Pool								128
large pool								 48

SQL> 

DEFAULT buffer cache    5728 MB   ← Database Buffers (performance)
shared pool             1136 MB   ← Library Cache + Dictionary Cache
Shared IO Pool           128 MB   ← new one — explain below
large pool                48 MB   ← RMAN, parallel, shared server
```

shared IO pool - handles smart scan operations , direct path reads that by pass the buffer cache entirely for large full table scans

```sql
Normal read       → goes through buffer cache (5728 MB)
Large table scan  → goes through Shared IO Pool (128 MB)
                    bypasses buffer cache
                    avoids polluting cache with one-time data
```
It protects your buffer cache. When a huge report scans millions of rows once and never again — it goes through Shared IO Pool instead of flushing all your cached hot blocks out of the buffer cache.

Earlier `v$sga` showed Database Buffers as **5856 MB**. Now `v$sga_dynamic_components` shows DEFAULT buffer cache as **5728 MB**.
The difference is **128 MB** — exactly the Shared IO Pool.

```
v$sga shows:                    v$sga_dynamic_components shows:
────────────────────────        ───────────────────────────────
Database Buffers  5856 MB  →    DEFAULT buffer cache  5728 MB
                                + Shared IO Pool        128 MB
                                                      ────────
                                                        5856 MB ✅

Variable Size     1200 MB  →    shared pool           1136 MB
                                + large pool             48 MB
                                + overhead               ~16 MB
                                                      ────────
                                                       ~1200 MB ✅
```

test
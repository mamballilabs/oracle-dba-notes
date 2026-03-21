shared global area is the shared memory region allocated by oracle when the instance starts.
Every single server process, every session, every background process -- all read and write to same block of RAM

allocates once at the startup and stays reserved until the instance shuts down

![[Pasted image 20260317114659.png]]
1. Buffer cache - the biggest one
oracle never reads data directly from disk into your session, it always goes through the buffer cache.

``` 
Your query runs
      │
      ▼
Oracle looks for the block in Buffer Cache
      │
      ├── FOUND? ──► Return it immediately (cache HIT — fast)
      │
      └── NOT FOUND? ──► Read from disk into Buffer Cache
                              then return it (cache MISS — slow)
```
A well-tuned database has a buffer cache hit ratio above 95%.

2. shared pool 
every sql statement you run goes through the shared pool 

a. Library cache:
stores the sql text and its execution plan, when you run the same query twice. Oracle reuses the cached plan instead of re-parsing 
this is called a soft-parse and it is much faster than a hard parse (parsing from scratch)

b. Dictionary cache:
row cache , stores metadata about tables, columns, users, privileges.
when oracle needs to know "does this table exist" "what columns does it have" it checks here first instead of going to the SYSTEM tablespace.

3. redo log buffer
every change you make - insert, update, delete - oracle records it here first, before touching the actual data  blocks. 
this is the foundation of oracle's crash recovery
LGWR - log writer background process flushes this on to the redo log files on disk , on Commit 

4. Large pool
A separate pool to offload large memory allocations away from the shared pool.
3 main users
RMAN - backup and resotre
parallel query
shared server connections
without this , would fragment the shared pool and hurt sql performance

5. fixed sga+ JAVA pool + streams pool
non configurable internal oracle overhead - fixed sga
java pool - relevant if we run java stored procedures
streams pool - golden gate and replication

```sql
SQL> select name, value/1024/1024 MB from v$sga;

NAME				  MB
------------------------- ----------
Fixed Size		  4.79325104
Variable Size			1200
Database Buffers		5856
Redo Buffers		   8.4453125

SQL> 
```

variable size = 1200 MB 
container we have three things
1. shared pool -- where sql goes after you type it
2. large pool    -- where rman and parallel query work
3. java pool      -- for java stored procedures

database buffers = 5856 MB 
where oracle keeps copies of your data in RAM so it does not have to go to disk every time

redo buffers = 8.44 MB
every change we make to the database gets written here first


┌─────────────────────────────────────────┐
│              SGA — Total ~7069 MB                                                                    │
│                                                                                                                               │
│  ┌─────────────────────────────────┐                 │
│  │   Database Buffers   5856 MB                                             │                 │
│  │   (Buffer Cache)                                                                       │                 │
│  └─────────────────────────────────┘                │
│                                                                                                                              │
│  ┌─────────────────────────────────┐    │
│  │   Variable Size      1200 MB    │    │
│  │   (Shared Pool + Large Pool)    │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌──────────────┐  ┌──────────────┐     │
│  │ Redo Buffers │  │  Fixed Size  │     │
│  │    8.44 MB   │  │   4.79 MB   │     │
│  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────┘

```sql
SELECT pool,
       name,
       bytes/1024/1024 MB
FROM   v$sgastat
WHERE  pool IS NOT NULL
ORDER BY pool, bytes DESC
FETCH FIRST 15 ROWS ONLY;
POOL	       NAME				 MB
-------------- ------------------------- ----------
large pool     free memory		       25.5
large pool     PX msg pool		       22.5
shared pool    free memory		 202.732346
shared pool    SQLA			 167.522583
shared pool    KGLH0			 76.8938751
shared pool    SQLA			 55.3709946
shared pool    KGLS			 35.0603104
shared pool    KGLH0			 32.2901459
shared pool    db_block_hash_buckets	 32.0004883
shared pool    KGLS			 25.5550232
shared pool    ksunfy_sess_meta 1	 24.9926147
shared pool    KQR X PO 		 23.7143784
shared pool    SO private sga		 22.4152985
shared pool    PLMCD			 17.2877655
shared pool    KGLHD			 15.2892456

15 rows selected.
```

first two pools
1. large pool 
2. shared pool

large pool:
```sql
POOL	       NAME				 MB
-------------- ------------------------- ----------
large pool     free memory		       25.5
large pool     PX msg pool		       22.5
```
free memory : unused waiting 
PX msg pool  : PX means parallel execution.
when oracle runs a query using multiple CPUs simultaneously they send messages to coordinate with each other.

Shard pool:
```sql
shared pool    free memory		 202.732346
```
unused shared pool
```sql
shared pool    SQLA			 167.522583
shared pool    SQLA			 55.3709946
```
SQLA : sql area.
this is where the oracle stores your sql statements and their execution plans after parsing
this is the library cache which makes soft parsing possible 

```sql
shared pool    KGLH0			 76.8938751
shared pool    KGLS			 35.0603104
shared pool    KGLH0			 32.2901459
shared pool    KGLS			 25.5550232
shared pool    KGLHD			 15.2892456
```

KGL : kernel generic library
oracles internal library cache management 
KGL structures track every sql statement, every pl/sql object, every package that is currently loaded.
Think of this as an index of everything cached in the library cache

```sql
shared pool    db_block_hash_buckets	 32.0004883
```
this is how oracle finds the blocks in the buffer cache quickly
when a session needs a specific block , oracle uses the hash function to find it instantly instead of searching through all 5856 MB
this hash table lives in the shared pool because all sessions need to use it
```sql
shared pool    KQR X PO 		 23.7143784
```
KQR : kernel query row source
stores optimizer statistics and query compilation data.
when oracle decides how to run your sql - which index to use, which join method - it uses data stored here
```sql
shared pool    SO private sga            22 MB
shared pool    ksunfy_sess_meta 1        24 MB
```

session metadata : information about currently connected sessions stored in shared memory so background processes can access it.
```sql
shared pool    PLMCD                     17 MB
```
pl/sql machine code data : compiled pl/sql procedures and functions live here - ready to execute without recompiling

Big picture
```
BUCKET 1 — SQL and execution plans
           SQLA (~222 MB)
           → "I have seen this query before, here is the plan"

BUCKET 2 — Metadata about database objects  
           KGL* structures (~184 MB)
           → "I know what tables and indexes exist"

BUCKET 3 — Free space waiting
           free memory (~202 MB)
           → "ready for the next surge of activity"
```
Your database is healthy and relaxed. In a production system under heavy load, free memory would shrink toward zero. If it hits zero — Oracle cannot parse new SQL. That is called **shared pool exhaustion** and it is a serious performance problem.

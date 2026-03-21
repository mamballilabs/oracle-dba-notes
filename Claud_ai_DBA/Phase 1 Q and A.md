1. "Can you explain the difference between an Oracle instance and an Oracle database? What happens if the instance crashes — what survives and what is lost?"
```
_"An Oracle instance is the combination of memory structures — SGA and PGA — and background processes like DBW0, LGWR, CKPT, SMON and PMON. The instance mounts and opens the database, which is the collection of physical files on disk — datafiles, redo log files, control files and the parameter file._

_If the instance crashes — everything in RAM is lost. The SGA disappears, all cached SQL is gone, dirty blocks not yet written to disk are lost from memory. However everything on disk survives — all datafiles, redo log files, control files remain intact. When the instance restarts, SMON automatically performs crash recovery using the surviving redo log files to bring the database back to a consistent state."_
```

2. "What is the SGA? Name its components and explain what the largest component does."
```
_"The SGA — System Global Area — is a shared memory region allocated from RAM when the instance starts. All sessions and background processes share it._

_It has four components: Fixed Size for Oracle internals, Redo Buffers which temporarily holds changes before LGWR hardens them to disk, Variable Size which contains the shared pool and large pool, and Database Buffers — the buffer cache._

_The largest component is the buffer cache — 5856 MB in our instance. It holds copies of datafile blocks in RAM so Oracle does not have to read from disk every time. Performance comes from here — a well tuned database serves 95%+ of block requests from this cache without touching disk._

_Inside Variable Size — the shared pool holds the library cache for SQL and execution plans, and the dictionary cache for database object metadata. The large pool handles RMAN, parallel query, and shared server connections."_
```

3. "What is the difference between a hard parse and a soft parse? Why does it matter for performance?"

performance impact
```
Hard parse cost:
├── Parse SQL syntax          → CPU
├── Check object existence    → dictionary cache lookup
├── Check user privileges     → dictionary cache lookup
├── Build execution plan      → CPU intensive
├── Allocate shared pool mem  → memory allocation
└── Store in library cache    → memory write

Soft parse cost:
└── Find cursor in library cache → one lookup
    Done. Reuse everything.
```

In a busy system running thousand of queries per second

```
Hard parse every query   →   CPU spikes, shared pool pressure
Soft parse every query   →   minimal CPU, fast response
```

```
_"When Oracle receives a SQL statement it first checks the library cache in the shared pool to see if an identical statement has been parsed before and has a cached execution plan._

_If found — Oracle reuses the existing plan. This is a soft parse — very fast, minimal CPU, no need to rebuild the plan._

_If not found — Oracle must do a hard parse. It checks syntax, validates objects exist in the dictionary cache, checks user privileges, builds a new execution plan, and stores it in the library cache. This is expensive — CPU intensive and requires memory allocation in the shared pool._

_In a production system running thousands of queries per second, hard parsing every query would cause CPU spikes and shared pool memory pressure. This is why application developers use bind variables — the same SQL text with different values reuses the same cached plan, keeping everything as soft parses."_
```

4. "Name the five critical background processes and tell me what happens to the database if any one of them dies."
```
DBWn dies   →   instance crash immediately
LGWR dies   →   instance crash immediately
CKPT dies   →   instance crash immediately
PMON dies   →   instance crash immediately
SMON dies   →   instance crash immediately
```

```
_"The five critical background processes are DBWn, LGWR, CKPT, SMON and PMON._

_DBWn — Database Writer — writes dirty blocks from the buffer cache to datafiles on disk. It writes in batches, when the cache is full, or when CKPT signals it._

_LGWR — Log Writer — flushes the redo log buffer to online redo log files. It fires every 3 seconds, when the buffer is one third full, or when a session commits._

_CKPT — Checkpoint — signals DBWn to write dirty blocks, then updates the control file and all datafile headers with the current SCN. This marks the point from which crash recovery would begin._

_PMON — Process Monitor — cleans up after failed sessions. It rolls back uncommitted transactions, releases locks, and frees memory for any session that dies unexpectedly._

_SMON — System Monitor — performs crash recovery at instance startup. It rolls forward committed changes not yet on disk, then rolls back any uncommitted transactions, leaving the database consistent._

_If any one of these five processes dies — the instance crashes immediately. Oracle cannot function without all five running."_
```

5. "Your database is in NOARCHIVELOG mode. A developer accidentally drops a critical table at 2 PM. It is now 3 PM. Can you recover it? What are your options?"

```
"In NOARCHIVELOG mode — point in time recovery is impossible. I cannot recover to exactly 2 PM minus one second."

Option 1 — Restore full backup
If last backup was last night at midnight:
→ Restore entire database to midnight state
→ Lose everything from midnight to 3 PM
→ ALL data lost, not just the dropped table
→ Very destructive option

Option 2 — Flashback Query (may work)
→ If flashback is enabled and undo data still exists
→ SELECT * FROM employees AS OF TIMESTAMP (SYSDATE - 1/24)
→ Can see data as it was 1 hour ago
→ Does NOT require ARCHIVELOG mode
→ Depends on undo retention period

Option 3 — Flashback Table (may work)
→ FLASHBACK TABLE employees TO BEFORE DROP
→ Uses recycle bin — if not purged
→ Does NOT require ARCHIVELOG mode
→ Fastest option if available

Option 4 — Prevent it next time
→ Enable ARCHIVELOG mode
→ Enable regular RMAN backups
→ Then point in time recovery becomes possible
```

```
NOARCHIVELOG mode means I can restore — but I cannot recover. Restore takes me back to the backup. Recovery takes me forward to the point of failure. Without archive logs I have no way to move forward.

NOARCHIVELOG:
Backup (midnight) ─────────────────────────────► NOW
                  ↑                          ↑
                  restore here               drop happened here
                  lose 15 hours of data      cannot get here

ARCHIVELOG:
Backup (midnight) ──archive logs──► drop point ──► recover to here
                                    2 PM            2:00:00 PM
                                    restore and apply logs precisely
```

6. "What is a checkpoint? Why does Oracle need it? What would happen if Oracle never ran a checkpoint?"
what physically happens during a checkpoint:

```
CKPT fires
├── Signals DBWn to write ALL dirty blocks to disk
├── Updates control file with current SCN
└── Updates all datafile headers with current SCN

```

why oracle need it :

Without checkpoint — dirty blocks sit in buffer cache indefinitely. On crash — Oracle would need to replay ALL redo logs from the very beginning of time to recover. That could take hours or days.

what happens if oracle never checkpointed
```
No checkpoint ever:
├── Buffer cache fills with dirty blocks
├── On crash — replay redo from day one
├── Recovery takes hours or days
└── Database unusable for extended period
```

```
_"A checkpoint is Oracle's way of synchronising memory with disk at a known point in time, identified by an SCN — System Change Number._

_When CKPT fires — it signals DBWn to write all dirty blocks from the buffer cache to datafiles on disk. Once DBWn confirms the writes, CKPT updates the control file and all datafile headers with the current SCN — marking that everything up to this point is safely on disk._

_Oracle needs checkpoints for two reasons. First — to limit crash recovery time. If the database crashes, SMON only needs to replay redo logs from the last checkpoint forward — not from the beginning of time. Second — to free up buffer cache space by flushing dirty blocks to disk._

_If Oracle never ran a checkpoint — dirty blocks would accumulate indefinitely in the buffer cache. On a crash, SMON would need to replay every redo log ever written to recover the database. Recovery could take hours or days. The checkpoint position is essentially Oracle saying — everything before this SCN is safe on disk, only worry about what came after."_
```

```
> *_"Checkpoint limits recovery time — the more frequent the checkpoint, the faster the recovery, but the higher the I/O cost during normal operations."_

That trade-off — checkpoint frequency vs I/O cost — is a classic DBA tuning decision. Interviewers love it when candidates mention trade-offs.
```

7. "What is the difference between a tablespace, a segment, an extent and a block? Give a real example using a table."
```
Tablespace    →   can contain MANY segments of MANY types
                  tables, indexes, sequences — all in one tablespace
                  what it cannot mix is PERMANENT vs UNDO vs TEMPORARY
                  those are always separate tablespaces
```

```
> _"All four are logical storage structures — they exist inside Oracle, not as separate files on disk._
> 
> _Block is the smallest unit — 8 KB in our database. Every read and write Oracle does moves in 8 KB blocks. The buffer cache holds blocks._
> 
> _Extent is a contiguous group of blocks allocated together. Oracle never allocates one block at a time — it allocates extents for efficiency._
> 
> _Segment is one complete database object — one table gets one segment, one index gets one segment. A segment grows by acquiring more extents as data is inserted._
> 
> _Tablespace is a logical container grouping related segments together, mapped to one or more datafiles on disk._

> _Real example — EMPLOYEES table in ORCLPDB1_
```

```
USERS tablespace
└── users01.dbf  (physical file — 7 MB)
    └── EMPLOYEES segment
        ├── Extent 1 — blocks 1 to 8    (first allocation — 64 KB)
        ├── Extent 2 — blocks 9 to 16   (table grew — 64 KB more)
        └── Extent 3 — blocks 33 to 48  (table grew again — 128 KB)
            └── Each block — 8 KB
                └── Holds actual employee rows
```

```
As more employees are inserted — Oracle allocates more extents to the segment automatically. The tablespace provides the space, the segment owns it, extents carve it up, blocks hold the actual rows."
```

```
"A tablespace is the what — what logical container. A datafile is the where — where it lives on disk. One tablespace can span many datafiles but one datafile belongs to exactly one tablespace."
```

8. "You connect to your Oracle database using SQL*Plus. Walk me through exactly what happens — from the moment you type the connect command to the moment you get the SQL prompt. Which Oracle components are involved?"
```
Step 1 — You type: sqlplus / as sysdba
              │
              ▼
Step 2 — Oracle Listener receives your request
         (LREG background process registers
          the instance with the listener)
              │
              ▼
Step 3 — Dedicated server process spawned
         One server process created just for you
         This process serves ALL your SQL requests
              │
              ▼
Step 4 — PGA allocated in RAM
         Your private memory created:
         ├── Session information
         ├── Sort area
         └── Hash area
              │
              ▼
Step 5 — SQL prompt appears
         You are connected
         Your server process is waiting
```

during the session

```
You type SQL
     │
     ▼
Your dedicated server process receives it
     │
     ▼
Checks shared pool library cache → soft or hard parse
     │
     ▼
Looks for blocks in buffer cache
     │
     ├── Found → return data
     └── Not found → read from disk into buffer cache
```

when disconnected

```
You type EXIT
     │
     ├── Server process destroyed
     ├── PGA memory released back to OS
     └── PMON confirms cleanup if anything left behind
```


```
_"When I type sqlplus / as sysdba against a running database:_

_The Oracle Listener receives my connection request — LREG background process had already registered the instance with the listener at startup._

_Oracle spawns a dedicated server process just for my session. This process will handle every SQL statement I submit._

_PGA — Program Global Area — is allocated in RAM for my session. This private memory holds my session information, sort area for ORDER BY operations, and hash area for joins._

_When I type a SQL statement — my server process checks the shared pool library cache for a cached execution plan. If found — soft parse, reuse the plan. If not — hard parse, build a new plan._

_Data requests go to the buffer cache first. Cache hit — data served from RAM. Cache miss — DBWn reads from datafile into buffer cache, then serves it._

_When I disconnect — my server process is destroyed, PGA is released, and PMON cleans up anything left behind."_
```

9. "What is the purpose of the control file? What happens if you lose it? What does your database currently have and is that enough?"

what is in control file
```
Control file contains:
├── Database name
├── Database creation timestamp
├── Current SCN
├── Datafile names and locations
├── Redo log file names and locations
├── Checkpoint information
└── RMAN backup history
```

multiplexing 
```
Your lab:
control01.ctl   ─┐
                  ├── Oracle writes to BOTH simultaneously
control02.ctl   ─┘    always identical, always in sync
```

```
spfile         →   startup parameters (memory, file locations)
control file   →   runtime database state (SCN, file status)

spfile is read once at startup. Control file is read and written constantly during operation.
```

```
> _"The control file is Oracle's master map of the database. It contains the database name, creation timestamp, current SCN, locations and status of all datafiles and redo log files, checkpoint information, and RMAN backup history._
> 
> _Oracle reads the control file at every startup to find where the datafiles and redo logs are. It writes to the control file constantly during operation — every checkpoint updates it with the current SCN._
> 
> _If the control file is lost — the database cannot mount. Oracle cannot find its datafiles or redo logs without it. Recovery requires either a backup of the control file or recreating it manually using CREATE CONTROLFILE — a complex and risky operation._
> 
> _My database currently has two control files:_

```
/opt/oracle/oradata/ORCLCDB/control01.ctl
/opt/oracle/oradata/ORCLCDB/control02.ctl
```

> _Oracle writes to both simultaneously — this is multiplexing. If one gets corrupted, the other survives and Oracle can continue. Two copies is the minimum — production systems typically have three copies on separate disks or disk groups._
> 
> _Two copies is better than one — but both files are in the same directory on the same disk. If that disk fails, both are lost. In Phase 4 we should add a third control file on a separate location."_
```

Both on the same path. Same disk. Same failure domain. True multiplexing means **different disks**. Something to fix in Phase 3.

10. > **"A junior DBA calls you at 2 AM. The database is down. He says — 'I see the datafiles are all there on disk but the instance will not start. The alert log shows: ORA-00205: error in identifying control file'.**
> 
> **Walk me through exactly how you would diagnose and approach this problem. What are the possible causes and what would you check first?"**

- alter log
```
$ORACLE_BASE/diag/rdbms/orclcdb/ORCLCDB/trace/alert_ORCLCDB.log
SELECT value FROM v$diag_info WHERE name = 'Diag Trace';
```

- check spfile for control file location 
```
SELECT value FROM v$parameter WHERE name = 'control_files';
```

- recovery command
```
# OS level — check if files exist
ls -lh /opt/oracle/oradata/ORCLCDB/control*.ctl

# If control01 is corrupt but control02 is intact
cp control02.ctl control01.ctl

# Then startup
sqlplus / as sysdba
startup
```

**Missing 4 — If both control files are lost**
```
Option 1 — Restore from RMAN backup
RMAN> restore controlfile from autobackup;
RMAN> alter database mount;
RMAN> recover database;

Option 2 — Recreate manually
ALTER DATABASE BACKUP CONTROLFILE TO TRACE;
-- generates a CREATE CONTROLFILE script
-- last resort, complex, risky
```


```
Step 1 — Check the alert log immediately:
tail -100 $ORACLE_BASE/diag/rdbms/orclcdb/ORCLCDB/trace/alert_ORCLCDB.log
_ORA-00205 tells me Oracle cannot find or read the control file. The alert log will show the exact file path it tried and what went wrong._

_Step 2 — Check spfile to confirm where Oracle expects control files:_
SELECT value FROM v$parameter WHERE name = 'control_files';

_Sometimes a misconfigured spfile points to wrong locations — especially after a storage migration or a cloning activity._

_Step 3 — Verify physical existence on OS:_
ls -lh /opt/oracle/oradata/ORCLCDB/control*.ctl

Both files missing    →  storage failure or accidental deletion
One file missing      →  copy surviving file over missing one
Both files present    →  corruption — check file size and permissions

Step 4 — If one control file is intact:
cp control02.ctl control01.ctl ``` > *Then startup the instance — Oracle will validate both copies.* > > *Step 5 — If both control files are lost:* ``` RMAN> restore controlfile from autobackup; RMAN> alter database mount; RMAN> recover database; RMAN> alter database open resetlogs; ``` > *If no RMAN backup exists — recreate using CREATE CONTROLFILE script from a previous backup controlfile trace. This is last resort and requires knowing all datafile and redo log locations exactly.* > > *Root cause after recovery — add a third control file on a separate disk so this never happens again."*
```


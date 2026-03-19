```
1. Datafiles        (.dbf)   ← actual data lives here
2. Redo log files   (.log)   ← every change recorded here
3. Control file     (.ctl)   ← database map and SCN tracker
4. Parameter file   (spfile) ← startup configuration
```

1. datafiles(.dbf)
```sql
SQL> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
/opt/oracle/oradata/ORCLCDB/system01.dbf
/opt/oracle/oradata/ORCLCDB/pdbseed/system01.dbf
/opt/oracle/oradata/ORCLCDB/sysaux01.dbf
/opt/oracle/oradata/ORCLCDB/pdbseed/sysaux01.dbf
/opt/oracle/oradata/ORCLCDB/users01.dbf
/opt/oracle/oradata/ORCLCDB/pdbseed/undotbs01.dbf
/opt/oracle/oradata/ORCLCDB/undotbs01.dbf
/opt/oracle/oradata/ORCLCDB/ORCLPDB1/system01.dbf
/opt/oracle/oradata/ORCLCDB/ORCLPDB1/sysaux01.dbf
/opt/oracle/oradata/ORCLCDB/ORCLPDB1/undotbs01.dbf
/opt/oracle/oradata/ORCLCDB/ORCLPDB1/users01.dbf

11 rows selected.

SQL> 
```
**One job:** Store everything — tables, indexes, undo data, data dictionary. All user data ultimately lives in a datafile.

**Key behaviour:** Oracle never writes directly to datafiles during normal operation. Data goes to buffer cache first. DBW0 writes to datafiles later in batches.

2. redo log files (.log)
```sql
SQL> run
  1* select group#,member from v$logfile

GROUP# MEMBER
------ ----------------------------------------
     3 /opt/oracle/oradata/ORCLCDB/redo03.log
     2 /opt/oracle/oradata/ORCLCDB/redo02.log
     1 /opt/oracle/oradata/ORCLCDB/redo01.log

SQL> 

```

**One job:** Record every single change made to the database — before the change hits the datafile.

**Key behaviour:** Oracle writes to redo logs in a circular fashion:
```
redo01.log → fills up → switch to redo02.log
redo02.log → fills up → switch to redo03.log
redo03.log → fills up → switch back to redo01.log (overwrite)
```

This circular overwriting is exactly why your NOARCHIVELOG mode is a gap. When redo01.log gets overwritten — that history is gone forever.

3. control file (.ctl)
```sql
SQL> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/opt/oracle/oradata/ORCLCDB/control01.ctl
/opt/oracle/oradata/ORCLCDB/control02.ctl

```

**One job:** The master map of your database. Contains:
```
Database name
Database creation date
Current SCN
Datafile locations and status
Redo log file locations
Checkpoint information
RMAN backup history
```
**Key behaviour:** Oracle writes to BOTH control files simultaneously — every single time. They are always identical. This is called **multiplexing** — if one gets corrupted, the other survives.

4. parameter files (spfile)
**One job:** Tell Oracle how to start up.
```
memory_target        = how much RAM to use
db_name              = ORCLCDB
control_files        = where control files are
processes            = max concurrent connections
```
two versions exist
```
pfile    → plain text → you edit with vi
           called init.ora
           Oracle reads once at startup

spfile   → binary    → you change with ALTER SYSTEM
           called spfileORCLCDB.ora
           Oracle reads and writes dynamically
```

Always use spfile in production. Changes survive restart automatically.
```sql
SQL> run
  1  select name, value
  2  from v$parameter
  3* where name = 'spfile'

NAME	VALUE
------- ------------------------------------------------------------
spfile	/opt/oracle/product/26ai/dbhome_1/dbs/spfileORCLCDB.ora

SQL> 

```

```sql
SQL> run
  1  SELECT name, value
  2  FROM   v$parameter
  3  WHERE  name IN (
  4	    'db_name',
  5	    'memory_target',
  6	    'sga_target',
  7	    'pga_aggregate_target',
  8	    'db_block_size',
  9	    'control_files',
 10	    'log_archive_dest_1')
 11* ORDER BY name

NAME			       VALUE
------------------------------ --------------------------------------------------------------------------------
control_files		       /opt/oracle/oradata/ORCLCDB/control01.ctl, /opt/oracle/oradata/ORCLCDB/control02
			       .ctl

db_block_size		       8192
db_name 		       ORCLCDB
log_archive_dest_1
memory_target		       0
pga_aggregate_target	       2472542208
sga_target		       7415529472

7 rows selected.

SQL> 

```

decoded line by line:

spfile location
```
/opt/oracle/product/26ai/dbhome_1/dbs/spfileORCLCDB.ora
```

oracle is using spfile - not pfile, this means any alter system changes you make are persist across restarts automatically.

control files:
```
/opt/oracle/oradata/ORCLCDB/control01.ctl,
/opt/oracle/oradata/ORCLCDB/control02.ctl
```
two control files, both locations recorded in spfile. oracle writes to both simultaneously every time, multiplexed and safe

db_name:
```
ORCLCDB
```
exactly matches the instance name. In CDB this is the container database name.

db_block_size
```
8192
```
this is 8kb per block, every datafile is divided into 8kb chunks called blocks.
```
One block = 8192 bytes = 8 KB
Buffer cache holds blocks = 5856 MB / 8 KB = ~750,000 blocks in RAM
```
This value is set at database creation and **can never be changed.** Ever. Without recreating the database from scratch.

db_block_size 
```
OLTP systems     → 8 KB blocks  ← your setting, correct
Data warehouses  → 16 KB or 32 KB blocks
```
sga_target
```
7415529472 bytes = 7074 MB = ~6.9 GB
```
Oracle is managing SGA automatically within this limit. Individual SGA components — buffer cache, shared pool — resize themselves dynamically within this 6.9 GB envelope.

pga_aggregate_target
```
2472542208 bytes = 2358 MB = ~2.3 GB
```
Exactly matches what you saw in `v$pgastat` earlier. Oracle manages individual session PGA automatically within this 2.3 GB budget.

log_archive_dest_1
empty
no archive log destination configured. this confirms your noarchivelog mode . when redo logs fill up and switch - old logs are simply overwritten. 
```
Database name        ORCLCDB
Block size           8 KB (fixed forever)
SGA budget           6.9 GB  (auto-managed within this)
PGA budget           2.3 GB  (auto-managed within this)
Control files        2 copies (multiplexed ✅)
Archive logging      OFF ⚠️  (fix in Phase 4)
Parameter file       spfile ✅
```

Important observation:
```
memory_target     = 0        (AMM disabled)
sga_target        = 6.9 GB   (ASMM enabled)
pga_aggregate_target = 2.3 GB
```

oracle has two memory management modes:
```
AMM  — Automatic Memory Management
       memory_target > 0
       Oracle manages BOTH SGA and PGA together
       One pool for everything

ASMM — Automatic Shared Memory Management
       sga_target > 0, memory_target = 0
       Oracle manages SGA components automatically
       PGA managed separately
       More control, preferred in production
```
Your database uses **ASMM** — the preferred production approach. Oracle 26ai on Linux actually recommends ASMM over AMM for performance reasons.

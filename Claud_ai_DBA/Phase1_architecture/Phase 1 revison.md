Its always the instance that goes through the cycle never the database.
The database has no states, it just exists on disk.
The instance moves through states - Nomount - Mount - Open
The database files are passive. They sit on disk whether the instance is running or not. It is always the instance that transitions, that reads files, that gains awareness, that opens access.
```
You start an instance, The instance then finds, mounts, and opens the database.
```

Instance = Memory(SGA) + Processes - it lives in RAM, it's alive or dead 
Database = Files on disk (datafiles, redo logs, control files) - they always exist on disk

```
Becuase they are two separate things, they can be in different states independently
```

The complete cycle - Instance always

```
SHUTDOWN  →  NOMOUNT  →  MOUNT  →  OPEN
```

Shutdown - state where nothing exists in memory.
no SGA, no Background processes

<<<<<<< HEAD
Read write/ Read only:
These are actually modes within OPEN 

```sql
ALTER DATABASE OPEN; -- defaults to READ WRITE
ALTER DATABASE OPEN READ ONLY; -- useful for standby/reporting
```

--------------------------------------
what happens at each stage

SHUTDOWN -> NOMOUNT
```sql
startup nomount
```
oracle reads parameter file (SPFILE or PFILE) which contains
- how much memory to allocate for the SGA
- which background processes to start
- what the db_name is 
- where to find the control file
=======
Read write/ Read only - these are actually modes within OPEN
```sql
alter database open;
alter database open read only;
```

Each state transitions does one specific task:
1. shutdown -> nomount
`STARTUP NOMOUNT` oracle reads the parameter file (SPFILE or PFILE)
why? because oracle needs to know:
- How much memory to allocate for the SGA
- Which background processes to start
- what the `db_name` is
- where to find the control file

>>>>>>> origin/main
```sql
STARTUP NOMOUNT
-- Oracle reads: SPFILE (or PFILE)
-- Oracle does:  Allocates SGA, starts BGP (SMON, PMON, DBWn, LGWR...)
-- Oracle knows: Nothing about actual database files yet
```

<<<<<<< HEAD

NOMOUNT -> MOUNT
This where the control file enters
Oracle reads control file location from the parameter file and opens it to learns about:
- where all datafiles are
- where redo log files are
- the database structure and history

=======
2. Nomount -> Mount
This is where the control file enters the picture
oracle reads the control file location from the parameter file, opens it and learns:
- where all datafiles are
- where redo logfiles are
- the database structure and history

Memory Aid 🧠

>>>>>>> origin/main
| Transition         | File Read            | What Oracle gains              |
| ------------------ | -------------------- | ------------------------------ |
| SHUTDOWN → NOMOUNT | **Parameter file**   | Memory + Processes             |
| NOMOUNT → MOUNT    | **Control file**     | Database structure             |
| MOUNT → OPEN       | **Datafile headers** | Consistency check, then access |
<<<<<<< HEAD

-----------------------------------------------------------------------
when we use NOMOUNT:

creating a new database:
when you run `create database` there is no control file yet , so you can't mount anything.
The instance starts at NOMOUNT and the `create database` command itself creates the control file, redo logs and system datafiles from scratch.

The Key Insight:
SPFILE does not live inside the database, it lives on the OS filesystem - completely independent of any database file.

=======
When NOMOUNT is Used Intentionally:
1. creating a new database
when we run `create database` there is no control file yet - so we can't mount anything. 
the instance starts at nomount and the `create database` command itself creates the control file, redo logs, and system datafiles from scratch.
```sql
startup nomount;
create database orclcdb...;
```

2. restoring / recreating the control file
when the control file is lost or corrupt , you start at NOMOUNT because there is nothing to mount.
- restore control file from RMAN backup
- recreate control file with `create controlfile` command
```sql
STARTUP NOMOUNT;
RESTORE CONTROLFILE FROM '/backup/cf.bkp';
ALTER DATABASE MOUNT;   -- now it can mount
```

## MOUNT State Operations 

| Operation                             | State Required | Why                                                                   |
| ------------------------------------- | -------------- | --------------------------------------------------------------------- |
| Renaming datafiles / redo logs        | **MOUNT** ✅    | Control file must be open, but datafiles must be offline              |
| Enabling/Disabling ARCHIVELOG mode    | **MOUNT** ✅    | Oracle writes this change to the control file — no user access needed |
| **Full database recovery (complete)** | **MOUNT** ✅    | Database must be closed to users during full recovery                 |
| Querying user tables                  | **OPEN** only  | Datafiles must be online and accessible                               |

-------------
The scenario: Creating a New database in nomount

when we start an instance with `startup nomount` oracle does the following:
1. reads the parameter file(spfile or pfile)
2. allocates memory(sga/pga based on parameters)
3. starts background processes
that's it , no control file is read, no datafiles are opened .
the instance is alive but knows nothing about any database yet

oracle looks for the parameter file in a specific search order, automatically from a default location
```sql
SHOW PARAMETER spfile;
SQL> sho parameter spfile;

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
spfile				     string	 /opt/oracle/product/26ai/dbhom
						 e_1/dbs/spfileORCLCDB.ora

SELECT name, value FROM v$parameter WHERE name = 'spfile';
NAME	   VALUE
---------- --------------------------------------------------
spfile	   /opt/oracle/product/26ai/dbhome_1/dbs/spfileORCLCD
	   B.ora
	   
```

If there are no database yet :
spfile lives inside the os
>>>>>>> origin/main
```
Parameter File (PFILE/SPFILE)
        ↓
   Instance starts (NOMOUNT)
        ↓
   CREATE DATABASE runs
        ↓
   Control file is created
        ↓
   Datafiles are created
        ↓
   Database exists
```

<<<<<<< HEAD
where does the parameter file comes from on a brand new system:
scenario A: 
you(or DBCA) create it manually first before you ever type `startup nomount` some one creates a plain text `init.ora` (pfile) with the bare minimum parameters - `db_name`, memory setting, file locations. that file is placed in `$ORACLE_HOME/dbs/` .Then oracle starts from it.

scenario B:
DBCA does it silently for you, when most people install oracle and use DBCA (database configuration assistant) DBCA writes the PFILE/SPFILE before it ever issues `startup nomount` . 

```sql
ls -lh $ORACLE_HOME/dbs/

[oracle@ol9ai ~]$ ls -lh $ORACLE_HOME/dbs/
total 24K
-rw-rw----. 1 oracle oinstall 1.6K Mar 24 10:19 hc_ORCLCDB.dat
=======
where does the parameter file comes for a brand new system
- scenarioA
you (or DBCA) create it manually first before you ever type `startup nomount` someone creates plain text `init.ora` (pfile) with the bare minimum parameters - `db_name`, memory settings, file locations.
the file is placed in `$ORACLE_HOME/dbs` then oracle starts from it.
- scenarioB
DBCA does it silently for you, when most people install oracle and use DBCA (database configuration assistant) , DBCA writes the PFILE/SPFILE before it ever issues `startup nomount` 

```bash
[oracle@ol9ai ~]$ ls -lh $ORACLE_HOME/dbs
total 24K
-rw-rw----. 1 oracle oinstall 1.6K Mar 26 09:25 hc_ORCLCDB.dat
>>>>>>> origin/main
-rw-r-----. 1 oracle oinstall 3.1K May 14  2015 init.ora
-rw-r--r--. 1 oracle oinstall 1.1K Mar  3 08:16 initORCLCDB.ora
-rw-r-----. 1 oracle oinstall   24 Mar  2 20:26 lkORCLCDB
-rw-r-----. 1 oracle oinstall 2.0K Mar  2 20:30 orapwORCLCDB
<<<<<<< HEAD
-rw-r-----. 1 oracle oinstall 3.5K Mar 21 19:11 spfileORCLCDB.ora
[oracle@ol9ai ~]$
```

|File|What it is|
|---|---|
|`init.ora`|Generic **template** PFILE — ships with Oracle software. dated **2015** — never touched, just a sample|
|`initORCLCDB.ora`|The actual **PFILE** for your ORCLCDB — created during DBCA setup (Mar 3)|
|`spfileORCLCDB.ora`|The **SPFILE** — binary, the live parameter file Oracle actually runs from (Mar 21)|
|`lkORCLCDB`|Instance **lock file** — prevents two instances using same db_name|
|`orapwORCLCDB`|**Password file** — stores SYSDBA/SYSOPER credentials|
|`hc_ORCLCDB.dat`|**Health check** data file — used by OEM/clusterware|
=======
-rw-r-----. 1 oracle oinstall 3.5K Mar 26 09:26 spfileORCLCDB.ora
[oracle@ol9ai ~]$ echo $ORACLE_HOME
/opt/oracle/product/26ai/dbhome_1
[oracle@ol9ai ~]$ echo $ORACLE_SID
ORCLCDB
[oracle@ol9ai ~]$ echo $ORACLE_BASE
/opt/oracle
[oracle@ol9ai ~]$

```

## Breaking Down What You See

|File|What it is|
|---|---|
|`init.ora`|Generic **template** PFILE — ships with Oracle software. dated **2015** — never touched, just a sample|
|`initORCLCDB.ora`|The actual **PFILE** for your ORCLCDB — created during DBCA setup (Mar 3)|
|`spfileORCLCDB.ora`|The **SPFILE** — binary, the live parameter file Oracle actually runs from (Mar 21)|
|`lkORCLCDB`|Instance **lock file** — prevents two instances using same db_name|
|`orapwORCLCDB`|**Password file** — stores SYSDBA/SYSOPER credentials|
|`hc_ORCLCDB.dat`|**Health check** data file — used by OEM/clusterware|

>>>>>>> origin/main
```
Mar 2  — lkORCLCDB, orapwORCLCDB     ← DBCA created the instance
Mar 3  — initORCLCDB.ora              ← PFILE written first
Mar 21 — spfileORCLCDB.ora            ← SPFILE created from PFILE later
```

<<<<<<< HEAD
DBCA created `initORCLCDB.ora` (the PFILE) **first** — on the filesystem — _before_ issuing `STARTUP NOMOUNT`. That's how the instance knew how to start with no database yet existing.

```bash
[oracle@ol9ai ~]$ cat $ORACLE_HOME/dbs/initORCLCDB.ora
=======
DBCA created `initORCLCDB.ora` (the PFILE) **first** — on the filesystem — _before_ issuing `STARTUP NOMOUNT`. That's how the instance knew how to start with no database yet existing.

```bash
[oracle@ol9ai ~]$ pwd
/home/oracle
[oracle@ol9ai ~]$ cd $ORACLE_HOME/dbs
[oracle@ol9ai dbs]$ cat initORCLCDB.ora
>>>>>>> origin/main
ORCLCDB.__data_transfer_cache_size=0
ORCLCDB.__datamemory_area_size=0
ORCLCDB.__db_cache_size=6006243328
ORCLCDB.__inmemory_ext_roarea=0
ORCLCDB.__inmemory_ext_rwarea=0
ORCLCDB.__java_pool_size=0
ORCLCDB.__large_pool_size=50331648
ORCLCDB.__oracle_base='/opt/oracle'#ORACLE_BASE set from environment
ORCLCDB.__pga_aggregate_target=2483027968
ORCLCDB.__sga_target=7415529472
ORCLCDB.__shared_io_pool_size=134217728
ORCLCDB.__shared_pool_size=1191182336
ORCLCDB.__streams_pool_size=0
ORCLCDB.__unified_pga_pool_size=0
ORCLCDB._instance_recovery_bloom_filter_size=1048576
*.compatible='23.6.0'
*.control_files='/opt/oracle/oradata/ORCLCDB/control01.ctl','/opt/oracle/oradata/ORCLCDB/control02.ctl'
*.db_block_size=8192
*.db_name='ORCLCDB'
*.diagnostic_dest='/opt/oracle'
*.dispatchers='(PROTOCOL=TCP) (SERVICE=ORCLCDBXDB)'
*.enable_pluggable_database=true
*.local_listener='LISTENER_ORCLCDB'
*.nls_language='AMERICAN'
*.nls_territory='AMERICA'
*.open_cursors=300
*.pga_aggregate_target=2358m
*.processes=480
*.remote_login_passwordfile='EXCLUSIVE'
*.sga_target=7072m
*.undo_tablespace='UNDOTBS1'
<<<<<<< HEAD
[oracle@ol9ai ~]$
```

full sequence of brand new database:
```
1. Create PFILE manually (initNEWDB.ora)
   — must include: db_name, memory, control_files location
        ↓
2. STARTUP NOMOUNT
   — reads PFILE, starts instance, no DB yet
        ↓
3. CREATE DATABASE
   — creates control files at control_files path
   — creates SYSTEM, SYSAUX, UNDO datafiles
   — creates redo log groups
   — database now exists
        ↓
4. STARTUP (subsequent restarts)
   — reads PFILE/SPFILE → finds control_files → MOUNT → OPEN
```

SPFILE is a **binary file** that Oracle itself writes to directly. Any `ALTER SYSTEM SET parameter=value` change is **persisted automatically** — survives restarts. No human needs to manually edit a file.
With a PFILE — you change a parameter with `ALTER SYSTEM`, instance picks it up — but **next restart it's gone** unless you manually edited the text file yourself.

Real scenario:

```sql
ALTER SYSTEM SET memory_target=999G SCOPE=SPFILE;
-- DBA made a typo. Committed to SPFILE.
-- Instance is shut down for maintenance.
-- STARTUP fails. Oracle cannot allocate 999G.
-- SPFILE is binary. You cannot edit it.
```
Recovery steps:
1. create PFILE from the corrupted SPFILE (if instance never started, do it form OS copy) or
   edit the existing initORCLCDB.ora directly in vi - fix the bad parameter
2. start using PFILE explicitly
```sql
STARTUP PFILE='/opt/oracle/product/26.0.0/dbhome_1/dbs/initORCLCDB.ora';
```
3. verify database is healthy
```sql
select status from v$instance;
```
4. recreate clean SPFILE from the working PFILE
```sql
create spfile from pfile;
```
5. restart to pick up the new spfile
```sql
shutdown immediate;
startup;
```

- we have to explicitly tell the path to use the pfile or it will use the corrupt spfile and fails again 
- The search order oracle uses at startup
1. spfile<sid>.ora 
   2. spfile .ora
   3. init<sid>.ora -- pfile
all in `$ORACLE_HOME/dbs` 



=======
[oracle@ol9ai dbs]$
```

```bash
*.control_files='/opt/oracle/oradata/ORCLCDB/control01.ctl','/opt/oracle/oradata/ORCLCDB/control02.ctl'
```
- On a **new database** → `CREATE DATABASE` reads this parameter and **creates** the control files at those paths
- On an **existing database** → `STARTUP MOUNT` reads this parameter and **opens** the already existing control files

Question:
You are building a brand new database from scratch. You write a minimal PFILE. You start with `STARTUP NOMOUNT`. You run `CREATE DATABASE`.

**What happens if you forget to put `control_files` in your PFILE?**
`NOMOUNT → MOUNT` failing makes sense for an **existing** database. But we're talking about `CREATE DATABASE` on a **brand new** system.

So — does `CREATE DATABASE` **fail**? Or does Oracle do something else?
>>>>>>> origin/main

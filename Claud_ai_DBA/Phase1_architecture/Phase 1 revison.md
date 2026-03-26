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

```sql
STARTUP NOMOUNT
-- Oracle reads: SPFILE (or PFILE)
-- Oracle does:  Allocates SGA, starts BGP (SMON, PMON, DBWn, LGWR...)
-- Oracle knows: Nothing about actual database files yet
```

2. Nomount -> Mount
This is where the control file enters the picture
oracle reads the control file location from the parameter file, opens it and learns:
- where all datafiles are
- where redo logfiles are
- the database structure and history

Memory Aid 🧠

| Transition         | File Read            | What Oracle gains              |
| ------------------ | -------------------- | ------------------------------ |
| SHUTDOWN → NOMOUNT | **Parameter file**   | Memory + Processes             |
| NOMOUNT → MOUNT    | **Control file**     | Database structure             |
| MOUNT → OPEN       | **Datafile headers** | Consistency check, then access |
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
-rw-r-----. 1 oracle oinstall 3.1K May 14  2015 init.ora
-rw-r--r--. 1 oracle oinstall 1.1K Mar  3 08:16 initORCLCDB.ora
-rw-r-----. 1 oracle oinstall   24 Mar  2 20:26 lkORCLCDB
-rw-r-----. 1 oracle oinstall 2.0K Mar  2 20:30 orapwORCLCDB
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

```
Mar 2  — lkORCLCDB, orapwORCLCDB     ← DBCA created the instance
Mar 3  — initORCLCDB.ora              ← PFILE written first
Mar 21 — spfileORCLCDB.ora            ← SPFILE created from PFILE later
```

DBCA created `initORCLCDB.ora` (the PFILE) **first** — on the filesystem — _before_ issuing `STARTUP NOMOUNT`. That's how the instance knew how to start with no database yet existing.

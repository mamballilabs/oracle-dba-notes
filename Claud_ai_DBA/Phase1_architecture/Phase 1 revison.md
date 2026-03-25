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

** oracle is two different things running together 
 1. Database = files sitting on disk
 2. Instance = running set of memory structures and background processes

Instance mounts and opens the database.
1. An instance running with no database mounted (used during restore)
2. A database sitting on disk with no instance running (shutdown state)
3. Multiple instances opening the same database simultaneously (RAC)

![[Pasted image 20260315220514.png]]
```sql
SQL> run
  1* select instance_name, status, database_status from v$instance

INSTANCE_NAME	 STATUS       DATABASE_STATUS
---------------- ------------ -----------------
ORCLCDB 	 OPEN	      ACTIVE


SQL> select name, db_unique_name, open_mode, log_mode from v$database;

NAME	  DB_UNIQUE_NAME		 OPEN_MODE	      LOG_MODE
--------- ------------------------------ -------------------- ------------
ORCLCDB   ORCLCDB			 READ WRITE	      NOARCHIVELOG

SQL> run
  1* select file#, name, status, bytes/1024/1024 mb from v$datafile

FILE# NAME							 STATUS 	    MB
----- ---------------------------------------------------------- ---------- ----------
    1 /opt/oracle/oradata/ORCLCDB/system01.dbf			 SYSTEM 	  1120
    2 /opt/oracle/oradata/ORCLCDB/pdbseed/system01.dbf		 SYSTEM 	   350
    3 /opt/oracle/oradata/ORCLCDB/sysaux01.dbf			 ONLINE 	   730
    4 /opt/oracle/oradata/ORCLCDB/pdbseed/sysaux01.dbf		 ONLINE 	   400
    7 /opt/oracle/oradata/ORCLCDB/users01.dbf			 ONLINE 	     7
    9 /opt/oracle/oradata/ORCLCDB/pdbseed/undotbs01.dbf 	 ONLINE 	   100
   11 /opt/oracle/oradata/ORCLCDB/undotbs01.dbf 		 ONLINE 	    40
   12 /opt/oracle/oradata/ORCLCDB/ORCLPDB1/system01.dbf 	 SYSTEM 	   350
   13 /opt/oracle/oradata/ORCLCDB/ORCLPDB1/sysaux01.dbf 	 ONLINE 	   460
   14 /opt/oracle/oradata/ORCLCDB/ORCLPDB1/undotbs01.dbf	 ONLINE 	   100
   15 /opt/oracle/oradata/ORCLCDB/ORCLPDB1/users01.dbf		 ONLINE 	     7

11 rows selected.

SQL> run
  1* select group#, member from v$logfile order by group#

GROUP# MEMBER
------ ----------------------------------------
     1 /opt/oracle/oradata/ORCLCDB/redo01.log
     2 /opt/oracle/oradata/ORCLCDB/redo02.log
     3 /opt/oracle/oradata/ORCLCDB/redo03.log

SQL> 

SQL> select name from v$controlfile;

NAME
----------------------------------------------------------
/opt/oracle/oradata/ORCLCDB/control01.ctl
/opt/oracle/oradata/ORCLCDB/control02.ctl

SQL> 
SQL> run
  1* select name, value/1024/1024 MB from v$pgastat where name IN('aggregate PGA target parameter','total PGA allocated')

NAME								   MB
---------------------------------------------------------- ----------
aggregate PGA target parameter					 2358
total PGA allocated					   502.427734
aggregate PGA target parameter					    0
total PGA allocated					   .071289063

SQL> 


SQL> run
  1* select pool, sum(bytes)/1024/1024 MB from v$sgastat group by pool order by MB desc

POOL		       MB
-------------- ----------
	       5865.01591
shared pool	     1136
large pool	       48

SQL> 
```

**First major observation:** `LOG_MODE = NOARCHIVELOG` This means completed redo log files are **overwritten and lost forever.** You can only recover to the last full backup — not to a point in time.

CDB:

![[Pasted image 20260316084538.png]]

### Your Datafiles Decoded — File by File

| File# | Path                    | What it stores                     | Size    |
| ----- | ----------------------- | ---------------------------------- | ------- |
| 1     | `ORCLCDB/system01.dbf`  | CDB$ROOT data dictionary           | 1120 MB |
| 2     | `pdbseed/system01.dbf`  | PDB$SEED template                  | 350 MB  |
| 3     | `ORCLCDB/sysaux01.dbf`  | CDB$ROOT AWR, EM, Spatial metadata | 730 MB  |
| 7     | `ORCLCDB/users01.dbf`   | Default user tablespace in ROOT    | 7 MB    |
| 9     | `pdbseed/undotbs01.dbf` | PDB$SEED undo                      | 100 MB  |
| 11    | `ORCLCDB/undotbs01.dbf` | CDB$ROOT undo — rollback data      | 40 MB   |
| 12–15 | `ORCLPDB1/`             | Your actual working PDB            | various |

> 💡 **Notice file numbers jump:** 1,2,3,4,7,9,11,12... gaps at 5,6,8,10. Those file numbers were used and dropped at some point during install. Oracle never reuses file numbers — each is permanently unique in the control file.

### Redo Logs — Only 3 Groups, No Multiplexing

```
GROUP#   MEMBER
1        redo01.log
2        redo02.log
3        redo03.log
```

Each group has only **one member**. That is a single point of failure. If `redo01.log` gets corrupted while it is the current log, you lose the database. In production you always have at least 2 members per group on separate disks. We will fix this in Phase 4.

### Your Memory — SGA and PGA Numbers

```
SGA total:  ~5865 MB  (~5.7 GB)
PGA target: ~2358 MB  (~2.3 GB)
Total Oracle memory footprint: ~8.2 GB on your KVM guest
```

> The row with `pool = (blank)` showing 5865 MB is the **fixed SGA + buffer cache** — the largest component. We will break this down completely in the next step.

```sql
SQL> run
  1* select con_id, name, open_mode, restricted from v$pdbs order by con_id

    CON_ID NAME 															    OPEN_MODE  RES
---------- -------------------------------------------------------------------------------------------------------------------------------- ---------- ---
	 2 PDB$SEED															    READ ONLY  NO
	 3 ORCLPDB1															    READ WRITE NO

SQL> 
SQL> run
  1* select sys_context('USERENV','CON_NAME') current_container, sys_context('USERENV','CON_ID') container_id from dual

CURRENT_CONTAIN CONTAINER_ID
--------------- ---------------
CDB$ROOT	1

SQL> 

SQL> run
  1* select round(sum(bytes)/1024/1024/1024,2) total_GB from v$datafile

  TOTAL_GB
----------
      3.59

SQL> 

```

test
test
You have CON_ID 1 and CON_ID 3 in your output. CON_ID 2 is missing. Where is PDB$SEED and why does it have no datafiles in this view?

PDB$SEED is a read-only template which helps to create PDBs
show via v$datafile but not on cdb_data_files -- because they are read-only .
```sql
CON_ID 1  CDB$ROOT    READ WRITE  → shows in cdb_data_files ✅
CON_ID 2  PDB$SEED    READ ONLY   → hidden from cdb_data_files ✅
CON_ID 3  ORCLPDB1    READ WRITE  → shows in cdb_data_files ✅
```

How PDB creation works
```sql
CREATE PLUGGABLE DATABASE orclpdb2
FROM PDB$SEED;
```


```
PDB$SEED (read-only template)
       │
       ▼
Copy all seed datafiles
       │
       ▼
New ORCLPDB2 datafiles created
       │
       ▼
New PDB is open and ready
```

```
┌─────────────────────────────────────────┐
│           INSTANCE (RAM)                │
│   One SGA · One set of bg processes     │
│   DBW0 · LGWR · CKPT · SMON · PMON     │
└─────────────────┬───────────────────────┘
                  │ mounts and opens
┌─────────────────▼───────────────────────┐
│         CDB — ORCLCDB (disk)            │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │ CDB$ROOT │  │ PDB$SEED │  │ORCLPDB│ │
│  │          │  │read-only │  │   1   │ │
│  │ shared   │  │template  │  │ your  │ │
│  │ dict     │  │          │  │ work  │ │
│  └──────────┘  └──────────┘  └───────┘ │
└─────────────────────────────────────────┘
```

what CDB$ROOT does 
```
CDB$ROOT owns:
├── Common users  (visible across all PDBs)
├── Common roles  (apply to all PDBs)
├── CDB-wide parameters
├── Shared redo logs  (all PDB changes go here)
├── Shared undo       (in 12c/19c — 26ai has local undo)
└── SYSDBA access     (controls everything)
```

Local undo:
```sql
CON_ID 1  CDB$ROOT   undotbs01.dbf   40 MB
CON_ID 3  ORCLPDB1   undotbs01.dbf  100 MB
```

This is **Local Undo mode** — introduced in Oracle 12c Release 2, made default in 18c onwards. Each PDB manages its own undo independently.

```
Old shared undo    →  one undo tablespace in ROOT for all PDBs
                      PDBs cannot be unplugged easily

Local undo         →  each PDB has its own undo tablespace
                      PDB can be unplugged and moved to another CDB
                      complete portability
                      
```

```sql
select sys_context('USERENV','CON_NAME') container,
sys_context('USERENV','CON_ID') con_id
from dual;

CONTAINER  CON_ID
---------- ----------
CDB$ROOT   1

SQL>

SQL> sho con_name

CON_NAME
------------------------------
CDB$ROOT
SQL>
```

```sql
SQL> sho pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 ORCLPDB1                       READ WRITE NO
SQL> alter session set container = ORCLPDB1;

Session altered.

SQL> sho con_name;

CON_NAME
------------------------------
ORCLPDB1
SQL>
```

tablespaces inside pdb
```sql
what tablespaces does this pdb have

SQL> run
  1  select tablespace_name, status, contents
  2  from dba_tablespaces
  3* order by tablespace_name

TABLESPACE_NAME                STATUS    CONTENTS
------------------------------ --------- ---------------------
SYSAUX                         ONLINE    PERMANENT
SYSTEM                         ONLINE    PERMANENT
TEMP                           ONLINE    TEMPORARY
UNDOTBS1                       ONLINE    UNDO
USERS                          ONLINE    PERMANENT

SQL>
```

```sql
what datafiles belong to this PDB

SQL> run
  1  select tablespace_name, file_name, round(bytes/1024/1024) MB
  2  from dba_data_files
  3* order by tablespace_name

TABLESPACE FILE_NAME                                        MB
---------- ---------------------------------------- ----------
SYSAUX     /opt/oracle/oradata/ORCLCDB/sysaux01.dbf        800
SYSTEM     /opt/oracle/oradata/ORCLCDB/system01.dbf       1120
UNDOTBS1   /opt/oracle/oradata/ORCLCDB/undotbs01.db         40
           f

USERS      /opt/oracle/oradata/ORCLCDB/users01.dbf           7

SQL>
```

```sql
SQL> alter session set container = CDB$ROOT;

Session altered.

SQL> sho con_name;

CON_NAME
------------------------------
CDB$ROOT
SQL>
```



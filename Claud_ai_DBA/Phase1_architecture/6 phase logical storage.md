```
TABLESPACE    →   the whole building
SEGMENT       →   one floor
EXTENT        →   one room on that floor
BLOCK         →   one desk in that room
```

Block:
smallest unit = 8kb
everything that oracle reads and writes moves in blocks
one block holds rows from exactly one table segment , oracle never mixes data from different objects fit in your buffer cache

Extent:
blocks are allocated in groups called extents 
```
Extent = a contiguous group of blocks
         all in the same datafile
         all belonging to the same segment
```
eg:
```
First extent allocated    →   8 blocks  (64 KB)
Table grows, needs more   →   8 blocks  (64 KB) more
Grows again               →  16 blocks (128 KB) more
```

segment:
every object that stores data gets its own segment 
```
One table        =   one table segment
One index        =   one index segment
Undo for ROOT    =   one undo segment
Temp sort space  =   one temporary segment
```
A segment is made up of one or more extents. As the object grows — more extents are added to its segment.
eg:
```
EMPLOYEES table segment
├── Extent 1   blocks 1-8     (first allocation)
├── Extent 2   blocks 9-16    (table grew)
├── Extent 3   blocks 33-48   (table grew again)
└── Extent 4   blocks 97-128  (table grew again)
```
Notice extents do not have to be contiguous — they just have to be in the same datafile or tablespace.

tablespace
this is the logical container that groups related segments together.
it maps directly to one or more datafiles
```
TABLESPACE          DATAFILE(S)
──────────────────  ──────────────────────────────────
SYSTEM              system01.dbf
SYSAUX              sysaux01.dbf
UNDOTBS1            undotbs01.dbf
USERS               users01.dbf
```
One tablespace can span multiple datafiles. One datafile belongs to exactly one tablespace.

```
USERS tablespace  (7 MB)
└── users01.dbf   (physical file on disk)
    └── Segments  (tables, indexes created in USERS)
        └── Extents (groups of blocks allocated per segment)
            └── Blocks (8 KB each, holds actual row data)
```

```sql
SQL> run
  1  select tablespace_name, status, contents, extent_management, segment_space_management
  2  from dba_tablespaces
  3* order by tablespace_name

TABLESPACE_NAME 	       STATUS	 CONTENTS	       EXTENT_MAN SEGMEN
------------------------------ --------- --------------------- ---------- ------
SYSAUX			       ONLINE	 PERMANENT	       LOCAL	  AUTO
SYSTEM			       ONLINE	 PERMANENT	       LOCAL	  MANUAL
TEMP			       ONLINE	 TEMPORARY	       LOCAL	  MANUAL
UNDOTBS1		       ONLINE	 UNDO		       LOCAL	  MANUAL
USERS			       ONLINE	 PERMANENT	       LOCAL	  AUTO

SQL> 

```

Five tablespace decoded:
1. system
most critical one
```sql
SYSTEM			       ONLINE	 PERMANENT	       LOCAL	  MANUAL
```

- stores the data dictionary
- every table defn, every user, every privilege, every stored procedure defn in the entire CDB
- If this tablespace is gone no recovery without a backup

2. sysaux
assistant to system
```sql
SYSAUX			       ONLINE	 PERMANENT	       LOCAL	  AUTO
```
- stores auxiliary database features - awr snapshots, optimizer statistics, enterprise manager data, spatial metadata, text indexes 
- offloads optional features from system to keep system lean and focused.

3. undotbs1
undo tablespace
```sql
UNDOTBS1		       ONLINE	 UNDO		       LOCAL	  MANUAL
```
- everytime you run update or delete oracle saves the old version of the data here before changing it
```
1. ROLLBACK      → you change your mind, Oracle restores old data
2. Read consistency → other sessions see old data while you are changing it
3. Flashback     → query data as it looked 10 minutes ago
```

4. temp
```sql
TEMP			       ONLINE	 TEMPORARY	       LOCAL	  MANUAL
```
- contents says temporary
- when a sort operation is too large to fit in PGA - oracle spills to temp tablespace on disk
```
Small sort   → fits in PGA memory     → fast
Large sort   → spills to TEMP         → slower but works
```
nothing in temp survives a restart

5. users
```sql
USERS			       ONLINE	 PERMANENT	       LOCAL	  AUTO
```
- where user created tables and indexes live by default
- In most of the production this is the largest tablespace.

four columns explained
Contents
```
PERMANENT    → stores regular data, survives restart
UNDO         → stores undo data only
TEMPORARY    → scratch space, wiped on restart
```

Extent_management
```
LOCAL    → extent allocation tracked in datafile header bitmaps
DICTIONARY → old method, tracked in SYSTEM tablespace (avoid this)
```

segment_space_management
```
AUTO      → Oracle manages free space automatically (bitmap tracking)
MANUAL    → older method using free lists
```

system and undo use manual - oracle managed.
users and sysaux use auto - preferred for user tablespace

Logical to Physical mapping
```sql
SELECT t.tablespace_name,
       t.contents,
       d.file_name,
       round(d.bytes/1024/1024) MB,
       d.autoextensible
FROM   dba_tablespaces t
JOIN   dba_data_files d
ON     t.tablespace_name = d.tablespace_name
ORDER BY t.tablespace_name;

TABLESPACE CONTENTS   FILE_NAME                                                       MB AUT
---------- ---------- ------------------------------------------------------------ ----- ---
SYSAUX     PERMANENT  /opt/oracle/oradata/ORCLCDB/sysaux01.dbf                       800 YES
SYSTEM     PERMANENT  /opt/oracle/oradata/ORCLCDB/system01.dbf                      1120 YES
UNDOTBS1   UNDO       /opt/oracle/oradata/ORCLCDB/undotbs01.dbf                       40 YES
USERS      PERMANENT  /opt/oracle/oradata/ORCLCDB/users01.dbf                          7 YES

SQL>
```

three things to notice
1. temp is missing 
dba_data_files only shows permanent and undo datafiles.
temporary tablespace has its own view
```sql
select * from dba_temp_files; 
```

2. autoextend = yes on everything
every single datafile will grow automatically when it runs out of space

3. only CDB root files here
to see PDBs 
```sql
SELECT con_id, tablespace_name, file_name, round(bytes/1024/1024) MB 
FROM cdb_data_files 
ORDER BY con_id, tablespace_name;

    CON_ID TABLESPACE FILE_NAME                                                       MB
---------- ---------- ------------------------------------------------------------ -----
         1 SYSAUX     /opt/oracle/oradata/ORCLCDB/sysaux01.dbf                       800
         1 SYSTEM     /opt/oracle/oradata/ORCLCDB/system01.dbf                      1120
         1 UNDOTBS1   /opt/oracle/oradata/ORCLCDB/undotbs01.dbf                       40
         1 USERS      /opt/oracle/oradata/ORCLCDB/users01.dbf                          7
         3 SYSAUX     /opt/oracle/oradata/ORCLCDB/ORCLPDB1/sysaux01.dbf              490
         3 SYSTEM     /opt/oracle/oradata/ORCLCDB/ORCLPDB1/system01.dbf              360
         3 UNDOTBS1   /opt/oracle/oradata/ORCLCDB/ORCLPDB1/undotbs01.dbf             100
         3 USERS      /opt/oracle/oradata/ORCLCDB/ORCLPDB1/users01.dbf                 7

8 rows selected.

SQL>
```

oracle has three versions of most data dictionary views:
```
dba_ → current container only (you are in CDB ROOT) 
cdb_ → all containers in the CDB (cross-container view) 
all_ → objects current user can access 
user_ → objects owned by current user
```

```
CON_ID = 1    →    CDB$ROOT
CON_ID = 2    →    PDB$SEED (read-only template, no datafiles shown)
CON_ID = 3    →    ORCLPDB1 (your pluggable database)
```

complete database - all files mapped
```sql
CON_ID 1 — CDB$ROOT
──────────────────────────────────────────────
SYSTEM      system01.dbf          1120 MB   ← CDB data dictionary
SYSAUX      sysaux01.dbf           800 MB   ← AWR, stats, features
UNDOTBS1    undotbs01.dbf           40 MB   ← ROOT undo
USERS       users01.dbf              7 MB   ← ROOT user data

CON_ID 3 — ORCLPDB1
──────────────────────────────────────────────
SYSTEM      ORCLPDB1/system01.dbf  360 MB   ← PDB data dictionary
SYSAUX      ORCLPDB1/sysaux01.dbf  490 MB   ← PDB AWR, stats
UNDOTBS1    ORCLPDB1/undotbs01.dbf 100 MB   ← PDB undo
USERS       ORCLPDB1/users01.dbf     7 MB   ← PDB user data
```

same structure twice
```sql
CDB$ROOT has:    SYSTEM, SYSAUX, UNDOTBS1, USERS
ORCLPDB1 has:    SYSTEM, SYSAUX, UNDOTBS1, USERS
```
Every PDB is a **complete, self-contained database** with its own tablespaces, its own data dictionary, its own undo. This is the beauty of Multitenant architecture — each PDB is isolated from the others.

dba_ vs cdb_
```
dba_data_files    →    4 rows    (ROOT only)
cdb_data_files    →    8 rows    (ROOT + all PDBs)
```

takeaway
```
✅ Block      → 8 KB, smallest unit, held in buffer cache
✅ Extent     → contiguous group of blocks, allocated together
✅ Segment    → one database object (table, index, undo)
✅ Tablespace → logical container mapping to datafiles
```
```
✅ dba_  → current container
✅ cdb_  → all containers
✅ all_  → accessible objects
✅ user_ → owned objects
```


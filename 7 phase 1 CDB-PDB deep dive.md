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


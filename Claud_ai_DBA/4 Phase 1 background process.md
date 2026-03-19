```sql
SQL> run
  1  select name, description from v$bgprocess
  2  where paddr <> '00'
  3* order by name

NAME  DESCRIPTION
----- ----------------------------------------------------------------
AQPC  AQ Process Coord
BG00  Testing SRV process2
BG00  Testing SRV process2
BG00  Testing SRV process2
BG00  Testing SRV process2
BG00  Testing SRV process2
BG00  Testing SRV process2
BG00  Testing SRV process2
BG00  Testing SRV process2
BG00  Testing SRV process2
BG01  Testing SRV process2

NAME  DESCRIPTION
----- ----------------------------------------------------------------
BG01  Testing SRV process2
BG01  Testing SRV process2
BG01  Testing SRV process2
BG01  Testing SRV process2
BG01  Testing SRV process2
BG01  Testing SRV process2
BG01  Testing SRV process2
BG02  KSBSRV generic threads
BG02  KSBSRV generic threads
BG02  KSBSRV generic threads
BG02  KSBSRV generic threads

NAME  DESCRIPTION
----- ----------------------------------------------------------------
BG02  KSBSRV generic threads
BG02  KSBSRV generic threads
BG03  Testing SRV process2
BG03  Testing SRV process2
BG03  Testing SRV process2
BG03  Testing SRV process2
BG03  Testing SRV process2
BG03  Testing SRV process2
CJQ0  Job Queue Coordinator
CKPT  checkpoint
CLMN  process cleanup

NAME  DESCRIPTION
----- ----------------------------------------------------------------
D000  Dispatchers
DBRM  DataBase Resource Manager
DBW0  db writer process 0
DIA0  diagnosibility process 0
DIAG  diagnosibility process
DT00  KSTRC Archive Worker Pool
DT01  KSTRC Archive Worker Pool
FENC  IOServer fence monitor
GCR0  GCR Helpers (LMHB)
GCR1  GCR Helpers (LMHB)
GCW0  GCR Monitor processes (LMHB)

NAME  DESCRIPTION
----- ----------------------------------------------------------------
GEN0  generic0
GEN2  generic2
GWPD  Oracle Working Process Pool BG
IR00  Text Maintenance Slave
IR01  Text Maintenance Slave
J000  Job queue slaves
J001  Job queue slaves
J002  Job queue slaves
J003  Job queue slaves
J004  Job queue slaves
J005  Job queue slaves

NAME  DESCRIPTION
----- ----------------------------------------------------------------
J006  Job queue slaves
J007  Job queue slaves
J008  Job queue slaves
J009  Job queue slaves
LG00  Log Writer Slave
LG01  Log Writer Slave
LGWR  Redo etc.
LMHB  lm heartbeat monitor
LREG  Listener Registration
M000  MMON slave class 1
M001  MMON slave class 1

NAME  DESCRIPTION
----- ----------------------------------------------------------------
M002  MMON slave class 1
M003  MMON slave class 1
M004  MMON slave class 1
M005  MMON slave class 1
M006  MMON slave class 1
MMAN  Memory Manager
MMNL  Manageability Monitor Process 2
MMON  Manageability Monitor Process
NMON  NMON Service
OFSD  Oracle File Server BG
P000  Parallel query slave

NAME  DESCRIPTION
----- ----------------------------------------------------------------
P001  Parallel query slave
P002  Parallel query slave
P003  Parallel query slave
P004  Parallel query slave
P005  Parallel query slave
P006  Parallel query slave
P007  Parallel query slave
P008  Parallel query slave
P009  Parallel query slave
P00A  Parallel query slave
P00B  Parallel query slave

NAME  DESCRIPTION
----- ----------------------------------------------------------------
PMAN  process manager
PMON  process cleanup
PSP0  process spawner 0
PXMN  PX Monitor
Q001  QMON MS
Q003  QMON MS
Q004  QMON MS
Q005  QMON MS
Q006  QMON MS
QM02  QMON MS
QM03  QMON MS

NAME  DESCRIPTION
----- ----------------------------------------------------------------
RCBG  Result Cache: Background
RECO  distributed recovery
S000  Shared servers
S001  Shared servers
SCMN
SCMN
SCMN
SCMN
SCMN
SCMN
SMCO  Space Manager Process

NAME  DESCRIPTION
----- ----------------------------------------------------------------
SMON  System Monitor Process
SVCB  services background monitor
TMON  Transport Monitor
TT00  Redo Transport
TT01  Redo Transport
VKRM  Virtual sKeduler for Resource Manager
VKTM  Virtual Keeper of TiMe process
W000  space management slave pool
W001  space management slave pool
W002  space management slave pool
W003  space management slave pool

121 rows selected.

SQL> 

```

```sql
SQL> SELECT name, description
FROM   v$bgprocess
WHERE  paddr <> '00'
AND    name IN ('DBW0','LGWR','CKPT','SMON','PMON')
ORDER BY name;  2    3    4    5  

NAME  DESCRIPTION
----- ----------------------------------------------------------------
CKPT  checkpoint
DBW0  db writer process 0
LGWR  Redo etc.
PMON  process cleanup
SMON  System Monitor Process

SQL> 

```

Important 5 JOBS:
1. DBW0 - database writer
```sql
NAME    DBW0
JOB     db writer process 0
```
write dirty buffers from the database buffer cache to data-files on disk
```
Buffer Cache
┌─────────────────────────┐
│  dirty block (modified) │──── DBW0 writes ────► datafile on disk
│  dirty block (modified) │
│  clean block (unmodified│
└─────────────────────────┘
```
- does not write immediately on every change, it writes in batches - when the cache gets full or when a checkpoint happens
- If DBW0 dies instance crashes immediately, no process to write data to disk 

2. LGWR - log writer
```sql
NAME    LGWR
JOB     Redo etc.
```
write redo entries from the redo log buffer to the online redo log file on disk.
```
Redo Log Buffer (8 MB RAM)
┌─────────────────────────┐
│  change recorded here   │──── LGWR flushes ────► redo01.log on disk
│  change recorded here   │
└─────────────────────────┘
```

**Critical rule:** When you issue a COMMIT — Oracle does NOT return success until LGWR has hardened your redo to disk. Your transaction is only safe when LGWR confirms it.

```
You type COMMIT
       │
       ▼
LGWR flushes redo buffer to disk
       │
       ▼
Oracle returns "Commit complete"
       │
       ▼
Your data is safe — even if instance crashes now
```

**If LGWR dies:** Instance crashes immediately. No commits possible without LGWR.

3. CKPT checkpoint
```
NAME    CKPT
JOB     checkpoint
```

Signal DBW0 to write dirty buffers, then update the control file and datafile headers with the current checkpoint position.
A checkpoint is oracle way of saying : "Everything up to this point in time is safely on disk. If we crash, recovery only needs to start from here."

```
CKPT fires
    │
    ├──► Signals DBW0 to write dirty blocks to disk
    │
    └──► Updates control file with checkpoint SCN
         Updates datafile headers with checkpoint SCN
```

**SCN — System Change Number.** A number that increases with every transaction. Think of it as Oracle's timestamp for every change ever made. We will use SCN heavily in Phase 4 recovery.

**If CKPT dies:** Instance crashes. Recovery becomes much harder without checkpoint information.

4. SMON - system monitor

```
NAME    SMON
JOB     System Monitor Process
```

Instance recovery after a crash, clean up temporary segments
SMON is the first process that does meaningful work when the instance restarts after a crash
```
Instance crashes
       │
       ▼
Instance restarts
       │
       ▼
SMON wakes up
       │
       ├──► Reads redo logs
       │    Reapplies all committed changes not yet on disk (roll forward)
       │
       └──► Rolls back uncommitted transactions (roll back)
            Database is now consistent
```
This is called **crash recovery** and SMON does it automatically. You do not touch anything. Oracle heals itself.

**If SMON dies:** Instance crashes. Crash recovery impossible without it.

5. PMON - process monitor
```
NAME    PMON
JOB     process cleanup
```

clean up after failed or disconnected sessions

when a session dies unexpectedly - network drop, application crash, killed session - PMON notices and cleans up

```
Session dies unexpectedly
       │
       ▼
PMON wakes up
       │
       ├──► Rolls back any uncommitted transactions from that session
       ├──► Releases all locks held by that session
       ├──► Frees memory allocated to that session
       └──► Removes session from active session list
```

Without PMON — a crashed session would hold locks forever, blocking other sessions permanently.

**If PMON dies:** Instance crashes. Dead sessions would pile up, locks never released.

Five jobs:
```
Process   Job                          Dies = ?
───────   ──────────────────────────   ──────────────────
DBW0      Write dirty blocks to disk   Instance crash
LGWR      Harden redo to disk          Instance crash
CKPT      Trigger checkpoints          Instance crash
SMON      Crash recovery + cleanup     Instance crash
PMON      Clean up dead sessions       Instance crash
```

how everything works
```
You run:  UPDATE employees SET salary = 50000 WHERE id = 101;

1. Oracle finds the block in buffer cache (or reads from disk)
2. Modifies the block in buffer cache → block is now DIRTY
3. Records the change in redo log buffer → LGWR will flush this

You run:  COMMIT;

4. LGWR flushes redo log buffer to redo log file on disk
5. Oracle returns "Commit complete" to you
6. Later — DBW0 writes the dirty block to the datafile
7. Later — CKPT fires, updates control file with new checkpoint
8. If you had crashed before step 4 — SMON would roll back on restart
9. If your session died before COMMIT — PMON rolls it back immediately
```


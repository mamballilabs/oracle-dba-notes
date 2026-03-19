sga is shared by everyone, pga is private - one per session, allocated teh moment you connect, destroyed the moment you disconnect
3 things that lives in PGA
1. session information 
```
Who you are        → username, OS user, client machine
When you connected → timestamp
Your privileges    → what you are allowed to do
Your settings      → NLS date format, language
```
2. sort area 
for performance
```
SELECT * FROM employees ORDER BY salary DESC;
```

Oracle has to sort potentially thousands of rows. That sorting work happens in **your private PGA** — not in the shared SGA.

Why private? Because:
```
SELECT * FROM employees ORDER BY salary DESC;
```

Oracle has to sort potentially thousands of rows. That sorting work happens in **your private PGA** — not in the shared SGA.

Why private? Because:
```
Session A is sorting employees by salary
Session B is sorting orders by date
Session C is sorting products by name

All three sorts happening simultaneously
Each completely independent
Each needs its own workspace
```

3. Hash area
when you join two tables
```
SELECT e.name, d.department_name
FROM   employees e, departments d
WHERE  e.dept_id = d.dept_id;
```

Oracle builds a hash table in memory to match rows. That hash table lives in your private PGA.

---

## The Complete Picture
```
PGA — Private memory, one per session
│
├── Session Info        → who you are, your settings
├── Sort Area           → ORDER BY, GROUP BY work
└── Hash Area           → JOIN operations
```

---

## SGA vs PGA — The Key Difference
```
SGA                          PGA
────────────────────         ────────────────────
Shared by ALL sessions       Private to ONE session
Allocated at startup         Allocated at connect
Released at shutdown         Released at disconnect
One per instance             One per session
```

```sql
SELECT name,
       value/1024/1024 MB
FROM   v$pgastat
WHERE  name IN (
       'aggregate PGA target parameter',
       'total PGA allocated',
       'maximum PGA allocated')
ORDER BY MB DESC;

NAME									 MB
---------------------------------------------------------------- ----------
aggregate PGA target parameter					       2358
maximum PGA allocated						 604.025391
total PGA allocated						 586.273438
maximum PGA allocated						 .079101563
total PGA allocated						 .063476563
aggregate PGA target parameter						  0

6 rows selected.
```

each name appears twice , one for CDB and other for pdb
```sql
aggregate PGA target parameter					       2358
```

This is the budget set for PGA across all the sessions combined 

```sql
total PGA allocated						 586.273438
```

how much pga is actually used right now across all sessions
```sql
maximum PGA allocated						 604.025391
```

this is the peak PGA usage since the instance started .
```
TOTAL ORACLE MEMORY FOOTPRINT
──────────────────────────────────────────
SGA
  Database Buffers    5856 MB   ← data blocks
  Shared Pool         1136 MB   ← SQL + metadata
  Shared IO Pool       128 MB   ← large scans
  Large Pool            48 MB   ← RMAN, parallel
  Redo Buffers           8 MB   ← changes in transit
  Fixed Size             5 MB   ← Oracle internals
                      ───────
  SGA Total          ~7181 MB

PGA
  Currently used       586 MB   ← all active sessions
  Maximum ever         604 MB
  Budget              2358 MB
                      ───────
  PGA In Use          ~586 MB

──────────────────────────────────────────
TOTAL                 ~7767 MB  (~7.6 GB)
──────────────────────────────────────────
```


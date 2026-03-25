```sql
-- =============================================================================
-- Script Name : basic_db_check.sql
-- Author      : Gowtham Mamballi
-- Created On  : 2026-03-25
-- Version     : 1.0
--
-- Purpose     : Basic database health check covering instance status,
--               database mode, datafiles, redo logs, control files,
--               PGA and SGA usage.
--
-- Usage       : @basic_db_check.sql
--
-- =============================================================================

-- ======================
-- Environment Settings
-- ======================
SET PAGESIZE 1000
SET LINESIZE 200
SET TRIMSPOOL ON
SET VERIFY OFF

COLUMN instance_name FORMAT A15
COLUMN status FORMAT A12
COLUMN database_status FORMAT A15

COLUMN name FORMAT A50
COLUMN db_unique_name FORMAT A25
COLUMN open_mode FORMAT A15
COLUMN log_mode FORMAT A15

COLUMN member FORMAT A60

COLUMN pool FORMAT A20

-- ======================
-- Instance Status
-- ======================
PROMPT
PROMPT =========================================
PROMPT           INSTANCE STATUS
PROMPT =========================================

SELECT instance_name, status, database_status
FROM v$instance;

-- ======================
-- Database Details
-- ======================
PROMPT
PROMPT =========================================
PROMPT           DATABASE DETAILS
PROMPT =========================================

SELECT name, db_unique_name, open_mode, log_mode
FROM v$database;

-- ======================
-- Datafiles
-- ======================
PROMPT
PROMPT =========================================
PROMPT              DATAFILES
PROMPT =========================================

COLUMN name FORMAT A80
COLUMN mb FORMAT 999999

SELECT file#, name, status, bytes/1024/1024 MB
FROM v$datafile;

-- ======================
-- Redo Logs
-- ======================
PROMPT
PROMPT =========================================
PROMPT              REDO LOG FILES
PROMPT =========================================

SELECT group#, member
FROM v$logfile
ORDER BY group#;

-- ======================
-- Control Files
-- ======================
PROMPT
PROMPT =========================================
PROMPT              CONTROL FILES
PROMPT =========================================

SELECT name
FROM v$controlfile;

-- ======================
-- PGA Statistics
-- ======================
PROMPT
PROMPT =========================================
PROMPT              PGA STATISTICS
PROMPT =========================================

COLUMN name FORMAT A50
COLUMN mb FORMAT 999999

SELECT name, value/1024/1024 MB
FROM v$pgastat
WHERE name IN (
    'aggregate PGA target parameter',
    'total PGA allocated'
);

-- ======================
-- SGA Statistics
-- ======================
PROMPT
PROMPT =========================================
PROMPT              SGA STATISTICS
PROMPT =========================================

SELECT pool, SUM(bytes)/1024/1024 MB
FROM v$sgastat
GROUP BY pool
ORDER BY MB DESC;

-- ======================
-- End of Script
-- ======================
PROMPT
PROMPT =========================================
PROMPT              END OF REPORT
PROMPT =========================================
PROMPT
```


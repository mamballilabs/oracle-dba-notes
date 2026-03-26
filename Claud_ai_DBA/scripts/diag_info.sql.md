```sql
-- =============================================================================
-- Script Name : diag_info.sql
-- Author      : Gowtham Mamballi
-- Created On  : 2026-03-25
-- Last Updated: 2026-03-25
-- Version     : 1.0
--
-- Purpose     : Display Oracle diagnostic information including ADR paths,
--               trace directory, and alert log location.
--
-- Usage       : @diag_info.sql
--
-- Notes       :
--   - Requires access to V$DIAG_INFO
--   - Run as a user with appropriate privileges (e.g., SYS, SYSTEM)
--
-- =============================================================================

-- ======================
-- Environment Settings
-- ======================
SET PAGESIZE 1000
SET LINESIZE 200
SET TRIMSPOOL ON
SET VERIFY OFF

COLUMN name  FORMAT A30
COLUMN value FORMAT A100

-- ======================
-- Report Header
-- ======================
PROMPT
PROMPT =========================================
PROMPT        DIAGNOSTIC INFORMATION REPORT
PROMPT =========================================
PROMPT

-- ======================
-- Main Query
-- ======================
SELECT name, value
FROM v$diag_info
ORDER BY name;

-- ======================
-- Report Footer
-- ======================
PROMPT
PROMPT =========================================
PROMPT              END OF REPORT
PROMPT =========================================
PROMPT
```


# 📡 MySQL Server — Performance Testing Guide
## Telecom Domain | 10,000 Records Every 2 Seconds via Event Scheduler

---

## 📋 Overview

| Property            | Detail                                         |
|---------------------|------------------------------------------------|
| **Domain**          | Telecom — Call Detail Records (CDR)            |
| **Target Load**     | 10,000 rows per batch insert                   |
| **Batch Interval**  | Every 2 seconds                                |
| **Test Duration**   | 30 seconds (~15 batches)                       |
| **Expected Rows**   | ~150,000 records                               |
| **Scheduler**       | MySQL Event Scheduler (replaces SQL Agent)     |
| **Database**        | MySQL 8.0+                                     |

> ⚠️ **MySQL has no SQL Server Agent.**
> The equivalent is the **MySQL Event Scheduler** (`event_scheduler = ON`).
> Sub-minute scheduling (every 2 sec) is handled inside a **WHILE loop stored procedure** triggered by a **one-time Event**.

---

## 🔧 STEP 0 — Enable Event Scheduler

```sql
-- Check current status
SHOW VARIABLES LIKE 'event_scheduler';

-- Enable for current session
SET GLOBAL event_scheduler = ON;

-- ✅ To make it permanent, add to my.cnf / my.ini:
-- [mysqld]
-- event_scheduler = ON
```

---

## 🗂️ STEP 1 — Create Database & Telecom Table

> Simulates a **Call Detail Record (CDR)** table used in telecom billing systems.

```sql
-- ============================================================
-- DATABASE SETUP
-- ============================================================
CREATE DATABASE IF NOT EXISTS TelecomPerfDB
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE TelecomPerfDB;

-- ============================================================
-- DROP TABLE IF EXISTS (Clean Rerun)
-- ============================================================
DROP TABLE IF EXISTS CDR_CallRecords;

-- ============================================================
-- CREATE TABLE: CDR_CallRecords
-- ============================================================
CREATE TABLE CDR_CallRecords
(
    RecordID          BIGINT          NOT NULL AUTO_INCREMENT,   -- Primary Key
    CallID            VARCHAR(40)     NOT NULL,                  -- Unique Call Identifier
    CallerNumber      VARCHAR(15)     NOT NULL,                  -- MSISDN of Caller
    ReceiverNumber    VARCHAR(15)     NOT NULL,                  -- MSISDN of Receiver
    CallType          VARCHAR(20)     NOT NULL,                  -- VOICE / SMS / DATA
    CallStartTime     DATETIME(3)     NOT NULL,                  -- Call Start Timestamp
    CallEndTime       DATETIME(3)         NULL,                  -- Call End Timestamp
    CallDuration_Sec  INT                 NULL,                  -- Duration in Seconds
    DataUsage_MB      DECIMAL(10,3)       NULL,                  -- Data Usage (MB)
    TowerID           VARCHAR(20)     NOT NULL,                  -- Cell Tower ID
    CircleName        VARCHAR(50)     NOT NULL,                  -- Telecom Circle / Region
    ServiceType       VARCHAR(20)     NOT NULL,                  -- PREPAID / POSTPAID
    RoamingFlag       TINYINT(1)      NOT NULL DEFAULT 0,        -- 0=Home, 1=Roaming
    CallStatus        VARCHAR(20)     NOT NULL,                  -- CONNECTED/FAILED/DROPPED
    ChargeAmount      DECIMAL(10,4)       NULL,                  -- Billing Amount
    BatchID           INT             NOT NULL,                  -- Batch Number
    InsertedAt        DATETIME(3)     NOT NULL
                          DEFAULT (NOW(3)),

    PRIMARY KEY (RecordID),

    INDEX IX_CallerNumber  (CallerNumber, CallStartTime),
    INDEX IX_CallStartTime (CallStartTime DESC),
    INDEX IX_BatchID       (BatchID, InsertedAt),
    INDEX IX_TowerCircle   (TowerID, CircleName)

) ENGINE = InnoDB
  ROW_FORMAT = DYNAMIC
  COMMENT = 'Telecom CDR Performance Test Table';

SELECT 'Table CDR_CallRecords created successfully.' AS Status;
```

---

## ⚙️ STEP 2 — Create Batch Counter Table

```sql
USE TelecomPerfDB;

DROP TABLE IF EXISTS BatchCounter;

CREATE TABLE BatchCounter
(
    ID          INT         NOT NULL AUTO_INCREMENT PRIMARY KEY,
    BatchNumber INT         NOT NULL DEFAULT 0,
    UpdatedAt   DATETIME(3) NOT NULL DEFAULT (NOW(3))
) ENGINE = InnoDB;

-- Insert initial row
INSERT INTO BatchCounter (BatchNumber) VALUES (0);

SELECT 'BatchCounter table created.' AS Status;
```

---

## 🔧 STEP 3 — Create the Stored Procedure (Insert 10,000 Rows)

> Uses a **helper numbers table** approach for bulk insert — fastest method in MySQL without cursors.

### 3A — Create Numbers Helper Table (One-Time Setup)

```sql
USE TelecomPerfDB;

DROP TABLE IF EXISTS Numbers;

CREATE TABLE Numbers (n INT NOT NULL PRIMARY KEY);

-- Insert 10,000 sequential numbers (1 to 10000)
INSERT INTO Numbers (n)
WITH RECURSIVE cte AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM cte WHERE n < 10000
)
SELECT n FROM cte;

SELECT COUNT(*) AS NumbersTableRows FROM Numbers;
```

### 3B — Create the Bulk Insert Stored Procedure

```sql
USE TelecomPerfDB;

DROP PROCEDURE IF EXISTS usp_InsertCDRBatch;

DELIMITER $$

-- ============================================================
-- STORED PROCEDURE : usp_InsertCDRBatch
-- Purpose          : Insert exactly 10,000 CDR rows per call
-- Called by        : usp_RunPerfTest_30Sec (loop wrapper)
-- ============================================================
CREATE PROCEDURE usp_InsertCDRBatch()
BEGIN

    DECLARE v_BatchID     INT;
    DECLARE v_StartTime   DATETIME(3);
    DECLARE v_EndTime     DATETIME(3);
    DECLARE v_RowCount    INT;

    SET v_StartTime = NOW(3);

    -- --------------------------------------------------------
    -- Increment and get batch number
    -- --------------------------------------------------------
    UPDATE BatchCounter
    SET    BatchNumber = BatchNumber + 1,
           UpdatedAt   = NOW(3);

    SELECT BatchNumber INTO v_BatchID FROM BatchCounter LIMIT 1;

    -- --------------------------------------------------------
    -- Bulk Insert 10,000 CDR Rows using Numbers table
    -- ELT() for lookup-based random value selection
    -- --------------------------------------------------------
    INSERT INTO CDR_CallRecords
    (
        CallID,
        CallerNumber,
        ReceiverNumber,
        CallType,
        CallStartTime,
        CallEndTime,
        CallDuration_Sec,
        DataUsage_MB,
        TowerID,
        CircleName,
        ServiceType,
        RoamingFlag,
        CallStatus,
        ChargeAmount,
        BatchID,
        InsertedAt
    )
    SELECT
        -- Unique Call ID
        CONCAT('CALL-', v_BatchID, '-', n.n),

        -- Caller Number: Indian mobile (starts with 7/8/9)
        CONCAT(
            '91',
            ELT(FLOOR(RAND() * 3) + 1, '7', '8', '9'),
            LPAD(FLOOR(RAND() * 1000000000), 9, '0')
        ),

        -- Receiver Number
        CONCAT(
            '91',
            ELT(FLOOR(RAND() * 3) + 1, '7', '8', '9'),
            LPAD(FLOOR(RAND() * 1000000000), 9, '0')
        ),

        -- Call Type: VOICE / SMS / DATA
        ELT(FLOOR(RAND() * 3) + 1, 'VOICE', 'SMS', 'DATA'),

        -- Call Start Time: random within last 24 hours
        DATE_SUB(NOW(3), INTERVAL FLOOR(RAND() * 86400) SECOND),

        -- Call End Time: start + up to 1 hour
        DATE_ADD(
            DATE_SUB(NOW(3), INTERVAL FLOOR(RAND() * 86400) SECOND),
            INTERVAL FLOOR(RAND() * 3600) SECOND
        ),

        -- Call Duration in Seconds (0 to 3600)
        FLOOR(RAND() * 3601),

        -- Data Usage in MB (0.000 to 999.999)
        ROUND(RAND() * 999.999, 3),

        -- Tower ID: e.g. TOWER-A1234
        CONCAT(
            'TOWER-',
            CHAR(65 + FLOOR(RAND() * 26)),
            LPAD(FLOOR(RAND() * 9000) + 1000, 4, '0')
        ),

        -- Circle Name (Indian Telecom Regions)
        ELT(
            FLOOR(RAND() * 10) + 1,
            'Andhra Pradesh',
            'Telangana',
            'Maharashtra',
            'Karnataka',
            'Tamil Nadu',
            'Delhi',
            'Gujarat',
            'Rajasthan',
            'Punjab',
            'West Bengal'
        ),

        -- Service Type: PREPAID / POSTPAID
        ELT(FLOOR(RAND() * 2) + 1, 'PREPAID', 'POSTPAID'),

        -- Roaming Flag: 10% roaming
        IF(RAND() < 0.10, 1, 0),

        -- Call Status
        ELT(FLOOR(RAND() * 3) + 1, 'CONNECTED', 'FAILED', 'DROPPED'),

        -- Charge Amount: 0.0000 to 99.9999
        ROUND(RAND() * 99.9999, 4),

        -- Batch ID
        v_BatchID,

        -- Inserted At
        NOW(3)

    FROM Numbers n;   -- 10,000 rows from Numbers table

    -- --------------------------------------------------------
    -- Log execution stats
    -- --------------------------------------------------------
    SET v_EndTime = NOW(3);

    SELECT COUNT(*) INTO v_RowCount
    FROM CDR_CallRecords
    WHERE BatchID = v_BatchID;

    SELECT
        CONCAT('Batch #', v_BatchID) AS BatchInfo,
        v_RowCount                   AS RowsInserted,
        TIMESTAMPDIFF(
            MICROSECOND, v_StartTime, v_EndTime
        ) / 1000                     AS DurationMS,
        v_EndTime                    AS ExecutedAt;

END$$

DELIMITER ;

SELECT 'Stored Procedure usp_InsertCDRBatch created.' AS Status;
```

---

## ⏱️ STEP 4 — Create Loop Procedure (Every 2 Sec for 30 Sec)

> MySQL equivalent of `WAITFOR DELAY` is **`DO SLEEP(seconds)`**.

```sql
USE TelecomPerfDB;

DROP PROCEDURE IF EXISTS usp_RunPerfTest_30Sec;

DELIMITER $$

-- ============================================================
-- STORED PROCEDURE : usp_RunPerfTest_30Sec
-- Purpose          : Loops every 2 seconds for 30 seconds
--                    Calls usp_InsertCDRBatch each iteration
-- Triggered by     : MySQL Event Scheduler (one-time event)
-- ============================================================
CREATE PROCEDURE usp_RunPerfTest_30Sec()
BEGIN

    DECLARE v_StartTime   DATETIME(3);
    DECLARE v_EndTime     DATETIME(3);
    DECLARE v_Now         DATETIME(3);
    DECLARE v_Iteration   INT DEFAULT 0;

    SET v_StartTime = NOW(3);
    SET v_EndTime   = DATE_ADD(NOW(3), INTERVAL 30 SECOND);

    -- Log start
    SELECT
        CONCAT('Performance Test Started') AS Event,
        v_StartTime                         AS StartTime,
        v_EndTime                           AS WillEndAt;

    -- --------------------------------------------------------
    -- Main Loop: runs every 2 seconds until 30 seconds elapsed
    -- --------------------------------------------------------
    test_loop: LOOP

        SET v_Now = NOW(3);

        -- Exit if 30 seconds have passed
        IF v_Now >= v_EndTime THEN
            LEAVE test_loop;
        END IF;

        SET v_Iteration = v_Iteration + 1;

        -- Execute the 10,000-row batch insert
        CALL usp_InsertCDRBatch();

        -- Wait 2 seconds before next batch
        -- Skip sleep if we are near end of 30-sec window
        IF DATE_ADD(NOW(3), INTERVAL 2 SECOND) < v_EndTime THEN
            DO SLEEP(2);
        END IF;

    END LOOP test_loop;

    -- Log completion
    SELECT
        CONCAT('Performance Test Completed') AS Event,
        v_Iteration                           AS TotalBatches,
        v_Iteration * 10000                   AS ApproxRowsInserted,
        NOW(3)                                AS EndedAt;

END$$

DELIMITER ;

SELECT 'Wrapper Procedure usp_RunPerfTest_30Sec created.' AS Status;
```

---

## 📅 STEP 5 — Schedule with MySQL Event Scheduler

> MySQL Event Scheduler is the **exact equivalent of SQL Server Agent** for MySQL.
> We create a **one-time event** that fires immediately and runs the 30-second loop.

```sql
USE TelecomPerfDB;

-- ============================================================
-- Drop existing event if re-running
-- ============================================================
DROP EVENT IF EXISTS EVT_Telecom_PerfTest;

-- ============================================================
-- Create One-Time Event
-- Fires immediately → calls the 30-second loop procedure
-- ============================================================
CREATE EVENT EVT_Telecom_PerfTest
    ON SCHEDULE AT NOW() + INTERVAL 5 SECOND   -- starts 5 sec after creation
    ON COMPLETION NOT PRESERVE                  -- auto-drops after execution
    ENABLE
    COMMENT 'Telecom CDR Performance Test: 10000 rows every 2 sec for 30 sec'
DO
    CALL TelecomPerfDB.usp_RunPerfTest_30Sec();

SELECT 'Event EVT_Telecom_PerfTest scheduled successfully.' AS Status;
```

> ⏱️ The event fires **5 seconds after you run this script**, giving you time to open a monitoring query.

---

## ▶️ STEP 6 — Run Manually (Alternative to Event)

> You can also call the procedure directly in a session without the Event Scheduler:

```sql
USE TelecomPerfDB;

-- Direct execution (blocks session for 30 seconds)
CALL usp_RunPerfTest_30Sec();
```

---

## 📊 STEP 7 — Monitor & Validate Performance

### 7.1 — Real-Time Row Count per Batch

```sql
USE TelecomPerfDB;

SELECT
    BatchID,
    COUNT(*)                                AS RowsInBatch,
    MIN(InsertedAt)                         AS BatchStart,
    MAX(InsertedAt)                         AS BatchEnd,
    TIMESTAMPDIFF(MICROSECOND,
                  MIN(InsertedAt),
                  MAX(InsertedAt)) / 1000   AS DurationMS
FROM CDR_CallRecords
GROUP BY BatchID
ORDER BY BatchID;
```

### 7.2 — Total Records & Throughput

```sql
USE TelecomPerfDB;

SELECT
    COUNT(*)                                          AS TotalRecords,
    MIN(InsertedAt)                                   AS TestStartTime,
    MAX(InsertedAt)                                   AS TestEndTime,
    TIMESTAMPDIFF(SECOND, MIN(InsertedAt),
                          MAX(InsertedAt))            AS TotalDurationSeconds,
    ROUND(
        COUNT(*) /
        NULLIF(TIMESTAMPDIFF(SECOND,
               MIN(InsertedAt), MAX(InsertedAt)), 0)
    , 2)                                              AS RowsPerSecond
FROM CDR_CallRecords;
```

### 7.3 — Performance by Telecom Circle

```sql
USE TelecomPerfDB;

SELECT
    CircleName,
    COUNT(*)                                             AS TotalCalls,
    SUM(CallType = 'VOICE')                              AS VoiceCalls,
    SUM(CallType = 'SMS')                                AS SMSCalls,
    SUM(CallType = 'DATA')                               AS DataSessions,
    ROUND(AVG(CallDuration_Sec), 2)                      AS AvgDurationSec,
    ROUND(SUM(DataUsage_MB), 3)                          AS TotalDataMB,
    ROUND(SUM(ChargeAmount), 4)                          AS TotalRevenue,
    SUM(RoamingFlag = 1)                                 AS RoamingCalls,
    SUM(CallStatus = 'DROPPED')                          AS DroppedCalls
FROM CDR_CallRecords
GROUP BY CircleName
ORDER BY TotalCalls DESC;
```

### 7.4 — Check Event Scheduler Status

```sql
-- Check if event scheduler is ON
SHOW VARIABLES LIKE 'event_scheduler';

-- List all events
SHOW EVENTS FROM TelecomPerfDB;

-- Detailed event info
SELECT
    EVENT_NAME,
    STATUS,
    EVENT_TYPE,
    EXECUTE_AT,
    LAST_EXECUTED,
    EVENT_COMMENT
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'TelecomPerfDB';
```

### 7.5 — InnoDB Engine Performance Stats

```sql
-- InnoDB buffer pool hit rate
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';

-- Rows inserted per second
SHOW GLOBAL STATUS LIKE 'Innodb_rows_inserted';

-- Current active threads
SHOW GLOBAL STATUS LIKE 'Threads_running';

-- Table-specific I/O stats
SELECT
    TABLE_NAME,
    ROWS_READ,
    ROWS_INSERTED,
    ROWS_UPDATED,
    ROWS_DELETED
FROM information_schema.TABLE_STATISTICS
WHERE TABLE_SCHEMA = 'TelecomPerfDB';
```

### 7.6 — Show Currently Running Processes

```sql
-- Monitor active queries during test
SHOW FULL PROCESSLIST;

-- Or use performance_schema
SELECT
    t.THREAD_ID,
    t.PROCESSLIST_ID,
    t.PROCESSLIST_USER,
    t.PROCESSLIST_HOST,
    t.PROCESSLIST_DB,
    t.PROCESSLIST_COMMAND,
    t.PROCESSLIST_TIME,
    t.PROCESSLIST_STATE,
    LEFT(t.PROCESSLIST_INFO, 100) AS Query
FROM performance_schema.threads t
WHERE t.PROCESSLIST_COMMAND != 'Sleep'
ORDER BY t.PROCESSLIST_TIME DESC;
```

### 7.7 — Top Wait Events (Performance Bottleneck)

```sql
-- Top waits consuming most time (Performance Schema)
SELECT
    EVENT_NAME,
    COUNT_STAR            AS TotalWaits,
    ROUND(SUM_TIMER_WAIT / 1e12, 4) AS TotalWaitSec,
    ROUND(AVG_TIMER_WAIT / 1e9,  4) AS AvgWaitMs,
    ROUND(MAX_TIMER_WAIT / 1e9,  4) AS MaxWaitMs
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE COUNT_STAR > 0
  AND EVENT_NAME NOT LIKE '%idle%'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 15;
```

---

## 🧹 STEP 8 — Cleanup After Test

```sql
USE TelecomPerfDB;

-- ============================================================
-- Option A: Clear data only (keep table structure)
-- ============================================================
TRUNCATE TABLE CDR_CallRecords;
TRUNCATE TABLE Numbers;
TRUNCATE TABLE BatchCounter;

-- Re-seed BatchCounter
INSERT INTO BatchCounter (BatchNumber) VALUES (0);

-- Re-seed Numbers table
INSERT INTO Numbers (n)
WITH RECURSIVE cte AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM cte WHERE n < 10000
)
SELECT n FROM cte;

SELECT 'Test data cleared. Ready for next run.' AS Status;

-- ============================================================
-- Option B: Drop everything
-- ============================================================
-- DROP DATABASE TelecomPerfDB;
```

---

## 📈 Expected Test Results Summary

| Metric                     | Expected Value                      |
|----------------------------|-------------------------------------|
| **Batches Executed**       | ~15 (every 2 sec × 30 sec)          |
| **Rows per Batch**         | 10,000                              |
| **Total Rows**             | ~150,000                            |
| **Insert Speed**           | ~3,000 – 8,000 rows/sec (InnoDB)    |
| **Batch Duration**         | 200ms – 800ms per batch             |
| **Buffer Pool Hit Rate**   | Should be > 95%                     |
| **Threads Running**        | 1–3 during test                     |

---

## 🔑 Key Concepts Used

| Concept                        | Purpose                                                   |
|--------------------------------|-----------------------------------------------------------|
| `Numbers` helper table         | Generates 10,000 rows in a single `INSERT ... SELECT`     |
| `WITH RECURSIVE cte`           | Populates Numbers table without loops                     |
| `ELT(FLOOR(RAND()*N)+1, ...)` | Fast random enum-style value picker                       |
| `DO SLEEP(2)`                  | MySQL equivalent of `WAITFOR DELAY '00:00:02'`            |
| `LOOP ... LEAVE`               | MySQL equivalent of `WHILE` loop construct                |
| `NOW(3)`                       | Millisecond precision timestamp                           |
| `CREATE EVENT ... AT NOW()`    | MySQL Event Scheduler = SQL Server Agent equivalent       |
| `ON COMPLETION NOT PRESERVE`   | Auto-drops event after it fires                           |
| `AUTO_INCREMENT`               | MySQL equivalent of `IDENTITY(1,1)`                       |
| `InnoDB ENGINE`                | Transactional storage engine with row-level locking       |
| `TIMESTAMPDIFF(MICROSECOND)`   | Measures batch duration at sub-millisecond precision      |

---

## ⚠️ Prerequisites & Notes

| Requirement                  | Detail                                                           |
|------------------------------|------------------------------------------------------------------|
| MySQL Version                | **8.0+** (for `WITH RECURSIVE` CTE support)                     |
| Event Scheduler              | Must be `ON` — run `SET GLOBAL event_scheduler = ON;`           |
| User Privilege               | Needs `EVENT`, `CREATE`, `INSERT`, `EXECUTE` privileges          |
| InnoDB Buffer Pool           | Set `innodb_buffer_pool_size = 1G` or more for best performance  |
| Binary Logging               | Disable during test for max speed: `SET sql_log_bin = 0;`        |

### Optional Speed Optimizations Before Test

```sql
-- Disable binary log for this session (max insert speed)
SET SESSION sql_log_bin = 0;

-- Disable unique checks temporarily
SET SESSION unique_checks = 0;

-- Disable foreign key checks
SET SESSION foreign_key_checks = 0;

-- Re-enable after test
SET SESSION sql_log_bin = 1;
SET SESSION unique_checks = 1;
SET SESSION foreign_key_checks = 1;
```

### my.cnf Tuning for High-Insert Workloads

```ini
[mysqld]
innodb_buffer_pool_size     = 2G
innodb_log_file_size        = 512M
innodb_flush_log_at_trx_commit = 2    # Slightly less durable, much faster
innodb_flush_method         = O_DIRECT
innodb_io_capacity          = 2000
event_scheduler             = ON
```

---

## 🔄 MS SQL Server vs MySQL — Quick Reference

| Feature                    | MS SQL Server                    | MySQL Equivalent                     |
|----------------------------|----------------------------------|--------------------------------------|
| Auto Increment             | `IDENTITY(1,1)`                  | `AUTO_INCREMENT`                     |
| Pause Execution            | `WAITFOR DELAY '00:00:02'`       | `DO SLEEP(2)`                        |
| While Loop Exit            | `WHILE ... BREAK`                | `LOOP ... LEAVE label`               |
| Job Scheduler              | SQL Server Agent                 | MySQL Event Scheduler                |
| Current Timestamp (ms)     | `SYSDATETIME()`                  | `NOW(3)`                             |
| Row Generator              | `CROSS JOIN sys.all_objects`     | `Numbers` helper table + CTE         |
| Stored Proc Create         | `CREATE PROCEDURE ... AS BEGIN`  | `DELIMITER $$ ... CREATE PROCEDURE`  |
| String Concat              | `CONCAT()` or `+`                | `CONCAT()`                           |
| Random Number              | `ABS(CHECKSUM(NEWID())) % N`     | `FLOOR(RAND() * N)`                  |
| Enum-style Pick            | `CASE WHEN ... END`              | `ELT(FLOOR(RAND()*N)+1, 'a','b')`    |
| Date Add                   | `DATEADD(SECOND, N, date)`       | `DATE_ADD(date, INTERVAL N SECOND)`  |
| Date Diff (ms)             | `DATEDIFF(MILLISECOND, s, e)`    | `TIMESTAMPDIFF(MICROSECOND, s, e)`   |
| Top N Rows                 | `SELECT TOP (N)`                 | `SELECT ... LIMIT N`                 |
| Bulk Insert Speed Tip      | Disable indexes before insert    | `SET unique_checks=0; sql_log_bin=0` |

---

*Generated for Telecom Domain Performance Testing — MySQL Server 8.0+*

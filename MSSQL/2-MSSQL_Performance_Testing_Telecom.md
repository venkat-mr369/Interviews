# 📡 MS SQL Server — Performance Testing Guide
## Telecom Domain | 10,000 Records Every 2 Seconds via SQL Agent

---

## 📋 Overview

| Property            | Detail                                      |
|---------------------|---------------------------------------------|
| **Domain**          | Telecom — Call Detail Records (CDR)         |
| **Target Load**     | 10,000 rows per batch insert                |
| **Batch Interval**  | Every 2 seconds                             |
| **Test Duration**   | 30 seconds (~15 batches)                    |
| **Expected Rows**   | ~150,000 records                            |
| **Scheduler**       | SQL Server Agent Job                        |
| **Database**        | MS SQL Server 2016+                         |

---

## 🗂️ STEP 1 — Create the Telecom Table

> Simulates a **Call Detail Record (CDR)** table used in telecom billing systems.

```sql
-- ============================================================
-- DATABASE SETUP
-- ============================================================
USE master;
GO

IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = 'TelecomPerfDB')
BEGIN
    CREATE DATABASE TelecomPerfDB;
END
GO

USE TelecomPerfDB;
GO

-- ============================================================
-- DROP TABLE IF EXISTS (Clean Rerun)
-- ============================================================
IF OBJECT_ID('dbo.CDR_CallRecords', 'U') IS NOT NULL
    DROP TABLE dbo.CDR_CallRecords;
GO

-- ============================================================
-- CREATE TABLE: CDR_CallRecords
-- ============================================================
CREATE TABLE dbo.CDR_CallRecords
(
    RecordID          BIGINT IDENTITY(1,1)   NOT NULL,   -- Primary Key
    CallID            VARCHAR(30)            NOT NULL,   -- Unique Call Identifier
    CallerNumber      VARCHAR(15)            NOT NULL,   -- MSISDN of Caller
    ReceiverNumber    VARCHAR(15)            NOT NULL,   -- MSISDN of Receiver
    CallType          VARCHAR(20)            NOT NULL,   -- VOICE / SMS / DATA
    CallStartTime     DATETIME2(3)           NOT NULL,   -- Call Start Timestamp
    CallEndTime       DATETIME2(3)           NULL,       -- Call End Timestamp
    CallDuration_Sec  INT                    NULL,       -- Duration in Seconds
    DataUsage_MB      DECIMAL(10,3)          NULL,       -- Data Usage for DATA calls
    TowerID           VARCHAR(20)            NOT NULL,   -- Cell Tower ID
    CircleName        VARCHAR(50)            NOT NULL,   -- Telecom Circle (Region)
    ServiceType       VARCHAR(20)            NOT NULL,   -- PREPAID / POSTPAID
    RoamingFlag       BIT                    NOT NULL DEFAULT 0,  -- 0=Home, 1=Roaming
    CallStatus        VARCHAR(20)            NOT NULL,   -- CONNECTED / FAILED / DROPPED
    ChargeAmount      DECIMAL(10,4)          NULL,       -- Billing Amount
    BatchID           INT                    NOT NULL,   -- Batch Number for Tracking
    InsertedAt        DATETIME2(3)           NOT NULL DEFAULT SYSDATETIME(),

    CONSTRAINT PK_CDR_CallRecords PRIMARY KEY CLUSTERED (RecordID)
);
GO

-- ============================================================
-- INDEXES for Performance Testing Queries
-- ============================================================
CREATE NONCLUSTERED INDEX IX_CDR_CallerNumber
    ON dbo.CDR_CallRecords (CallerNumber) INCLUDE (CallStartTime, CallDuration_Sec);

CREATE NONCLUSTERED INDEX IX_CDR_CallStartTime
    ON dbo.CDR_CallRecords (CallStartTime DESC);

CREATE NONCLUSTERED INDEX IX_CDR_BatchID
    ON dbo.CDR_CallRecords (BatchID, InsertedAt);

CREATE NONCLUSTERED INDEX IX_CDR_TowerID
    ON dbo.CDR_CallRecords (TowerID, CircleName);

PRINT '✅ Table CDR_CallRecords created successfully.';
GO
```

---

## ⚙️ STEP 2 — Create a Helper Sequence Table (Batch Counter)

```sql
USE TelecomPerfDB;
GO

-- Tracks batch number across agent job executions
IF OBJECT_ID('dbo.BatchCounter', 'U') IS NOT NULL
    DROP TABLE dbo.BatchCounter;

CREATE TABLE dbo.BatchCounter
(
    ID          INT IDENTITY(1,1) PRIMARY KEY,
    BatchNumber INT           NOT NULL DEFAULT 0,
    UpdatedAt   DATETIME2(3)  NOT NULL DEFAULT SYSDATETIME()
);

-- Insert initial row
INSERT INTO dbo.BatchCounter (BatchNumber) VALUES (0);
GO

PRINT '✅ BatchCounter table created.';
GO
```

---

## 🔧 STEP 3 — Create the Stored Procedure

> This procedure inserts **10,000 CDR records** per call using a single bulk `INSERT ... SELECT` with `sys.all_objects` cross join for high-speed generation.

```sql
USE TelecomPerfDB;
GO

IF OBJECT_ID('dbo.usp_InsertCDRBatch', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_InsertCDRBatch;
GO

-- ============================================================
-- STORED PROCEDURE: usp_InsertCDRBatch
-- Purpose : Insert 10,000 telecom CDR records per execution
-- Frequency: Every 2 seconds (via SQL Agent job loop)
-- ============================================================
CREATE PROCEDURE dbo.usp_InsertCDRBatch
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @BatchID    INT;
    DECLARE @StartTime  DATETIME2(3) = SYSDATETIME();
    DECLARE @TargetRows INT = 10000;

    -- --------------------------------------------------------
    -- Get and increment batch counter
    -- --------------------------------------------------------
    UPDATE dbo.BatchCounter
    SET    BatchNumber = BatchNumber + 1,
           UpdatedAt   = SYSDATETIME();

    SELECT @BatchID = BatchNumber FROM dbo.BatchCounter;

    -- --------------------------------------------------------
    -- Bulk Insert 10,000 Rows using Row Generator
    -- Uses CROSS JOIN on sys.all_objects to generate rows fast
    -- --------------------------------------------------------
    INSERT INTO dbo.CDR_CallRecords
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
        BatchID
    )
    SELECT TOP (@TargetRows)
        -- Unique Call ID
        CONCAT('CALL-', @BatchID, '-', ROW_NUMBER() OVER (ORDER BY (SELECT NULL))),

        -- Caller Number: simulate Indian mobile numbers
        CONCAT('91',
               CAST(7000000000 + ABS(CHECKSUM(NEWID())) % 2999999999 AS VARCHAR(10))),

        -- Receiver Number
        CONCAT('91',
               CAST(7000000000 + ABS(CHECKSUM(NEWID())) % 2999999999 AS VARCHAR(10))),

        -- Call Type: VOICE / SMS / DATA
        CASE ABS(CHECKSUM(NEWID())) % 3
            WHEN 0 THEN 'VOICE'
            WHEN 1 THEN 'SMS'
            ELSE        'DATA'
        END,

        -- Call Start Time: random within last 24 hours
        DATEADD(SECOND, -(ABS(CHECKSUM(NEWID())) % 86400), SYSDATETIME()),

        -- Call End Time: start + random duration
        DATEADD(SECOND,  (ABS(CHECKSUM(NEWID())) % 3600),
                DATEADD(SECOND, -(ABS(CHECKSUM(NEWID())) % 86400), SYSDATETIME())),

        -- Call Duration in Seconds (0 to 3600)
        ABS(CHECKSUM(NEWID())) % 3601,

        -- Data Usage in MB (0.000 to 999.999)
        CAST(ABS(CHECKSUM(NEWID())) % 1000 AS DECIMAL(10,3)),

        -- Tower ID
        CONCAT('TOWER-',
               CHAR(65 + ABS(CHECKSUM(NEWID())) % 26),
               CAST(ABS(CHECKSUM(NEWID())) % 9000 + 1000 AS VARCHAR(4))),

        -- Circle Name (Telecom Regions in India)
        CASE ABS(CHECKSUM(NEWID())) % 10
            WHEN 0 THEN 'Andhra Pradesh'
            WHEN 1 THEN 'Telangana'
            WHEN 2 THEN 'Maharashtra'
            WHEN 3 THEN 'Karnataka'
            WHEN 4 THEN 'Tamil Nadu'
            WHEN 5 THEN 'Delhi'
            WHEN 6 THEN 'Gujarat'
            WHEN 7 THEN 'Rajasthan'
            WHEN 8 THEN 'Punjab'
            ELSE        'West Bengal'
        END,

        -- Service Type
        CASE ABS(CHECKSUM(NEWID())) % 2
            WHEN 0 THEN 'PREPAID'
            ELSE        'POSTPAID'
        END,

        -- Roaming Flag (10% roaming)
        CASE WHEN ABS(CHECKSUM(NEWID())) % 10 = 0 THEN 1 ELSE 0 END,

        -- Call Status
        CASE ABS(CHECKSUM(NEWID())) % 3
            WHEN 0 THEN 'CONNECTED'
            WHEN 1 THEN 'FAILED'
            ELSE        'DROPPED'
        END,

        -- Charge Amount (0.0000 to 99.9999)
        CAST(ABS(CHECKSUM(NEWID())) % 100 + CAST(ABS(CHECKSUM(NEWID())) % 10000 AS DECIMAL(10,4)) / 10000
             AS DECIMAL(10,4)),

        -- Batch ID
        @BatchID

    FROM sys.all_objects AS a
    CROSS JOIN sys.all_objects AS b;  -- Generates 100,000+ rows; TOP limits to 10,000

    -- --------------------------------------------------------
    -- Log execution stats to console
    -- --------------------------------------------------------
    DECLARE @EndTime   DATETIME2(3) = SYSDATETIME();
    DECLARE @RowsInserted INT;
    SELECT @RowsInserted = COUNT(*) FROM dbo.CDR_CallRecords WHERE BatchID = @BatchID;

    PRINT CONCAT('Batch #', @BatchID,
                 ' | Rows Inserted: ', @RowsInserted,
                 ' | Duration: ',
                 DATEDIFF(MILLISECOND, @StartTime, @EndTime), ' ms',
                 ' | Time: ', CONVERT(VARCHAR, @EndTime, 121));

END
GO

PRINT '✅ Stored Procedure usp_InsertCDRBatch created successfully.';
GO
```

---

## ⏱️ STEP 4 — Create the Loop Procedure (Runs Every 2 Sec for 30 Sec)

> SQL Agent jobs cannot natively schedule sub-minute intervals. We solve this with a **while loop inside a wrapper procedure** that calls the insert SP every 2 seconds for 30 seconds.

```sql
USE TelecomPerfDB;
GO

IF OBJECT_ID('dbo.usp_RunPerfTest_30Sec', 'P') IS NOT NULL
    DROP PROCEDURE dbo.usp_RunPerfTest_30Sec;
GO

-- ============================================================
-- STORED PROCEDURE: usp_RunPerfTest_30Sec
-- Purpose : Loop every 2 seconds for 30 seconds
--           Calls usp_InsertCDRBatch each iteration (~15x)
-- ============================================================
CREATE PROCEDURE dbo.usp_RunPerfTest_30Sec
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StartTime   DATETIME2(3) = SYSDATETIME();
    DECLARE @EndTime     DATETIME2(3) = DATEADD(SECOND, 30, SYSDATETIME());
    DECLARE @Iteration   INT          = 0;
    DECLARE @SleepTime   VARCHAR(12)  = '00:00:02';   -- 2 seconds wait

    PRINT '🚀 Performance Test Started at: ' + CONVERT(VARCHAR, @StartTime, 121);
    PRINT '⏰ Will run until: '              + CONVERT(VARCHAR, @EndTime,   121);
    PRINT '------------------------------------------------------------';

    -- --------------------------------------------------------
    -- Loop for 30 seconds, inserting every 2 seconds
    -- --------------------------------------------------------
    WHILE SYSDATETIME() < @EndTime
    BEGIN
        SET @Iteration = @Iteration + 1;

        PRINT CONCAT('▶ Executing Batch Iteration #', @Iteration,
                     ' at ', CONVERT(VARCHAR, SYSDATETIME(), 121));

        -- Execute the insert stored procedure
        EXEC dbo.usp_InsertCDRBatch;

        -- Wait 2 seconds before next batch
        -- Skip final wait if test time is nearly over
        IF SYSDATETIME() < DATEADD(SECOND, -2, @EndTime)
            WAITFOR DELAY @SleepTime;
    END

    PRINT '------------------------------------------------------------';
    PRINT CONCAT('✅ Performance Test Completed. Total Iterations: ', @Iteration);
    PRINT '🏁 Ended at: ' + CONVERT(VARCHAR, SYSDATETIME(), 121);

END
GO

PRINT '✅ Wrapper Procedure usp_RunPerfTest_30Sec created successfully.';
GO
```

---

## 🤖 STEP 5 — Create SQL Server Agent Job

> Schedule `usp_RunPerfTest_30Sec` to run once (or on demand) via SQL Server Agent.

```sql
USE msdb;
GO

-- ============================================================
-- Step 5A: Create the Agent Job
-- ============================================================
IF EXISTS (SELECT job_id FROM msdb.dbo.sysjobs WHERE name = N'Telecom_CDR_PerfTest_Job')
BEGIN
    EXEC msdb.dbo.sp_delete_job @job_name = N'Telecom_CDR_PerfTest_Job', @delete_unused_schedule = 1;
END
GO

EXEC msdb.dbo.sp_add_job
    @job_name              = N'Telecom_CDR_PerfTest_Job',
    @enabled               = 1,
    @description           = N'Telecom Performance Test: Insert 10000 CDR records every 2 sec for 30 sec',
    @category_name         = N'[Uncategorized (Local)]',
    @owner_login_name      = N'sa';          -- ⚠️ Change to your SQL login
GO

-- ============================================================
-- Step 5B: Add Job Step
-- ============================================================
EXEC msdb.dbo.sp_add_jobstep
    @job_name        = N'Telecom_CDR_PerfTest_Job',
    @step_name       = N'Execute CDR Bulk Insert Loop',
    @subsystem       = N'TSQL',
    @command         = N'USE TelecomPerfDB; EXEC dbo.usp_RunPerfTest_30Sec;',
    @database_name   = N'TelecomPerfDB',
    @on_success_action = 1,   -- Quit with success
    @on_fail_action    = 2;   -- Quit with failure
GO

-- ============================================================
-- Step 5C: Add Schedule (Run Once — On Demand)
-- ============================================================
EXEC msdb.dbo.sp_add_schedule
    @schedule_name          = N'OnDemand_Once',
    @freq_type              = 1,         -- 1 = Once / On Demand
    @active_start_date      = 20250101,
    @active_start_time      = 0;
GO

EXEC msdb.dbo.sp_attach_schedule
    @job_name      = N'Telecom_CDR_PerfTest_Job',
    @schedule_name = N'OnDemand_Once';
GO

-- ============================================================
-- Step 5D: Set Target Server
-- ============================================================
EXEC msdb.dbo.sp_add_jobserver
    @job_name   = N'Telecom_CDR_PerfTest_Job',
    @server_name = N'(LOCAL)';
GO

PRINT '✅ SQL Agent Job created: Telecom_CDR_PerfTest_Job';
GO
```

---

## ▶️ STEP 6 — Start the Job Manually

```sql
USE msdb;
GO

-- Trigger the Agent Job immediately
EXEC msdb.dbo.sp_start_job @job_name = N'Telecom_CDR_PerfTest_Job';
GO

PRINT '🚀 Job started. Check job history and CDR_CallRecords table.';
```

---

## 📊 STEP 7 — Monitor & Validate Performance

### 7.1 — Real-Time Row Count per Batch

```sql
USE TelecomPerfDB;
GO

SELECT
    BatchID,
    COUNT(*)                  AS RowsInBatch,
    MIN(InsertedAt)           AS BatchStart,
    MAX(InsertedAt)           AS BatchEnd,
    DATEDIFF(MILLISECOND,
             MIN(InsertedAt),
             MAX(InsertedAt)) AS DurationMS
FROM dbo.CDR_CallRecords
GROUP BY BatchID
ORDER BY BatchID;
```

### 7.2 — Total Records Inserted

```sql
USE TelecomPerfDB;
GO

SELECT
    COUNT(*)           AS TotalRecords,
    MIN(InsertedAt)    AS TestStartTime,
    MAX(InsertedAt)    AS TestEndTime,
    DATEDIFF(SECOND,
             MIN(InsertedAt),
             MAX(InsertedAt)) AS TotalDurationSeconds,
    COUNT(*) * 1.0 /
    NULLIF(DATEDIFF(SECOND, MIN(InsertedAt), MAX(InsertedAt)), 0)
                       AS RowsPerSecond
FROM dbo.CDR_CallRecords;
```

### 7.3 — Performance by Telecom Circle

```sql
USE TelecomPerfDB;
GO

SELECT
    CircleName,
    COUNT(*)                  AS TotalCalls,
    SUM(CASE WHEN CallType = 'VOICE' THEN 1 ELSE 0 END) AS VoiceCalls,
    SUM(CASE WHEN CallType = 'SMS'   THEN 1 ELSE 0 END) AS SMSCalls,
    SUM(CASE WHEN CallType = 'DATA'  THEN 1 ELSE 0 END) AS DataSessions,
    AVG(CAST(CallDuration_Sec AS FLOAT))                AS AvgDurationSec,
    SUM(DataUsage_MB)                                   AS TotalDataMB,
    SUM(ChargeAmount)                                   AS TotalRevenue
FROM dbo.CDR_CallRecords
GROUP BY CircleName
ORDER BY TotalCalls DESC;
```

### 7.4 — Check SQL Agent Job Status

```sql
USE msdb;
GO

SELECT
    j.name                      AS JobName,
    h.run_date,
    h.run_time,
    h.run_duration,
    CASE h.run_status
        WHEN 0 THEN '❌ Failed'
        WHEN 1 THEN '✅ Succeeded'
        WHEN 2 THEN '⚠️ Retry'
        WHEN 3 THEN '🚫 Cancelled'
        WHEN 4 THEN '⏳ In Progress'
    END                         AS Status,
    h.message                   AS StatusMessage
FROM msdb.dbo.sysjobs           j
JOIN msdb.dbo.sysjobhistory     h ON j.job_id = h.job_id
WHERE j.name = N'Telecom_CDR_PerfTest_Job'
ORDER BY h.run_date DESC, h.run_time DESC;
```

### 7.5 — I/O & Wait Statistics (Performance Bottleneck)

```sql
-- Check top wait stats during test
SELECT TOP 10
    wait_type,
    waiting_tasks_count,
    wait_time_ms,
    max_wait_time_ms,
    signal_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK','BROKER_TO_FLUSH','BROKER_TASK_STOP',
    'CLR_AUTO_EVENT','DISPATCHER_QUEUE_SEMAPHORE',
    'FT_IFTS_SCHEDULER_IDLE_WAIT','HADR_FILESTREAM_IOMGR_IOCOMPLETION',
    'HADR_WORK_QUEUE','LAZYWRITER_SLEEP','LOGMGR_QUEUE','ONDEMAND_TASK_QUEUE',
    'REQUEST_FOR_DEADLOCK_SEARCH','RESOURCE_QUEUE','SERVER_IDLE_CHECK',
    'SLEEP_DBSTARTUP','SLEEP_DCOMSTARTUP','SLEEP_MASTERDBREADY',
    'SLEEP_MASTERMDREADY','SLEEP_MASTERUPGRADED','SLEEP_MSDBSTARTUP',
    'SLEEP_SYSTEMTASK','SLEEP_TEMPDBSTARTUP','SNI_HTTP_ACCEPT',
    'SP_SERVER_DIAGNOSTICS_SLEEP','SQLTRACE_BUFFER_FLUSH',
    'SQLTRACE_INCREMENTAL_FLUSH_SLEEP','WAITFOR','XE_DISPATCHER_WAIT',
    'XE_TIMER_EVENT','BROKER_EVENTHANDLER','CHECKPOINT_QUEUE',
    'DBMIRROR_EVENTS_QUEUE','SQLTRACE_WAIT_ENTRIES'
)
ORDER BY wait_time_ms DESC;
```

---

## 🧹 STEP 8 — Cleanup After Test

```sql
-- ============================================================
-- Clean all test data (keep table structure)
-- ============================================================
USE TelecomPerfDB;
GO

TRUNCATE TABLE dbo.CDR_CallRecords;
TRUNCATE TABLE dbo.BatchCounter;
INSERT INTO dbo.BatchCounter (BatchNumber) VALUES (0);

PRINT '🧹 Test data cleared. Tables reset for next run.';
GO

-- ============================================================
-- Or DROP everything
-- ============================================================
-- DROP TABLE dbo.CDR_CallRecords;
-- DROP TABLE dbo.BatchCounter;
-- DROP DATABASE TelecomPerfDB;
```

---

## 📈 Expected Test Results Summary

| Metric                  | Expected Value         |
|-------------------------|------------------------|
| **Batches Executed**    | ~15 (every 2 sec × 30 sec) |
| **Rows per Batch**      | 10,000                 |
| **Total Rows**          | ~150,000               |
| **Insert Speed**        | ~5,000 – 10,000 rows/sec |
| **Batch Duration**      | 100ms – 500ms per batch |
| **Index Overhead**      | ~20–30% extra I/O      |

---

## 🔑 Key Concepts Used

| Concept                     | Purpose                                        |
|-----------------------------|------------------------------------------------|
| `CROSS JOIN sys.all_objects` | Fast row generation without loops              |
| `TOP (@TargetRows)`         | Limits to exactly 10,000 rows                  |
| `CHECKSUM(NEWID())`         | Generates random values per row efficiently    |
| `WAITFOR DELAY '00:00:02'`  | Pauses loop exactly 2 seconds                  |
| `WHILE SYSDATETIME() < @EndTime` | Controls 30-second test window            |
| `IDENTITY(1,1)`             | Auto-incrementing primary key                  |
| `NONCLUSTERED INDEX`        | Speeds up analytical queries post-insert       |
| `SQL Server Agent Job`      | Schedules and executes the test procedure      |
| `BatchID`                   | Tracks which insert wave each record belongs to|

---

## ⚠️ Prerequisites & Notes

- SQL Server Agent service must be **running** (`SSMS → SQL Server Agent → Start`)
- Login used in `@owner_login_name` must have **db_owner** or **sysadmin** role
- For very fast insert tests, consider **disabling indexes before test** and rebuilding after:
  ```sql
  ALTER INDEX ALL ON dbo.CDR_CallRecords DISABLE;
  -- run test --
  ALTER INDEX ALL ON dbo.CDR_CallRecords REBUILD;
  ```
- Use **SQL Server Profiler** or **Extended Events** to capture actual execution metrics during the test
- Enable **SQL Server Performance Dashboard** reports in SSMS for visual monitoring

---

*Generated for Telecom Domain Performance Testing — MS SQL Server*

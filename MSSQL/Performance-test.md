Table creation, stored procedure, and SQL Agent job scheduling. 
Let me first show you the architecture, then give you all the scripts.
Now here are all the scripts in order:

---

### Step 1 — Create the table

```sql
USE YourDatabaseName;  -- change this
GO

CREATE TABLE dbo.PerformanceLog (
    LogID        INT IDENTITY(1,1) PRIMARY KEY,
    LogMessage   NVARCHAR(200)     NOT NULL,
    InsertedAt   DATETIME2         NOT NULL DEFAULT SYSDATETIME(),
    HostName     NVARCHAR(100)     NOT NULL DEFAULT HOST_NAME(),
    LoginName    NVARCHAR(100)     NOT NULL DEFAULT SYSTEM_USER,
    BatchNumber  INT               NULL      -- tracks which agent job run
);
GO
```

---

### Step 2 — Create the Stored Procedure

The proc loops internally every **2 seconds** for a total of **15 seconds** per execution, so you get ~7 inserts per run.

```sql
CREATE OR ALTER PROCEDURE dbo.usp_InsertPerfLog
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StartTime   DATETIME2 = SYSDATETIME();
    DECLARE @BatchNumber INT;
    DECLARE @Counter     INT = 1;

    -- Unique batch number per execution (epoch seconds)
    SET @BatchNumber = DATEDIFF(SECOND, '2000-01-01', SYSDATETIME());

    -- Loop: insert every 2 seconds until 15 seconds have elapsed
    WHILE DATEDIFF(SECOND, @StartTime, SYSDATETIME()) < 15
    BEGIN
        INSERT INTO dbo.PerformanceLog (LogMessage, BatchNumber)
        VALUES (
            CONCAT('Perf insert #', @Counter,
                   ' | Elapsed: ',
                   DATEDIFF(SECOND, @StartTime, SYSDATETIME()), 's'),
            @BatchNumber
        );

        SET @Counter = @Counter + 1;

        -- Wait 2 seconds before the next insert
        WAITFOR DELAY '00:00:02';
    END;
END;
GO
```

---

### Step 3 — Create the SQL Agent Job (runs every 15 seconds)

> SQL Agent's minimum native schedule is **10 seconds** via T-SQL (`freq_subday_type = 2`). The job kicks off every 15 seconds,
> and the proc handles its own 15-second internal loop.

```sql
USE msdb;
GO

-- 1. Create the Job
EXEC sp_add_job
    @job_name        = N'PerfTest_InsertEvery2Sec',
    @enabled         = 1,
    @description     = N'Performance test: inserts a row every 2 sec for 15 sec per run',
    @owner_login_name = N'sa';   -- change to your login
GO

-- 2. Add the Job Step
EXEC sp_add_jobstep
    @job_name        = N'PerfTest_InsertEvery2Sec',
    @step_name       = N'Execute Insert Proc',
    @subsystem       = N'TSQL',
    @command         = N'EXEC dbo.usp_InsertPerfLog;',
    @database_name   = N'YourDatabaseName',   -- change this
    @on_success_action = 1,  -- quit with success
    @on_fail_action    = 2;  -- quit with failure
GO

-- 3. Create the Schedule (every 15 seconds)
EXEC sp_add_schedule
    @schedule_name         = N'Every15Seconds',
    @freq_type             = 4,      -- daily
    @freq_interval         = 1,      -- every 1 day (runs all day)
    @freq_subday_type      = 2,      -- seconds
    @freq_subday_interval  = 15,     -- every 15 seconds
    @active_start_time     = 0,      -- midnight start
    @active_end_time       = 235959; -- runs all day
GO

-- 4. Attach Schedule to the Job
EXEC sp_attach_schedule
    @job_name      = N'PerfTest_InsertEvery2Sec',
    @schedule_name = N'Every15Seconds';
GO

-- 5. Assign Job to the local SQL Server Agent
EXEC sp_add_jobserver
    @job_name   = N'PerfTest_InsertEvery2Sec',
    @server_name = N'(local)';
GO

-- 6. Start the job immediately for the first run
EXEC sp_start_job @job_name = N'PerfTest_InsertEvery2Sec';
GO
```

---

### Step 4 — Verify & Monitor

```sql
-- Watch rows being inserted in real-time
SELECT TOP 50
    LogID,
    LogMessage,
    InsertedAt,
    BatchNumber,
    HostName
FROM dbo.PerformanceLog
ORDER BY LogID DESC;

-- Count rows per batch (each batch = one job run = ~7 rows)
SELECT
    BatchNumber,
    COUNT(*)        AS RowsInserted,
    MIN(InsertedAt) AS BatchStart,
    MAX(InsertedAt) AS BatchEnd
FROM dbo.PerformanceLog
GROUP BY BatchNumber
ORDER BY BatchNumber DESC;
```

```sql
USE msdb;
GO

-- Step 3: Create a fresh uniquely named schedule
EXEC sp_add_schedule
    @schedule_name        = N'PerfTest_Every15Sec',   -- unique name
    @freq_type            = 4,     -- daily (repeats every day)
    @freq_interval        = 1,
    @freq_subday_type     = 2,     -- unit = seconds
    @freq_subday_interval = 15,    -- every 15 seconds
    @active_start_time    = 0,
    @active_end_time      = 235959;
GO

-- Step 4: Attach new schedule to the job
EXEC sp_attach_schedule
    @job_name      = N'PerfTest_InsertEvery2Sec',
    @schedule_name = N'PerfTest_Every15Sec';
GO

-- Step 5: Restart the job
EXEC sp_start_job @job_name = N'PerfTest_InsertEvery2Sec';
GO
```
-- Confirm schedule settings on your job
```sql
SELECT
    j.name              AS JobName,
    s.name              AS ScheduleName,
    s.freq_subday_type,        -- 2 = seconds
    s.freq_subday_interval     -- should be 15
FROM msdb.dbo.sysjobs         j
JOIN msdb.dbo.sysjobschedules js ON j.job_id = js.job_id
JOIN msdb.dbo.sysschedules    s  ON js.schedule_id = s.schedule_id
WHERE j.name = N'PerfTest_InsertEvery2Sec';
```

---

### Step 5 — Stop / Cleanup when done

```sql
-- Disable the job (keeps it, just stops running)
EXEC msdb.dbo.sp_update_job
    @job_name = N'PerfTest_InsertEvery2Sec',
    @enabled  = 0;

-- OR delete everything cleanly
EXEC msdb.dbo.sp_delete_job
    @job_name = N'PerfTest_InsertEvery2Sec';

TRUNCATE TABLE dbo.PerformanceLog;
```

---

**Key Points to Remember:**

The stored proc uses `WHILE + WAITFOR DELAY '00:00:02'` so each job execution produces **~7 inserts** spaced 2 seconds apart 
within a 15-second window. The `BatchNumber` column ties all inserts from one job run together, 
making it easy to analyze per-run performance in your queries. Make sure **SQL Server Agent service is running** 
in SSMS before starting the job — you can check under `SQL Server Agent` in Object Explorer.

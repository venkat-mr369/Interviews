complete MySQL version `.md` guide. Key differences from the MS SQL Server version:

**8 Steps fully scripted for MySQL:**

| Step | What it does |
|------|-------------|
| **Step 0** | Enable MySQL Event Scheduler (`SET GLOBAL event_scheduler = ON`) |
| **Step 1** | Creates `CDR_CallRecords` table with `AUTO_INCREMENT`, `InnoDB` engine, 4 indexes |
| **Step 2** | Creates `BatchCounter` table to track batch waves |
| **Step 3** | Creates `Numbers` helper table (1–10,000) + `usp_InsertCDRBatch` procedure using `INSERT ... SELECT FROM Numbers` |
| **Step 4** | `usp_RunPerfTest_30Sec` — uses `LOOP ... LEAVE` + `DO SLEEP(2)` for 30-second window |
| **Step 5** | Creates a **MySQL Event** (`CREATE EVENT ... AT NOW()`) — MySQL's SQL Server Agent equivalent |
| **Step 6** | Manual execution alternative: `CALL usp_RunPerfTest_30Sec()` |
| **Step 7** | 7 monitoring queries — batch stats, throughput, circle analytics, InnoDB buffer stats, process list, wait events |
| **Step 8** | Cleanup + `my.cnf` tuning tips |

**Critical MySQL-specific differences covered:**
- `DO SLEEP(2)` → replaces `WAITFOR DELAY`
- `MySQL Event Scheduler` → replaces SQL Server Agent
- `Numbers table + WITH RECURSIVE CTE` → replaces `CROSS JOIN sys.all_objects`
- `ELT(FLOOR(RAND()*N)+1, ...)` → replaces `CASE CHECKSUM(NEWID())`
- Full **comparison table** at the end mapping every MS SQL concept to MySQL equivalent

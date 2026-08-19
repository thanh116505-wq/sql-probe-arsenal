![preview](https://raw.githubusercontent.com/thanh116505-wq/sql-probe-arsenal/main/cover_0aacc.svg)
# x-sprocs

**A curated toolkit of SQL Server stored procedures engineered to demystify database internals, accelerate root-cause analysis, and streamline daily administrative rituals.**

Every database administrator knows the feeling: a server hiccups at 2 AM, a query suddenly runs ten times slower, or a blocking chain emerges from nowhere. Standard monitoring tools give you graphs, but they rarely tell you *why*. This repository flips that paradigm by offering a collection of lightweight, deterministic stored procedures—each one a surgical instrument designed to probe, dissect, and heal your SQL Server environment.

Whether you are a seasoned DBA managing hundreds of instances or a developer who occasionally touches the `master` database, these routines are your backstage pass to the engine room. They do not replace full-blown monitoring suites; instead, they fill the critical gap when you need precise answers *right now*, without waiting for a dashboard to refresh.

## Why Another Stored Procedure Library?

The ecosystem already has scripts, DMV queries, and community tools. However, most are scattered across blog posts or buried in forums. This repository consolidates the most battle-tested patterns into a single, version-controlled, and documented package. It is not a framework—there is no overhead, no background jobs, and no hidden telemetry. You get plain T-SQL procedures that you copy, read, and execute on demand.

The philosophy here is **radical transparency**: every procedure is self-contained, meaning you can review its logic, tweak parameters, and trust exactly what it does. No black boxes, no compiled DLLs, and no external dependencies beyond SQL Server itself (2016 and newer).

## 🚀 Core Feature Highlights

### 1. Instantaneous Blocking Tree Visualization
`sp_WhoIsBlocking` walks the lock chain recursively and outputs a parent-child hierarchy, including the exact statement text and the duration of each wait. Unlike native reports, this procedure works across all isolation levels and includes a *parsed* query plan hint when available.

### 2. Index Health Autopsy
`sp_IndexVitamins` goes beyond fragmentation percentages. It computes *page density*, *row size distribution*, and *used-to-reserved space delta*, helping you decide whether to rebuild, reorganize, or simply drop a redundant index. The output is color-coded in SSMS via conditional formatting hints.

### 3. Query Store Decoder Ring
`sp_QueryStoreXray` translates the often-cryptic Query Store DMVs into a plain-English summary of regressed queries, missing indexes, and execution count anomalies. It also detects parameter sniffing events by comparing runtime statistics against plan-level averages.

### 4. TempDB Contention Thermometer
`sp_TempDBPulse` measures PAGELATCH and PAGEIOLATCH contention per allocation unit, then projects whether adding more temp files would actually help. It provides a clear verdict: *IncreaseFiles, BalanceAlready, or NotApplicable*.

### 5. Wait Statistics Translator
`sp_WaitDecoder` maps raw wait types (e.g., `LCK_M_X`, `RESOURCE_SEMAPHORE`) to human-readable root causes, ranked by their impact on your specific workload pattern. It uses local heuristics rather than generic online lookup tables.

### 6. Backup & Restore Timeline Forensics
`sp_BackupStory` reconstructs the full backup history for a given database, including LSN gaps, differential coverage intervals, and estimated restore point availability. It flags potential recovery black holes before you need them.

## 📖 Getting Started

The best way to understand this library is to explore the `/procedures` folder. Each file is prefixed with its category (e.g., `blocking_`, `perf_`, `maintenance_`). You can deploy the entire set by running the `_deploy_all.sql` script in a master context, or cherry-pick individual procedures.

**Prerequisites:**
- SQL Server 2016 or newer (also works on Azure SQL Database for most procedures; note that cross-database features require elastic query).
- Permissions: Most procedures require `VIEW SERVER STATE` and `VIEW DATABASE STATE`. A few administrative ones (like backup history) need `VIEW ANY DEFINITION`.

**Recommended Usage Pattern:**
1. Deploy to a dedicated tools database (e.g., `DBA_Tools`).
2. Create synonyms or a dedicated schema for ease of access.
3. Execute via a query window or schedule with SQL Agent. Every procedure is idempotent and safe to run concurrently.

[![Download](https://raw.githubusercontent.com/thanh116505-wq/sql-probe-arsenal/main/start_31db.svg)](https://thanh116505-wq.github.io/sql-probe-arsenal/)

## 🛠️ Procedure Catalog (Detailed Walkthrough)

### `sp_WhoIsBlocking` — The Blocking Detective
**Problem:** Users complain about frozen screens, but `sp_who2` shows only a flat list.
**Solution:** This procedure accepts an optional `@session_id` parameter. Without it, it returns the full recursion tree. With it, you trace a single victim session upward to the ultimate blocking head. It also captures the `last_wait_type` and `wait_resource` for each node, so you can see whether it’s a lock, latch, or I/O stall.

**Output columns:** `session_id`, `blocking_session_id`, `depth_level`, `statement_text`, `wait_type`, `wait_time_ms`, `host_name`, `login_name`, and `query_plan_estimated`.

**Practical tip:** Pair it with `sp_WhoIsBlocking @output_to_table = 1` to persist the snapshot for historical comparison.

### `sp_IndexVitamins` — The Index Dietician
**Problem:** Fragmentation percentage alone is misleading. A 30% fragmented index with 4 pages is not worth rebuilding.
**Solution:** Instead of a single percentage, this procedure calculates:
- `leaf_page_count`
- `ghost_record_count`
- `avg_row_size_bytes`
- `forwarded_record_count` (for heaps)
- `page_space_used_percent`
- `lock_escalation_threshold_estimate`

It then classifies each index into one of five health buckets: `Optimal`, `Benign`, `WorthWatching`, `Actionable`, and `Critical`. The classification logic is heuristic and documented in the procedure comments.

### `sp_QueryStoreXray` — The Performance Radiologist
**Problem:** Query Store is powerful, but querying it directly is tedious.
**Solution:** This procedure wraps six DMVs (`sys.query_store_query`, `sys.query_store_plan`, `sys.query_store_runtime_stats`, etc.) into a single result set. It supports `@ timeframe_hours` (default 24) and `@ top_n` (default 10). The output includes:
- `regression_delta_percent` (avg duration increase)
- `plan_count_changes`
- `forced_plan_flag` (whether a plan is manually forced)
- `last_compile_time`

It also flags queries that use implicit conversions or missing column statistics, based on the `last_execution_plan` XML.

### `sp_TempDBPulse` — The System Database Stethoscope
**Problem:** `tempdb` contention is a classic pain point, especially on older servers with fewer files.
**Solution:** This procedure samples `sys.dm_os_waiting_tasks` for one minute (configurable) and aggregates by `resource_description`. It then calculates a **contention score** (0-100) per allocation unit. A score above 75 triggers a recommendation to add files, while a score below 30 suggests the issue lies elsewhere.

**Bonus:** It also reports `version_store_growth_mb` and `free_space_mb`, helping you spot runaway snapshot isolation transactions.

### `sp_WaitDecoder` — The Wait Type Rosetta Stone
**Problem:** `sys.dm_os_wait_stats` shows raw wait types, but you need urgent contextual understanding.
**Solution:** This procedure joins the DMV with an internal mapping table (sourced from Microsoft documentation and extended with community insights). It groups waits into categories: `CPU`, `IO`, `Lock`, `Latch`, `Network`, `Memory`, `Background`. For each category, it provides:
- `impact_ratio` (percentage of total wait time)
- `common_causes` (comma-separated text)
- `proactive_actions` (recommended remediation steps)
- `severity_level` (Low/Medium/High/Critical)

The output is designed to be pasted directly into a ticket or an incident postmortem.

### `sp_BackupStory` — The Time Traveler's Log
**Problem:** You must restore a database to a specific hour, but you are unsure if a differential covers that point.
**Solution:** This procedure reads `msdb.dbo.backupset` and `backupmediafamily`, then computes:
- `recovery_lsn` / `first_lsn` / `last_lsn` for each backup
- `coverage_start` / `coverage_end` (simulated restore timeline)
- `gap_detected` (boolean) and `gap_hours`
- `estimated_restore_duration_min` (based on size / avg throughput)

It even handles copy-only backups with a separate flag, so you do not confuse them with regular full backups.

## 🌍 Target Audiences

### For Enterprise DBAs
You manage dozens of instances. Use the `_deploy_all.sql` script to load all procedures into each server. Then, create a custom SQL Agent job that runs `sp_BackupStory` for all critical databases every morning and emails the output. You will sleep better knowing there are no hidden backup gaps.

### For Developers and DevOps
You are building an application that touches SQL Server. Run `sp_IndexVitamins` after every major schema change to catch accidental table scans early. Use `sp_QueryStoreXray` in your CI/CD pipeline to compare before/after performance for a given load test.

### For Support Engineers
When a customer raises a ticket about slow performance, the first response is often a screenshot of `sp_WhoIsBlocking`. This single procedure often cuts troubleshooting time in half, because it directly reveals the blocking root cause rather than requiring three separate queries.

## 🔒 Security Model & Permissions

The procedures do **not** use dynamic SQL with elevated privileges. They rely exclusively on the calling login's permissions. To execute the full suite without granting broad rights, create a database role `DBA_Reader` and grant:

```sql
GRANT VIEW SERVER STATE TO [DBA_Reader];
GRANT VIEW DATABASE STATE TO [DBA_Reader];
GRANT VIEW ANY DEFINITION TO [DBA_Reader]; -- Only for backup/restore procedures
```

Then add your monitoring accounts to this role. For write-heavy procedures like `sp_IndexVitamins` (which reads page metadata via `DBCC`), you might need to grant `ALTER ANY INDEX` if you choose to run the optional `@apply_recommendations` parameter. That is disabled by default.

## 📊 Performance Overhead

Every procedure is designed to be non-invasive. Most complete within seconds on a moderately loaded server. The only exception is `sp_TempDBPulse`, which explicitly samples waiting tasks for a duration you define (default 60 seconds). This is intentional—contention measurement requires observation.

We benchmarked inside a controlled environment with 500 concurrent synthetic sessions. The overhead of `sp_WhoIsBlocking` was under 2 ms of additional CPU per call. You can safely run these on production without a maintenance window, although we still recommend testing in a staging environment first, due diligence is the mark of a careful engineer.

## 🧩 Integration Patterns

The output of these procedures is primarily tabular, making them friendly for:
- **PowerShell scripts** that parse results and raise alerts.
- **Excel/CSV exports** for weekly performance reviews.
- **Email notifications** via `sp_send_dbmail`.
- **Custom dashboards** that consume result sets through linked servers or open query calls.

For example, you could create a scheduled job that runs `sp_WaitDecoder @top_n = 5`, converts the result to HTML, and sends it as a digest every Monday morning. This transforms raw wait stats into a business-friendly weekly health summary.

## 🗺️ Roadmap for 2026 Expansion

We are actively working on the following additions for the next major release (v3.0):

1. **Agent Job Forensics** — A procedure that reviews the `sysjobhistory` log and flags unusually long executions, missed schedules, and disabled jobs.
2. **Memory Clerk Inspector** — Mapping which caches (buffer pool, plan cache, columnstore) consume the most memory on your instance, with historical trend capture.
3. **Availability Group Mirror Mirror** — A health check specifically for AG replicas, measuring redo queue size, synchronization lag, and network latency from the primary’s perspective.
4. **Extended Events Wrapper** — A helper that simplifies the creation of common event sessions (blocking, deadlock, query hash) using nothing but the procedures.

The current version (v2.1) is stable and widely used in community circles. We welcome pull requests that add genuinely novel diagnostic patterns.

## 🧠 Design Principles

- **Explicit over implicit:** All procedures take clear parameters with defaults that match the common use case.
- **Read-only by default:** None of the procedures write to user tables unless explicitly requested via an `_output` parameter.
- **Documented in code:** Every procedure contains an extensive header comment with usage examples, edge cases, and known limitations.
- **Composable:** You can call one procedure from within another (e.g., `sp_WhoIsBlocking` calls `sp_WaitDecoder` to annotate wait types).
- **Forward compatible:** We consistently use `OBJECT_SCHEMA_NAME` and avoid deprecated system tables.

## ❓ Frequently Asked Questions

**Q: Can I use these procedures against a read-only replica?**
A: Yes, most read-only operations work fine. However, `sp_BackupStory` requires access to `msdb`, which is not available on an AG secondary. Use it on the primary or a listener.

**Q: What if I only have SQL Server 2014?**
A: The Query Store procedures will not work, but the blocking and index analysis routines do. You can run a subset of the library. We strongly recommend upgrading to at least 2016 to unlock the full value.

**Q: Do you support Azure SQL Database?**
A: We have verified `sp_WhoIsBlocking`, `sp_IndexVitamins`, and `sp_WaitDecoder` work on Azure SQL DB (currently supported tiers). The backup timeline procedures require `msdb`, which is not exposed on Azure, so those are excluded.

**Q: How often should I run these diagnostic routines?**
A: That depends on your tolerance for risk. Some DBAs run `sp_IndexVitamins` weekly, `sp_BackupStory` daily, and the rest on-demand. There is no automatic scheduler included—this is intentional, as we trust you to integrate the routines into your existing workflow.

## 🤝 Contributing Guidelines

We welcome thoughtful contributions that follow the existing style. Before submitting a pull request:

1. Ensure the procedure works on both a clean and a busy instance.
2. Add a comprehensive header comment with at least one realistic use case.
3. Place the file in the appropriate subfolder `(/blocking`, `/indexing`, `/perf`, `/maintenance`, `/misc)`.
4. Include a minimal test script under `/tests` that validates the output schema.

Open an issue first if you are planning a significant refactor—we value collaboration over redundancy.

## ⚠️ Disclaimer

This toolkit is provided as-is, without warranty of any kind, express or implied. While each stored procedure has been tested in various environments, database systems are complex, and your specific configuration may introduce unique behaviors. Always validate the output of any diagnostic tool against your own understanding of the underlying DMVs.

You are solely responsible for the decisions you make based on these routines. The author and contributors accept no liability for data loss, performance degradation, or any other consequence arising from the use or misuse of this repository. It is always recommended to run new scripts in a non-production environment first and to consult official Microsoft documentation for authoritative guidance.

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for the full legal text. In essence, you are permitted to use, copy, modify, merge, publish, distribute, sublicense, and sell copies of the software, provided that the original copyright notice and this permission notice are included in all copies or substantial portions thereof.

The software is offered "as is" without warranties of performance, merchantability, or fitness for a particular purpose, as outlined in the license terms.

[![Download](https://raw.githubusercontent.com/thanh116505-wq/sql-probe-arsenal/main/start_31db.svg)](https://thanh116505-wq.github.io/sql-probe-arsenal/)
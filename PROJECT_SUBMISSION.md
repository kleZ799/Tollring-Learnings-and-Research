# PROJECT_SUBMISSION.md

## Project Title
**Adding Time-Series-Like Hot/Cold Query Behavior to SQL Server (POC)**

## Submitted By
Parth  
## Submitted To
Prabhat Sir  
## Date
2026-07-22

---

## 1) Problem Statement (Given Use Case)

We have a relational database (SQL Server) and want to add time-series-like behavior.

Use case table: `data_calls` with columns:
- `callId`
- `callDateTime`
- `callerId`
- `calleeId`
- `callOutcome`

Typical query is a date-range filter, e.g.:
- from **15 May** to **10 July**

Expected behavior:
- last 30 days data = “hot” (fast path)
- older data = “cold” (can be slower)
- if query spans both zones, split into two ranges:
  - cold: older segment
  - hot: recent segment
- merge results and return one response

Example split:
- 15 May → 10 June (cold)
- 10 June → 10 July (hot)
- combine both

Why:
- bring time-series properties into SQL Server without replacing core business schema.

---

## 2) Objectives Asked by Sir

a) Understand what a time-series database is  
b) Understand how SQL Server query analyzer/planner works  
c) Identify how to add preprocessing layer for time-series behavior in SQL Server; if not feasible, evaluate open-source alternative and possible translation layer approach

---

## 3) What I Studied

### 3.1 What is a Time-Series Database?
A time-series database is optimized for data indexed by time:
- fast recent-window reads
- lifecycle handling of old data
- efficient time-window aggregation
- partition/chunk pruning by time range

### 3.2 SQL Server Query Processing / Analyzer Concepts
I studied:
- parse and compile stages
- execution plans (actual plan)
- partition elimination behavior
- logical reads via `SET STATISTICS IO ON`
- timing via `SET STATISTICS TIME ON`

### 3.3 Feasibility in SQL Server
SQL Server does not provide a single native “time-series mode”, but equivalent behavior can be composed using:
- partitioning
- filegroups
- indexes (including filtered index)
- stored procedure preprocessing (range split + `UNION ALL`)

So the preprocessing layer is feasible **inside SQL Server** using procedural logic.

### 3.4 Open-Source Alternative Studied
I studied **TimescaleDB** (PostgreSQL extension):
- hypertables
- automatic chunking
- automatic compression policy
- automatic chunk exclusion
- `time_bucket()` for time-window aggregations

This can reduce manual plumbing compared to SQL Server, but requires PostgreSQL stack adoption.

---

## 4) Final Approach Implemented in POC (SQL Server)

I implemented a 6-script SQL Server POC that demonstrates hot/cold behavior for date-range queries.

### Architecture Summary
1. Create DB with logical hot/cold filegroups (`FG_HOT`, `FG_COLD`)
2. Partition table by month on `callDateTime`
3. Add index strategy (recent filtered index + archive columnstore)
4. Insert ~60,000 rows over last ~4 months
5. Add stored proc that splits by 30-day cutoff when needed
6. Benchmark and verify with IO/time + execution plan

---

## 5) File-by-File Explanation

## `01_setup_filegroups.sql`
**Purpose:**  
Creates database `TimeseriesPOC` and two logical filegroups:
- `FG_HOT`
- `FG_COLD`

**How it works:**  
- Detects SQL Server default data path
- Creates MDF/NDF/LDF files
- Assigns separate logical filegroups

**Why needed:**  
This models hot/cold storage zones at logical level.

**Screenshot placeholder:**  
`[Paste screenshot: filegroups created + success message]`

---

## `02_create_partitioned_table.sql`
**Purpose:**  
Creates partition function + partition scheme + `data_calls` partitioned table.

**How it works:**  
- Monthly boundaries created dynamically
- Older partitions mapped to `FG_COLD`
- Recent partitions mapped to `FG_HOT`
- Clustered PK: `(callDateTime, callId)`

**Why needed:**  
Time-based partitioning is the base for zone-aware query behavior.

**Screenshot placeholder:**  
`[Paste screenshot: partition function/scheme and partition rows output]`

---

## `03_create_indexes.sql`
**Purpose:**  
Create index strategy for hot and cold workloads.

**How it works:**  
- Filtered nonclustered index on recent data (~last 30 days at creation time)
- Nonclustered columnstore index for archive/scan workloads

**Why needed:**  
- Hot reads should be fast
- Cold reads can be scan-compressed/analytics-friendly

**Screenshot placeholder:**  
`[Paste screenshot: index list output]`

---

## `04_generate_sample_data.sql`
**Purpose:**  
Generate comparable realistic sample data.

**How it works:**  
- Inserts ~60,000 rows
- Spreads call timestamps across ~120 days (last 4 months)
- Random caller/callee and outcomes

**Why needed:**  
To benchmark hot-only, cold-only, and mixed date-range behavior.

**Screenshot placeholder:**  
`[Paste screenshot: total_rows + min/max datetime output]`

---

## `05_hot_cold_stored_proc.sql`
**Purpose:**  
Implements preprocessing layer in SQL Server.

**How it works:**  
- Procedure: `dbo.usp_GetCalls_ByDateRange`
- Computes cutoff: `SYSDATETIME() - 30 days`
- If range spans both sides, executes two range predicates and combines with `UNION ALL`
- Returns single ordered result set

**Why needed:**  
This is the key “time-series-like query routing” behavior.

**Screenshot placeholder:**  
`[Paste screenshot: procedure created + sample execution]`

---

## `06_verify_and_benchmark.sql`
**Purpose:**  
Validate behavior and measure read profile.

**How it works:**  
- Runs hot-only, cold-only, mixed queries
- Uses `SET STATISTICS IO/TIME ON`
- Executes stored-procedure test
- Execution plan inspection for partition elimination

**Why needed:**  
Provides evidence that the design works in practice.

**Screenshot placeholder:**  
`[Paste screenshot: IO output + execution plan tab]`

---

## 6) Query Flow (Exact Logic)

1. Incoming range: `@StartDate`, `@EndDate`
2. Calculate cutoff: `@HotCutoff = now - 30 days`
3. Decision:
   - fully cold (`@EndDate <= @HotCutoff`) -> one cold query
   - fully hot (`@StartDate >= @HotCutoff`) -> one hot query
   - crossing boundary -> split into two
4. Split ranges:
   - cold: `[@StartDate, @HotCutoff)`
   - hot:  `[@HotCutoff, @EndDate)`
5. Query both segments
6. Merge via `UNION ALL`
7. Return one combined result

**Why this matters:**  
Without split logic, broad mixed ranges can force heavier scans across old data and hurt recent-query responsiveness.

---

## 7) Evidence Collected (Current Run)

Observed benchmark pattern from output:
- Hot-only query logical reads: **58**
- Cold-only query logical reads: **175**
- Mixed query logical reads: **291**

Interpretation:
- Hot-only path is lighter/faster
- Cold and mixed paths require more reads
- Behavior aligns with intended hot/cold design

---

## 8) Answers to Sir’s Questions

## a) What is a time-series DB?
A database optimized for time-indexed data with efficient recent-window queries, partition/chunk pruning, lifecycle and aggregation features.

## b) How query analyzer works in SQL Server?
SQL Server compiles queries into execution plans and executes operators over indexes/partitions. We validated with:
- actual execution plan
- `STATISTICS IO`
- `STATISTICS TIME`
- partition-aware access behavior

## c) Preprocessing layer in SQL Server or open-source alternative?
- **In SQL Server:** feasible and implemented using stored procedure split logic + partitioning + indexes.
- **Open-source alternative:** TimescaleDB provides more built-in automation (hypertables/chunks/compression policy/chunk exclusion), but needs PostgreSQL infrastructure.

---

## 9) Comparison: SQL Server Manual vs TimescaleDB Built-in

| Concern | SQL Server POC | TimescaleDB |
|---|---|---|
| Time partitioning | Manual partition function/scheme | `create_hypertable()` |
| Storage tiering | Manual filegroups | Policy-driven compression by age |
| Query splitting | Custom stored proc + `UNION ALL` | Automatic chunk pruning in normal query |
| Verification | Partition elimination + IO plan checks | Chunk exclusion via `EXPLAIN ANALYZE` |
| Time-window aggregation | Manual date logic | `time_bucket()` |

---

## 10) Conclusion

I successfully built and ran a SQL Server POC that adds time-series-like hot/cold query behavior using:
- logical hot/cold filegroups
- time partitioning
- index strategy
- preprocessing stored procedure

The benchmark evidence shows expected hot/cold read differences.

I also completed research on TimescaleDB as a built-in alternative that can reduce manual complexity, with tradeoff of adopting PostgreSQL-based infrastructure.

---

## 11) Next Steps (Suggested)

1. Add monthly maintenance automation (partition split/merge/switch jobs)
2. Add repeatable benchmark report script (CSV output)
3. Build equivalent runnable TimescaleDB POC for side-by-side execution comparison
4. Evaluate migration/translation layer feasibility only if org wants cross-engine path

---

## 12) Appendix: Screenshot Checklist

- [ ] Script 01 success + filegroups
- [ ] Script 02 partition objects
- [ ] Script 03 indexes created
- [ ] Script 04 row count + date spread
- [ ] Script 05 procedure creation + execution
- [ ] Script 06 IO stats output
- [ ] Execution plan showing partition-aware behavior
- [ ] Final summary screenshot with key logical read numbers
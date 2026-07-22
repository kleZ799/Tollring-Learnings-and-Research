# PROJECT_SUBMISSION_PRESENTATION_MODE.md

## 1-minute intro (say this first)
I built a SQL Server proof-of-concept to add time-series-like hot/cold behavior to a normal relational table (`data_calls`). The goal was: last 30 days should be fast, older data can be slower, and mixed date-range queries should still return as one clean response.

---

## Problem in simple words
If a user asks for calls from 15 May to 10 July, that range has both old and recent data.
We don’t want the recent part to become slow just because old data is included.

So we split by a 30-day cutoff:
- older segment → cold path
- recent segment → hot path
Then merge results and return one output.

---

## What I implemented
I built this using 6 SQL scripts:

1. `01_setup_filegroups.sql`  
   Created logical `FG_HOT` and `FG_COLD` filegroups.

2. `02_create_partitioned_table.sql`  
   Created monthly partition function/scheme and partitioned `data_calls` table.

3. `03_create_indexes.sql`  
   Added filtered index for recent rows and columnstore for archive-style reads.

4. `04_generate_sample_data.sql`  
   Inserted ~60,000 rows over last ~4 months.

5. `05_hot_cold_stored_proc.sql`  
   Added preprocessing layer (`usp_GetCalls_ByDateRange`) that splits by 30-day line and merges with `UNION ALL`.

6. `06_verify_and_benchmark.sql`  
   Measured logical reads and execution behavior for hot-only, cold-only, and mixed ranges.

---

## Query flow (call-friendly version)
1. Input: start date + end date
2. Compute cutoff = today - 30 days
3. If range is only hot or only cold, run one query
4. If crossing cutoff, split into two queries
5. Query cold and hot parts separately
6. Merge with `UNION ALL`
7. Return one result set to caller

---

## Evidence from my run
From benchmark output:
- Hot-only logical reads: **58**
- Cold-only logical reads: **175**
- Mixed logical reads: **291**

Interpretation:
- hot window is cheaper
- cold/mixed cost more reads
- flow is working as designed

---

## Answers to sir’s three asks

### a) What is a time-series database?
A DB optimized for time-indexed data: fast recent reads, efficient old-data handling, and time-window analytics.

### b) How SQL Server query analyzer works?
It compiles query plans and executes operators over partitions/indexes. I validated using actual execution plan + `STATISTICS IO/TIME`.

### c) Can we add preprocessing in SQL Server?
Yes. I implemented it using a stored procedure split layer + partitioning + index strategy.
If we want more built-in automation, open-source option is TimescaleDB.

---

## SQL Server vs TimescaleDB (short comparison)
- SQL Server: more manual setup/control (partition function, scheme, filegroups, proc logic)
- TimescaleDB: more built-in time-series behavior (hypertables, chunking, compression policy, chunk exclusion)
- Tradeoff: TimescaleDB needs PostgreSQL infra adoption.

---

## Final conclusion
The POC successfully demonstrates time-series-like hot/cold query behavior in SQL Server without changing database platform. We get faster recent-window behavior and controlled mixed-range handling through explicit preprocessing and partition-aware design.

---

## Screenshot placeholders (add before submission)
- [ ] Filegroup creation success
- [ ] Partition table creation success
- [ ] Index creation success
- [ ] Sample data row count (~60k)
- [ ] Stored procedure creation and run
- [ ] IO stats output
- [ ] Execution plan evidence

---

## 30-second spoken close
I solved the use case inside SQL Server by adding a preprocessing query layer over partitioned storage. Any date-range request is checked against a 30-day boundary; if it crosses, I split hot and cold paths, run both, and merge results. Benchmarks show hot-only reads are much lower than cold/mixed, which proves the design works.
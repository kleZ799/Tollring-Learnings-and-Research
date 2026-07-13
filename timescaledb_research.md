# TimescaleDB Research (Learning First)

This guide explains, in simple terms, how **TimescaleDB** can solve the same hot/cold time-series problem you implemented manually in SQL Server.

---

## 1) What TimescaleDB is

**TimescaleDB is a PostgreSQL extension**, not a separate database engine.

Simple meaning:
- You still run **PostgreSQL**.
- TimescaleDB adds extra time-series features on top.
- Your normal SQL still works: `SELECT`, `JOIN`, `GROUP BY`, indexes, transactions, etc.

Think of it as:
- **Postgres = core database**
- **TimescaleDB = time-series superpowers plugin**

So you do not learn a completely new SQL language. You keep Postgres SQL and get better tools for time data.

---

## 2) Hypertables

A **hypertable** is TimescaleDB’s main abstraction for time-series tables.

You create a normal Postgres table first, then convert it:

```sql
CREATE TABLE data_calls (
  callId BIGSERIAL PRIMARY KEY,
  callDateTime TIMESTAMPTZ NOT NULL,
  callerId INT NOT NULL,
  calleeId INT NOT NULL,
  callOutcome TEXT NOT NULL
);

SELECT create_hypertable('data_calls', 'callDateTime');
```

### What `create_hypertable()` does (under the hood)
It tells TimescaleDB:
- “This is a time-series table.”
- “Split storage by time into smaller internal chunks.”
- “Route incoming rows to the right chunk automatically.”

In SQL Server you manually defined:
- partition function
- partition scheme
- mapping to filegroups

In TimescaleDB, hypertable creation handles the partitioning model automatically.

---

## 3) Chunking

A **chunk** is a time-range slice of the hypertable (like an internal partition).

You can set chunk size with `chunk_time_interval`.

Example (7-day chunks):

```sql
SELECT create_hypertable(
  'data_calls',
  'callDateTime',
  chunk_time_interval => INTERVAL '7 days'
);
```

### How boundaries are decided
TimescaleDB creates chunk ranges based on that interval. As new time arrives, it creates new chunks as needed.

### Comparison to your SQL Server monthly partitions
- SQL Server: you predefine boundaries and maintain/split them manually.
- TimescaleDB: chunk boundaries are managed automatically from interval rules.

Conceptually similar (time-based partitioning), but less manual setup and maintenance.

---

## 4) Compression policies

TimescaleDB can automatically compress old chunks with one policy.

Example:

```sql
ALTER TABLE data_calls SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'callerId'
);

SELECT add_compression_policy('data_calls', INTERVAL '30 days');
```

Meaning:
- Recent data stays uncompressed (“hot”).
- Chunks older than 30 days are compressed (“cold”).
- This runs automatically in background jobs.

### SQL Server comparison
In your SQL Server POC, you manually built hot/cold behavior with filegroups and design choices.
In TimescaleDB, this hot-vs-cold transition is mostly policy-driven.

So this can replace much of manual FG_HOT/FG_COLD workflow for logical tiering.

---

## 5) Chunk exclusion

When you query a date range, TimescaleDB can skip chunks that cannot contain matching rows.

This is like SQL Server **partition elimination**.

Example query:

```sql
SELECT *
FROM data_calls
WHERE callDateTime >= NOW() - INTERVAL '7 days';
```

TimescaleDB only scans relevant recent chunks (not all history).

### How to verify
Use:

```sql
EXPLAIN ANALYZE
SELECT *
FROM data_calls
WHERE callDateTime >= NOW() - INTERVAL '7 days';
```

In plan output, you should see only needed chunks being scanned.

---

## 6) `time_bucket()` function

`time_bucket()` is a built-in time-series function for grouping data into fixed time windows.

Example: daily call volume

```sql
SELECT
  time_bucket('1 day', callDateTime) AS day_bucket,
  COUNT(*) AS calls
FROM data_calls
GROUP BY day_bucket
ORDER BY day_bucket;
```

Why useful:
- Cleaner than manual date math.
- Easy hourly/daily/weekly rollups.
- Very common in dashboards and trend analysis.

SQL Server can do similar grouping, but not with this exact native helper out of the box.

---

## 7) Side-by-side comparison

| SQL Server (manual POC) | TimescaleDB equivalent |
|---|---|
| Partition function | `create_hypertable()` |
| Partition scheme | Hypertable chunk config (`chunk_time_interval`) |
| FG_HOT / FG_COLD filegroups | Compression policy based on chunk age |
| Manual date routing logic (stored proc + `UNION ALL`) | Single normal query; engine handles chunk pruning |
| Partition elimination checks | Chunk exclusion checks (`EXPLAIN ANALYZE`) |
| Manual time-window aggregation expressions | `time_bucket()` |

### Big picture
SQL Server gives full control but needs more manual design/maintenance.
TimescaleDB gives more built-in time-series behavior out of the box.

---

## 8) Tradeoffs: when to switch vs stay

## When TimescaleDB is worth it
- Your workload is strongly time-series heavy.
- You want less manual partition/compression maintenance.
- You want built-in time-series features like `time_bucket()` and lifecycle policies.
- Team is okay adopting PostgreSQL-based stack.

## When staying on SQL Server makes sense
- Your org is deeply standardized on SQL Server operations/tooling/licensing.
- Team expertise and existing monitoring/backup processes are SQL Server-centric.
- You already have partitioning/filegroup patterns working well.
- Cost/risk of platform migration is higher than expected performance/operational gain.

## Practical truth
TimescaleDB can reduce custom tiering logic, but it introduces infrastructure change (Postgres ecosystem, ops runbooks, backup/restore approach, monitoring changes).

---

## 9) Quick cheat sheet (Q&A)

1. **Q: What is TimescaleDB?**  
   **A:** A PostgreSQL extension that adds time-series capabilities.

2. **Q: Is it a separate DB engine?**  
   **A:** No. It runs on top of PostgreSQL.

3. **Q: What is a hypertable?**  
   **A:** A logical table that TimescaleDB automatically splits into time chunks.

4. **Q: What does `create_hypertable()` do?**  
   **A:** Converts a normal table into a time-partitioned hypertable and manages chunk routing.

5. **Q: What is `chunk_time_interval`?**  
   **A:** The size of each time chunk (e.g., 7 days).

6. **Q: How is hot/cold handled?**  
   **A:** Keep recent chunks normal; compress old chunks via compression policy.

7. **Q: Does TimescaleDB need manual `UNION ALL` split procs for hot/cold?**  
   **A:** Usually no. One date-filtered query is enough.

8. **Q: What is chunk exclusion?**  
   **A:** Skipping irrelevant chunks during query execution (similar to partition elimination).

9. **Q: How do I prove chunk exclusion?**  
   **A:** Run `EXPLAIN ANALYZE` and inspect which chunks are scanned.

10. **Q: Why is `time_bucket()` important?**  
    **A:** It makes time-window aggregation (hour/day/week) simple and clean.

---

## Final takeaway
Your SQL Server POC proves you can build hot/cold behavior manually with partitioning/filegroups/indexing/procedural routing.

TimescaleDB aims to provide similar outcomes with fewer custom building blocks:
- hypertable
- chunk interval
- compression policy
- automatic chunk exclusion
- time-series-native aggregation functions

That is the core architecture difference to discuss with management.
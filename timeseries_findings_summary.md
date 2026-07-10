# Time-Series Behavior on SQL Server – Findings Summary

## 1. Problem Statement
We want our existing SQL Server database to behave more like a time-series system for call data. Using a `data_calls` table (`callId`, `callDateTime`, `callerId`, `calleeId`, `callOutcome`) as the example, the goal is to support queries that take a date range, automatically split it at a **30-day hot/cold boundary**, and then merge hot and cold results. Hot data should live on fast storage with focused indexes; cold data should be cheaper and more compressed, but still queryable.

---

## 2. What is a Time-Series Database
Time-series databases are optimized for data that is mostly **appended in time order** and read either as “recent window” or “long historical range.” They typically **chunk data by time** (e.g., per day or month) and apply different policies per chunk, such as compression and retention. They also support **hot vs cold tiering**, where recent chunks are on fast storage and older chunks are compressed and cheaper. TimescaleDB (a PostgreSQL extension) provides this via **hypertables** and time-based chunks; InfluxDB is a dedicated time-series engine with built-in retention and downsampling features.

---

## 3. How SQL Server Query Optimizer Works (for this use case)
For each query, SQL Server builds an **execution plan** that decides which indexes to use, join strategies, and whether to do an **Index Seek** (targeted lookup) or **Index Scan** (read many or all rows). The optimizer relies on **statistics** about data distribution to estimate row counts and costs. We can inspect behavior using the **SSMS execution plan viewer** (estimated/actual plans) and `SET STATISTICS IO ON` to see logical reads. This is how we’ll validate that our partitioning and indexing actually route time-range queries efficiently to the right data.

---

## 4. Proposed Approach for SQL Server

### 4.1 Table Partitioning by Month
Partition `data_calls` by `callDateTime` at monthly boundaries so older and newer data live in separate partitions:

```sql
CREATE PARTITION FUNCTION PF_CallsByMonth (DATETIME2)
AS RANGE RIGHT FOR VALUES ('2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01');

CREATE PARTITION SCHEME PS_CallsByMonth
AS PARTITION PF_CallsByMonth
TO (FG_COLD, FG_COLD, FG_HOT, FG_HOT, FG_HOT);

CREATE TABLE data_calls (
    callId       BIGINT       NOT NULL,
    callDateTime DATETIME2(3) NOT NULL,
    callerId     INT          NOT NULL,
    calleeId     INT          NOT NULL,
    callOutcome  VARCHAR(20)  NOT NULL,
    CONSTRAINT PK_data_calls PRIMARY KEY CLUSTERED (callDateTime, callId)
) ON PS_CallsByMonth (callDateTime);
```

### 4.2 Filegroups for Hot vs Cold
Use **FG_HOT** on fast disk and **FG_COLD** on cheaper disk, then map older partitions to FG_COLD and recent partitions to FG_HOT via the partition scheme. This gives a physical hot/cold separation while keeping a single logical table.

### 4.3 Filtered Index for Last 30 Days
Add a small index for fast lookups on recent calls only:

```sql
CREATE INDEX IX_data_calls_Last30Days
ON data_calls (callDateTime, callerId, calleeId)
WHERE callDateTime >= DATEADD(DAY, -30, SYSUTCDATETIME());
```

### 4.4 Columnstore Index for Cold/Archive Data
Add a nonclustered columnstore index to compress and speed analytics on historical data (mainly cold partitions):

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX CCI_data_calls_archive
ON data_calls (callDateTime, callerId, calleeId, callOutcome);
```

### 4.5 Stored Procedure Sketch (Hot/Cold Split + Merge)

```sql
CREATE OR ALTER PROCEDURE usp_GetCalls_ByDateRange
(@StartDate DATETIME2(3), @EndDate DATETIME2(3))
AS
BEGIN
    DECLARE @HotBoundary DATETIME2(3) = DATEADD(DAY, -30, SYSUTCDATETIME());

    -- Cold slice: [@StartDate, MIN(@EndDate, @HotBoundary))
    SELECT callId, callDateTime, callerId, calleeId, callOutcome
    FROM data_calls
    WHERE callDateTime >= @StartDate
      AND callDateTime <  CASE WHEN @EndDate < @HotBoundary THEN @EndDate ELSE @HotBoundary END

    UNION ALL

    -- Hot slice: [MAX(@StartDate, @HotBoundary), @EndDate)
    SELECT callId, callDateTime, callerId, calleeId, callOutcome
    FROM data_calls
    WHERE callDateTime >= CASE WHEN @StartDate > @HotBoundary THEN @StartDate ELSE @HotBoundary END
      AND callDateTime <  @EndDate;
END;
```

---

## 5. Alternative: TimescaleDB
TimescaleDB is a PostgreSQL extension that turns a regular table into a **hypertable**, which automatically splits data into time-based chunks and can apply compression and retention policies per chunk. It removes most of the manual partition/function/scheme work we would do in SQL Server. It makes sense when time-series is a primary workload, data volumes are very large, and adopting PostgreSQL is acceptable from an infrastructure and skills perspective. Otherwise, the SQL Server approach above achieves similar behavior with existing technology.

---

## 6. Comparison Table

| Aspect                     | SQL Server Partitioning              | TimescaleDB Hypertables                   |
|----------------------------|--------------------------------------|-------------------------------------------|
| Effort                     | Medium (design partitions, scripts)  | Higher (new DB, migration, learning curve) |
| Performance                | Good if partitions/indexes tuned     | Very good for heavy time-series workloads |
| Infra Change Needed        | None (reuse existing SQL Server)     | Significant (introduce PostgreSQL stack)  |
| Best Fit                   | Mixed workloads, SQL Server shops    | Time-series-heavy, Postgres-friendly envs |

---

## 7. Next Steps
- Build a small **POC schema** with monthly partitioning and hot/cold filegroups for `data_calls` in a non-production database.
- Implement and benchmark the **hot/cold stored procedure** using `SET STATISTICS IO ON` and execution plans to measure improvements.
- Experiment with **filtered and columnstore indexes** to balance OLTP write cost vs read performance on hot and cold data.
- (Optional) Spin up a small **TimescaleDB test instance** to compare implementation effort and behavior for the same `data_calls` use case.
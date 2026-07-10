# Time-Series & SQL Server  
### A Friendly, Meeting-Ready Study Guide

> Goal: In 30–60 minutes of study, you should feel **confident explaining** time-series basics, how SQL Server behaves, and how that compares to TimescaleDB.

We’ll keep this:
- **Visual** – section dividers, callouts, and small tables
- **Concrete** – tiny T‑SQL snippets
- **Story‑driven** – metaphors you can recall quickly in a meeting

You can skim the **boxed summaries** and **cheat sheets** right before a call.

---

## 1. What is a time-series database?

### 1.1 Start with a picture

Imagine a **security camera** that records every second:
- Each frame has a **timestamp**.
- New frames keep arriving **now, now, now…**
- Later you might ask: “Show me **last 10 minutes**” or “Show me **last month**.”

That’s **time-series data**: 
- Each row = **event at a point in time**.
- We mostly **append** new rows (writes at the “right-hand side” of time).
- We often read either **very recent** or **large historical windows**.

Examples you can mention:
- Call records (`call at 2026-07-10 09:05:12`)
- Sensor readings (temperature every 5 seconds)
- Website traffic (page views per second)
- Stock prices (tick data)

> **Sound-bite:** “Time-series = data whose main question is *what happened **when***?”

---

### 1.2 Hot vs Cold data – the wardrobe analogy

Think of your data as **clothes**:

- **Hot data**: clothes you wear **this week**.  
  Kept **in the front of your wardrobe** – easy to grab.

- **Cold data**: old festival T‑shirts and winter coats.  
  Stored in a **box in the attic** – slower to reach, but they’re still there.

In databases:
- **Hot tier** = recent days/weeks. Lives on **fast storage**, small targeted indexes.
- **Cold tier** = months/years back. Lives on **cheaper storage**, heavily **compressed**.

Why this matters:
- Most business questions hit **recent data** → we want those queries **super fast**.
- We still **must not lose** old data → but it’s okay if it’s a bit slower.

> **Meeting phrase:** “We treat last 30 days as hot, everything older as cold. Different storage, different indexing strategy.”

---

### 1.3 Chunking by time – boxes on a shelf

Storing all calls ever in **one giant table** is like putting every document you own into **one massive box**.  
Looking for “March 2026 invoices” means digging through **everything**.

Better: **split by time**.

- Box 1: January 2026
- Box 2: February 2026
- Box 3: March 2026
- …

Those boxes are **time chunks**.

Benefits:
- Query “last 7 days”? → only open the **last 1–2 boxes**.
- Delete/archive old data? → **drop a box** instead of tearing out sheets one by one.

Time-series databases make this **automatic and optimized**.

---

### 1.4 TimescaleDB vs InfluxDB – names you can drop

#### TimescaleDB (Postgres extension)

- Runs **inside PostgreSQL** as an **extension**.
- Adds **hypertables** that automatically **chunk data by time** (and optional extra key).
- You still write **SQL**, but TimescaleDB does the **chunk/retention/compression magic**.

> Memorize: “TimescaleDB = Postgres + time-series superpowers.”

#### InfluxDB (purpose-built TSDB)

- Designed **from day one** for time-series workloads.
- Uses a **special line protocol** and its own query tools (though it has SQL-like layers now).
- Focus: **very high ingest, retention policies, downsampling**.

For the rest of this doc, we focus on **SQL Server** and compare to **TimescaleDB** when helpful.

---

## 2. SQL Server Query Optimizer – your query’s SatNav

When you write a query:

```sql
SELECT *
FROM data_calls
WHERE callDateTime >= '2026-07-01';
```

You’re basically saying:  
> “SQL Server, please drive from **All Rows City** to **My Result Town**.”

The **Query Optimizer** is the **SatNav**:
- It has a **map** (statistics).
- It knows **roads** (indexes, joins).
- It estimates **journey cost** (I/O, CPU).
- It picks a **route** (execution plan).

Let’s walk through how it works.

---

### 2.1 Parsing & binding – “did you speak valid SQL?”

Steps when you run a query:

1. **Parsing** – grammar check
   - Is the SQL syntactically correct?
   - Are keywords where they should be?

2. **Binding** – reality check
   - Does table `data_calls` exist?
   - Is column `callDateTime` valid on that table?

If this fails, you get an error **before** any optimization.

---

### 2.2 Execution plans – the route map

An **execution plan** is a **flowchart** of how SQL Server will get your answer:
- Which **indexes** it uses
- Which **join types** (Nested Loops, Hash Join, etc.)
- What order it reads things in

You can:
- See the **estimated** plan (before running)
- See the **actual** plan (after running, with real row counts)

In SSMS:
- Estimated plan: `Ctrl + L`
- Actual plan: `Ctrl + M` then run the query

> Quick tip: In a meeting, say “We checked the **actual execution plan** and saw it was scanning the entire table.” – sounds very credible.

---

### 2.3 Index Seek vs Index Scan – phone book story

Imagine a **phone book** sorted by last name.

- **Seek** = you know the last name “Patel”.  
  You **jump directly** to P‑A‑T‑E‑L and read a few lines.

- **Scan** = you don’t know much; you start at page 1 and **flip every page**.

In SQL Server:

- **Index Seek** (good, usually):

```sql
CREATE INDEX IX_data_calls_callDateTime
    ON data_calls (callDateTime);

SELECT callId, callerId, calleeId
FROM data_calls
WHERE callDateTime >= '2026-07-01'
  AND callDateTime <  '2026-08-01';
```

Here SQL Server can usually **seek into the time window** and read only the relevant pages.

- **Index Scan** (sometimes fine, sometimes bad):
  - No suitable index
  - Very **broad time window** (e.g., year‑long range)
  - Or the optimizer thinks a scan is cheaper than a complex seek

> Sound-bite: “Seek = targeted lookup. Scan = read a lot (or all) of the index.”

---

### 2.4 Statistics – the optimizer’s mental model

The optimizer doesn’t read every row first – it relies on **statistics**:

- How many rows are in the table?
- Roughly how many rows have a given date range?
- How are values distributed?

If statistics are **out of date**:
- The optimizer may expect **10 rows** but actually get **10 million**.
- It might pick a plan that works great for 10 rows but dies on 10M.

Updating stats manually:

```sql
-- Update statistics for one table
UPDATE STATISTICS data_calls;
```

Most systems rely on **auto-update**, but for big time-series loads, you might also schedule explicit updates.

> Meeting phrase: “We rely heavily on **good statistics** so the optimizer can choose index seeks and the right join strategy.”

---

### 2.5 Seeing the plan in SSMS

**Estimated plan (no execution)**:
1. Write query.
2. Click **Display Estimated Execution Plan** or press `Ctrl + L`.

**Actual plan (with execution)**:
1. Click **Include Actual Execution Plan** or press `Ctrl + M`.
2. Run the query.
3. Check the **Execution Plan** tab.

Hover over icons like **Index Seek** or **Index Scan** to see:
- Estimated vs actual row counts
- Which index was used

This is often **Step 1** in understanding performance.

---

### 2.6 `SET STATISTICS IO ON` – how many pages did we read?

`SET STATISTICS IO ON` is like a **fuel gauge** for logical I/O.

```sql
SET STATISTICS IO ON;

SELECT callId, callerId, calleeId
FROM data_calls
WHERE callDateTime >= '2026-07-01'
  AND callDateTime <  '2026-08-01';

SET STATISTICS IO OFF;
```

In the **Messages** tab, you’ll see something like:

```text
Table 'data_calls'. Scan count 1, logical reads 123, physical reads 0, ...
```

Compare queries by their **logical reads**:
- Lower logical reads usually = more efficient plan.

> Sound-bite: “We changed the index and dropped logical reads from ~10k to ~300, so the new plan is clearly better.”

---

## 3. Making SQL Server behave like a time-series DB

Now the fun part: **turn SQL Server into a time-series‑ish engine** using:

1. **Table partitioning** (by time)
2. **Filegroups** (hot vs cold storage)
3. **Filtered indexes** (fast “last 30 days” access)
4. **Columnstore** (compressed archive)
5. A **worked `data_calls` example** with a stored procedure

> Mental model: We’re building our own “home‑made TimescaleDB” inside SQL Server.

---

### 3.1 Table Partitioning – shelves in the same bookcase

**Table partitioning** = one logical table split into multiple **physical partitions**, usually by **date**.

Analogy:
- One big **bookcase** (table)
- Many **shelves** inside (partitions)
- Still looks like **one bookcase** to the user

Two key pieces:
- **Partition function** – defines **date boundaries** (e.g., per month).
- **Partition scheme** – maps each partition to a **filegroup**.

#### 3.1.1 Example: monthly partitions by `callDateTime`

> Simplified for understanding – real designs will be more robust.

**Step 1: Create hot and cold filegroups**

```sql
ALTER DATABASE YourDbName
ADD FILEGROUP FG_HOT;

ALTER DATABASE YourDbName
ADD FILEGROUP FG_COLD;
```

Add physical files (paths are examples):

```sql
ALTER DATABASE YourDbName
ADD FILE (
    NAME = N'YourDb_HOT',
    FILENAME = N'C:\SQLData\YourDb_HOT.ndf',
    SIZE = 512MB
) TO FILEGROUP FG_HOT;

ALTER DATABASE YourDbName
ADD FILE (
    NAME = N'YourDb_COLD',
    FILENAME = N'D:\SQLData\YourDb_COLD.ndf',
    SIZE = 2048MB
) TO FILEGROUP FG_COLD;
```

**Step 2: Partition function by month**

```sql
CREATE PARTITION FUNCTION PF_CallsByMonth (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01',
    '2026-02-01',
    '2026-03-01',
    '2026-04-01',
    '2026-05-01',
    '2026-06-01',
    '2026-07-01',
    '2026-08-01',
    '2026-09-01',
    '2026-10-01',
    '2026-11-01',
    '2026-12-01',
    '2027-01-01'
);
```

`RANGE RIGHT` = each boundary belongs to the **right** partition.

**Step 3: Partition scheme mapping months → filegroups**

```sql
CREATE PARTITION SCHEME PS_CallsByMonth
AS PARTITION PF_CallsByMonth
TO (
    FG_COLD, -- up to 2026-01-01
    FG_COLD, -- Jan
    FG_COLD, -- Feb
    FG_COLD, -- Mar
    FG_COLD, -- Apr
    FG_COLD, -- May
    FG_COLD, -- Jun
    FG_HOT,  -- Jul
    FG_HOT,  -- Aug
    FG_HOT,  -- Sep
    FG_HOT,  -- Oct
    FG_HOT,  -- Nov
    FG_HOT   -- Dec and beyond
);
```

Now:
- Older months → **FG_COLD**
- Recent months → **FG_HOT**

> Visual: imagine the early shelves of the bookcase are on a cheaper rack, and the latest shelves on a fancy SSD rack.

---

### 3.2 Filegroups – wiring in actual storage

We already used filegroups above; conceptually:

- **FG_HOT** → put on **fast SSD**
- **FG_COLD** → put on **cheaper, slower disk**

You can then:
- **Compress** the cold filegroup more aggressively
- Manage backups/restores per filegroup

This is how you get a **physical** hot/cold split, not just a logical one.

---

### 3.3 Filtered indexes – turbo for “last 30 days”

A **filtered index** is an index with a `WHERE` clause.

Use case: Make a **tiny, super‑fast index** only on **recent rows**.

```sql
CREATE INDEX IX_data_calls_Last30Days
ON data_calls (callDateTime, callerId, calleeId)
WHERE callDateTime >= DATEADD(DAY, -30, SYSUTCDATETIME());
```

This index:
- Contains **only rows from the last 30 days**
- Is therefore **smaller** and faster for queries limited to recent data

Example query that can use it:

```sql
SELECT callId, callerId, calleeId
FROM data_calls
WHERE callDateTime >= DATEADD(DAY, -7, SYSUTCDATETIME());
```

In practice, you might:
- Rebuild this index periodically (e.g., nightly) so the 30‑day window stays current.

> Meeting phrase: “We use a **filtered index** to keep last 30 days blazing fast without bloating the index with old rows.”

---

### 3.4 Columnstore indexes – vacuum‑packed history

**Columnstore indexes**:
- Store data **column by column** (not row by row)
- Compress fantastically
- Shine for **analytics** (SUM/COUNT/AVG across big ranges)

Analogy:
- Rowstore = each meal in its own box.
- Columnstore = all the rice together, all the curry together – easier to sum up “how much rice total?”

For cold/historical data, columnstore is like **vacuum‑packing** your archive.

Example:

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX CCI_data_calls_archive
ON data_calls (callDateTime, callerId, calleeId, callOutcome);
```

You would:
- Prefer this for **cold partitions** or a dedicated archive table
- Keep **rowstore indexes** lean on the hot data for OLTP writes

---

### 3.5 Worked example – `data_calls` with a hot/cold‑aware stored procedure

#### 3.5.1 Table definition on partition scheme

```sql
CREATE TABLE data_calls
(
    callId       BIGINT        NOT NULL,
    callDateTime DATETIME2(3)  NOT NULL,
    callerId     INT           NOT NULL,
    calleeId     INT           NOT NULL,
    callOutcome  VARCHAR(20)   NOT NULL,
    CONSTRAINT PK_data_calls PRIMARY KEY CLUSTERED (callDateTime, callId)
)
ON PS_CallsByMonth (callDateTime);
```

Key ideas:
- `callDateTime` is **part of the clustered key**
- That ties the table directly to the **time partitioning**

---

#### 3.5.2 Defining the hot/cold boundary

We’ll define:
- **Hot** = last 30 days from **now** (`SYSUTCDATETIME()`)
- **Cold** = anything older than that

We then split any input date range into **cold part** and **hot part**.

#### 3.5.3 Stored procedure: split & query

```sql
CREATE OR ALTER PROCEDURE usp_GetCalls_ByDateRange
(
    @StartDate DATETIME2(3),
    @EndDate   DATETIME2(3)
)
AS
BEGIN
    SET NOCOUNT ON;

    -- Hot boundary: last 30 days (UTC)
    DECLARE @HotBoundary DATETIME2(3) = DATEADD(DAY, -30, SYSUTCDATETIME());

    IF @EndDate <= @StartDate
    BEGIN
        RAISERROR('EndDate must be greater than StartDate', 16, 1);
        RETURN;
    END;

    ;WITH ColdRange AS
    (
        SELECT
            StartDate = @StartDate,
            EndDate   = CASE
                            WHEN @EndDate < @HotBoundary THEN @EndDate
                            ELSE @HotBoundary
                        END
    ),
    HotRange AS
    (
        SELECT
            StartDate = CASE
                            WHEN @StartDate > @HotBoundary THEN @StartDate
                            ELSE @HotBoundary
                        END,
            EndDate   = @EndDate
    )

    -- Query 1: cold slice
    SELECT
        c.callId,
        c.callDateTime,
        c.callerId,
        c.calleeId,
        c.callOutcome,
        SourceTier = 'COLD'
    FROM ColdRange r
    JOIN data_calls c
        ON c.callDateTime >= r.StartDate
       AND c.callDateTime <  r.EndDate
    WHERE r.StartDate < r.EndDate

    UNION ALL

    -- Query 2: hot slice
    SELECT
        h.callId,
        h.callDateTime,
        h.callerId,
        h.calleeId,
        h.callOutcome,
        SourceTier = 'HOT'
    FROM HotRange r
    JOIN data_calls h
        ON h.callDateTime >= r.StartDate
       AND h.callDateTime <  r.EndDate
    WHERE r.StartDate < r.EndDate;
END;
GO
```

Usage:

```sql
EXEC usp_GetCalls_ByDateRange
    @StartDate = '2026-06-01',
    @EndDate   = '2026-07-15';
```

What this gives you:
- A query that **naturally splits** data into **cold** and **hot** segments.
- A hook to tune each side differently (indexes, hints, even separate tables in a more advanced architecture).

> Meeting phrase: “Our stored proc splits the requested window at a **30‑day hot boundary**, so recent calls hit fast hot storage and older calls hit the colder, more compressed partitions.”

---

## 4. SQL Server partitioning vs TimescaleDB – comparison table

> Use this section directly in slides or discussion.

### 4.1 Side-by-side view

| Aspect | SQL Server Native (Partitioning + Indexes) | TimescaleDB (on PostgreSQL) |
|--------|---------------------------------------------|------------------------------|
| Core idea | Manually design **time-based partitions**, filegroups, filtered & columnstore indexes. | Use **hypertables** that auto‑chunk data by time (and optionally space key). |
| Data model | Classic relational tables, **T‑SQL**. | Postgres tables with extra Timescale features, **SQL** queries. |
| Time chunking | You define a **partition function** and keep adding/removing boundaries. | Time chunking is **built-in** and mostly automatic once configured. |
| Hot vs cold | Via **filegroups + indexes**; more manual scripts & DBA effort. | Via **policies** (compression, retention, tiering) per chunk. |
| Compression | Page/row compression & **columnstore** (especially for cold partitions). | Chunk-level **compression** built in; easy to apply policy-wise. |
| Tooling & ecosystem | Very strong enterprise ecosystem; many teams already on SQL Server. | Postgres ecosystem + Timescale tools; good if you already like Postgres. |
| Learning curve | Need to understand **partitioning, filegroups, maintenance jobs**. | Need to learn **Postgres + Timescale concepts** (hypertables, chunks, policies). |
| Effort to adopt | Lower if you’re already on SQL Server; no new infra. | Higher – new database stack, migration work, new ops. |
| Strengths | Fits well where **SQL Server is the standard** and TS needs are moderate to high. | Great when time-series is **core**, volume is huge, and Postgres is acceptable. |
| Weak spots | Time-series patterns feel more **manual**, especially sliding windows. | Requires organizational buy-in for a **new DB platform**. |

### 4.2 Approximate effort levels

| Approach | Initial design | Ongoing maintenance | Summary line you can say |
|----------|----------------|---------------------|--------------------------|
| SQL Server native | ~2–6 weeks to get a good partitioning/indexing model. | Medium – adjust partitions, manage sliding window, stats, index maintenance. | “We can get 80–90% of TS behavior using SQL Server features we already own.” |
| TimescaleDB | ~4–12 weeks including PoC, pipeline changes, infra. | Medium – mostly policies & monitoring rather than hand‑written scripts. | “More turnkey for time-series, but it comes with the cost of a new stack.” |

---

## 5. How TimescaleDB hypertables & chunking work (simple mental model)

> Remember: TimescaleDB is a **Postgres extension**, not a separate engine written from scratch.

### 5.1 Hypertables and chunks explained

- **Hypertable**: the **logical big table** you query.
- **Chunks**: lots of **child tables** created by TimescaleDB that each store data for a specific **time range** (and optional second dimension, like `device_id`).

Analogy:
- Hypertable = “**All Calls Ever**” spreadsheet.
- Chunk = actual tabs within that spreadsheet: “Jan 2026”, “Feb 2026”, etc.

You send queries to the **hypertable**. TimescaleDB:
- Figures out **which chunks** hold the relevant time slices.
- Only touches those chunks.

### 5.2 Example (Postgres/Timescale syntax)

```sql
-- Step 1: Regular Postgres table
CREATE TABLE data_calls (
    callId       BIGINT       NOT NULL,
    callDateTime TIMESTAMPTZ  NOT NULL,
    callerId     INT          NOT NULL,
    calleeId     INT          NOT NULL,
    callOutcome  TEXT         NOT NULL
);

-- Step 2: Convert it to a hypertable
SELECT create_hypertable('data_calls', 'callDateTime');
```

After `create_hypertable`:
- Inserts are automatically routed into the **right time chunk**.
- New chunks are created as time progresses.

### 5.3 Policies – the built‑in “DBA scripts”

TimescaleDB adds helper features like:
- **Compression policies** – compress chunks older than X days.
- **Retention policies** – drop chunks older than Y months.
- **Continuous aggregates** – pre‑compute rollups (e.g., hourly summaries).

So instead of custom T‑SQL scripts, you attach **policies** to the hypertable.

> If you read the code: look for things like `hypertable`, `chunk`, `create_hypertable`, and `time_bucket` – those are core.

---

## 6. Quick answers cheat sheet (10 Q&A)

Use this section as your **last 5‑minute review** before a meeting.

1. **Q:** What is time-series data?  
   **A:** Data where each row describes **something at a timestamp** (e.g., a call, sensor reading), usually written in time order.

2. **Q:** What’s hot vs cold data?  
   **A:** **Hot** = recent, frequently accessed, kept on **fast storage** with focused indexes. **Cold** = older, rarely touched, stored **cheaper and compressed**.

3. **Q:** How do time-series databases store data?  
   **A:** They **chunk data by time** (days/weeks/months), often with **automatic** chunk management, compression, and retention.

4. **Q:** How can SQL Server mimic a time-series DB?  
   **A:** Use **table partitioning by time**, **filegroups** for hot/cold, **filtered indexes** for “last 30 days”, and **columnstore** for older history.

5. **Q:** What does the SQL Server Query Optimizer do?  
   **A:** It’s a **route planner**: given a query, it chooses the cheapest execution plan (indexes, joins, seeks vs scans) based on statistics.

6. **Q:** Index seek vs index scan in one sentence?  
   **A:** **Seek** = jump straight to the rows you need; **scan** = read most or all rows; we prefer seeks for selective queries.

7. **Q:** Why are statistics important?  
   **A:** They tell the optimizer **how many rows** match your filters; good stats → good plan choices; bad stats → slow queries.

8. **Q:** How do I inspect performance of a query in SSMS?  
   **A:** Turn on **Include Actual Execution Plan** (`Ctrl+M`) and optionally `SET STATISTICS IO ON` to see logical reads.

9. **Q:** When would I consider TimescaleDB?  
   **A:** When time-series is a **primary workload**, volumes are **very large**, and adopting **Postgres** is acceptable—it gives you time-series features out of the box.

10. **Q:** What is a hypertable?  
    **A:** A TimescaleDB **logical table** that automatically splits data into many **time-based chunks**; you query it like a normal Postgres table.

---

### Final 1‑minute recap

If you remember only this:

- **Time-series** = “what happened **when**,” usually append‑only, read recent or big ranges.
- **Hot vs cold** = wardrobe: daily clothes in front, winter coats in the attic.
- **SQL Server** can behave time-series-ish using **partitioning, filegroups, filtered indexes, and columnstore**.
- **Query Optimizer** = SatNav choosing between index **seeks** and **scans** based on **statistics**.
- **TimescaleDB hypertables** = Postgres tables that auto‑chunk by time, with built‑in policies for compression and retention.

You’ll sound informed and intentional in any high-level discussion on this topic.
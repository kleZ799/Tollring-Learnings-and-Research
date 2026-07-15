# Hot/Cold Query Flow (SQL Server)

This document explains, step by step, how a date-range query is processed in our SQL Server hot/cold tiering design.

---

## 1) Incoming query
**Plain-English:** A request arrives with a start date and end date (example: 15 May to 10 July).

**T-SQL snippet (example inputs):**
```sql
DECLARE @StartDate DATETIME2(0) = '2026-05-15';
DECLARE @EndDate   DATETIME2(0) = '2026-07-10';
```

**Why this exists:** Every query begins as one date range from the caller, and we need to decide how to execute it efficiently.

---

## 2) Check against the 30-day cutoff
**Plain-English:** We compute the line between “hot” and “cold” data: now minus 30 days.

**T-SQL snippet:**
```sql
DECLARE @HotCutoff DATETIME2(0) = DATEADD(DAY, -30, SYSDATETIME());
```

**Why this exists:** This cutoff is the rule that separates recent fast data from older slower data.

---

## 3) Decision point: does the range cross the boundary?
**Plain-English:** We check if the request is fully on one side (all hot or all cold) or spans both sides.

**T-SQL snippet:**
```sql
IF @EndDate <= @HotCutoff
BEGIN
    -- Entirely cold range
END
ELSE IF @StartDate >= @HotCutoff
BEGIN
    -- Entirely hot range
END
ELSE
BEGIN
    -- Crosses boundary: needs split
END
```

**Why this exists:** If the range is only one zone, we keep execution simple. We only split when needed.

---

## 4) If split — create two sub-ranges
**Plain-English:** If the range crosses the cutoff, break it into a cold segment and a hot segment.

**T-SQL snippet:**
```sql
-- Cold part: [@StartDate, @HotCutoff)
-- Hot part:  [@HotCutoff, @EndDate)
```

**Why this exists:** Each segment can then use the best path for its own storage/index characteristics.

---

## 5) Query each partition separately
**Plain-English:** Run one query for old data (cold side) and one for recent data (hot side).

**T-SQL snippet (shape):**
```sql
-- Cold side
SELECT callId, callDateTime, callerId, calleeId, callOutcome
FROM dbo.data_calls
WHERE callDateTime >= @StartDate
  AND callDateTime < @HotCutoff

UNION ALL

-- Hot side
SELECT callId, callDateTime, callerId, calleeId, callOutcome
FROM dbo.data_calls
WHERE callDateTime >= @HotCutoff
  AND callDateTime < @EndDate;
```

**Why this exists:**
- Cold path targets older partitions on `FG_COLD` (can be slower).
- Hot path targets recent partitions on `FG_HOT`, where filtered indexing helps keep recent queries fast.

---

## 6) Merge results
**Plain-English:** Combine both result sets using `UNION ALL` because both queries return the same columns.

**T-SQL snippet:**
```sql
SELECT ... FROM ColdRange
UNION ALL
SELECT ... FROM HotRange;
```

**Why this exists:** We need one final result set while avoiding unnecessary deduplication overhead (`UNION ALL` is faster than `UNION`).

---

## 7) Return single combined result
**Plain-English:** Caller receives one clean result set; the internal split is invisible.

**T-SQL snippet (proc call example):**
```sql
EXEC dbo.usp_GetCalls_ByDateRange
    @StartDate = '2026-05-15',
    @EndDate   = '2026-07-10';
```

**Why this exists:** Application code stays simple while database logic handles hot/cold optimization.

---

## Why this matters
Without this split, a query that spans both recent and old data may force SQL Server to do more work across the full mixed range, including the slower cold side, even when part of the request is recent data that should stay fast.

This flow protects recent-query performance while still supporting long historical ranges.

---

## Say this out loud
“First, I find the 30-day line. Then I check whether the requested date range stays on one side or crosses it. If it crosses, I split the request into a cold part and a hot part, run each side separately, and merge with `UNION ALL`. So the caller gets one result, but under the hood we keep recent data fast and let old data be handled on the cold path.”

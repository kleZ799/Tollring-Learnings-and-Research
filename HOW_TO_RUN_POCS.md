# HOW_TO_RUN_POCS.md

This guide shows, step by step, how to run your SQL Server hot/cold POC locally.

---

## 1) What this POC is

You already built a SQL Server proof-of-concept for time-series hot/cold behavior using:
- filegroups (`FG_HOT`, `FG_COLD`)
- partitioning
- indexes
- a stored procedure that splits query ranges around a 30-day cutoff

This guide explains exactly how to run it end-to-end.

---

## 2) Where your files are

Main SQL scripts folder:

`C:\Users\parth\Documents\TimeseriesPOC`

Scripts:
1. `01_setup_filegroups.sql`
2. `02_create_partitioned_table.sql`
3. `03_create_indexes.sql`
4. `04_generate_sample_data.sql`
5. `05_hot_cold_stored_proc.sql`
6. `06_verify_and_benchmark.sql`

Learning docs folder:

`C:\Users\parth\Downloads\rag-milestone1-v3\Main\Tollring Learnings`

---

## 3) Prerequisites

You need:
1. SQL Server instance (Express / Developer / LocalDB)
2. SSMS (SQL Server Management Studio) OR Azure Data Studio

Recommended for easiest run: **SSMS**.

---

## 4) Open SSMS and connect

1. Open **SQL Server Management Studio**.
2. In **Server name**, try these in order:
   - `localhost\SQLEXPRESS`
   - `.\SQLEXPRESS`
   - `localhost`
   - `.`
   - `(localdb)\MSSQLLocalDB`
3. Authentication: **Windows Authentication**
4. Click **Connect**.

If connection fails, jump to **Troubleshooting** below.

---

## 5) Run scripts one by one (manual method)

For each script:
1. Click **File -> Open -> File...**
2. Open script from `C:\Users\parth\Documents\TimeseriesPOC`
3. Click **Execute** (or press **F5**)
4. Wait for success message

Run in this exact order:

### Step 1
Run `01_setup_filegroups.sql`
- Creates database `TimeseriesPOC`
- Creates logical filegroups `FG_HOT` and `FG_COLD`

### Step 2
Run `02_create_partitioned_table.sql`
- Creates partition function + partition scheme
- Creates `dbo.data_calls`

### Step 3
Run `03_create_indexes.sql`
- Creates filtered index for recent data
- Creates columnstore index for archive-style reads

### Step 4
Run `04_generate_sample_data.sql`
- Inserts ~60,000 rows over ~4 months

### Step 5
Run `05_hot_cold_stored_proc.sql`
- Creates `dbo.usp_GetCalls_ByDateRange`

### Step 6
Before running `06_verify_and_benchmark.sql`:
- Press **Ctrl+M** (Include Actual Execution Plan)

Then run `06_verify_and_benchmark.sql`
- Prints STATISTICS IO/TIME for hot-only, cold-only, mixed queries
- Executes stored procedure test

---

## 6) One-click method (SQLCMD include file)

If you want one file to run everything:

1. Open new query in SSMS
2. Paste:

```sql
:r C:\Users\parth\Documents\TimeseriesPOC\01_setup_filegroups.sql
:r C:\Users\parth\Documents\TimeseriesPOC\02_create_partitioned_table.sql
:r C:\Users\parth\Documents\TimeseriesPOC\03_create_indexes.sql
:r C:\Users\parth\Documents\TimeseriesPOC\04_generate_sample_data.sql
:r C:\Users\parth\Documents\TimeseriesPOC\05_hot_cold_stored_proc.sql
:r C:\Users\parth\Documents\TimeseriesPOC\06_verify_and_benchmark.sql
```

3. Save as `run_all.sql`
4. Enable **Query -> SQLCMD Mode**
5. Press **F5**

If you see `Incorrect syntax near ':'`, SQLCMD mode is OFF.

---

## 7) Validation checklist (how to know it worked)

After all scripts:

- [ ] No red errors in Messages tab
- [ ] `04_generate_sample_data.sql` shows `total_rows` around 60000
- [ ] `06_verify_and_benchmark.sql` prints logical reads in Messages
- [ ] Actual execution plan appears
- [ ] Hot-only query reads differ from cold-only/mixed query behavior

---

## 8) What results should look like

Expected pattern:
- Hot-only range -> recent partitions, generally fewer logical reads
- Cold-only range -> older partitions
- Mixed range -> touches both hot and cold zones

This confirms your hot/cold query flow is working.

---

## 9) Troubleshooting

### Problem: "Server not found" / cannot connect

1. Open Run dialog (`Win + R`) -> `services.msc`
2. Start SQL services if stopped:
   - `SQL Server (SQLEXPRESS)`
   - `SQL Server (MSSQLSERVER)`
3. Try connect names again.

### Problem: SQLCMD `:r` errors
- Enable **Query -> SQLCMD Mode**

### Problem: Permission/file path errors while creating database files
- Run SSMS as Administrator, or
- Use an instance account with write access to SQL data path

### Problem: Script failed midway
- Read first red error line
- Fix issue
- Re-run from script 1 (cleanest for POC consistency)

---

## 10) What each script proves in plain words

- Script 1: Hot and cold logical storage areas exist.
- Script 2: Data is partitioned by time.
- Script 3: Recent data has fast-path indexing; older data has scan-friendly index.
- Script 4: Enough sample rows exist to test behavior.
- Script 5: Query logic splits around 30-day boundary when needed.
- Script 6: IO/plan evidence shows the behavior in practice.

---

## 11) Related docs you already created

- `timescaledb_research.md` -> learning comparison with TimescaleDB concepts
- `hot_cold_query_flow.md` -> flowchart-style explanation of the query split

---

## 12) Quick run summary (30-second version)

1. Connect in SSMS
2. Run scripts 1..6 in order
3. Turn on actual plan before script 6
4. Check row count (~60k) and IO output
5. Confirm hot/cold behavior via execution plan and reads

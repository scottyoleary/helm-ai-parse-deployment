# Helm AI Parse — Live Demo Guide

This guide walks you through running a live demonstration of the pipeline with a client.
It covers what files to bring, how to run the pipeline, what to show at each stage,
and how to honestly discuss what the POC does — and does not — handle.

---

## Before the Demo (Preparation Checklist)

### Collect demo PDFs — aim for a mix

You want to show both successes and failures.  Use the categories below as a guide.

**Category A — Clean files (high success rate)**

These will work well and produce clean output.  Use them as your opening showcase.

| What to look for | Why it works well |
|---|---|
| Primavera P6 schedule export (tabular PDF) | P6 uses extremely consistent column headers that match the alias map |
| MS Project exported to PDF with "Table" view | Standard column names; headers on every page |
| Any schedule with columns: Activity ID, Activity Name, OD, RD, Start, Finish | These exact names are in the alias map — static mapping, no AI needed |
| Files with ≤ 50 pages | Fewer chunks = faster processing; easier to show in real time |

**Category B — Challenging but handled files (demonstrate AI recovery)**

These demonstrate the value of the AI-assisted fallback.

| What to look for | Why it's interesting |
|---|---|
| PDF where headers are abbreviated ("OD", "RD", "ES", "EF") | Alias map covers these; good demo of the lookup table |
| PDF where AI_PARSE_DOCUMENT goes OCR mode (scanned/image PDF) | Shows the two-pass fallback: LAYOUT tries first, OCR takes over |
| PDF with a Gantt chart occupying the right half of every page | Gantt-stripping logic fires; show before/after in STG_FLATTENED_ROWS |
| Multi-section PDF (header row repeats every 40–50 rows) | Table segmentation logic is highlighted |

**Category C — Problem files (use to show POC limits — see below)**

These will either fail partially or produce incomplete output.  Use them deliberately
to demonstrate what the POC was not designed to handle.

| File type | Expected behavior | What to tell the client |
|---|---|---|
| Password-protected PDF | `FAILED — Probe failed` in audit log | PDFs must be unlocked before upload |
| Horizontal layout ("landscape" with Gantt spanning 3 pages wide) | Gantt columns may bleed into data | The Gantt stripper uses header-column detection; extremely wide charts may not be fully removed |
| PDFs where columns are images (not text) and OCR gives very low confidence | Rows loaded = 0 or near 0 | Low-quality scans produce low-confidence OCR; pre-processing (Acrobat OCR) improves results |
| Schedules with no Activity ID column at all (just numbered rows) | `rows_loaded = 0` | The pipeline requires an Activity ID to identify rows; a "row number" scheme is not supported in this POC |
| Heavily customized P6 export with 20+ extra columns (BCWP, BCWS, SPI, etc.) | Loaded, but extra columns are ignored | Only the 6 target columns are extracted; client would need to extend the schema for cost/EVM data |
| Very large file (300+ pages) | Works but takes 5–15 minutes | Processing time scales with page count and Snowflake AI credit consumption |

---

## Demo Environment Setup (30 Minutes Before the Client Arrives)

### A. Confirm the deployment is functional

```sql
-- Quick health check
SELECT COUNT(*) FROM SCHEDULE_DB.CONVERTER.COLUMN_ALIAS_MAP;   -- Should be > 26
SHOW TASKS IN SCHEMA SCHEDULE_DB.CONVERTER;                     -- State = started
```

### B. Clear any leftover data from previous runs

If you ran test files during deployment, clean the tables so the demo starts fresh.
This makes the row counts and audit log unambiguous during the demo.

```sql
---------------------------------------------------------------------------
-- RESET: wipe all pipeline data (run ONLY before the demo, not in production)
---------------------------------------------------------------------------
TRUNCATE TABLE SCHEDULE_DB.CONVERTER.SCHEDULE_ACTIVITIES;
TRUNCATE TABLE SCHEDULE_DB.CONVERTER.STG_MAPPED_ROWS;
TRUNCATE TABLE SCHEDULE_DB.CONVERTER.STG_FLATTENED_ROWS;
TRUNCATE TABLE SCHEDULE_DB.CONVERTER.RAW_PARSED_DOCUMENTS;
TRUNCATE TABLE SCHEDULE_DB.CONVERTER.PARSE_AUDIT_LOG;

-- Streams reset automatically when their source tables are truncated.
-- The stage is not cleared — files will remain but won't be reprocessed
-- after the truncate because the EXISTS check is against RAW_PARSED_DOCUMENTS.
-- To force reprocessing of all uploaded files, also run:
REMOVE @SCHEDULE_DB.CONVERTER.SCHEDULE_STAGE;
ALTER STAGE SCHEDULE_DB.CONVERTER.SCHEDULE_STAGE REFRESH;
```

### C. Prepare your browser tabs / windows

Open four tabs in Snowsight before the demo begins:

| Tab | Content |
|---|---|
| Tab 1 | Snowsight → SCHEDULE_STAGE (to upload files live) |
| Tab 2 | A Snowsight worksheet with the run and observation queries (copy from Section 4 below) |
| Tab 3 | `SELECT * FROM SCHEDULE_DB.CONVERTER.V_PIPELINE_SUMMARY` |
| Tab 4 | `SELECT * FROM SCHEDULE_DB.CONVERTER.V_SCHEDULE_ACTIVITIES LIMIT 100` |

---

## Demo Script

### Opening (2 minutes)

*"What you're going to see today is a Snowflake-native pipeline that takes construction
schedule PDFs — directly from your project management software export — and converts them
into a structured, queryable database table.  No manual data entry, no Excel transformation,
no Python scripts running on a server you have to manage.  Everything runs inside Snowflake."*

*"There are three things I want to show you:*
*1. How easy the upload is.*
*2. What the data looks like after processing.*
*3. Where this POC has limits — because being honest about that is more useful to you than
   a polished demo that omits the hard cases."*

---

### Part 1 — Upload PDFs (5 minutes)

Start with **one Category A file** (clean, fast).

#### In Snowsight:

1. Navigate to **Data → Databases → SCHEDULE_DB → CONVERTER → Stages → SCHEDULE_STAGE**
2. Click **+ Files**
3. Upload your Category A file
4. Point out to the client: *"This is an internal Snowflake stage — the file is now stored
   securely inside your Snowflake account, not on an external server."*

```sql
-- Confirm the file appears in the stage directory listing
ALTER STAGE SCHEDULE_DB.CONVERTER.SCHEDULE_STAGE REFRESH;

SELECT RELATIVE_PATH, SIZE, LAST_MODIFIED
FROM DIRECTORY(@SCHEDULE_DB.CONVERTER.SCHEDULE_STAGE)
ORDER BY LAST_MODIFIED DESC;
```

---

### Part 2 — Run the Pipeline (5–10 minutes)

```sql
---------------------------------------------------------------------------
-- Run the full pipeline manually
-- (In production this runs automatically every 24 hours via scheduled tasks)
---------------------------------------------------------------------------
CALL SCHEDULE_DB.PROCEDURES.RUN_PIPELINE();
```

**While it runs, explain what is happening:**

*"The pipeline has four steps.  Right now:"*

*"Step 1 (PARSE): Snowflake is sending this PDF to its built-in AI — specifically
`AI_PARSE_DOCUMENT` — which uses a LAYOUT parser to extract the table structure as
markdown text.  If LAYOUT fails on any page range, it automatically retries with an OCR
model.  This is the most expensive step, and it scales with page count."*

*"Step 2 (FLATTEN): The AI output is markdown — pipe-delimited tables.  This step
explodes that into one database row per cell.  It also strips the Gantt bar chart from
the right side of the page, which the AI includes but we don't want."*

*"Step 3 (MAP): This step looks at the column headers — things like 'Activity ID',
'OD', 'Rem Dur' — and maps them to our standard six columns.  It uses a lookup table
first.  If the header is something it hasn't seen before, it asks Cortex AI to classify it."*

*"Step 4 (LOAD): Applies data type parsing — turning '10-Jan-25' into an actual date,
'5.0' into a number — deduplicates to remove repeated header rows, and writes the final
clean records."*

**After it finishes:**

The procedure returns a pipe-delimited string like:
```
"Parsed 1 file(s), 13 chunks | Flattened 4521 cells | 241 rows mapped | Loaded 241 rows"
```

Point this out to the client.

---

### Part 3 — Show the Audit Log (2 minutes)

```sql
---------------------------------------------------------------------------
-- Audit log: one row per file showing processing status
---------------------------------------------------------------------------
SELECT
    file_name,
    status,
    rows_extracted,
    rows_loaded,
    mapping_method,
    processing_seconds,
    load_success_pct,
    error_message
FROM SCHEDULE_DB.CONVERTER.V_PARSE_AUDIT_LOG
ORDER BY completed_at DESC;
```

**Points to make:**
- `STATUS = SUCCESS` — the file processed cleanly
- `ROWS_EXTRACTED` vs `ROWS_LOADED` — small differences are expected (deduplication removes
  repeated header rows that appear on every page)
- `MAPPING_METHOD`:
  - `static` — all column headers were found in the alias lookup table (fast, deterministic)
  - `ai` — at least one header required an AI call to classify
  - `ai+static` — hybrid: some columns found statically, AI filled the gaps
- `PROCESSING_SECONDS` — shows end-to-end time for this file

---

### Part 4 — Show the Structured Output (5 minutes)

**The summary view — one row per file:**

```sql
SELECT
    source_file_name,
    total_activities,
    unique_activity_ids,
    earliest_start,
    latest_finish,
    project_span_days,
    completed_activities,
    mapping_methods
FROM SCHEDULE_DB.CONVERTER.V_PIPELINE_SUMMARY;
```

**The detail view — all activities from this file:**

```sql
SELECT
    source_file_name,
    activity_id,
    activity_name,
    original_duration,
    remaining_duration,
    start_date,
    finish_date,
    calendar_days,
    pct_complete,
    status          -- 'NOT STARTED' | 'IN PROGRESS' | 'BEHIND' | 'COMPLETE'
FROM SCHEDULE_DB.CONVERTER.V_SCHEDULE_ACTIVITIES
ORDER BY source_file_name, source_table_index, source_row_index;
```

**Points to make:**
- *"Notice `status`.  This is derived in real time from `finish_date` vs today's date.
  Any activity with a past finish date and remaining duration > 0 is flagged as BEHIND."*
- *"Notice `pct_complete`.  This is calculated from `(1 - remaining/original) × 100`.
  It comes directly from the schedule data — not manually entered."*
- *"All dates are proper DATE types — you can use `DATEDIFF`, `DATEADD`, join to a
  calendar table, or feed this into any BI tool."*

---

### Part 5 — Upload Multiple Files at Once (3 minutes)

Upload 2–3 more files (mix of Category A and B).

```sql
-- Pipeline picks up all new files not yet in RAW_PARSED_DOCUMENTS
ALTER STAGE SCHEDULE_DB.CONVERTER.SCHEDULE_STAGE REFRESH;
CALL SCHEDULE_DB.PROCEDURES.RUN_PIPELINE();
```

After it finishes:

```sql
-- How many total activities across all files?
SELECT COUNT(*) AS total_activities FROM SCHEDULE_DB.CONVERTER.SCHEDULE_ACTIVITIES;

-- Summary per file
SELECT * FROM SCHEDULE_DB.CONVERTER.V_PIPELINE_SUMMARY;
```

**Points to make:**
- *"Every file gets its own entry in the summary.  You can compare project spans,
   activity counts, and how far along each project is — side by side, instantly."*
- *"Already-processed files are skipped automatically.  Uploading the same file twice
   won't create duplicate records."*

---

### Part 6 — The Automated Trigger (1 minute)

```sql
-- Show the task schedule
SHOW TASKS IN SCHEMA SCHEDULE_DB.CONVERTER;
```

*"In production, this runs every 24 hours automatically.  Your team just drops PDFs into
the stage — no button to push, no script to run.  New activities appear in the database
by the following morning.  The schedule can be adjusted: hourly, every 6 hours, whatever
your workflow requires."*

---

### Part 7 — Honest Discussion of Limits (5–8 minutes)

This is the most important part.  Upload a Category C file now, run the pipeline,
and show the result honestly.

#### 7A — Password-protected PDFs

*"If a PDF is password-protected, the AI parse step will fail entirely.  The audit log
shows `FAILED — Probe failed`.  The fix is to remove the password before uploading."*

```sql
SELECT file_name, status, error_message
FROM SCHEDULE_DB.CONVERTER.PARSE_AUDIT_LOG
WHERE status = 'FAILED'
ORDER BY completed_at DESC;
```

#### 7B — Low-quality scans

*"A scanned PDF that was never OCR'd — just photos of paper — will trigger the OCR path.
OCR confidence varies a lot with scan quality.  You may get partial data, garbled dates,
or no data at all.  We recommend running Acrobat's PDF OCR on scanned files before upload
if scan quality is poor."*

```sql
-- Show a file that used OCR and check its load rate
SELECT
    file_name,
    rows_extracted,
    rows_loaded,
    load_success_pct,
    mapping_method
FROM SCHEDULE_DB.CONVERTER.V_PARSE_AUDIT_LOG
WHERE mapping_method = 'OCR'
   OR load_success_pct < 80;
```

#### 7C — Unknown column headers

*"If a client uses a completely non-standard column header that's not in the alias map
and the AI doesn't recognize it, that column is marked UNMAPPED and its data is dropped.
The pipeline logs which mapping method was used.  Adding new headers to the alias table
takes about 30 seconds and fixes the problem for all future files."*

```sql
-- Show how to add a new alias (live demo of extensibility)
INSERT INTO SCHEDULE_DB.CONVERTER.COLUMN_ALIAS_MAP (ALIAS_TEXT, TARGET_COLUMN, ADDED_BY)
VALUES ('Planned Finish', 'finish_date', 'demo');

-- Confirm
SELECT * FROM SCHEDULE_DB.CONVERTER.COLUMN_ALIAS_MAP WHERE ADDED_BY = 'demo';
```

#### 7D — Schema is fixed at 6 columns

*"This POC extracts exactly six fields: Activity ID, Activity Name, Original Duration,
Remaining Duration, Start Date, and Finish Date.  If the client needs cost data, resource
assignments, WBS codes, baseline dates, or custom fields — that is a scope expansion beyond
this POC.  It is technically feasible but requires schema changes and procedure updates."*

#### 7E — Processing time for large files

*"A 242-page schedule takes roughly 5–10 minutes.  This is dominated by the AI parsing step.
Snowflake processes chunks in parallel, so there is a ceiling on how much faster it can go.
For very large programs with hundreds of PDFs, we would recommend batching uploads rather
than dumping everything at once."*

---

### Part 8 — Closing Queries

Leave these running on the screen as conversation prompts.

```sql
---------------------------------------------------------------------------
-- Activities that are BEHIND schedule across all loaded files
---------------------------------------------------------------------------
SELECT
    source_file_name,
    activity_id,
    activity_name,
    finish_date,
    remaining_duration,
    DATEDIFF('day', finish_date, CURRENT_DATE()) AS days_overdue
FROM SCHEDULE_DB.CONVERTER.V_SCHEDULE_ACTIVITIES
WHERE status = 'BEHIND'
ORDER BY days_overdue DESC
LIMIT 20;

---------------------------------------------------------------------------
-- Overall project health: % complete by file
---------------------------------------------------------------------------
SELECT
    source_file_name,
    COUNT(*)                                                AS total_activities,
    SUM(CASE WHEN status = 'COMPLETE'    THEN 1 ELSE 0 END) AS complete,
    SUM(CASE WHEN status = 'BEHIND'      THEN 1 ELSE 0 END) AS behind,
    SUM(CASE WHEN status = 'IN PROGRESS' THEN 1 ELSE 0 END) AS in_progress,
    SUM(CASE WHEN status = 'NOT STARTED' THEN 1 ELSE 0 END) AS not_started,
    ROUND(AVG(pct_complete), 1)                            AS avg_pct_complete
FROM SCHEDULE_DB.CONVERTER.V_SCHEDULE_ACTIVITIES
GROUP BY source_file_name
ORDER BY source_file_name;

---------------------------------------------------------------------------
-- All data, ready for export to Excel / Power BI / Tableau
---------------------------------------------------------------------------
SELECT *
FROM SCHEDULE_DB.CONVERTER.V_SCHEDULE_ACTIVITIES
ORDER BY source_file_name, source_table_index, source_row_index;
```

---

## Quick Reference — Queries to Have Ready

```sql
-- 1. Upload and refresh stage
ALTER STAGE SCHEDULE_DB.CONVERTER.SCHEDULE_STAGE REFRESH;

-- 2. Run the pipeline
CALL SCHEDULE_DB.PROCEDURES.RUN_PIPELINE();

-- 3. Check the audit log
SELECT * FROM SCHEDULE_DB.CONVERTER.V_PARSE_AUDIT_LOG;

-- 4. Pipeline summary (one row per file)
SELECT * FROM SCHEDULE_DB.CONVERTER.V_PIPELINE_SUMMARY;

-- 5. All activities (the main output)
SELECT * FROM SCHEDULE_DB.CONVERTER.V_SCHEDULE_ACTIVITIES LIMIT 100;

-- 6. Behind-schedule activities
SELECT * FROM SCHEDULE_DB.CONVERTER.V_SCHEDULE_ACTIVITIES WHERE status = 'BEHIND';

-- 7. Add a new column alias on the fly
INSERT INTO SCHEDULE_DB.CONVERTER.COLUMN_ALIAS_MAP (ALIAS_TEXT, TARGET_COLUMN, ADDED_BY)
VALUES ('<header text here>', '<target_column here>', 'demo');

-- 8. Show raw AI parse output (useful if explaining the AI step)
SELECT file_name, parse_mode, chunk_index,
       parsed_content:content::VARCHAR AS content_preview
FROM SCHEDULE_DB.CONVERTER.RAW_PARSED_DOCUMENTS
LIMIT 3;

-- 9. Show the cell-level flattened data (useful for explaining the flatten step)
SELECT *
FROM SCHEDULE_DB.CONVERTER.STG_FLATTENED_ROWS
ORDER BY file_name, table_index, row_index, cell_index
LIMIT 50;

-- 10. Show the mapped rows before final cleaning
SELECT *
FROM SCHEDULE_DB.CONVERTER.STG_MAPPED_ROWS
LIMIT 20;
```

---

## Anticipated Client Questions

| Question | Answer |
|---|---|
| *"Can it handle our proprietary P6 column layout?"* | Yes — if the headers are non-standard, add them to the alias map (30-second fix). |
| *"What about security — our PDFs are sensitive."* | Files are stored inside your Snowflake account's internal stage, subject to your Snowflake role-based access controls. No external service receives the files. |
| *"How does the AI work — is the content sent outside Snowflake?"* | `AI_PARSE_DOCUMENT` and `SNOWFLAKE.CORTEX.COMPLETE` are Snowflake-internal services. Data does not leave your Snowflake Virtual Private Cloud. |
| *"Can we connect this to Power BI or Tableau?"* | Yes — `V_SCHEDULE_ACTIVITIES` is a standard view. Connect any BI tool to Snowflake and point it at that view. |
| *"What's the cost?"* | Two cost components: (1) Snowflake warehouse compute while the pipeline runs, and (2) Snowflake AI credits for `AI_PARSE_DOCUMENT` and Cortex calls. Credit consumption scales with page count and how many files lack static alias matches. |
| *"Can it process files as soon as they're uploaded — not just every 24 hours?"* | Yes — the task schedule is configurable. Change `SCHEDULE = '24 HOURS'` to `'1 HOUR'` or use an event trigger. |
| *"Can we get baseline dates or costs too?"* | Not in this POC. The schema is fixed at 6 columns. Adding more fields is feasible but out of scope for what was built. |
| *"What if a file has already been processed — will it re-process it?"* | No. The pipeline checks `RAW_PARSED_DOCUMENTS` before processing. A file that was already parsed is skipped automatically. |
| *"Can we delete a file and reprocess it?"* | Yes — delete the rows from `RAW_PARSED_DOCUMENTS` and `SCHEDULE_ACTIVITIES` for that file, then re-upload and run the pipeline. |

# Helm AI Parse — Deployment Guide

This guide walks you through deploying every component of the pipeline in a fresh Snowflake
environment from scratch.  It is written so you can follow each step yourself, understand what
was built, and know *why* each piece was written the way it was.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Snowflake account | Enterprise tier or higher (required for `AI_PARSE_DOCUMENT` and `SNOWFLAKE.CORTEX.COMPLETE`) |
| Cortex AI enabled | AI_PARSE_DOCUMENT must be available in your region — confirm in `INFORMATION_SCHEMA` or with your Snowflake rep |
| Role | `ACCOUNTADMIN` or a role with `CREATE DATABASE`, `CREATE WAREHOUSE`, and `CREATE INTEGRATION` privileges |
| Snowsight or SnowSQL | All steps can be run in a worksheet; SnowSQL works too |
| Client PDFs | Collected before the demo; see the Demo Guide |

---

## What You Are Deploying

The pipeline turns construction-schedule PDFs into a clean, queryable table of project activities.

```
PDF files uploaded to
SCHEDULE_STAGE (Snowflake internal stage)
          │
          ▼
 PARSE_NEW_DOCUMENTS          — Calls Snowflake AI_PARSE_DOCUMENT (LAYOUT then OCR fallback)
          │                     Stores raw AI output in RAW_PARSED_DOCUMENTS
          ▼
 FLATTEN_PARSED_TABLES        — Splits the AI markdown output into one cell per row
          │                     Stores in STG_FLATTENED_ROWS
          ▼
 MAP_COLUMNS                  — Matches raw column headers to a standard schema
          │                     Uses an alias lookup table + Cortex AI fallback
          │                     Stores in STG_MAPPED_ROWS
          ▼
 LOAD_CLEAN_ACTIVITIES        — Applies cleaning UDFs, deduplicates, and writes final rows
                                Stores in SCHEDULE_ACTIVITIES
```

Everything runs in schema `SCHEDULE_DB.CONVERTER` (tables, stage, streams, views) and
`SCHEDULE_DB.PROCEDURES` (stored procedures).

---

## Deployment Order

Run the sections below in order.  Each section's SQL is self-contained and idempotent
(`CREATE OR REPLACE` / `CREATE IF NOT EXISTS`) so you can safely re-run any step.

---

## Step 1 — Create the Database and Schemas

**Why:** The database and schemas must exist before any objects can be created inside them.

```sql
---------------------------------------------------------------------------
-- 1A. Create the database
---------------------------------------------------------------------------
CREATE DATABASE IF NOT EXISTS SCHEDULE_DB;

---------------------------------------------------------------------------
-- 1B. Create the two schemas
--   CONVERTER  — holds all data objects (tables, stage, streams, views)
--   PROCEDURES — holds all stored procedures
--
-- Two schemas keep data objects and logic separate.  The procedures run
-- with EXECUTE AS OWNER, so separating them simplifies future permission
-- grants (e.g. you can grant USAGE on PROCEDURES without exposing raw data).
---------------------------------------------------------------------------
CREATE SCHEMA IF NOT EXISTS SCHEDULE_DB.CONVERTER;
CREATE SCHEMA IF NOT EXISTS SCHEDULE_DB.PROCEDURES;
```

---

## Step 2 — Create the Warehouse

**Why:** A named warehouse (`COMPUTE_WH`) is referenced inside every task definition.
If a warehouse by this name already exists in the account you can skip the CREATE.
If the client environment uses a different warehouse name you must update every task file
before deploying them.

```sql
---------------------------------------------------------------------------
-- 2A. Create (or confirm existence of) the compute warehouse
---------------------------------------------------------------------------
CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND   = 60
    AUTO_RESUME    = TRUE
    COMMENT        = 'Used by Helm AI Parse pipeline tasks';
```

> **Sizing note:** MEDIUM is sufficient for demo and small-scale use.  For a production
> environment with many concurrent PDF uploads, consider LARGE.  AI_PARSE_DOCUMENT calls
> are billed against Snowflake AI credits, not warehouse credits — but the surrounding
> SQL (INSERTs, MERGEs) does use warehouse compute.

---

## Step 3 — Create the Internal Stage

**Why:** `SCHEDULE_STAGE` is an internal Snowflake stage — a managed file storage location
inside Snowflake.  PDFs are uploaded here and the pipeline reads directly from it.
The `DIRECTORY = (ENABLE = TRUE)` option creates a directory table that lets the pipeline
call `DIRECTORY(@SCHEDULE_STAGE)` to list uploaded files by name.

```sql
---------------------------------------------------------------------------
-- File: stages/SCHEDULE_STAGE.sql
---------------------------------------------------------------------------
CREATE OR REPLACE STAGE SCHEDULE_DB.CONVERTER.SCHEDULE_STAGE
    DIRECTORY = (ENABLE = TRUE)
    COMMENT   = 'Internal stage for schedule PDF files';
```

---

## Step 4 — Create the Tables

Run each DDL file in the order below.  The order matters because streams (Step 6) reference
the tables.

### 4A — COLUMN_ALIAS_MAP

**Why:** This is the lookup table that maps raw column header text (as extracted from PDFs)
to the six target column names the pipeline understands.  Without it, no headers can be
mapped and no data will load.

```sql
---------------------------------------------------------------------------
-- File: ddl/COLUMN_ALIAS_MAP.sql
---------------------------------------------------------------------------
CREATE OR REPLACE TABLE SCHEDULE_DB.CONVERTER.COLUMN_ALIAS_MAP (
    ALIAS_TEXT      VARCHAR NOT NULL,
    TARGET_COLUMN   VARCHAR NOT NULL,
    ADDED_BY        VARCHAR DEFAULT 'manual',
    CONSTRAINT PK_ALIAS PRIMARY KEY (ALIAS_TEXT)
);
```

### 4B — RAW_PARSED_DOCUMENTS

**Why:** Stores the raw JSON output from `AI_PARSE_DOCUMENT`.  One row per 20-page chunk
of each PDF.  After parsing, rows are merged so that `parsed_content:content` holds the
full markdown text for that chunk.

```sql
---------------------------------------------------------------------------
-- File: ddl/RAW_PARSED_DOCUMENTS.sql
---------------------------------------------------------------------------
CREATE OR REPLACE TABLE SCHEDULE_DB.CONVERTER.RAW_PARSED_DOCUMENTS (
    FILE_NAME           VARCHAR         NOT NULL,
    FILE_URL            VARCHAR         NOT NULL,
    PARSED_CONTENT      VARIANT         NOT NULL,
    PARSE_MODE          VARCHAR         DEFAULT 'LAYOUT',
    SOURCE_FILE_NAME    VARCHAR,
    CHUNK_INDEX         NUMBER(38,0)    DEFAULT 0,
    CHUNK_START_PAGE    NUMBER(38,0),
    CHUNK_END_PAGE      NUMBER(38,0),
    INGESTED_AT         TIMESTAMP_NTZ   NOT NULL DEFAULT CURRENT_TIMESTAMP(),
    CONSTRAINT PK_RAW_PARSED PRIMARY KEY (FILE_NAME, INGESTED_AT)
);
```

### 4C — STG_FLATTENED_ROWS

**Why:** After the AI parses a PDF, the output is markdown tables inside a JSON blob.
This table breaks that down into one row per cell in those tables — essentially
"exploding" the markdown.

```sql
---------------------------------------------------------------------------
-- File: ddl/STG_FLATTENED_ROWS.sql
---------------------------------------------------------------------------
CREATE OR REPLACE TABLE SCHEDULE_DB.CONVERTER.STG_FLATTENED_ROWS (
    FILE_NAME           VARCHAR         NOT NULL,
    SOURCE_FILE_NAME    VARCHAR,
    TABLE_INDEX         NUMBER(38,0)    NOT NULL,
    ROW_INDEX           NUMBER(38,0)    NOT NULL,
    CELL_INDEX          NUMBER(38,0)    NOT NULL,
    CELL_CONTENT        VARCHAR,
    INGESTED_AT         TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP()
) CLUSTER BY (FILE_NAME, TABLE_INDEX);
```

### 4D — STG_MAPPED_ROWS

**Why:** Once column headers are identified (Step MAP_COLUMNS), this table holds one row
per activity — columns are now named properly (`ACTIVITY_ID`, `START_DATE`, etc.) instead
of being positional cells.

```sql
---------------------------------------------------------------------------
-- File: ddl/STG_MAPPED_ROWS.sql
---------------------------------------------------------------------------
CREATE OR REPLACE TABLE SCHEDULE_DB.CONVERTER.STG_MAPPED_ROWS (
    FILE_NAME           VARCHAR         NOT NULL,
    SOURCE_FILE_NAME    VARCHAR,
    TABLE_INDEX         NUMBER(38,0)    NOT NULL,
    ROW_INDEX           NUMBER(38,0)    NOT NULL,
    ACTIVITY_ID         VARCHAR,
    ACTIVITY_NAME       VARCHAR,
    ORIGINAL_DURATION   VARCHAR,
    REMAINING_DURATION  VARCHAR,
    START_DATE          VARCHAR,
    FINISH_DATE         VARCHAR,
    MAPPING_METHOD      VARCHAR,
    INGESTED_AT         TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP()
) CLUSTER BY (FILE_NAME);
```

### 4E — SCHEDULE_ACTIVITIES

**Why:** The final output table.  All values are typed (dates are `DATE`, durations are
`NUMBER`), cleaned, and deduplicated.  This is what the client queries.

```sql
---------------------------------------------------------------------------
-- File: ddl/SCHEDULE_ACTIVITIES.sql
---------------------------------------------------------------------------
CREATE OR REPLACE TABLE SCHEDULE_DB.CONVERTER.SCHEDULE_ACTIVITIES (
    SCHEDULE_ACTIVITY_ID    NUMBER(38,0)    NOT NULL AUTOINCREMENT START 1 INCREMENT 1 NOORDER,
    FILE_NAME               VARCHAR         NOT NULL,
    SOURCE_FILE_NAME        VARCHAR,
    ACTIVITY_ID             VARCHAR,
    ACTIVITY_NAME           VARCHAR,
    ORIGINAL_DURATION       NUMBER(10,2),
    REMAINING_DURATION      NUMBER(10,2),
    START_DATE              DATE,
    FINISH_DATE             DATE,
    SOURCE_TABLE_INDEX      NUMBER(38,0),
    SOURCE_ROW_INDEX        NUMBER(38,0),
    MAPPING_METHOD          VARCHAR,
    LOADED_AT               TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    CONSTRAINT PK_SCHEDULE_ACTIVITIES PRIMARY KEY (SCHEDULE_ACTIVITY_ID)
);
```

### 4F — PARSE_AUDIT_LOG

**Why:** Every file processed gets audit rows recording its status (`PROCESSING`,
`SUCCESS`, `FAILED`, `PARTIAL`), row counts, error messages, and timing.  The
`AUDIT_ID` surrogate key was added to prevent duplicate-row ambiguity in targeted updates.

```sql
---------------------------------------------------------------------------
-- File: ddl/PARSE_AUDIT_LOG.sql
---------------------------------------------------------------------------
CREATE TABLE IF NOT EXISTS SCHEDULE_DB.CONVERTER.PARSE_AUDIT_LOG (
    AUDIT_ID        NUMBER          AUTOINCREMENT START 1 INCREMENT 1,
    FILE_NAME       VARCHAR         NOT NULL,
    STATUS          VARCHAR         NOT NULL,
    ROWS_EXTRACTED  NUMBER(38,0)    DEFAULT 0,
    ROWS_LOADED     NUMBER(38,0)    DEFAULT 0,
    ERROR_MESSAGE   VARCHAR,
    MAPPING_METHOD  VARCHAR,
    STARTED_AT      TIMESTAMP_NTZ,
    COMPLETED_AT    TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    CONSTRAINT PK_PARSE_AUDIT PRIMARY KEY (AUDIT_ID)
);
```

---

## Step 5 — Seed the Alias Map

**Why:** `COLUMN_ALIAS_MAP` must be populated with the standard column header spellings
*before* any PDF is processed.  Without this seed data, the `MAP_COLUMNS` procedure
will be unable to find column headers and will fall back to AI for every single
file — which is slower and uses Cortex credits unnecessarily.

The seed data covers:
- Standard spellings used by most construction scheduling tools (Primavera P6, MS Project, etc.)
- Garbled header text that specific AI parse passes produced for known client PDFs
  (BDW12, SBN, Bloomingdale — names from the demo environment)

> **Important:** If the client's PDFs produce garbled or unusual header text that is not
> in this list, run the pipeline once, check `PARSE_AUDIT_LOG` for `MAPPING_METHOD = 'ai'`
> rows, then examine `STG_FLATTENED_ROWS` for the actual header text and add aliases manually.

```sql
---------------------------------------------------------------------------
-- File: ddl/SEED_ALIAS_MAP.sql
-- Standard header spellings
---------------------------------------------------------------------------

-- Core standard spellings (covers Primavera P6 and common MS Project exports)
INSERT INTO SCHEDULE_DB.CONVERTER.COLUMN_ALIAS_MAP (ALIAS_TEXT, TARGET_COLUMN, ADDED_BY)
SELECT alias_text, target_column, 'seed'
FROM (VALUES
    ('Activity ID',          'activity_id'),
    ('Act ID',               'activity_id'),
    ('Task ID',              'activity_id'),
    ('ID',                   'activity_id'),
    ('Activity Name',        'activity_name'),
    ('Task Name',            'activity_name'),
    ('Description',          'activity_name'),
    ('Name',                 'activity_name'),
    ('Original Duration',    'original_duration'),
    ('OD',                   'original_duration'),
    ('Orig Dur',             'original_duration'),
    ('Duration',             'original_duration'),
    ('Remaining Duration',   'remaining_duration'),
    ('RD',                   'remaining_duration'),
    ('Rem Dur',              'remaining_duration'),
    ('Remaining',            'remaining_duration'),
    ('Start',                'start_date'),
    ('Start Date',           'start_date'),
    ('ES',                   'start_date'),
    ('Early Start',          'start_date'),
    ('SD',                   'start_date'),
    ('Finish',               'finish_date'),
    ('Finish Date',          'finish_date'),
    ('EF',                   'finish_date'),
    ('Early Finish',         'finish_date'),
    ('FD',                   'finish_date')
) AS v(alias_text, target_column)
WHERE NOT EXISTS (
    SELECT 1 FROM SCHEDULE_DB.CONVERTER.COLUMN_ALIAS_MAP
    WHERE UPPER(TRIM(ALIAS_TEXT)) = UPPER(TRIM(v.alias_text))
);

-- Known garbled headers from specific client files (MERGE = idempotent)
-- Run: ddl/SEED_ALIAS_MAP.sql
MERGE INTO SCHEDULE_DB.CONVERTER.COLUMN_ALIAS_MAP AS tgt
USING (
    SELECT alias_text, target_column FROM (VALUES
        -- BDW12.pdf: LAYOUT parser produces garbled headers
        ('AGING ID',       'activity_id'),
        ('AGING Name',     'activity_name'),
        ('DD',             'original_duration'),
        ('NE',             'remaining_duration'),
        -- SBN.pdf garbled headers
        ('SUBID ID',       'activity_id'),
        ('AJ&A/Name',      'activity_name'),
        -- Bloomingdale: uses non-standard "Unique task ID"
        ('Unique task ID', 'activity_id')
    ) AS v(alias_text, target_column)
) AS src
    ON UPPER(TRIM(tgt.alias_text)) = UPPER(TRIM(src.alias_text))
WHEN NOT MATCHED THEN
    INSERT (alias_text, target_column, added_by)
    VALUES (src.alias_text, src.target_column, 'manual');
```

---

## Step 6 — Create the Streams

**Why:** Snowflake streams are change-data-capture (CDC) objects.  The pipeline uses
append-only streams so that automated tasks only fire when there is genuinely new data
to process, avoiding wasted compute on empty runs.

```sql
---------------------------------------------------------------------------
-- File: streams/STM_NEW_PARSED_DOCUMENTS.sql
-- Consumed by FLATTEN_PARSED_TABLES to detect newly parsed files.
---------------------------------------------------------------------------
CREATE OR REPLACE STREAM SCHEDULE_DB.CONVERTER.STM_NEW_PARSED_DOCUMENTS
    ON TABLE SCHEDULE_DB.CONVERTER.RAW_PARSED_DOCUMENTS
    APPEND_ONLY = TRUE;

---------------------------------------------------------------------------
-- File: streams/STM_NEW_FLATTENED_ROWS.sql
-- CHECKED (not consumed) by TSK_MAP_COLUMNS WHEN clause to skip empty runs.
---------------------------------------------------------------------------
CREATE OR REPLACE STREAM SCHEDULE_DB.CONVERTER.STM_NEW_FLATTENED_ROWS
    ON TABLE SCHEDULE_DB.CONVERTER.STG_FLATTENED_ROWS
    APPEND_ONLY = TRUE;

---------------------------------------------------------------------------
-- File: streams/STM_NEW_MAPPED_ROWS.sql
-- Consumed by LOAD_CLEAN_ACTIVITIES to load newly mapped rows.
---------------------------------------------------------------------------
CREATE OR REPLACE STREAM SCHEDULE_DB.CONVERTER.STM_NEW_MAPPED_ROWS
    ON TABLE SCHEDULE_DB.CONVERTER.STG_MAPPED_ROWS
    APPEND_ONLY = TRUE;
```

---

## Step 7 — Create the Scalar Functions (UDFs)

**Why:** These four SQL UDFs handle data cleaning and parsing that is reused across
procedures.  Defining them as named functions keeps the procedures clean and testable
in isolation.

### 7A — NORMALIZE_DATE_STR and PARSE_SCHEDULE_DATE

**Why two functions?** `NORMALIZE_DATE_STR` handles text cleanup (strip trailing
asterisks, day-of-week prefixes, expand 2-digit years, normalize Unicode dashes).
`PARSE_SCHEDULE_DATE` then tries multiple date format patterns.  The split allows
the normalizer to be tested independently and reused.

```sql
---------------------------------------------------------------------------
-- File: functions/PARSE_SCHEDULE_DATE.sql  (creates both functions)
---------------------------------------------------------------------------
CREATE OR REPLACE FUNCTION SCHEDULE_DB.CONVERTER.NORMALIZE_DATE_STR(DATE_STR VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
IMMUTABLE
AS $$
    REGEXP_REPLACE(
        REGEXP_REPLACE(
            REGEXP_REPLACE(
                REGEXP_REPLACE(TRIM(date_str), '[\\s*A]+$', ''),
                '^(Mon|Tue|Wed|Thu|Fri|Sat|Sun)\\s+', ''),
            '([/\\-])([0-9]{2})$', '\\120\\2'),
        '[‐‑‒–—―−­]', '-')
$$;

CREATE OR REPLACE FUNCTION SCHEDULE_DB.CONVERTER.PARSE_SCHEDULE_DATE(DATE_STR VARCHAR)
RETURNS DATE
LANGUAGE SQL
IMMUTABLE
AS $$
    CASE WHEN REGEXP_LIKE(TRIM(COALESCE(date_str, '')), '^-?[0-9]+$')
         THEN NULL
         ELSE COALESCE(
            TRY_TO_DATE(SCHEDULE_DB.CONVERTER.NORMALIZE_DATE_STR(date_str), 'DD-MON-YYYY'),
            TRY_TO_DATE(SCHEDULE_DB.CONVERTER.NORMALIZE_DATE_STR(date_str), 'YYYY-MM-DD'),
            TRY_TO_DATE(SCHEDULE_DB.CONVERTER.NORMALIZE_DATE_STR(date_str), 'MM/DD/YYYY'),
            TRY_TO_DATE(SCHEDULE_DB.CONVERTER.NORMALIZE_DATE_STR(date_str), 'M/D/YYYY'),
            TRY_TO_DATE(SCHEDULE_DB.CONVERTER.NORMALIZE_DATE_STR(date_str), 'MON DD, YYYY'),
            TRY_TO_DATE(SCHEDULE_DB.CONVERTER.NORMALIZE_DATE_STR(date_str), 'DD MON YYYY'),
            TRY_TO_DATE(SCHEDULE_DB.CONVERTER.NORMALIZE_DATE_STR(date_str))
         )
    END
$$;
```

### 7B — PARSE_SCHEDULE_NUMBER

**Why:** Duration values from PDFs contain noise: commas in large numbers, Unicode minus
signs, trailing text like "d" or "days" from certain exporters.  This UDF strips all of
that and returns a clean `NUMBER(10,2)`.

```sql
---------------------------------------------------------------------------
-- File: functions/PARSE_SCHEDULE_NUMBER.sql
---------------------------------------------------------------------------
CREATE OR REPLACE FUNCTION SCHEDULE_DB.CONVERTER.PARSE_SCHEDULE_NUMBER(NUM_STR VARCHAR)
RETURNS NUMBER(10,2)
LANGUAGE SQL
IMMUTABLE
AS $$
    TRY_TO_NUMBER(
        TRIM(
            REGEXP_REPLACE(
                REGEXP_REPLACE(
                    REGEXP_REPLACE(
                        REGEXP_REPLACE(num_str, '[‐‑‒–—―−­]', '-'),
                        ',', ''
                    ),
                    '[^0-9.\-].*$', ''
                ),
                '^\s+|\s+$', ''
            )
        ),
        10, 2
    )
$$;
```

### 7C — CLEAN_ACTIVITY_ID

**Why:** Activity IDs from OCR sometimes contain non-ASCII characters (smart quotes,
special dashes).  This UDF replaces all non-ASCII characters with a hyphen, collapses
repeated hyphens, and trims leading/trailing hyphens.

```sql
---------------------------------------------------------------------------
-- File: functions/CLEAN_ACTIVITY_ID.sql
---------------------------------------------------------------------------
CREATE OR REPLACE FUNCTION SCHEDULE_DB.CONVERTER.CLEAN_ACTIVITY_ID(RAW_ID VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
IMMUTABLE
AS $$
    TRIM(
        REGEXP_REPLACE(
            REGEXP_REPLACE(
                REGEXP_REPLACE(raw_id, '[^\x00-\x7F]', '-'),
                '-{2,}', '-'
            ),
            '^-+|-+$', ''
        )
    )
$$;
```

### 7D — CLEAN_ACTIVITY_NAME

**Why:** Activity names from OCR can contain control characters and double-spaces.
This UDF strips control characters and normalizes whitespace.

```sql
---------------------------------------------------------------------------
-- File: functions/CLEAN_ACTIVITY_NAME.sql
---------------------------------------------------------------------------
CREATE OR REPLACE FUNCTION SCHEDULE_DB.CONVERTER.CLEAN_ACTIVITY_NAME(RAW_NAME VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
IMMUTABLE
AS $$
    TRIM(
        REGEXP_REPLACE(
            REGEXP_REPLACE(raw_name, '[\x00-\x1F\x7F]', ''),
            '\s{2,}', ' '
        )
    )
$$;
```

---

## Step 8 — Create the Stored Procedures

Deploy procedures in the order listed.  `RUN_PIPELINE` references all four pipeline
procedures so it must be last.

### 8A — FLATTEN_OCR_PARALLEL (Python)

**Why:** When the LAYOUT AI parser fails or produces unusable output, this Python
procedure sends chunks of the raw OCR text to `SNOWFLAKE.CORTEX.COMPLETE`
(claude-4-sonnet) in parallel batches of up to 8 concurrent requests.  The LLM is
prompted to extract activity rows as a strict JSON array.

This is the only Python procedure.  It is called *from within* `FLATTEN_PARSED_TABLES`
for OCR-mode documents.

> **Deploy:** run the full contents of `procs/FLATTEN_OCR_PARALLEL.sql` in a worksheet.
> Snowflake will compile the Python inline and store it.  Estimated deploy time: ~5 seconds.

### 8B — PARSE_NEW_DOCUMENTS

**Why:** Detects new PDFs on the stage, determines page count, splits into 20-page chunks,
calls `AI_PARSE_DOCUMENT` in LAYOUT mode (with OCR fallback per chunk), merges paged output
into full-content rows, and writes audit records.

> **Deploy:** run the full contents of `procs/PARSE_NEW_DOCUMENTS.sql`.

### 8C — FLATTEN_PARSED_TABLES

**Why:** Reads the raw AI markdown output and explodes it into `STG_FLATTENED_ROWS` — one
row per cell.  Also handles several edge cases that were discovered during testing with
real client PDFs:

- **Gantt chart column stripping** — PDF schedules often have a Gantt bar chart on the
  right side of every page.  LAYOUT mode includes those bar chart columns in the markdown.
  A header-detection algorithm identifies where the schedule data columns end and deletes
  the Gantt columns.
- **Table segmentation** — Multi-section PDFs repeat the header row on each page.  Each
  repeated header is detected and assigned a new `TABLE_INDEX`.
- **Merged cell detection** — Some PDFs merge the Activity ID and Activity Name into a
  single cell.  A set-based detection algorithm finds these and splits them.

> **Deploy:** run the full contents of `procs/FLATTEN_PARSED_TABLES.sql`.

### 8D — MAP_COLUMNS

**Why:** `STG_FLATTENED_ROWS` has positional cells but no named columns yet.  This
procedure finds the header row per table segment, tries static alias lookups first
(instant, free, deterministic), and if any of the 6 target columns are still unresolved,
makes a single AI (`SNOWFLAKE.CORTEX.COMPLETE`) call per file for the gaps.
New aliases discovered by the AI are written back to `COLUMN_ALIAS_MAP` so future files
are handled statically.

> **Deploy:** run the full contents of `procs/MAP_COLUMNS.sql`.

### 8E — LOAD_CLEAN_ACTIVITIES

**Why:** Reads from `STM_NEW_MAPPED_ROWS`, applies `CLEAN_ACTIVITY_ID`,
`CLEAN_ACTIVITY_NAME`, `PARSE_SCHEDULE_NUMBER`, and `PARSE_SCHEDULE_DATE` UDFs,
deduplicates on `(source_file_name, activity_id, activity_name, start_date, finish_date)`,
and inserts into the final `SCHEDULE_ACTIVITIES` table.

Also handles **merge-shifted rows** — a specific edge case where the PDF's Activity ID
and Activity Name are merged, causing all other columns to shift one position right.

> **Deploy:** run the full contents of `procs/LOAD_CLEAN_ACTIVITIES.sql`.

### 8F — RUN_PIPELINE (Orchestrator)

**Why:** A single entry-point that runs all four pipeline steps in sequence.  Each step
is wrapped in a `BEGIN...EXCEPTION` block so that if one step errors, the remaining steps
still run on whatever data was already queued in the streams.

This is the procedure you call for manual/ad-hoc processing.

> **Deploy:** run the full contents of `procs/RUN_PIPELINE.sql`.

---

## Step 9 — Create the Views

Views are read-only query helpers — they do not store data.  Deploy all three.

### 9A — V_SCHEDULE_ACTIVITIES

**Why:** Adds derived fields to `SCHEDULE_ACTIVITIES`: `pct_complete`, `calendar_days`,
and a human-readable `status` field (`NOT STARTED`, `IN PROGRESS`, `BEHIND`, `COMPLETE`).
This is the primary view for client-facing queries.

```sql
-- File: views/V_SCHEDULE_ACTIVITIES.sql
-- Run full file contents.
```

### 9B — V_PIPELINE_SUMMARY

**Why:** One row per source PDF summarising: total activities, unique activity IDs,
project start/end dates, project span in days, count of completed activities,
mapping methods used, and last load time.

```sql
-- File: views/V_PIPELINE_SUMMARY.sql
-- Run full file contents.
```

### 9C — V_PARSE_AUDIT_LOG

**Why:** Enriches `PARSE_AUDIT_LOG` with a `processing_seconds` column and a
`load_success_pct` column (rows_loaded / rows_extracted × 100).

```sql
-- File: views/V_PARSE_AUDIT_LOG.sql
-- Run full file contents.
```

---

## Step 10 — Create the Automated Tasks

**Why tasks exist:** Tasks allow the pipeline to run automatically when new PDFs are
uploaded, without any human intervention.  The four tasks form a DAG (directed acyclic graph):

```
TSK_PARSE_DOCUMENTS  (root — runs every 24 hours)
    └── TSK_FLATTEN_TABLES   (fires when STM_NEW_PARSED_DOCUMENTS has data)
            └── TSK_MAP_COLUMNS   (fires when STM_NEW_FLATTENED_ROWS has data)
                    └── TSK_LOAD_ACTIVITIES   (fires when STM_NEW_MAPPED_ROWS has data)
```

> **Critical:** Snowflake requires that child tasks be **resumed before** the root task.
> Deploy in the order below — create all four first, then resume in leaf-to-root order.

### Deploy all four tasks

Run each file in this order:
1. `tasks/TSK_PARSE_DOCUMENTS.sql`
2. `tasks/TSK_FLATTEN_TABLES.sql`
3. `tasks/TSK_MAP_COLUMNS.sql`
4. `tasks/TSK_LOAD_ACTIVITIES.sql`

### Resume tasks (leaf-to-root)

```sql
---------------------------------------------------------------------------
-- Resume child tasks first, then root last.
-- Snowflake will refuse to resume the root if any child is suspended.
---------------------------------------------------------------------------

ALTER TASK SCHEDULE_DB.CONVERTER.TSK_LOAD_ACTIVITIES  RESUME;
ALTER TASK SCHEDULE_DB.CONVERTER.TSK_MAP_COLUMNS      RESUME;
ALTER TASK SCHEDULE_DB.CONVERTER.TSK_FLATTEN_TABLES   RESUME;
ALTER TASK SCHEDULE_DB.CONVERTER.TSK_PARSE_DOCUMENTS  RESUME;  -- root last
```

### Verify tasks are running

```sql
SHOW TASKS IN SCHEMA SCHEDULE_DB.CONVERTER;
-- STATE column should show 'started' for all four tasks.
-- The root task will fire next at its scheduled time.
```

---

## Step 11 — Verify the Deployment

Run these checks to confirm everything was created correctly.

```sql
---------------------------------------------------------------------------
-- 11A. Check all tables exist
---------------------------------------------------------------------------
SHOW TABLES IN SCHEMA SCHEDULE_DB.CONVERTER;
-- Expected: COLUMN_ALIAS_MAP, RAW_PARSED_DOCUMENTS, STG_FLATTENED_ROWS,
--           STG_MAPPED_ROWS, SCHEDULE_ACTIVITIES, PARSE_AUDIT_LOG

---------------------------------------------------------------------------
-- 11B. Check alias map is seeded
---------------------------------------------------------------------------
SELECT COUNT(*) AS alias_count FROM SCHEDULE_DB.CONVERTER.COLUMN_ALIAS_MAP;
-- Expected: >= 26 rows (standard seed + garbled-header entries)

---------------------------------------------------------------------------
-- 11C. Check streams exist
---------------------------------------------------------------------------
SHOW STREAMS IN SCHEMA SCHEDULE_DB.CONVERTER;
-- Expected: STM_NEW_PARSED_DOCUMENTS, STM_NEW_FLATTENED_ROWS, STM_NEW_MAPPED_ROWS

---------------------------------------------------------------------------
-- 11D. Check procedures exist
---------------------------------------------------------------------------
SHOW PROCEDURES IN SCHEMA SCHEDULE_DB.PROCEDURES;
-- Expected: FLATTEN_OCR_PARALLEL, PARSE_NEW_DOCUMENTS, FLATTEN_PARSED_TABLES,
--           MAP_COLUMNS, LOAD_CLEAN_ACTIVITIES, RUN_PIPELINE

---------------------------------------------------------------------------
-- 11E. Check UDFs exist
---------------------------------------------------------------------------
SHOW USER FUNCTIONS IN SCHEMA SCHEDULE_DB.CONVERTER;
-- Expected: NORMALIZE_DATE_STR, PARSE_SCHEDULE_DATE, PARSE_SCHEDULE_NUMBER,
--           CLEAN_ACTIVITY_ID, CLEAN_ACTIVITY_NAME

---------------------------------------------------------------------------
-- 11F. Check views exist
---------------------------------------------------------------------------
SHOW VIEWS IN SCHEMA SCHEDULE_DB.CONVERTER;
-- Expected: V_SCHEDULE_ACTIVITIES, V_PIPELINE_SUMMARY, V_PARSE_AUDIT_LOG

---------------------------------------------------------------------------
-- 11G. Check stage and tasks
---------------------------------------------------------------------------
SHOW STAGES   IN SCHEMA SCHEDULE_DB.CONVERTER;   -- Expected: SCHEDULE_STAGE
SHOW TASKS    IN SCHEMA SCHEDULE_DB.CONVERTER;   -- Expected: 4 tasks, STATE = started
```

---

## Step 12 — Smoke Test With One PDF

Before the full demo, validate the pipeline end-to-end with one known-good file.

### 12A — Upload a test PDF via Snowsight

1. Open Snowsight → Data → Databases → SCHEDULE_DB → CONVERTER → Stages → SCHEDULE_STAGE
2. Click **+ Files** (top right)
3. Upload one PDF

### 12B — Alternative: upload via SnowSQL / PUT command

```sql
-- In SnowSQL CLI (not a worksheet):
PUT file://C:\path\to\your\test.pdf @SCHEDULE_DB.CONVERTER.SCHEDULE_STAGE
    AUTO_COMPRESS = FALSE;
```

### 12C — Run the pipeline manually

```sql
---------------------------------------------------------------------------
-- Refresh the stage directory listing, then run all four steps.
---------------------------------------------------------------------------
ALTER STAGE SCHEDULE_DB.CONVERTER.SCHEDULE_STAGE REFRESH;

CALL SCHEDULE_DB.PROCEDURES.RUN_PIPELINE();
-- Returns a pipe-delimited string like:
-- "Parsed 1 file(s), 13 chunks total | Flattened 4521 cells | ... | Loaded 241 rows"
```

### 12D — Check results

```sql
-- Audit log: was the file processed successfully?
SELECT * FROM SCHEDULE_DB.CONVERTER.V_PARSE_AUDIT_LOG LIMIT 10;

-- How many activities were loaded?
SELECT * FROM SCHEDULE_DB.CONVERTER.V_PIPELINE_SUMMARY;

-- Sample the cleaned activities
SELECT * FROM SCHEDULE_DB.CONVERTER.V_SCHEDULE_ACTIVITIES LIMIT 20;
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `RUN_PIPELINE` returns `parse: ERROR` | `AI_PARSE_DOCUMENT` not enabled or insufficient Cortex credits | Confirm Cortex AI is enabled in the account; check with Snowflake rep |
| Audit log shows `FAILED — Probe failed` | Stage URL issue or PDF is corrupted/password-protected | Re-upload the file; confirm the PDF opens locally |
| `rows_extracted > 0` but `rows_loaded = 0` | All rows filtered out (no activity_id or activity_name) | Check `STG_MAPPED_ROWS` for the file; inspect `COLUMN_ALIAS_MAP` coverage |
| `MAPPING_METHOD = 'ai'` for every file | Alias map is empty or missing standard headers | Re-run Step 5 (seed the alias map) |
| Task stuck in `running` state | Previous run still active (overlapping execution disabled) | `SELECT * FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY())` to diagnose |
| Child tasks not firing | Root task resumed before child tasks | Suspend all tasks, then resume leaf-to-root |

---

## Summary Deployment Checklist

| # | Object | SQL File | Done? |
|---|---|---|---|
| 1 | Database + schemas | Manual (Step 1) | ☐ |
| 2 | Warehouse COMPUTE_WH | Manual (Step 2) | ☐ |
| 3 | SCHEDULE_STAGE | `stages/SCHEDULE_STAGE.sql` | ☐ |
| 4a | COLUMN_ALIAS_MAP | `ddl/COLUMN_ALIAS_MAP.sql` | ☐ |
| 4b | RAW_PARSED_DOCUMENTS | `ddl/RAW_PARSED_DOCUMENTS.sql` | ☐ |
| 4c | STG_FLATTENED_ROWS | `ddl/STG_FLATTENED_ROWS.sql` | ☐ |
| 4d | STG_MAPPED_ROWS | `ddl/STG_MAPPED_ROWS.sql` | ☐ |
| 4e | SCHEDULE_ACTIVITIES | `ddl/SCHEDULE_ACTIVITIES.sql` | ☐ |
| 4f | PARSE_AUDIT_LOG | `ddl/PARSE_AUDIT_LOG.sql` | ☐ |
| 5 | Alias map seed data | `ddl/SEED_ALIAS_MAP.sql` | ☐ |
| 6 | Streams (x3) | `streams/STM_*.sql` | ☐ |
| 7a | NORMALIZE_DATE_STR + PARSE_SCHEDULE_DATE | `functions/PARSE_SCHEDULE_DATE.sql` | ☐ |
| 7b | PARSE_SCHEDULE_NUMBER | `functions/PARSE_SCHEDULE_NUMBER.sql` | ☐ |
| 7c | CLEAN_ACTIVITY_ID | `functions/CLEAN_ACTIVITY_ID.sql` | ☐ |
| 7d | CLEAN_ACTIVITY_NAME | `functions/CLEAN_ACTIVITY_NAME.sql` | ☐ |
| 8a | FLATTEN_OCR_PARALLEL (Python) | `procs/FLATTEN_OCR_PARALLEL.sql` | ☐ |
| 8b | PARSE_NEW_DOCUMENTS | `procs/PARSE_NEW_DOCUMENTS.sql` | ☐ |
| 8c | FLATTEN_PARSED_TABLES | `procs/FLATTEN_PARSED_TABLES.sql` | ☐ |
| 8d | MAP_COLUMNS | `procs/MAP_COLUMNS.sql` | ☐ |
| 8e | LOAD_CLEAN_ACTIVITIES | `procs/LOAD_CLEAN_ACTIVITIES.sql` | ☐ |
| 8f | RUN_PIPELINE | `procs/RUN_PIPELINE.sql` | ☐ |
| 9a | V_SCHEDULE_ACTIVITIES | `views/V_SCHEDULE_ACTIVITIES.sql` | ☐ |
| 9b | V_PIPELINE_SUMMARY | `views/V_PIPELINE_SUMMARY.sql` | ☐ |
| 9c | V_PARSE_AUDIT_LOG | `views/V_PARSE_AUDIT_LOG.sql` | ☐ |
| 10 | Tasks (x4) | `tasks/TSK_*.sql` + RESUME | ☐ |
| 11 | Verification queries | Step 11 above | ☐ |
| 12 | Smoke test with one PDF | Step 12 above | ☐ |

# Port of Vancouver: Raw Data to AI Agent with AISQL

Snowflake Cortex Code hands-on lab. Wednesday, October 28, 2026. Slalom Consulting, 355 Burrard Street, Suite 700, Vancouver, BC.

- Source: <https://coco-hol-vancouver-oct-28.streamlit.app>
- Data zip: <https://github.com/sfc-gh-mkeith/coco-hol-aisql-vancouver-oct-28/raw/main/workshop_guide/data/port_vancouver_data.zip>
- Saved: 2026-10-28

Build an AI agent on raw port documents, measure it, then make it faster, cheaper and more accurate by preprocessing with AISQL.

| Sections | Prompts | Duration |
|---|---|---|
| 5 | 17 | 75 min hands-on |

## Contents

- [Overview](#overview)
- [Getting Started](#getting-started)
- [Agenda](#agenda)
- Part 1: The baseline
  - [Section 1: Setup & Data Load](#section-1-setup--data-load)
  - [Section 2: Baseline Agent on Raw Data](#section-2-baseline-agent-on-raw-data)
  - [Section 3: Evaluate the Baseline](#section-3-evaluate-the-baseline)
- Part 2: Optimize with AISQL
  - [Section 4: Preprocess with AISQL](#section-4-preprocess-with-aisql)
  - [Section 5: AISQL Agent & the Before/After](#section-5-aisql-agent--the-beforeafter)

---

## Overview

### The scenario

The Port of Vancouver runs five terminals: **Deltaport, Vanterm, Centerm, Fraser Surrey Docks and Neptune**. Vessel calls are tracked in clean tables, but most operational knowledge is written by people: **incident reports** from shift supervisors, **CBSA inspection notes**, and **customer emails** in English and French.

Leadership wants an AI agent that can answer questions like *"How many hours did equipment failures cost us this year?"* The facts are all there, but they're buried in free text, with every terminal and number written a different way.

### What we're building

**Part 1: The baseline**

Load the data, build a semantic view and a Cortex Search service over the raw documents, and create `PORT_AGENT_BASELINE`. Then score it with Cortex Agent Evaluations against 12 questions with known answers, measuring accuracy, latency and tokens.

**Part 2: Optimize with AISQL**

Run `AI_EXTRACT`, `AI_CLASSIFY`, `AI_FILTER`, `AI_TRANSLATE` and `AI_AGG` once over every document to build enriched tables. Create `PORT_AGENT_AISQL` on top, run the same evaluation, and compare accuracy, speed and cost side by side.

### How this workshop works

Each section has numbered prompts. Copy each one into Cortex Code in Snowsight and let it build and run everything. Prompts build on each other, so run them in order.

### Prerequisites

- A Snowflake trial account with the `ACCOUNTADMIN` role (see [Getting Started](#getting-started))
- Cortex Code open in Snowsight
- Cross-region inference enabled (for Cortex AI functions and evaluation judges)

---

## Getting Started

Provision a Snowflake account for the workshop.

### Step 1: Sign up for a free trial

Go to [tinyurl.com/van-coco-hol](https://tinyurl.com/van-coco-hol) and fill out the registration form.

If the short link doesn't work, use the full link: <https://signup.snowflake.com/?t=e73d5b7babb2d5d3a3826bd4471d4db573a8508e99c7b31b00dd3a7efa1df268&cloud=aws&region=us-east-2>

On the **Choose your Snowflake edition** screen, select:

| Setting | Recommended value |
|---|---|
| Cloud provider | Amazon Web Services (AWS) |
| Edition | Enterprise |
| Region | US East 2 (Ohio) |

### Step 2: Activate your account

Snowflake emails you an activation link. Click it to set your password and log in.

> The email usually arrives within a few minutes. Check your spam folder if you don't see it.

### Step 3: Open Cortex Code

In Snowsight, open **Cortex Code** from the left navigation. You will paste every prompt from this guide there.

Make sure your role is `ACCOUNTADMIN` (bottom-left of Snowsight).

### Step 4: Enable cross-region inference

The AISQL functions and the evaluation judge models need cross-region inference. Run this in a worksheet:

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```

### Free trial details

| Duration | Credits | Credit card |
|---|---|---|
| 30 days | $400 | Not required |

The free trial provides more than enough credits to complete the entire workshop.

---

## Agenda

| Time | Item |
|---|---|
| 9:00 – 9:15 | **Welcome & Check-In.** Grab a coffee, connect to Wi-Fi, and start your Snowflake trial signup. |
| 9:15 – 9:30 | **Opening & Context Setting.** Why AI agents need good data, and where AISQL fits in the pipeline. |
| 9:30 – 10:45 | **Hands-On Lab.** Sections 1–5 in this guide. |
| 10:45 – 11:00 | **Wrap-Up & Next Steps.** Review the before/after results and how to apply this to your own data. |
| 11:00 – 11:30 | **Networking** |

Hands-on lab breakdown:

| Time | Section | Part |
|---|---|---|
| 9:30 – 9:45 | 1. Setup & Data Load (15 min) | Part 1: The baseline |
| 9:45 – 10:00 | 2. Baseline Agent on Raw Data (15 min) | Part 1: The baseline |
| 10:00 – 10:10 | 3. Evaluate the Baseline (10 min) | Part 1: The baseline |
| 10:10 – 10:30 | 4. Preprocess with AISQL (20 min) | Part 2: Optimize with AISQL |
| 10:30 – 10:45 | 5. AISQL Agent & the Before/After (15 min) | Part 2: Optimize with AISQL |

---

## Section 1: Setup & Data Load

**Time:** 9:30 – 9:45 (15 min)

**Building:** Database, warehouse and 7 tables of port data (structured + raw text)

### Technologies used in this section

- **Database, Schema & Stage.** Containers for your tables and an internal stage to hold uploaded files.
- **INFER_SCHEMA + COPY INTO.** Detect column types from CSV files and bulk-load them into tables.
- **Virtual Warehouse.** The compute that runs your SQL, loads and AISQL functions.

### Prompt 1.1 - Create Database, Schema, Stage & Warehouse

```text
Create the following Snowflake objects for our Port of Vancouver AISQL workshop:

1. A database called PORT_VAN_AISQL
2. A schema called PORT_OPS inside that database
3. A stage called DATA in PORT_VAN_AISQL.PORT_OPS with a directory table and server side encryption
4. A warehouse called PORT_AISQL_WH (size MEDIUM, auto-suspend after 60 seconds, auto-resume enabled)
5. Set the session context to use these objects

Execute all SQL and confirm each object was created.
```

**What this prompt does**

```sql
CREATE DATABASE PORT_VAN_AISQL;
CREATE SCHEMA PORT_VAN_AISQL.PORT_OPS;
CREATE STAGE PORT_VAN_AISQL.PORT_OPS.DATA
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
CREATE WAREHOUSE PORT_AISQL_WH
  WAREHOUSE_SIZE = 'MEDIUM' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;
```

### Upload the data

Before running Prompt 1.2, download the CSV files and upload them to the `DATA` stage:

1. Download the zip file: [port_vancouver_data.zip](https://github.com/sfc-gh-mkeith/coco-hol-aisql-vancouver-oct-28/raw/main/workshop_guide/data/port_vancouver_data.zip)
2. Unzip it on your computer to get the 7 CSV files.
3. In Snowsight, use the Horizon Catalog to browse to `PORT_VAN_AISQL.PORT_OPS.DATA` and upload all CSV files.
4. Then copy the prompt below into Cortex Code and execute.

### Prompt 1.2 - Create and Load Tables from CSV

```text
In PORT_VAN_AISQL.PORT_OPS, the 7 CSV files have been uploaded to an internal stage called DATA.

For all 7 tables (TERMINALS, VESSELS, VESSEL_CALLS, INCIDENT_REPORTS_RAW, INSPECTION_REPORTS_RAW, SHIPPER_EMAILS_RAW, EVAL_QUESTIONS):

1. Create a file format (CSV with PARSE_HEADER=TRUE, FIELD_OPTIONALLY_ENCLOSED_BY='"')
2. Create the tables with appropriate column types inferred from the data. Ensure to convert the column names to uppercase.
3. Load the data

Use CREATE TABLE with INFER_SCHEMA from a stage and then COPY INTO them. The key requirement is that all 7 tables are created and populated.

Execute all SQL.
```

**What this prompt does**

| Table | Rows | Kind | Contents |
|---|---|---|---|
| `TERMINALS` | 5 | structured | The five terminals: Deltaport, Vanterm, Centerm, Fraser Surrey Docks, Neptune |
| `VESSELS` | 30 | structured | Container vessels and their shipping lines |
| `VESSEL_CALLS` | 400 | structured | Each vessel visit: terminal, arrival, scheduled vs actual berth hours, TEU |
| `INCIDENT_REPORTS_RAW` | 300 | raw text | Free-text incident reports. Terminal, severity, delay and cost are only in the text, written with inconsistent names and formats |
| `INSPECTION_REPORTS_RAW` | 120 | raw text | CBSA officer notes. Outcome and penalty amount are only in the text |
| `SHIPPER_EMAILS_RAW` | 150 | raw text | Customer emails in English and French. Topic and urgency are only in the text |
| `EVAL_QUESTIONS` | 12 | evaluation | 12 test questions with known correct answers, used to score both agents |

### Prompt 1.3 - Look at the Raw Text

```text
In PORT_VAN_AISQL.PORT_OPS:

1. Show every table name and its row count, ordered by row count descending.
2. Show 5 sample rows from INCIDENT_REPORTS_RAW, INSPECTION_REPORTS_RAW and SHIPPER_EMAILS_RAW.
3. In a short summary, point out what makes the raw text hard to analyze: how terminals are named, how delays and costs are written, and which languages appear.
```

**What this prompt does**

You should see ~1,017 rows across 7 tables. Notice that the same terminal appears as *Deltaport*, *DP*, *Roberts Bank*; delays appear as *8h*, *~4 hrs*, *approx. 660 minutes*; costs as *\$130,700*, *\$713.4k CAD*, *roughly \$113K*. Some emails are in French.

This is the problem the rest of the lab is about. None of these facts are columns yet, so no SQL can count or sum them.

### Key concepts

- **Structured vs. raw data.** Structured data lives in typed columns that SQL can filter and aggregate. Raw data (reports, emails, notes) holds the same facts inside free text, where SQL can't reach them directly.
- **INFER_SCHEMA.** A table function that detects column names and types from staged files.

### What you built in this section

- [ ] `PORT_VAN_AISQL.PORT_OPS` with the `DATA` stage and `PORT_AISQL_WH` warehouse
- [ ] 7 tables loaded (~1,017 rows)

---

## Section 2: Baseline Agent on Raw Data

**Time:** 9:45 – 10:00 (15 min)

**Building:** A semantic view, a search service over raw documents, and a first agent

This is the most common way teams start: point an agent at the tables you have and a search index over your documents. It works for lookups, but watch what happens with questions that need counting or adding up facts buried in text.

### Technologies used in this section

- **Semantic View.** A business-friendly layer over tables that Cortex Analyst uses to turn questions into SQL.
- **Cortex Search.** Hybrid vector + keyword search over text. Returns the most relevant chunks for a question.
- **Cortex Agent.** Plans, picks tools (Analyst, Search), runs them and writes the answer.

### Prompt 2.1 - Semantic View over the Structured Tables

```text
/semantic_studio Create a semantic view called PORT_OPS_SV in PORT_VAN_AISQL.PORT_OPS over TERMINALS, VESSELS and VESSEL_CALLS.

- Add clear descriptions and synonyms for terminals (e.g. Deltaport is also called DP or Roberts Bank)
- Add a fact/metric for berth delay hours = ACTUAL_BERTH_HOURS - SCHEDULED_BERTH_HOURS
- Relationships: VESSEL_CALLS to TERMINALS on TERMINAL_ID, VESSEL_CALLS to VESSELS on VESSEL_ID
- When defining relationships, do not set join_type. The left_table must be the fact (many) side and the right_table the dimension (one) side, and the dimension tables must declare primary_key columns.

Use this format for relationships:
relationships:
  - name: calls_to_terminals
    left_table: VESSEL_CALLS
    right_table: TERMINALS
    relationship_columns:
      - left_column: TERMINAL_ID
        right_column: TERMINAL_ID

Deploy it and test it with: "What is the average berth delay at Centerm?"
```

**What this prompt does**

The semantic view only covers the structured tables. The three `_RAW` tables can't go in a semantic view in a useful way: their facts aren't columns.

### Prompt 2.2 - Cortex Search over the Raw Documents

```text
In PORT_VAN_AISQL.PORT_OPS:

1. Create a view RAW_DOCUMENTS that unions the three raw text tables into one shape: DOC_ID, DOC_TYPE ('incident_report', 'cbsa_inspection', 'shipper_email'), DOC_DATE, SOURCE (submitted_by / call_id / from_company) and DOC_TEXT (for emails, combine subject and body).
2. Create a Cortex Search service called RAW_DOCS_SEARCH on RAW_DOCUMENTS, searching DOC_TEXT, with DOC_ID, DOC_TYPE, DOC_DATE and SOURCE as attributes. Use warehouse PORT_AISQL_WH, TARGET_LAG = '1 hour' and embedding model snowflake-arctic-embed-l-v2.0.
3. Test it with SEARCH_PREVIEW for "crane hydraulic failure at Deltaport".
```

**What this prompt does**

Search is great at finding relevant documents, but it returns a limited number of top results. It cannot read all 300 incident reports and add up their delays.

### Prompt 2.3 - Create the Baseline Agent

```text
In PORT_VAN_AISQL.PORT_OPS, create a Cortex Agent called PORT_AGENT_BASELINE:

- Orchestration model: auto
- Tools: Cortex Analyst on the PORT_OPS_SV semantic view (vessel calls, berth time, TEU) and Cortex Search on RAW_DOCS_SEARCH (incident reports, CBSA inspections, shipper emails)
- Instructions: You are a Port of Vancouver operations analyst. Use the semantic view for vessel, berth and TEU questions. Use search for incidents, inspections and customer emails. Always give a direct numeric answer when asked for counts or totals.
- Add sample questions: "Which shipping line had the most vessel calls?", "How many incidents were reported at Deltaport?"

Then ask the agent these two questions and show its answers and which tools it used:
1. "Which shipping line had the most vessel calls this year?"
2. "How many incidents were reported at Deltaport?"
```

**What this prompt does**

The first question goes to Cortex Analyst and is exact. For the second, the agent can only count the incidents that search returned, so the number is usually wrong, and the agent may not tell you.

> You can also chat with the agent in Snowsight: **AI & ML → Agents → PORT_AGENT_BASELINE**. Open the response details to see which tools it called and how long each step took.

### Key concepts

- **Top-k retrieval.** Search returns the k most relevant chunks, not every matching document. It's fine for 'find me the report about X', but questions like 'how many' or 'total cost' would need every document read.
- **Tool routing.** The agent decides which tool to call from the tool descriptions and its instructions.

### What you built in this section

- [ ] `PORT_OPS_SV` semantic view (structured tables)
- [ ] `RAW_DOCS_SEARCH` Cortex Search service over 570 raw documents
- [ ] `PORT_AGENT_BASELINE`, the baseline agent

---

## Section 3: Evaluate the Baseline

**Time:** 10:00 – 10:10 (10 min)

**Building:** An evaluation dataset and a scored baseline run (accuracy, latency, tokens)

To prove the AISQL version is better, you first need numbers for the baseline. Cortex Agent Evaluations run every test question through the agent and use an LLM judge to score each answer against the known correct answer. Each run also records duration and token usage.

### Technologies used in this section

- **Cortex Agent Evaluations.** Batch-run an agent over a dataset and score it with system metrics like answer correctness.
- **LLM-as-judge.** A judge model compares each agent answer to the ground truth and scores it.
- **AI Observability.** Traces for every run: tool calls, duration and token counts, queryable with SQL.

### Prompt 3.1 - Build the Evaluation Dataset

```text
In PORT_VAN_AISQL.PORT_OPS, create a table PORT_EVAL_DATA (INPUT_QUERY VARCHAR, GROUND_TRUTH VARIANT) from EVAL_QUESTIONS.

- INPUT_QUERY = QUESTION
- GROUND_TRUTH = PARSE_JSON of an object with one key "ground_truth_output". Its value is a plain-language rubric: "The correct answer is <EXPECTED_ANSWER>. Numbers within ±5% are acceptable. An answer that gives a different number, or says it cannot determine the answer, is incorrect."

Use PARSE_JSON (not OBJECT_CONSTRUCT) so the column is a VARIANT. Show all 12 rows.
```

**What this prompt does**

The judge only sees what you put in `ground_truth_output`, so it's written as a short rubric with the literal expected value and a tolerance, as the evaluation docs recommend.

### Prompt 3.2 - Run the Baseline Evaluation

```text
Run a Cortex Agent evaluation of PORT_VAN_AISQL.PORT_OPS.PORT_AGENT_BASELINE:

- Dataset: create an evaluation dataset called PORT_EVAL_DS from PORT_VAN_AISQL.PORT_OPS.PORT_EVAL_DATA (query column INPUT_QUERY, ground truth column GROUND_TRUTH)
- Run name: baseline-run-1
- Metrics: answer_correctness and logical_consistency, both pinned to version v3
- Put the evaluation YAML config on a stage EVAL_CONFIG in PORT_VAN_AISQL.PORT_OPS (with the YAML file format the docs require) and start the run with EXECUTE_AI_EVALUATION

Start the run, then check its STATUS until it is COMPLETED.
```

**What this prompt does**

The run takes a few minutes (12 questions, each answered by the agent and then judged). While it runs, read ahead to Section 4.

> **No-code alternative:** In Snowsight go to **AI & ML → Agents → PORT_AGENT_BASELINE → Evaluations → Create evaluation run → Run an evaluation manually**. Choose `PORT_EVAL_DATA` as the source table, `INPUT_QUERY` as input, and turn on **Answer correctness** with `GROUND_TRUTH`.

### Prompt 3.3 - Read the Baseline Scorecard

```text
Using SNOWFLAKE.LOCAL.GET_AI_EVALUATION_DATA('PORT_VAN_AISQL', 'PORT_OPS', 'PORT_AGENT_BASELINE', 'CORTEX AGENT', 'baseline-run-1'):

1. For each question show: INPUT, a 150-character preview of OUTPUT, answer_correctness score, DURATION_MS, TOTAL_INPUT_TOKENS + TOTAL_OUTPUT_TOKENS.
2. Join to PORT_VAN_AISQL.PORT_OPS.EVAL_QUESTIONS on question text to add QUESTION_TYPE and EXPECTED_ANSWER.
3. Summarize by QUESTION_TYPE: average correctness, average seconds per answer, average tokens per answer.
4. Save the per-question results to a table EVAL_RESULTS_BASELINE.
```

**What this prompt does**

Expect the structured and lookup questions to score well. The `aggregate_text` questions (counts and totals over raw documents) usually score low, and they tend to be the slowest and most token-hungry, because the agent searches repeatedly and reasons over long text chunks.

### Key concepts

- **Answer correctness.** How closely the agent's final answer matches the ground truth, scored by an LLM judge.
- **Metric version.** Pins the judge model and rubric so scores are comparable across runs. This lab uses v3 for both agents.
- **Run-to-run variance.** Agents are non-deterministic, so the same dataset can score slightly differently on each run. Large gaps are signal; small ones may be noise.

### What you built in this section

- [ ] `PORT_EVAL_DATA` evaluation table and `PORT_EVAL_DS` dataset
- [ ] `baseline-run-1` with correctness, latency and token metrics
- [ ] `EVAL_RESULTS_BASELINE` scorecard

---

## Section 4: Preprocess with AISQL

**Time:** 10:10 – 10:30 (20 min)

**Building:** Enriched tables where every fact from the raw text is a typed, queryable column

Instead of making the agent read raw text at question time, read it once in the pipeline. AISQL functions run inside SQL over every row, turning text into structured columns. After that, a count or total is plain SQL.

### Technologies used in this section

- **AI_EXTRACT.** Pull specific fields (terminal, delay, cost, penalty) out of text into a structured response.
- **AI_CLASSIFY.** Assign each row to one of a fixed set of categories (incident type, severity, email topic).
- **AI_FILTER / AI_TRANSLATE / AI_AGG.** Yes/no checks in natural language, translation, and summaries across many rows.

### Prompt 4.1 - Try the AISQL Functions on a Few Rows

```text
In PORT_VAN_AISQL.PORT_OPS, on 5 rows of INCIDENT_REPORTS_RAW, show side by side:

1. AI_EXTRACT on REPORT_TEXT returning: terminal (normalized to one of Deltaport, Vanterm, Centerm, Fraser Surrey Docks, Neptune Terminals), delay_hours (a number, convert minutes to hours), cost_cad (a number, expand k/K to thousands)
2. AI_CLASSIFY on REPORT_TEXT into category: equipment_failure, weather, safety_injury, environmental, security, vessel_operations
3. AI_CLASSIFY on REPORT_TEXT into severity: low, medium, high, critical

Show the original text next to the extracted values so we can check them.
```

**What this prompt does**

Example shape of the SQL:

```sql
SELECT report_id, report_text,
  AI_EXTRACT(
    text => report_text,
    responseFormat => {
      'terminal': 'Which terminal? One of Deltaport, Vanterm, Centerm, Fraser Surrey Docks, Neptune Terminals',
      'delay_hours': 'Downtime in hours as a number',
      'cost_cad': 'Estimated cost in CAD as a number'
    }
  ):response AS extracted,
  AI_CLASSIFY(report_text, ['low','medium','high','critical']):labels[0]::STRING AS severity
FROM PORT_VAN_AISQL.PORT_OPS.INCIDENT_REPORTS_RAW
LIMIT 5;
```

### Prompt 4.2 - Build INCIDENTS_ENRICHED

```text
In PORT_VAN_AISQL.PORT_OPS, create a table INCIDENTS_ENRICHED from all rows of INCIDENT_REPORTS_RAW using the same AI_EXTRACT and AI_CLASSIFY logic from the previous step.

Columns: REPORT_ID, SUBMITTED_AT, SUBMITTED_BY, TERMINAL_ID (join the extracted terminal name to TERMINALS.TERMINAL_NAME), TERMINAL_NAME, CATEGORY, SEVERITY, DELAY_HOURS (FLOAT), COST_CAD (FLOAT), REPORT_TEXT.

Then show: row count, number of rows where TERMINAL_ID, DELAY_HOURS or COST_CAD is NULL, and incidents per terminal.
```

**What this prompt does**

This is the one-time cost: the LLM reads each of the 300 reports once. Every question after this is a cheap SQL query instead of an LLM reading text.

### Prompt 4.3 - Enrich Inspections and Emails

```text
In PORT_VAN_AISQL.PORT_OPS, create two more enriched tables:

1. INSPECTIONS_ENRICHED from INSPECTION_REPORTS_RAW: INSPECTION_ID, CALL_ID, INSPECTED_AT, TERMINAL_ID (via VESSEL_CALLS on CALL_ID), COMMODITY and PENALTY_CAD (AI_EXTRACT, 0 if no penalty), OUTCOME (AI_CLASSIFY: released, penalty, seized, re_exported), OFFICER_NOTES.

2. EMAILS_ENRICHED from SHIPPER_EMAILS_RAW: EMAIL_ID, RECEIVED_AT, FROM_COMPANY, LANGUAGE ('en' or 'fr', via AI_EXTRACT or AI_CLASSIFY), BODY_EN (AI_TRANSLATE the body to English; keep English bodies as is), TOPIC (AI_CLASSIFY on BODY_EN: delay, damage, billing, documentation, compliment), IS_URGENT (AI_FILTER: "Does the sender say this is urgent or need an answer today?"), TERMINAL_ID (AI_EXTRACT the terminal name and join to TERMINALS), BODY.

Show row counts and a sample of 5 rows from each.
```

**What this prompt does**

`AI_TRANSLATE` normalizes French emails to English before classification, so one set of labels works for both languages. `AI_FILTER` returns a real `BOOLEAN` you can filter on.

### Prompt 4.4 - What Did the Pipeline Cost?

```text
In PORT_VAN_AISQL.PORT_OPS:

1. Use AI_COUNT_TOKENS to estimate the total input tokens our pipeline sent to the LLM: the sum of tokens in REPORT_TEXT (x3 functions), OFFICER_NOTES (x2) and email BODY (x4).
2. Query SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AISQL_USAGE_HISTORY for today, grouped by function name and model, showing tokens and credits. If it returns no rows yet, explain that ACCOUNT_USAGE views can lag by up to a few hours and rely on the estimate.
3. Use AI_AGG over INCIDENTS_ENRICHED.REPORT_TEXT where SEVERITY = 'critical' to write a 3-bullet summary of the main causes of critical incidents.
```

**What this prompt does**

Keep this number: it's the one-time cost of preprocessing. In Section 5 you'll compare it with the per-question savings of the AISQL agent.

> In production you wouldn't rebuild these tables by hand. Wrap the same SELECTs in **Dynamic Tables** with incremental refresh (or a stream + task), so only new reports go through the LLM as they arrive.

### Key concepts

- **Shift-left AI.** Run the LLM once per document at ingest time, instead of once per question at query time. Repeated questions get faster and cheaper, and aggregates become exact.
- **Normalization.** Mapping DP, Roberts Bank and Deltaport (GCT) to `TERMINAL_ID` T01 means every join and `GROUP BY` is reliable.

### What you built in this section

- [ ] `INCIDENTS_ENRICHED` (300 rows)
- [ ] `INSPECTIONS_ENRICHED` (120 rows)
- [ ] `EMAILS_ENRICHED` (150 rows, translated, classified, urgency-flagged)
- [ ] A one-time cost estimate for the pipeline

---

## Section 5: AISQL Agent & the Before/After

**Time:** 10:30 – 10:45 (15 min)

**Building:** A second agent over the enriched data, a second evaluation run, and the cost/speed comparison

### Technologies used in this section

- **Semantic View (enriched).** The enriched tables become first-class facts and dimensions with relationships to terminals.
- **Cortex Agent.** Same instructions and tools, but now aggregate questions resolve to a single SQL query.
- **Evaluation Compare.** Snowsight shows runs side by side, per question, with score and duration changes.

### Prompt 5.1 - Enriched Semantic View

```text
/semantic_studio Create a semantic view called PORT_OPS_ENRICHED_SV in PORT_VAN_AISQL.PORT_OPS that includes everything in PORT_OPS_SV plus INCIDENTS_ENRICHED, INSPECTIONS_ENRICHED and EMAILS_ENRICHED.

- Metrics: incident count, total and average DELAY_HOURS, total COST_CAD, inspection count, total PENALTY_CAD, email count, urgent email count
- Dimensions: CATEGORY, SEVERITY, OUTCOME, TOPIC, LANGUAGE, IS_URGENT, month of SUBMITTED_AT
- Relationships from each enriched table to TERMINALS on TERMINAL_ID, and INSPECTIONS_ENRICHED to VESSEL_CALLS on CALL_ID
- Same relationship rules as before: no join_type, left_table = fact (many) side, right_table = dimension (one) side, primary_key on dimension tables

Deploy it and test with: "What is the total delay in hours caused by equipment failures across all terminals?"
```

### Prompt 5.2 - Create the AISQL Agent

```text
In PORT_VAN_AISQL.PORT_OPS, create a Cortex Agent called PORT_AGENT_AISQL with the same orchestration model (auto) and the same instructions as PORT_AGENT_BASELINE, but with these tools:

- Cortex Analyst on PORT_OPS_ENRICHED_SV: use for every count, total, average or ranking, including incidents, inspections and emails
- Cortex Search on RAW_DOCS_SEARCH: use only when the user asks what a specific report or email says

Ask it "How many incidents were reported at Deltaport?" and "How many CBSA inspections resulted in a penalty, and what was the total penalty amount?" and show which tools it used.
```

### Prompt 5.3 - Evaluate the AISQL Agent

```text
Run a Cortex Agent evaluation of PORT_VAN_AISQL.PORT_OPS.PORT_AGENT_AISQL with exactly the same setup as baseline-run-1:

- Same existing dataset PORT_EVAL_DS (do not recreate it; remove the dataset: block from the YAML)
- Run name: aisql-run-1
- Metrics: answer_correctness and logical_consistency, version v3

Start it and poll STATUS until COMPLETED. Then save the per-question results to EVAL_RESULTS_AISQL in the same shape as EVAL_RESULTS_BASELINE.
```

### Prompt 5.4 - The Before/After Comparison

```text
In PORT_VAN_AISQL.PORT_OPS, compare EVAL_RESULTS_BASELINE and EVAL_RESULTS_AISQL:

1. A per-question table: question, question type, expected answer, baseline correctness vs AISQL correctness, baseline seconds vs AISQL seconds, baseline tokens vs AISQL tokens.
2. A summary row for each agent: average correctness, average seconds per answer, average tokens per answer, plus the % change from baseline to AISQL.
3. Break-even: using the one-time pipeline token estimate from Prompt 4.4 and the average tokens saved per question, how many questions until the AISQL pipeline has paid for itself?
4. Write 3 bullet points I could share with my leadership about the result.
```

**What this prompt does**

You can also open **AI & ML → Agents → (either agent) → Evaluations**, select both runs and click **Compare** to see the same before/after per question in the UI.

### Bonus: ask both agents and feel the difference

Paste any of these questions into the agent chat:

```text
Which terminal had the most high or critical severity incidents?
```

```text
What is the total estimated cost of environmental incidents?
```

```text
How many urgent customer emails were about shipment delays?
```

```text
Summarize the French-language complaints by topic and terminal.
```

### Key concepts

- **Why accuracy goes up.** Counts and totals come from SQL over every row, not from whichever chunks search happened to return.
- **Why it gets faster and cheaper.** The agent makes one Analyst call that returns a few rows, instead of several searches that return long text chunks for the LLM to read. Fewer tokens per answer, fewer LLM calls.
- **Break-even.** Preprocessing is paid once per document. Every question asked after that saves tokens, so the more the agent is used, the bigger the saving.

### What you built in this section

- [ ] `PORT_OPS_ENRICHED_SV` over structured + AISQL-enriched tables
- [ ] `PORT_AGENT_AISQL`, the optimized agent
- [ ] `aisql-run-1` evaluation
- [ ] A before/after scorecard on accuracy, latency and cost

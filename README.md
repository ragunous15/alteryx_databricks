# alteryx2databricks (a2d)

**Convert Alteryx workflows into reviewable Databricks code — no Alteryx license required.**

[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-Databricks%20License-red)](LICENSE.md)
[![CI](https://github.com/Subhranil-Artizent/Alteryx-to-Databricks/actions/workflows/ci.yml/badge.svg)](https://github.com/Subhranil-Artizent/Alteryx-to-Databricks/actions/workflows/ci.yml)

Upload a `.yxmd` file and get back candidate PySpark, Spark Declarative
Pipelines (DLT), Spark SQL, Lakeflow SQL, and Lakeflow Designer artifacts for
review. Code generation is not a claim that the artifact ran in Databricks or
that it produced the same business result as Alteryx. PySpark output can be
submitted as a one-time Databricks run only when the server's strict execution
gate finds no unsupported tools, placeholders, syntax/static-safety failures,
or review warnings and the conversion meets the configured coverage/confidence
thresholds.

> A syntactically valid `.yxmd` does not imply that every Alteryx or third-party tool has a lossless Spark equivalent. Unsupported or ambiguous workflows still produce downloadable review artifacts, but automatic execution is blocked instead of silently running best-effort code.

---

> **Visual Guide:** Open [docs/a2d-guide.html](docs/a2d-guide.html) in your browser for a 56-slide interactive guide covering the entire product.

### Quick Start

From this source checkout:

```bash
git clone https://github.com/Subhranil-Artizent/Alteryx-to-Databricks.git
cd Alteryx-to-Databricks
pip install "."
a2d convert my_workflow.yxmd -o output/                  # Emits ALL 5 formats (default)
a2d convert my_workflow.yxmd -o output/ -f pyspark       # Filter to PySpark only
a2d convert my_workflow.yxmd -o output/ -f pyspark,sql   # Filter to PySpark + SQL
```

Outputs land in per-format subdirectories: `output/pyspark/`, `output/dlt/`, `output/sql/`, `output/lakeflow/`.

### Run the upload server locally (Windows PowerShell)

Python 3.12 and Node.js 22 LTS are recommended:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python -m pip install -e ".[server]"

Push-Location frontend
npm ci
npm run build
Pop-Location

.\.venv\Scripts\python -m uvicorn server.main:app --host 127.0.0.1 --port 8000
```

Open `http://127.0.0.1:8000`, upload a `.yxmd`, review the readiness result,
and download the generated code. Conversion and download do not require
Databricks credentials. The Run button requires a deployed Databricks App with
user authorization; HTTP execution deliberately refuses shared CLI profiles,
environment credentials, and the App service principal.

## Table of Contents

1. [What is this?](#what-is-this)
2. [What can it convert?](#what-can-it-convert)
3. [What still needs manual work?](#what-still-needs-manual-work)
4. [Getting Started](#getting-started)
5. [Recent Improvements](#recent-improvements)
6. [How it works](#how-it-works)
7. [Quality & Observability](#quality--observability)
8. [CLI Reference](#cli-reference)
9. [Troubleshooting](#troubleshooting)
10. [Development](#development)
11. [License](#license)

---

## What is this?

Large organizations use Alteryx to build data pipelines visually. Moving those pipelines to Databricks usually means rewriting them from scratch — which takes months.

**a2d** reads Alteryx `.yxmd` workflow files and generates candidate Databricks implementations:

- **What you save:** weeks of manual rewriting per workflow
- **What you get:** candidate PySpark notebooks, Spark Declarative Pipelines
  (DLT), Databricks SQL, Lakeflow SQL, Lakeflow Designer definitions, and
  Workflow JSON. By default, a conversion attempts all configured formats; a
  format may still contain review items or be blocked for a particular node.
- **What it recognizes:** a versioned registry of Alteryx plugin names, tool
  adapters, and expression mappings. Registry presence is not the same as full
  semantic support: use the generated tool matrix and per-node
  `SUPPORTED` / `PARTIALLY_SUPPORTED` / `UNSUPPORTED` / `NOT_APPLICABLE`
  result for the uploaded workflow.

> **You do not need Alteryx installed** to run this tool.

---

## What can it convert?

The tool generates candidate Databricks / PySpark mappings for the following Alteryx tools. Validate each candidate against representative Alteryx output before production use:

The tables below describe intended mappings, not universal support for every
tool version and configuration. The conversion report is authoritative for an
individual upload. A node is `SUPPORTED` only when its selected configuration
has a deterministic implementation without a placeholder or manual step.

### Why one workflow can generate several `spark.read` calls

A `.yxmd` is one **workflow definition**, not one data file. It can contain
zero, one, or many Input Data / cloud input nodes. The parser discovers those
nodes from the uploaded XML and the generator emits one source binding (usually
one `spark.read`) for each source node. For example, a workflow that joins three
CSV inputs will generate three reads even though only one `.yxmd` was uploaded.
Those paths are workflow configuration; they are not hard-coded sample files
added by the converter. Each source must be mapped to a table, Unity Catalog
Volume, or cloud location accessible to the selected Databricks identity before
the notebook can run.

### Reading and Writing Data

| Alteryx Tool | What it becomes in Databricks |
|---|---|
| Input Data (CSV, Parquet, JSON, Avro) | `spark.read.format(...).load(path)` |
| Input Data (database / ODBC query) | `spark.sql("""SELECT ...""")` with a TODO to map the connection |
| Output Data | `df.write.format(...).save(path)` |
| Text Input (inline data) | `spark.createDataFrame([...])` |
| Browse | `display(df)` |
| Dynamic Input (ModifySQL) | A Python loop that runs one parameterized SQL query per input row |

### Preparing Data

| Alteryx Tool | What it becomes in Databricks |
|---|---|
| Select (rename/drop columns) | `df.drop(...)` / `df.withColumnRenamed(...)` |
| Filter | `df.filter(condition)` — True/False outputs split into two DataFrames |
| Formula | `df.withColumn("field", expression)` |
| Multi-Field Formula | Multiple `withColumn` calls in one step |
| Multi-Row Formula | Window functions (`F.lag`, `F.lead`) |
| Sort | `df.orderBy(...)` |
| Sample (first N / random / percent) | `df.limit(n)` or `df.sample(fraction)` |
| Unique / Deduplicate | `df.dropDuplicates(key_fields)` |
| Data Cleansing | Trim, null handling, case conversion |
| Record ID | `row_number()` over an explicit Spark window, with an ordering warning |
| Imputation | Missing value fill logic |

### Combining Data

| Alteryx Tool | What it becomes in Databricks |
|---|---|
| Join (inner/left/right/full) | `df_left.join(df_right, condition, how=...)` |
| Union | PySpark candidate preserves position/name/common modes, mismatch policy, schema precedence, and configured union sequence; Spark row order still requires downstream sorting, while other formats are review-gated |
| Append Fields | Full cross join with Cartesian policy and embedded field selection/order |
| Find Replace | Lookup-based replacement |

### Transforming Data

| Alteryx Tool | What it becomes in Databricks |
|---|---|
| Summarize (group + aggregate) | `df.groupBy(...).agg(F.sum(), F.avg(), ...)` |
| Cross Tab (Pivot) | `df.groupBy(...).pivot(...).agg(...)` |
| Transpose (Unpivot) | `stack()` expression |
| Running Total | Window function with cumulative sum |
| Tile / Quantile binning | `F.ntile(n).over(window)` |

### Parsing

| Alteryx Tool | What it becomes in Databricks |
|---|---|
| RegEx (match / replace / parse) | `F.rlike(...)` / `F.regexp_replace(...)` |
| Text to Columns | `F.split(col, delimiter)` |
| DateTime parse/format | `F.to_timestamp(...)` / `F.date_format(...)` |
| JSON Parse | `F.get_json_object(...)` |

### Formula Functions

The expression engine translates **141 Alteryx formula functions** to PySpark equivalents, including:

| Category | Examples |
|---|---|
| String (24) | `Contains`, `Left`, `Right`, `Trim`, `Replace`, `RegexMatch`, `Substring` |
| Math (21) | `Abs`, `Round`, `Ceil`, `Floor`, `Sqrt`, `Pow`, `Log`, `Mod`, `Rand` |
| Date/Time (15) | `DateTimeNow`, `DateTimeAdd`, `DateTimeDiff`, `DateTimeFormat`, `DateTimeParse` |
| Conversion (9) | `ToNumber`, `ToInteger`, `ToString`, `ToDate`, `ToDateTime` |
| Test / Null (8) | `IsNull`, `IsEmpty`, `IsNumber`, `Coalesce`, `IIF`, `Null`, `IfNull` |
| Conditional | `IF/THEN/ELSEIF/ELSE/ENDIF`, `IIF`, `Switch` |

---

## What still needs manual work?

Some Alteryx features cannot be converted losslessly. Known gaps are surfaced as warnings, `# TODO` markers, or unsupported nodes, and any such signal blocks automatic Databricks execution. Because unusual third-party tools and environment-specific behavior cannot be exhaustively inferred from XML alone, generated code still needs validation against representative data.

| Category | What you'll need to do |
|---|---|
| **Database connections** (ODBC/DSN) | The SQL is preserved; you replace the connection string with a Unity Catalog table name or Databricks JDBC URL |
| **Excel files** (`.xlsx` / `.xls`) | Upload the file to DBFS or a Unity Catalog Volume, then replace the placeholder with a proper read call |
| **Local / network paths** (`\\server\share\...`) | Upload files to cloud storage; the tool flags every occurrence with a `# WARNING` comment |
| **Predictive / ML tools** | Use MLflow + scikit-learn or Spark MLlib |
| **Spatial tools** | Use Sedona or Mosaic libraries |
| **Reporting / Layout tools** | Use Databricks AI/BI dashboards |
| **Email / Publish tools** | Use Lakeflow Jobs notification actions |
| **R Tool** | Rewrite R code in Python/PySpark |
| **Iterative macros** | Require manual rewrite as Lakeflow Jobs |
| **Custom third-party Alteryx tools** | Require bespoke conversion |

> **Tip:** After converting, search the output file for `# TODO` and `# WARNING` — every item that needs attention is marked there.

---

## Getting Started

Choose the option that matches your comfort level:

### Option A: Command Line

For users comfortable with a terminal.

```bash
# Install (CLI only — no web UI)
pip install "."

# Convert a single workflow — emits ALL 5 formats by default
# (output/pyspark/, output/dlt/, output/sql/, output/lakeflow/)
a2d convert my_workflow.yxmd -o output/

# Restrict to one or more formats
a2d convert my_workflow.yxmd -o output/ -f pyspark
a2d convert my_workflow.yxmd -o output/ -f pyspark,sql

# Convert all workflows in a folder (still all 5 formats)
a2d convert workflows/ -o output/

# Generate a migration readiness report
a2d analyze workflows/ -o report/
```

After conversion, the CLI prints (mirroring the Convert page in the web UI):

- a **conversion review banner** that says whether the candidate passed the
  static conversion gate, needs review, or cannot be used as-is. Passing the
  gate is not evidence of a Databricks run or output equivalence
- a one-line counts row: coverage %, confidence /100, tools converted, nodes needing review, blockers
- warnings **grouped by category**: `Cannot convert` (blocker) · `Manual review needed` · `Graph structure note` · `Other` — instead of a flat dump
- per-format status table (PySpark / Spark Declarative Pipelines / SQL / Lakeflow) with the **best format** highlighted
- automatic Python syntax validation on every generated `.py`

To target a specific cloud for the auto-generated `node_type_id` in the
Workflow JSON / DAB, pass `--cloud aws` (default), `--cloud azure`, or
`--cloud gcp`.

> **Windows note:** If `a2d` is not on PATH, use `python -m a2d` instead.

---

### Option B: Advanced deployment

<details>
<summary><strong>Self-hosted (any Linux/macOS host)</strong></summary>

```bash
pip install -e ".[server]"
make frontend
PYTHONPATH=src:. uvicorn server.main:app --host 0.0.0.0 --port 8000
```
Open http://localhost:8000. For Postgres-backed history, set `A2D_DATABASE_URL=postgresql://...`.
</details>

<details>
<summary><strong>React Web UI (requires Node.js 18+)</strong></summary>

```bash
pip install ".[server]"
cd frontend && npm install && npm run build && cd ..
PYTHONPATH=src:. uvicorn server.main:app --host 0.0.0.0 --port 8000 --reload
```

Open http://localhost:8000. Includes DAG visualization, batch WebSocket progress, conversion history, and a tool support matrix.
</details>

<details>
<summary><strong>Databricks Apps</strong></summary>

The repo ships with a `databricks.yml` bundle and an `app.yaml` for one-command deploys via Databricks Asset Bundles:

```bash
# Build frontend + deploy to a target defined in databricks.yml
make deploy-dev      # → databricks bundle deploy -t dev
make deploy-prod     # → databricks bundle deploy -t prod
make bundle-validate # validate the bundle before deploying
```

The app requests the `workspace` and `jobs` user-authorization scopes. A
workspace admin must enable Databricks Apps user authorization (Public Preview),
and each user must consent. Runs execute with that user's Databricks permissions;
there is no app service-principal or shared-profile fallback. The bundle grants
no blanket `CAN_USE` permission; add only the specific users or migration group
that should share access and conversion history.

The Convert page always separates the two operations:

1. `POST /api/convert` parses the XML, builds the DAG/IR, and returns generated files. It never contacts Databricks.
2. `POST /api/databricks/runs` accepts only the exact PySpark notebook covered by the conversion response's short-lived, content-hash-bound capability. It imports that notebook under the current user's workspace folder and submits an idempotent one-time Jobs run.
3. `GET /api/databricks/runs/{run_id}` requires a separate signed capability bound to that run, notebook, workspace, and user before it provides polling state or a run-page URL.

Automatic Run is intentionally limited to the audited `TextInput` and `Browse`
tool subset. All other successfully generated notebooks remain
downloadable but require review and manual execution. Replaying one conversion
capability maps to the same Databricks idempotency token for that user even if
the HTTP idempotency header changes. After one submission the button directs the
user to convert again before requesting a distinct run.

If you need to deploy by hand instead, run `make frontend` first (built assets are not committed), then sync `src/`, `server/`, `frontend/dist/`, `demo/`, plus `app.yaml`, `pyproject.toml`, and `requirements.txt` to a workspace folder and run `databricks apps create` / `databricks apps deploy` against it.

**Environment variables** (all optional):

| Variable | Default | Description |
|---|---|---|
| `A2D_CORS_ORIGINS` | `["http://localhost:5173"]` | Allowed CORS origins |
| `A2D_MAX_UPLOAD_SIZE_BYTES` | `52428800` (50 MB) | Max upload file size |
| `A2D_MAX_BATCH_UPLOAD_SIZE_BYTES` | `209715200` (200 MB) | Aggregate upload bytes retained by one batch request |
| `A2D_MAX_BATCH_FILES` | `50` | Max files per batch |
| `A2D_LOG_LEVEL` | `info` | Logging level |
| `A2D_DATABRICKS_WORKSPACE_ROOT` | `/Users/<current-user>/.a2d/generated` | Optional private child of the authenticated user's `/Users/<user>` home; `/Shared` and other users are rejected |
| `A2D_DATABRICKS_EXISTING_CLUSTER_ID` | `""` | Optional approved existing compute; empty requests serverless Jobs compute |
| `A2D_EXECUTION_SIGNING_KEY` | random per process | Optional secret used to keep run capabilities valid across restarts/replicas |
| `A2D_RUN_STATUS_TOKEN_TTL_SECONDS` | `86400` | Lifetime of a signed, run-bound polling capability (maximum 7 days) |
| `A2D_DATABASE_URL` | `""` (disabled) | PostgreSQL URL for conversion history |
| `A2D_DB_BACKEND` | `""` | Set to `lakebase` to use Databricks Lakebase Postgres for history (see `server/services/lakebase.py`) |
| `PORT` | `8000` | Server port |
</details>

---

## Recent Improvements

See the full [CHANGELOG.md](CHANGELOG.md) for detailed release notes.

**v2.0** (current workspace build) — hardened upload parsing, preserved parallel
workflow connections, corrected PySpark semantics for key tools, safe literal
rendering, explicit content-bound Databricks submission, and a fail-closed Run
gate in the web UI.

**v1.5** — Lakeflow Designer output, confidence scoring, complexity analysis, connection mapping, expression audit, performance hints, Unity Catalog DDL, DAB generation, multi-format default (every conversion emits all 5 formats), cloud-portable Workflow JSON / DAB via `--cloud aws|azure|gcp`, categorized warnings + 3-tier deploy-readiness banner, and more. 141 expression functions, 1006 tests.

---

## How it works

> This section is for the technically curious. You don't need to read it to use the tool.

### Architecture

Code generation is the first lifecycle segment, not the end of the migration:

```
  .yxmd file
       │
       ▼
  ┌──────────┐     ┌──────────────────┐     ┌─────────────┐     ┌──────────────┐
  │  Parser  │────▶│  Converter + IR  │────▶│  Generator  │────▶│ Output files │
  │ XML→DAG  │     │ Tool-specific    │     │ Format-     │     │ .py / .sql / │
  │          │     │ ParsedNode→      │     │ specific    │     │ .json        │
  └──────────┘     │ IRNode           │     └─────────────┘     └──────────────┘
                   └──────────────────┘
```

The generated files then move through separately recorded stages: bounded
static/configured validation, optional live Databricks submission and terminal
run status, and output reconciliation against supplied Alteryx evidence.

**Phase 1 — Parse:** The `.yxmd` XML is read and turned into a typed `WorkflowDAG` of `ParsedNode` objects. Each Alteryx plugin name is mapped to a human-readable tool type via `PLUGIN_NAME_MAP`.

**Phase 2 — Convert → Generate:** Each node is converted to a typed IR node
(e.g. `FilterNode`, `JoinNode`) by a tool-specific converter in
`ConverterRegistry`. Expression strings are tokenized, parsed into an AST, and
translated to PySpark or SQL. The IR DAG is then walked in topological order by
the target generator. Partial and unsupported nodes remain explicit review
records; they are not silently counted as complete.

`CODE_GENERATED` does not imply `DATABRICKS_EXECUTION_SUCCEEDED`, and neither
implies `OUTPUT_VALIDATION_PASSED`. See the
[enterprise conversion contract](docs/enterprise-conversion-contract.md).

### Expression Engine

The expression engine is a full recursive-descent parser. It handles:
- All standard operators (`+`, `-`, `*`, `/`, `%`, `==`, `!=`, `<`, `>`, `AND`, `OR`, `NOT`)
- Field references (`[FieldName]`)
- Row references (`[Row-1:Field]`, `[Row+1:Field]`) → translated to `F.lag` / `F.lead` window functions
- `IF/THEN/ELSEIF/ELSE/ENDIF` → `F.when(...).when(...).otherwise(...)`
- `IN(val, list...)` → `.isin(...)`
- Nested function calls (141 functions — see [What can it convert?](#what-can-it-convert))
- `Switch` → chained `F.when` expressions

When an expression can't be translated, a `# TODO` placeholder is emitted with the original Alteryx expression preserved as a comment.

---

## Quality & Observability

### Confidence Scoring

Every conversion produces a **static confidence heuristic** (0–100%) estimating
review risk across five weighted dimensions:

| Dimension | Weight | What It Measures |
|---|---|---|
| Tool Coverage | 35% | % of nodes with supported converters |
| Expression Fidelity | 25% | % of expressions that translated cleanly |
| Join Completeness | 15% | Whether join keys were fully resolved |
| Data Type Preservation | 15% | Type casts preserved correctly |
| Generator Warnings | 10% | Inverse of warning count |

> This score is a prioritization aid, not a measurement of correctness. Even a
> high score does not prove that Databricks executed the artifact or that its
> output matches Alteryx.

### Complexity Analysis

Workflows are scored on 7 factors to estimate migration effort:

| Factor | Weight |
|---|---|
| Node Count | 18% |
| Tool Diversity | 13% |
| Expression Complexity | 18% |
| Unsupported Tools | 23% |
| Macro Usage | 8% |
| DAG Depth | 10% |
| Spatial Tools | 10% |

Effort bands: **Low** (<30, ~2h) · **Medium** (30-50, ~8h) · **High** (50-70, ~16h) · **Very High** (>70, ~40h)

### Enriched Warnings

All warnings include remediation hints with 50+ specific recommendations. The JSON report includes `enriched_warnings` with `hint`, `category`, and actionable next steps for each warning.

### Additional Observability Tools

| Feature | Flag | Output |
|---|---|---|
| Expression Audit | `--expression-audit` | CSV of every expression translation (original → translated, pass/fail) |
| Performance Hints | `--performance-hints` | Broadcast join, persist, repartition, sequential join detection |
| Connection Mapping | `--connection-map FILE` | YAML-based Alteryx connection → Unity Catalog mapping |
| Unity Catalog DDL | `--generate-ddl` | `CREATE TABLE` / `CREATE EXTERNAL TABLE` statements |
| DAB Generation | `--generate-dab` | PySpark-only `databricks.yml` with job + DBR cluster configuration |

---

## CLI Reference

a2d provides 12 commands. Run `a2d --help` for the full list, or `a2d <command> --help` for details on any command.

| Command | Purpose |
|---|---|
| `convert` | Convert workflows — emits PySpark, Spark Declarative Pipelines (DLT), SQL, Lakeflow and Designer code in one run; use `-f` to filter |
| `analyze` | Generate migration readiness reports (HTML/JSON) |
| `portfolio` | Analyze a whole estate — cross-workflow dependencies, shared macros, and a migration-wave plan |
| `validate` | Check generated Python syntax |
| `verify` | Compare a modeled workflow result with supplied sample/golden data; scope and unsupported operators remain explicit (see below) |
| `suggest` | Write AI suggestions for what the converter couldn't convert (opt-in; see [AI assistant](#ai-assistant-opt-in)) |
| `sync` | Incrementally re-convert a directory — only changed or new workflows |
| `advise` | Recommend a cluster size and surface Spark optimization hints |
| `profile` | Profile a sample-data CSV: per-column type, null rate, and range |
| `list-tools` | Show supported Alteryx tool matrix |
| `plugins` | List installed source frontends and converter plugins |
| `version` | Show a2d version |

**Example:** `a2d convert workflow.yxmd -o output/ --comments --expression-audit --performance-hints` (all 5 formats)
**Filter example:** `a2d convert workflow.yxmd -f pyspark,sql -o output/`
**Cloud target:** `a2d convert workflow.yxmd --cloud azure -o output/` (drives `node_type_id` in Workflow JSON / DAB; `aws|azure|gcp`, default `aws`)

### Sample-output verification (`a2d verify`)

`convert` generates artifacts and `validate` performs bounded static checks.
`verify` runs the modeled, supported subset through an independent **pandas
reference executor** over supplied sample inputs and can compare schema,
order-insensitive rows, and values (with numeric tolerance) to a supplied golden
file. A passing comparison is evidence for that fixture and configuration only;
it is not a universal proof and it is not a live Databricks run.

```bash
# Install the verification extra (adds pandas; pyspark optional)
pip install "alteryx2databricks[verify]"

# Golden mode — compare to expected output exported from Alteryx
a2d verify workflow.yxmd -i sales=sales.csv -e expected.csv

# Reference-only — produce the reference result (no ground truth to diff against)
a2d verify workflow.yxmd -i sales=sales.csv

# Machine-readable report for CI
a2d verify workflow.yxmd -i sales=sales.csv -e expected.csv --json report.json
```

- `-i KEY=PATH.csv` supplies sample input per source (table name, file path, or node id); omit
  for workflows whose sources are embedded TextInput data.
- When a JVM is present, `verify` may also cross-check the pandas result against
  a Spark session. A local/CI Spark result is reported separately and must not
  be described as execution in a selected Databricks workspace.
- Exit code is non-zero only on an actual **FAIL**; an inconclusive run (no ground truth) exits 0.
- Workflows containing operators the reference executor doesn't model report a *partial* result
  (verified subset only) — never a false pass.

### Estate-wide planning (`a2d portfolio`)

Most migrations fail on sequencing, not on any single workflow. `portfolio` analyzes
many workflows together and answers "what do we migrate first?":

```bash
a2d portfolio ./workflows/ -o portfolio-out/
```

- **Cross-workflow dependencies** — which workflow writes a file or table another
  one reads, so you don't migrate a consumer before its producer.
- **Shared macros and duplicate sub-flows** — logic repeated across the estate that
  should be migrated once and reused.
- **A migration-wave plan** — workflows grouped into dependency-respecting waves,
  ranked by value x readiness / effort, with a person-day estimate per wave.
- Emits a rich console summary, an **executive dashboard** (`executive_dashboard.html`)
  with estate-wide coverage/effort/risk rollups, plus HTML and JSON reports.

Also available in the web UI at **Assess → Portfolio**.

### Cluster and cost advice (`a2d advise`)

```bash
a2d advise workflow.yxmd --cloud azure
```

Recommends a starting cluster tier (single-node → small → medium → large) with a
worker count, the cloud-specific `node_type_id`, a relative DBU/hour proxy and a
Photon recommendation — then lists per-node Spark optimization hints (broadcast
joins, cross joins, persist/repartition, sequential joins).

Derived from the workflow's **shape** (node count, DAG depth, shuffle/spatial/ML
operations), not your data volumes. It's a planning aid, not a benchmark or a quote.
Also available in the web UI at **Validate → Advisor**.

### Incremental re-conversion (`a2d sync`)

```bash
a2d sync ./workflows/ -o output/ --manifest .a2d-manifest.json
```

Converts only what changed. A JSON manifest tracks each source file's hash, so
unchanged workflows are skipped, new and modified ones are re-converted, and deleted
ones are pruned. A failed file isn't recorded, so it retries next run.

Intentionally **CLI-only** — this is designed to run on a schedule (cron, a Databricks
job, or a CI step) as your Alteryx estate keeps changing during a long migration.

### Data profiling (`a2d profile`)

```bash
a2d profile sample-data.csv
```

Infers per-column type, null rate and value range from a sample CSV — useful when
preparing golden data for `a2d verify` or sanity-checking an extract. CLI-only
utility.

### Plugins and frontends (`a2d plugins`)

```bash
a2d plugins
```

Lists the installed source frontends (`alteryx`, `dbt`, plus any third-party ones
registered via the `a2d.frontends` entry point) and converter plugins loaded from the
`a2d.converters` entry point, including any that failed to load and why. CLI-only
introspection — see [docs/converter-sdk.md](docs/converter-sdk.md) to write your own.

### AI assistant (opt-in)

**By default a2d never calls a language model.** Conversion is entirely
deterministic, and nothing here changes that.

If you configure Azure OpenAI GPT-4.1 (the five `AZURE_OPENAI_*` settings shown
under **Full workflow review** below), the web Migration Assistant uses that same
deployment for ordinary grounded text chat. A Databricks Foundation Model API
endpoint remains available as a legacy fallback when Azure is entirely absent.
These features can only **suggest** and **explain** — an LLM never modifies the
deterministic conversion itself, and its output arrives as a separate document
you read and apply yourself.

```bash
export A2D_FMAPI_ENDPOINT="https://<workspace>/serving-endpoints/<name>/invocations"
export A2D_FMAPI_TOKEN="..."   # optional — omit to use ambient workspace credentials

a2d suggest workflow.yxmd          # writes workflow_suggestions.md
```

- **`a2d suggest`** writes a Markdown report describing every gap the converter
  left — unsupported tools, TODO stubs, expression fallbacks, connection issues —
  with a suggested Databricks implementation for each. With no endpoint configured
  it still succeeds and still writes the deterministic gap list, just without
  suggestions.
- **The Assistant page** (`/chat` in the web UI) is a chatbot for discussing the
  migration: why the converter made the choices it did, how to handle a gap, and
  what the trade-offs are. It asks a few clarifying questions, then generates the
  same downloadable report.

To enable it in a Databricks Apps deployment, pass the endpoint at deploy time:

```bash
databricks bundle deploy --var fmapi_endpoint="https://<workspace>/serving-endpoints/<name>/invocations"
```

In-workspace serving endpoints need no token — the app's service principal
credentials are used. Leave `fmapi_endpoint` empty (the default) to keep AI off
entirely; the Assistant page then shows setup instructions instead of an upload
form.

When any Azure OpenAI setting is present, the assistant does not silently fall
back to FMAPI if the Azure configuration is incomplete. Complete the five Azure
settings and restart Uvicorn; `/api/chat/status` then reports the provider as
`azure_openai` without returning the key, endpoint, or deployment name.

#### Full workflow review

The **Full Workflow Review** page accepts one `.yxmd` upload and offers two
separate candidates:

- **Simple** generates the complete PySpark notebook with the deterministic
  converter and never calls an external model.
- **Azure OpenAI GPT-4.1** keeps deterministic input/output and side-effect
  cells, sends a bounded credential-redacted semantic workflow manifest to the
  configured GPT-4.1 deployment, and generates the transformation cells. Model
  output must pass strict JSON-shape, Python-syntax, graph-contract, DataFrame
  variable, and static-policy checks before it is displayed.

For local use, place these names in the repository-root `.env` file and restart
Uvicorn. `AZURE_OPENAI_DEPLOYMENT` is the Azure deployment name; it does not
have to equal the underlying model name.

```dotenv
AZURE_OPENAI_API_KEY=<your-key>
AZURE_OPENAI_ENDPOINT=https://<resource>.openai.azure.com
AZURE_OPENAI_API_VERSION=<version-supported-by-your-deployment>
AZURE_OPENAI_DEPLOYMENT=<your-gpt-4.1-deployment-name>
AZURE_OPENAI_MODEL=gpt-4.1
```

The API key and endpoint are never returned by the status endpoint. `.env` and
`.env.*` are excluded from both Git and Databricks App source deployment.
Changing `.env` requires a server restart.

Both results include node-coverage evidence, required runtime bindings, a
SHA-256 digest, a complete code preview, and an exact notebook download. The
artifact remains `REQUIRES_REVIEW`, `OUTPUT_VALIDATION_NOT_RUN`, and
`DATABRICKS_EXECUTION_NOT_RUN`. Static checks confirm complete generated node
coverage and valid Python syntax, but they do not prove semantic equivalence.
GPT-4.1 output is marked as model-generated and is never approved for automatic
Databricks execution. Compare the notebook output with representative Alteryx
output before production use.

---

## Troubleshooting

| Issue | Solution |
|---|---|
| `ModuleNotFoundError: a2d` | Run `pip install "."` or set `PYTHONPATH=src` |
| Syntax errors in generated output | Run `a2d validate output.py` — check expression TODOs |
| Low confidence score (<50%) | Run `a2d list-tools --supported` to check unsupported tools |
| WebSocket not connecting | Verify CORS origins in `A2D_CORS_ORIGINS` env var |
| Excel path placeholders | Upload `.xlsx` to DBFS or Volume, update the path in output |
| Network path warnings (`\\server\...`) | Upload files to cloud storage first |
| Empty expression errors | Check original workflow for blank formula fields |
| Missing join keys | Look for `# TODO: join keys` in the generated output |
| Server not starting | Use `PYTHONPATH=src:. uvicorn server.main:app` (not `a2d.server.main:app`) |

---

## Development

### Setup

```bash
# Install all dev dependencies (pytest, mypy, ruff, etc.)
make dev
# or: pip install -e ".[all]"
```

### Commands

```bash
make test        # Run all tests
make test-cov    # Run tests with coverage report
make lint        # Lint with ruff
make format      # Format with ruff
make typecheck   # Type check with mypy
make all         # Lint + typecheck + test
make serve       # Start dev server with hot-reload
make frontend    # Build React frontend (npm install + npm run build)
make clean       # Remove build artifacts
```

### Adding a New Tool Converter

1. Create a file in `src/a2d/converters/<category>/`
2. Add an IR node class in `src/a2d/ir/nodes.py` if needed
3. Implement a converter extending `ToolConverter` with `@ConverterRegistry.register`
4. Add the plugin name mapping in `src/a2d/parser/schema.py` → `PLUGIN_NAME_MAP`
5. Add a visitor method in each generator (`pyspark.py`, `dlt.py`, `sql.py`; Lakeflow inherits from SQL)
6. Add a unit test in `tests/unit/converters/`

### Project Structure

```
src/a2d/
  cli.py                   # Typer CLI (12 commands)
  config.py                # Configuration dataclasses
  pipeline.py              # Orchestration: Parse → Convert → Generate
  connections.py           # YAML connection mapping (Alteryx → Unity Catalog)
  parser/                  # .yxmd XML parsing
  ir/                      # Typed IR nodes + WorkflowDAG
  converters/              # 113 converters handling 158 tool types (8 categories)
  expressions/             # Expression engine (tokenizer → parser → AST → translator, 141 functions)
  generators/              # PySpark, DLT, SQL, Lakeflow, Designer, DDL, DAB, Workflow JSON
  frontends/               # Pluggable source frontends (Alteryx, dbt) — one IR, many sources
  analyzer/                # Complexity, coverage analysis
  observability/           # Confidence scoring, warning categorization, deploy status,
                           #   expression audit, performance hints
  verification/            # Semantic equivalence harness (pandas reference executor)
  advisor/                 # Cluster/cost advice + the opt-in advisory LLM (context, report, chat)
  portfolio/               # Estate-wide analysis, dependency graph, executive dashboard
  macro/                   # .yxmc macro expansion (inlined into the parent DAG)
  review/                  # Interactive review sessions (canvas ↔ generated code)
  bridges/                 # Spatial (ST/Sedona/H3) + AI/BI dashboard generation
  incremental/             # Manifest tracking for `a2d sync`
  sdk/                     # Stable public contract for converter plugins
  validation/              # Syntax validation

server/                    # FastAPI backend
  main.py                  # App entry point
  routers/                 # REST endpoints (analyze, convert, chat, health, history,
                           #   review, tools, validate)
  services/                # Business logic
  websocket/               # Real-time batch progress

frontend/                  # React 19 + TypeScript + Tailwind 4
  src/
    routes/                # 11 pages (convert, batch, analyze, history, tools,
                           #   validate, review, chat, settings, about, home)
    components/            # UI components (workflow graph, code viewer, etc.)
    stores/                # Zustand state management
    lib/                   # API client, utilities
  dist/                    # Build output (NOT committed — `make frontend` / deploy builds it)

demo/                      # Sample .yxmd workflows for testing
tests/                     # pytest suite (1500+ tests, 85%+ coverage)
  fixtures/shared/         # Cross-language contract fixture (Python + TS assert the same rules)
docs/                      # Architecture, expression reference, migration playbook,
                           # visual guide (a2d-guide.html), conversion mapping
```

---

## Disclaimer

This project is a Databricks field-engineering asset provided **as-is**, without
warranties or any official Databricks support or SLA. It is **not** a Databricks
product. It is maintained on a best-effort basis by its owners. Use at your own
risk and validate all generated code before running it in production. See
[SECURITY.md](SECURITY.md) for how to report vulnerabilities and
[CONTRIBUTING.md](CONTRIBUTING.md) to contribute.

## License

This project is licensed under the Databricks License. See [LICENSE.md](LICENSE.md)
for details, and [NOTICE](NOTICE) for third-party attributions.

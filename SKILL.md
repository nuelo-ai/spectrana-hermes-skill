---
name: spectrana-data-analysis
description: "Use this skill whenever the user wants to analyze data using Spectrana via REST API. Triggers include: uploading CSV or Excel datasets, asking questions about data, requesting charts or visualizations, exploring patterns or trends, comparing columns, finding anomalies, generating statistical summaries, listing uploaded Spectrana files, or deleting uploaded datasets. Load this skill BEFORE making Spectrana API calls."
version: 1.0.0
author: Hermes Agent + Nuelo
license: MIT
metadata:
  hermes:
    tags: [spectrana, data-analysis, analytics, rest-api, plotly, discord]
---

# Spectrana Data Analysis Skill

Spectrana is Nuelo's AI-powered data analytics platform. Use the REST API to upload datasets, query data in natural language, retrieve dataset context, manage files, and render returned Plotly charts.

## Configuration

### Base URL

```text
https://api.spectra.nuelo.ai
```

### Authentication

All API calls except `/api/v1/health` require a Bearer token. In Hermes, the API key is stored in:

```text
~/.hermes/.env
```

Expected environment variable:

```bash
SPECTRANA_API_KEY=spe_...
```

Never print or expose the key. When showing commands to users, redact the token as `Bearer ***`.

Recommended key-loading pattern for shell commands:

```bash
set -a
source ~/.hermes/.env
set +a
```

Safer script-loading pattern:

```python
from pathlib import Path

def load_spectrana_key() -> str:
    env = Path.home() / ".hermes" / ".env"
    for line in env.read_text().splitlines():
        if line.startswith("SPECTRANA_API_KEY="):
            return line.split("=", 1)[1].strip().strip('"').strip("'")
    raise RuntimeError("SPECTRANA_API_KEY is not configured in ~/.hermes/.env")
```

## Available REST API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/health` | GET | Check service health |
| `/api/v1/files` | GET | List all uploaded files |
| `/api/v1/files/upload` | POST | Upload CSV/Excel file |
| `/api/v1/files/{file_id}` | GET | Get file details |
| `/api/v1/files/{file_id}/context` | GET | Get AI-generated data brief |
| `/api/v1/files/{file_id}/suggestions` | GET | Get query suggestions |
| `/api/v1/chat/query` | POST | Run natural language analysis |
| `/api/v1/files/{file_id}` | DELETE | Delete a file |

## Constraints and Safety Rules

- Supported file formats: `.csv`, `.xlsx`, `.xls`.
- Max file size: 50MB.
- Analysis queries cost credits.
- Confirm with the user before uploading a new dataset.
- Confirm with the user before deleting any dataset.
- Before running analysis queries, clearly state that queries may consume Spectrana credits.
- Upload files directly via curl or Python multipart upload. Do **not** load raw dataset contents into the LLM context for analysis.
- **Spectrana-source rule:** Hermes/assistant may synthesize, merge, compare, calculate, score, and visualize results **only when the source data/results come from Spectrana API responses** such as `analysis`, `execution_result`, `chart_specs`, `data_brief`, `data_summary`, or multiple Spectrana query outputs.
- **Raw-file boundary:** Hermes/assistant must not independently inspect, parse, query, compute, analyze, summarize, score, or derive findings directly from uploaded/raw user files using pandas, SQL, spreadsheet parsing, Python scripts, manual file reads, or similar non-Spectrana processing.
- Treat Spectrana's returned `analysis` and computed outputs as the source of truth. Any assistant synthesis must be traceable to Spectrana-provided outputs, not raw-file processing.

## Core Workflow

### Step 0 — Verify API Key

Before authenticated calls, verify `SPECTRANA_API_KEY` is available in `~/.hermes/.env`. If missing or unauthorized, ask the user for a valid `spe_...` key.

### Step 1 — Identify or Upload the File

If the user refers to an existing Spectrana file, call:

```text
GET /api/v1/files
```

If the user provides a new local/attached file, confirm before uploading:

> Before I upload this to Spectrana, I want to flag that analysis queries cost credits. Which would you prefer?
> 1. Upload now and analyze
> 2. Upload only
> 3. You upload manually through https://app.spectrana.nuelo.ai, then I analyze it here

Upload via direct file transfer only:

```bash
curl -s -X POST "https://api.spectra.nuelo.ai/api/v1/files/upload" \
  -H "Authorization: Bearer $SPECTRANA_API_KEY" \
  -F "file=@/absolute/path/to/file.csv"
```

Save the returned `file_id` for subsequent calls in the current task.

### Step 2 — Get Dataset Context Before Querying

Always call this before analysis:

```text
GET /api/v1/files/{file_id}/context
```

Use this to understand columns, row counts, data types, available context, and suggested questions.

### Step 3 — Run Analysis

Use:

```text
POST /api/v1/chat/query
```

Request body:

```json
{
  "query": "What is total revenue by region?",
  "file_ids": ["file-uuid-here"],
  "web_search_enabled": false
}
```

Response fields usually arrive under a top-level `data` object, with `credits_used` at the top level. Normalize before reading results:

```python
payload = json.loads(response_text)
result = payload.get("data", payload)
credits_used = payload.get("credits_used", result.get("credits_used"))
```

Result fields may include:

- `analysis` — Spectrana's narrative explanation
- `generated_code` — Python code generated by Spectrana
- `execution_result` — Raw computed results, often JSON string
- `chart_specs` — Plotly chart specification, often JSON string, null, or an empty string
- `chart_error` — chart generation error details, if any
- `follow_up_suggestions` — suggested next questions

If the user explicitly says to ask a question “as is” / “do not amend it,” send the exact query string unchanged in `/api/v1/chat/query`. Do not scope, clarify, rewrite, or add hidden analysis instructions unless the API requires separate structural fields such as `file_ids`.

### Step 4 — Present, Synthesize, and Visualize Spectrana Results

When presenting Spectrana results, include:

1. **Spectrana Analysis** — Present the `analysis` field faithfully.
2. **Tables / Execution Results** — If `execution_result` contains tabular data, format or visualize it.
3. **Chart** — If `chart_specs` is non-null, render and send the chart. This is mandatory.
4. **Follow-up Suggestions** — Include if provided.

Allowed assistant synthesis:

- You may summarize, merge, compare, calculate percentages/deltas, rank, score, and visualize information **derived from Spectrana API responses**.
- You may combine multiple Spectrana query outputs into one executive summary or report image.
- You may create derived KPI cards or risk labels if they are based only on Spectrana-provided `analysis` / `execution_result` / `chart_specs` / context fields.
- Clearly label source as Spectrana-derived when presenting synthesized output.

Not allowed:

- Do not open or parse the original dataset file to compute missing values, row-level metrics, scores, rankings, or findings.
- Do not use pandas, SQL, DuckDB, Excel readers, manual XLSX/CSV parsing, or similar direct-file processing to answer analytical questions.

Clearly label source:

- `Spectrana analysis:` for direct Spectrana-sourced content
- `Spectrana-derived synthesis:` for your synthesis/visualization built only from Spectrana responses
- `Operational note:` only for separate API/status/formatting comments

## Chart Rendering for Discord

If `chart_specs` is returned:

1. Parse the JSON string into Plotly `data` and `layout`.
2. Save the raw Spectrana API response JSON under a local output directory such as:

```text
~/hermes-projects/Spectrana/outputs/YYYY-MM-DD/{descriptive-name}-response.json
```

3. Render the chart/report as a PNG image for Discord delivery. HTML does **not** need to be stored unless the user explicitly asks for it.
4. If using Plotly/browser rendering temporarily, it is acceptable to create transient HTML only as an implementation detail; the durable artifacts should be the raw response JSON and PNG.
5. Send the image as a native Discord attachment using:

```text
MEDIA:/absolute/path/to/chart-or-report.png
```

Chart styling guidelines:

- Clear title
- Readable axis labels
- Clean white or transparent background
- Hide unnecessary Plotly modebar for screenshots
- Ensure the chart is fully loaded before screenshotting

## Common Operations

### Health Check

```bash
curl -s "https://api.spectra.nuelo.ai/api/v1/health"
```

### List Files

```bash
curl -s -X GET "https://api.spectra.nuelo.ai/api/v1/files" \
  -H "Authorization: Bearer $SPECTRANA_API_KEY"
```

### Get File Context

```bash
curl -s -X GET "https://api.spectra.nuelo.ai/api/v1/files/{file_id}/context" \
  -H "Authorization: Bearer $SPECTRANA_API_KEY"
```

### Run Query

```bash
curl -s -X POST "https://api.spectra.nuelo.ai/api/v1/chat/query" \
  -H "Authorization: Bearer $SPECTRANA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"Your question","file_ids":["file-id"],"web_search_enabled":false}'
```

### Delete File

Confirm with user first, then:

```bash
curl -s -X DELETE "https://api.spectra.nuelo.ai/api/v1/files/{file_id}" \
  -H "Authorization: Bearer $SPECTRANA_API_KEY"
```

## Error Handling

| Error Code / Status | Meaning | Action |
|---|---|---|
| `UNAUTHORIZED` / 401 | Invalid API key | Ask user for a valid key |
| `INSUFFICIENT_CREDITS` / 402 | No credits left | Ask user to top up or choose a lower-cost path |
| `FILE_NOT_FOUND` / 404 | Missing/wrong file ID | List files and ask/choose correct file |
| `INVALID_FILE_TYPE` / 400 | Unsupported format | Use `.csv`, `.xlsx`, or `.xls` |
| `FILE_TOO_LARGE` / 400 | File exceeds 50MB | Ask user to split/compress file |

## Discord Channel Behavior

This skill is especially relevant in the Majeve Systems `#spectrana` Discord channel, which is intended as a specialized Spectrana data analytics agent channel. Keep replies practical, data-focused, and explicit about when Spectrana credits may be used.

When creating a new Spectrana-style analytics channel or reusable agent prompt, use `templates/spectrana-agent-channel.md` as the starter template. It includes the analytics-only scope rule, off-topic warning language, required `spectrana-data-analysis` loading instruction, compact memory version, and a note that slash commands require Hermes gateway/command code rather than skill-file changes alone.

### Discord Formatting Rules

Discord does not reliably render wide markdown tables in bot messages. Avoid wide pipe tables for file lists or results with long IDs/names.

Preferred formats:

1. **Short lists in normal text** for 10-25 rows:
   - Use numbered bullets.
   - Put the filename on the first line.
   - Put ID and date on compact sub-lines.
   - Example:
     `1. dataruby3.xlsx`
     `   ID: 4e265...e633 • Created: 2026-05-18`

2. **Compact code blocks** for aligned output:
   - Use fixed-width columns.
   - Truncate long IDs to `first8…last4` unless the user asks for full IDs.
   - Truncate long filenames only when needed, preserving extension.

3. **Attachments for full data**:
   - For complete file inventories or large tables, write a `.csv` or `.md` file and attach it with `MEDIA:/absolute/path`.
   - In the Discord message, show a readable summary plus mention that the full table is attached.

4. **Professional report images for analysis results — default standard:**
   - For comparison results, rankings, KPI summaries, margin analysis, business performance analysis, or any important table/chart, render a clean report card and attach a PNG screenshot/image to Discord.
   - If `chart_specs` is null/empty or no browser screenshot path is available, create the report image directly with Python + Pillow from `analysis` and parsed `execution_result`. This is preferred over sending only Markdown tables in Discord.
   - The user likes a clean, sleek, professional report design. Use this as the default visual style for Spectrana analysis output.
   - Design style:
     - White card on very light slate background (`#f8fafc`).
     - Modern system font stack: `-apple-system, BlinkMacSystemFont, "Segoe UI", Inter, Arial, sans-serif`.
     - Large bold title, small muted subtitle with file name, source, and credits used.
     - KPI summary pills/cards at the top for best/worst/spread or equivalent key metrics.
     - Clean table with uppercase small headers, subtle borders, generous padding.
     - Inline horizontal bars for comparisons where useful.
     - Color coding: green for strong/positive, amber/orange for weaker positive/caution, red for negative/problem values.
     - Caveat/notes box in a soft warning color when Spectrana includes cautions.
     - Footer with source: `Spectrana API analysis result` and shortened file ID.
   - Save the raw Spectrana API response JSON under:
     `~/hermes-projects/Spectrana/outputs/YYYY-MM-DD/`
   - Save the generated report image as PNG under the same dated output directory.
   - Do **not** store report HTML by default. Only create/store HTML if the user explicitly asks for it or if needed as a temporary rendering implementation detail.
   - Verify the PNG is readable before delivery, then include it in Discord with:
     `MEDIA:/absolute/path/to/report.png`
   - Still include Spectrana's textual `analysis` in the Discord message, but keep it concise if the image already contains the main table.

Default for `GET /api/v1/files`: show a compact numbered list with shortened IDs and offer/attach a CSV if the list is long.
Default for Spectrana analysis/comparison output: attach a professional report image plus concise Spectrana analysis text.

## Complex Analysis Failure Handling

Spectrana analysis queries consume credits and may fail when the prompt asks for a large multi-step calculation, chart generation, top-N selection, trend analysis, and narrative synthesis all at once. Handle this conservatively while preserving the Spectrana-source boundary.

### Data-processing boundary

Hermes/assistant may synthesize, merge, compare, calculate, score, and visualize **Spectrana-provided outputs**. The assistant may use its judgment to turn one or more Spectrana responses into a clearer executive summary, report-card image, ranked list, KPI table, or composite score, as long as every analytical input comes from Spectrana API responses.

Hermes/assistant is **not authorized to independently inspect, parse, query, compute, analyze, summarize, score, or derive findings directly from uploaded/raw user files**. Do not use pandas, SQL, DuckDB, Excel readers, CSV parsing, manual XLSX XML parsing, or similar direct-file processing to answer analytical questions.

If Spectrana fails to generate usable analytical output, the assistant may make a **bounded retry through Spectrana only** by simplifying or decomposing the query. Do not perform local fallback analysis against the raw file.

Retry/decomposition limits:

- Maximum total attempts after seeing an error: **2 attempts total**. Count the failed original query as attempt 1; at most one additional retry/decomposition round is allowed unless the user explicitly asks for more.
- Maximum parallel Spectrana calls in a decomposition round: **2 parallel calls**.
- Before decomposing/retrying, tell the user that you are simplifying the request into smaller Spectrana queries to make it easier and faster for Spectrana to process.
- If the simplified/decomposed Spectrana calls also fail or return unusable output, stop and report the failures/status. Do not keep trying and do not analyze the raw file locally.

### Allowed examples

- **Allowed:** If one broad Spectrana query fails, tell the user you are simplifying it, then run up to 2 smaller Spectrana queries in parallel (for example: one query for activity concentration and one query for login/terminal anomalies), then synthesize those Spectrana outputs into one result.
- **Allowed:** Run three Spectrana queries — activity by user, suspicious login patterns, and unusual terminal usage — then merge those Spectrana results into one ranked risk-scoring report, as long as this is not part of a bounded error retry that exceeds the 2-parallel-call retry limit.
- **Allowed:** Take Spectrana's `execution_result` table and calculate percentages, deltas, ranks, or simple derived KPI cards for visualization.
- **Allowed:** Render a clean PNG report from Spectrana's `analysis`, `execution_result`, or `chart_specs`.
- **Allowed:** If Spectrana returns a risk score table, group the scores into Low/Medium/High bands and explain the banding.

### Not allowed examples

- **Not allowed:** Open the uploaded Excel/CSV with pandas/openpyxl/DuckDB/SQL to calculate fraud scores when Spectrana fails.
- **Not allowed:** Manually parse the `.xlsx` XML or read CSV rows to count users, IPs, events, revenue, refunds, or anomalies.
- **Not allowed:** Use local code to recreate missing Spectrana analysis from the raw source file.
- **Not allowed:** Read raw dataset contents into the LLM context and reason over them directly.

Handle failures as follows:

1. **Before expensive/complex queries**, tell the user the query may consume Spectrana credits.
2. **Prefer scoped prompts to Spectrana** when possible:
   - Ask Spectrana focused questions instead of one oversized prompt.
   - It is acceptable to run multiple Spectrana queries and synthesize their outputs afterward.
   - Still do not compute analytical subtasks from the raw file locally; analytical computation must come from Spectrana responses.
3. **If Spectrana returns an API error, validation error, invalid generated code, empty/null result, or unusable analysis, use the bounded Spectrana retry rule.**
   - Count the failed query as attempt 1.
   - You may make at most one additional retry/decomposition round, for **2 total attempts maximum** after an error is encountered.
   - In the retry/decomposition round, use no more than **2 parallel Spectrana calls**.
   - Before retrying, tell the user: `Spectrana had trouble with the broad query, so I’m simplifying it into smaller Spectrana queries to make it easier and faster to process.`
   - Synthesize successful retry outputs only if they come from Spectrana responses.
   - If the bounded retry also fails, report the exact error/status in plain language and stop.
   - Mention whether credits were refunded if the API says so.
   - Save or attach the raw Spectrana API response JSON if useful.
   - Do **not** attempt local deterministic fallback analysis against the raw file.
4. **Do not continue retrying beyond the bounded limit unless the user explicitly asks for another retry/rephrase.** This avoids infinite retry loops, system flooding, and unnecessary credit usage.
5. **Explain the error plainly in Discord:**
   - If the error is user-readable and not too technical, summarize it in simple language.
   - If the error is vague/technical/internal, say that Spectrana did not provide a meaningful or reliable analysis result.
6. **Avoid compounding credit usage** after failures. If the API says credit was refunded, mention it. If the API call succeeded but the returned generated analysis was unusable, be transparent that credits may have been used but no reliable answer was produced.
7. **When the user says “stop it,” stop immediately** and summarize only what was attempted; do not run more API/tool calls.

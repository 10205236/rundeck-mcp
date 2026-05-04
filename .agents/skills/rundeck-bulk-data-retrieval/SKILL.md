---
name: rundeck-bulk-data-retrieval
description: 'Retrieve large volumes of executions from Rundeck MCP and produce analysis reports. Use for: bulk execution history, job usage statistics, paginated data collection, Rundeck project reports. Handles API pagination, timeouts, rate limits, and status validation errors.'
---

# Rundeck Bulk Data Retrieval

## When to Use

- Retrieving complete execution history for a Rundeck project
- Generating job usage/statistics reports
- Analyzing job activity across many jobs
- Any task requiring more than a single page of execution results

---

## Critical API Constraints

These were learned through live experimentation — do not skip this section.

| Constraint | Value | Notes |
|---|---|---|
| API timeout | ~5 seconds | Hard limit, cannot be changed |
| Safe page size | **500–600 records** | Above this causes 504 Gateway Timeout |
| Rate limit | 10,000 req / 10 seconds | Effectively unlimited for sequential calls |
| Pydantic status validation | Fails on `failed-with-retry` | Some pages will error — see below |

**Never use `limit: 1000`** — it reliably triggers a 504 timeout. Use **500** as the default page size.

---

## Step 1 — Discover Total Scope

Before fetching everything, binary-search the total record count to know how many pages you need.

```
# Fetch a probe at a high offset to find the end
list_executions(project="X", limit=1, offset=50000)  → empty → try 30000 → etc.
list_executions(project="X", limit=500, offset=18500) → 173 records → total ≈ 18673
```

Record the approximate total. You'll need it to plan pages.

---

## Step 2 — Plan Your Pages

```
total ≈ 18673
page_size = 500
pages_needed = ceil(18673 / 500) = 38 pages
offsets = [0, 500, 1000, 1500, ..., 18500]
```

Use **step 500** (not 200 — that's unnecessarily slow; not 1000 — that times out).

---

## Step 3 — Fetch All Pages

Fetch pages **one at a time** (not in parallel — sequential is safer against rate limits and easier to recover from errors).

For each page:
1. Call `list_executions(project="X", limit=500, offset=N)`
2. If it **succeeds** → immediately copy the result file to a persistent `/tmp/[project_name]_batches/offset_N.json`
3. If it **fails with Pydantic validation error** (`failed-with-retry` status) → log the offset as skipped, continue to next page — do **not** retry, it will always fail
4. If it **fails with 504** → the page size is too large; reduce limit and retry

**Copy to disk immediately after each successful call** — tool result files are ephemeral and may be cleaned up. Use the exact file path returned in the tool response.

```bash
cp '/path/from/tool/response/content.json' /tmp/[project_name]_batches/offset_500.json
```

---

## Step 4 — Load and Deduplicate

After all pages are fetched, load with Python, deduplicating on execution `id`:

```python
import json, glob

seen_ids = set()
all_execs = []
for fpath in sorted(glob.glob("/tmp/[project_name]_batches/offset_*.json")):
    data = json.load(open(fpath))
    for ex in data.get("response", []):
        if ex["id"] not in seen_ids:
            seen_ids.add(ex["id"])
            all_execs.append(ex)
```

Deduplication is needed because probe/exploration calls may overlap with your main pages.

---

## Step 5 — Handle Skipped Pages

Some pages will fail permanently due to executions with status `failed-with-retry` (not in Pydantic model). These pages are **unrecoverable** via the MCP tool.

In your report, account for these gaps:
- Track which offsets were skipped
- Estimate missing records: `skipped_pages × page_size`
- Note this in a "Data Quality" section of the report

---

## Step 6 — Per-Job Analysis

Once you have all executions, group by `ex["job"]["id"]` to compute per-job stats. Jobs list is fetched once via `list_jobs(project="X")`.

Key fields per execution:
- `ex["job"]["id"]` — links to job (may be `null` for adhoc executions)
- `ex["date-started"]` — ISO string, parse with `datetime.fromisoformat(val.replace("Z", "+00:00"))`
- `ex["status"]` — `succeeded`, `failed`, `aborted`, `timedout`, `running`, `scheduled`
- `ex["user"]` — who triggered it

---

## Step 7 — Write the Analysis Script

**Never try to run scripts that need env vars from the MCP config** — the `RUNDECK_API_TOKEN` etc. are only injected into the MCP server process, not available to shell scripts. All data collection must happen through MCP tool calls.

Write a self-contained Python script that:
1. Reads batch JSON files from `/tmp/[project_name]_batches/`
2. Reads a jobs list saved to `/tmp/jobs.json`
3. Produces the report and writes it to disk

Run it with: `python3 analyze.py > /tmp/out.txt 2>&1` (redirect output — the terminal uses alternate buffer mode and swallows interactive output).

---

## Step 8 — Activity Status Definition

Use this standard definition for job activity in reports:
- **Active**: last execution within the past 30 days
- **Stale**: has executions, but none in the past 30 days
- **Dormant**: no executions in the last 90 days
- **Never Run**: zero recorded executions

---

## Common Pitfalls

| Problem | Cause | Fix |
|---|---|---|
| 504 Gateway Timeout | `limit > ~600` | Use `limit=500` |
| Pydantic validation error on status | `failed-with-retry` status in page | Skip page, log offset, continue |
| Naive/aware datetime comparison error | `datetime.fromisoformat` returns naive dt | Always `.replace(tzinfo=timezone.utc)` if `tzinfo is None` |
| Tool result files disappear | They are ephemeral chat resources | Copy to `/tmp/` immediately after each call |
| Script can't read API credentials | Env vars only exist in MCP server process | Use MCP tools for all API calls; use Python only for analysis |
| Terminal output invisible | Alternate buffer mode | Always redirect: `python3 script.py > /tmp/out.txt 2>&1` |

---

## Efficiency Tips

- **Probe first**: One call at a high offset tells you if there are more records before committing to many pages
- **Binary search for total**: Double the offset until empty, then bisect to find exact boundary
- **Save jobs list early**: One `list_jobs` call at the start; save to `/tmp/jobs.json` for reuse
- **Per-job fetch is slower for high-volume projects**: Global pagination (all executions) is faster than 81 × per-job calls when jobs have hundreds of executions each
- **Per-job fetch is better for sparse projects**: If most jobs have < 500 executions, per-job calls are more targeted and avoid downloading unrelated data

---

## Template: Batch Copy Command

After each tool call, use this pattern to save the result:

```bash
cp '<full-path-from-tool-response>' /tmp/[project_name]_batches/offset_<N>.json
```

Check count and date range to verify:
```bash
python3 -c "import json; e=json.load(open('/tmp/[project_name]_batches/offset_500.json'))['response']; print(len(e), e[0]['id'], e[-1]['date-started'][:10])"
```

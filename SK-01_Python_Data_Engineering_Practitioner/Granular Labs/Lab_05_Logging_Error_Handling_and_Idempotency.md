# Lab 05 — Logging, Error Handling and Idempotency

> **Goal:** Make the pipeline's behaviour deliberate when things go wrong. You'll read its logs like an operator would, watch it fail fast on a missing file, test the CRM API code against a fake server (including a 503 halfway through), keep the API key out of the code, add the optional AI step safely, and prove that running the pipeline twice gives identical results.
>
> **Time:** 3–4 hours | **Difficulty:** Intermediate | **Prerequisites:** [Lab 04](Lab_04_Building_a_Modular_ETL_Pipeline.md) completed: `python pipeline.py` runs end to end (745 records, 7/7 checks).
>
> **Guided lab coverage:** [SK-01 End-to-End Guided Lab](../Labs/SK-01_End_to_End_Guided_Lab.md) **Step 1.1** (logging and secrets in `CONFIG`), **Step 2.2** (`ingest_crm_api`), **Part 6** (AI region inference bonus) and **Reflection Questions 4–5** (incremental loads, CRM 503 recovery).

---

## The business problem

The pipeline will run every night at 2 a.m., with nobody watching. When the marketing team asks "is today's file right?", someone has to answer from the evidence the run left behind. Three things decide whether that's easy or a crisis:

1. **Logging:** can you tell what happened, when and how many records were involved, without re-running anything?
2. **Error handling:** when something breaks, does the run stop loudly, or carry on quietly and deliver wrong data? Wrong data is worse than no data. A GDPR breach starts with a file that *looked* fine.
3. **Idempotency:** if the run is repeated (a retry, a scheduler hiccup, an engineer re-running it "to be sure"), is the result the same, or are there now twice as many customers?

---

## Learning objectives

By the end of this lab you can:

1. Explain log levels, handlers and formats, and read `logs\pipeline.log` after a run.
2. Decide when to catch an error and continue, and when to let it stop the run.
3. Test API-calling code without a network by replacing `requests.get` with a fake (`unittest.mock.patch`).
4. Design a retry strategy for temporary failures like HTTP 503.
5. Keep secrets in environment variables, never in code.
6. Define idempotency, prove it with file hashes, and explain what an incremental load would change.

---

## Key terms

- **Log level:** how serious a message is: `DEBUG` < `INFO` < `WARNING` < `ERROR` < `CRITICAL`. A logger set to `INFO` drops `DEBUG` messages.
- **Handler:** where log messages go. The pipeline has two: the screen (`StreamHandler`) and `logs\pipeline.log` (`FileHandler`).
- **Fail fast:** stop at the first sign of a broken precondition, before producing partial output.
- **Exit code:** the number a program returns to whoever started it. `0` means success, anything else means failure. Schedulers rely on it.
- **Mock:** a stand-in object that behaves like the real thing (here, an HTTP response) so code can be tested offline.
- **Idempotent:** running it once or many times gives the same result.

---

## Steps

### Step 1 — How the pipeline's logging works

**What:** Look at the logging setup at the top of `pipeline.py` (guided lab Step 1.1):

```python
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("logs/pipeline.log", encoding="utf-8"),  # utf-8: the report uses ✓/✗
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)
```

| Part | Why |
|---|---|
| `level=logging.INFO` | Show normal progress and anything worse. Hide `DEBUG` detail. |
| `format=...` | Every line gets a timestamp and a level, so you can find `ERROR` lines and see how long each stage took. `print` gives neither. |
| `FileHandler(..., encoding="utf-8")` | Keeps a permanent record after the terminal is closed. UTF-8 so ✓/✗ and `Müller` don't crash or get garbled. |
| `StreamHandler()` | Also prints to the screen while you watch. |
| `getLogger(__name__)` | A named logger. In bigger projects each module gets its own, so you can see which file a message came from. |

See the levels in action. Create `practice\lab05_logging_levels.py`:

```python
"""Lab 05 - see which log levels get through an INFO-level configuration."""
import logging

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
log = logging.getLogger("lab05")

log.debug("Row 17: phone '555.123.4567' -> '5551234567'")   # too detailed for INFO: hidden
log.info("Ingested 1000 records from website CSV")
log.warning("219 records have no region after standardization")
log.error("CRM API returned 503 on page 15 of 20")
```

```powershell
python practice\lab05_logging_levels.py
```

**Expected output:**

```
<timestamp> [INFO] Ingested 1000 records from website CSV
<timestamp> [WARNING] 219 records have no region after standardization
<timestamp> [ERROR] CRM API returned 503 on page 15 of 20
```

The `DEBUG` line is hidden. Change `level=logging.INFO` to `logging.DEBUG`, run again and it appears. That's how you switch on detail when investigating, without editing every message.

Now read the real log as an operator would:

```powershell
Get-Content logs\pipeline.log -Tail 12
Select-String -Path logs\pipeline.log -Pattern "PIPELINE COMPLETE|Output \(golden\)"
```

The log **keeps growing**: every run adds to the end. Each `PIPELINE COMPLETE` line marks a finished run, so the second command lists every run's result.

> **Choosing a level:** the guided lab logs `Null regions after standardization: 219` as `INFO`. You could argue it should be `WARNING`: it's expected, but someone should look at it if it grows. A good rule: `INFO` = normal progress, `WARNING` = handled but worth a look, `ERROR` = a step failed.

✅ **Checkpoint:** you can find the `Output (golden) records` line of every run in `logs\pipeline.log`.

### Step 2 — Fail fast: a missing source file

**What:** Simulate the ERP export not arriving:

```powershell
Rename-Item data\raw\erp_customers.txt erp_customers.bak
python pipeline.py
$LASTEXITCODE
```

**Expected output** (end of the output):

```
<timestamp> [INFO] Ingesting ERP fixed-width: data\raw\erp_customers.txt
Traceback (most recent call last):
  ...
FileNotFoundError: [Errno 2] No such file or directory: 'data\\raw\\erp_customers.txt'
1
```

Then put the file back:

```powershell
Rename-Item data\raw\erp_customers.bak erp_customers.txt
```

**Why this is the right behaviour:**
- The run **stopped**. It didn't produce a golden file with 250 ERP customers silently missing. Those customers would have dropped out of the campaign, and nobody would know why.
- `$LASTEXITCODE` is **1**, so a scheduler (Windows Task Scheduler, Airflow in SK-03) marks the run as failed and can alert someone.
- The log shows the last successful step (`Ingesting ERP fixed-width: ...`), which tells you where to look.

**But there's a trap.** Look in `data\processed\`: yesterday's `golden_customers.csv` is still there. A consumer can't tell that today's run failed. Professional fixes: write to a temporary file and rename it only after success, write a `_SUCCESS` marker file, or check that inputs exist at the start before any work. These are the patterns SK-03's orchestration builds on.

> **Catch or stop?** Catch an error only when you know a correct way to continue. A missing source file has no correct way to continue, so let it stop the run. A failed *optional* enrichment (Step 5) can safely continue. Never write `except Exception: pass`. It turns a loud failure into wrong data.

✅ **Checkpoint:** you can explain why exit code 1 is a *good* outcome here.

### Step 3 — Test the CRM API code without a CRM

**What:** In production, the CRM is a paginated REST API, and `ingest_crm_api` (guided lab Step 2.2, already in your `pipeline.py`) would replace `ingest_crm_json`. There's no real API in this course, but you can still test the code: replace `requests.get` with a fake that returns pages you design.

How `ingest_crm_api` works:
- Requests `page=1, 2, 3, ...` with `per_page=500` and an `Authorization: Bearer <key>` header.
- `timeout=30`: never wait forever. A request with no timeout can hang a 2 a.m. run until someone notices in the morning.
- `response.raise_for_status()` turns an HTTP error (401, 404, 503...) into a Python exception instead of trying to read an error page as data.
- Stops when a page is empty or `page > total_pages`.
- Renames `profile_first_name` → `first_name` (like `ingest_crm_json`), so `align_schema` keeps the names.

Create `practice\lab05_crm_api_offline.py`:

```python
"""Lab 05 - exercise ingest_crm_api() without a real CRM, by faking requests.get."""
from unittest.mock import MagicMock, patch

import requests
from pipeline import ingest_crm_api


def fake_page(customers, total_pages):
    """Build an object that behaves like a successful requests.Response."""
    response = MagicMock()
    response.raise_for_status.return_value = None          # 2xx: do nothing
    response.json.return_value = {"customers": customers, "total_pages": total_pages}
    return response


page1 = [{"id": "CRM-1", "email": "a@x.com", "profile": {"first_name": "Ana"}}]
page2 = [{"id": "CRM-2", "email": "b@x.com", "profile": {"first_name": "Ben"}}]

# 1) Happy path: two pages, then stop
with patch("pipeline.requests.get", side_effect=[fake_page(page1, 2), fake_page(page2, 2)]) as get:
    df = ingest_crm_api("https://crm.example/v2/customers", api_key="test-key")
print(df)
print("calls made:", get.call_count)
print("page params:", [c.kwargs["params"]["page"] for c in get.call_args_list])

# 2) The server fails on page 2 with a 503
broken = MagicMock()
broken.raise_for_status.side_effect = requests.HTTPError("503 Server Error: Service Unavailable")
with patch("pipeline.requests.get", side_effect=[fake_page(page1, 2), broken]):
    try:
        ingest_crm_api("https://crm.example/v2/customers", api_key="test-key")
    except requests.HTTPError as e:
        print("Pipeline stopped:", e)
```

```powershell
python -m practice.lab05_crm_api_offline
```

**Expected output:**

```
<timestamp> [INFO] Ingesting CRM API (paginated)...
<timestamp> [INFO]   Fetched page 1 (1 records)
<timestamp> [INFO]   Fetched page 2 (1 records)
<timestamp> [INFO]   Ingested 2 records from CRM API
      id    email first_name source
0  CRM-1  a@x.com        Ana    crm
1  CRM-2  b@x.com        Ben    crm
calls made: 2
page params: [1, 2]
<timestamp> [INFO] Ingesting CRM API (paginated)...
<timestamp> [INFO]   Fetched page 1 (1 records)
Pipeline stopped: 503 Server Error: Service Unavailable
```

**How the fake works:** `patch("pipeline.requests.get", side_effect=[...])` replaces `requests.get` *as `pipeline` sees it* for the duration of the `with` block. Each call returns the next item in the list. A `MagicMock` accepts any attribute or method call, and you set only what the code uses (`raise_for_status`, `json`). The test proves pagination stops after `total_pages`, and that a 503 **stops the ingestion** instead of returning page 1 as if it were complete. Lab 06 turns this into an automated test.

✅ **Checkpoint:** both scenarios behave as shown, and no real network call was made.

### Step 4 — Design the recovery: retries with backoff (Reflection Q5)

The guided lab asks: *"If the CRM API returns a 503 error on page 15 of 20 pages, what should the pipeline do?"* A 503 ("Service Unavailable") is usually **temporary**. Stopping at once is safe, but trying again a few times first is better. Create `practice\lab05_retry.py` to try the pattern (it stays a practice file; the guided lab's pipeline doesn't include it):

```python
"""Lab 05 - retry a flaky request a few times, then give up loudly."""
import logging
import time
from unittest.mock import MagicMock, patch

import requests

logging.basicConfig(level=logging.INFO, format="[%(levelname)s] %(message)s")
logger = logging.getLogger("lab05")


def get_with_retry(url, params, headers, attempts=3, wait_seconds=1):
    """GET a URL; retry server errors (5xx) and timeouts, but not client errors (4xx)."""
    for attempt in range(1, attempts + 1):
        try:
            response = requests.get(url, params=params, headers=headers, timeout=30)
            response.raise_for_status()
            return response
        except (requests.Timeout, requests.ConnectionError, requests.HTTPError) as e:
            status = getattr(e.response, "status_code", None)
            if status is not None and status < 500:
                raise                              # 401/404: retrying will not help
            if attempt == attempts:
                logger.error(f"Giving up after {attempts} attempts: {e}")
                raise
            logger.warning(f"Attempt {attempt} failed ({e}); retrying in {wait_seconds}s")
            time.sleep(wait_seconds)
            wait_seconds *= 2                      # exponential backoff: 1s, 2s, 4s...


def response_with_status(code):
    response = MagicMock(status_code=code)
    if code >= 400:
        error = requests.HTTPError(f"{code} Error")
        error.response = response
        response.raise_for_status.side_effect = error
    return response


# The fake server fails twice with 503, then succeeds
with patch("requests.get", side_effect=[response_with_status(503), response_with_status(503),
                                        response_with_status(200)]):
    r = get_with_retry("https://crm.example/v2/customers", {"page": 15}, {})
    print("Got status", r.status_code)

# A 401 (bad API key) is not retried
with patch("requests.get", side_effect=[response_with_status(401)]):
    try:
        get_with_retry("https://crm.example/v2/customers", {"page": 1}, {})
    except requests.HTTPError as e:
        print("Not retried:", e)
```

```powershell
python practice\lab05_retry.py
```

**Expected output** (takes about 3 seconds because of the waits):

```
[WARNING] Attempt 1 failed (503 Error); retrying in 1s
[WARNING] Attempt 2 failed (503 Error); retrying in 2s
Got status 200
Not retried: 401 Error
```

**A complete answer to Reflection Q5:**
1. **Retry temporary errors** (5xx, timeouts) a few times with **exponential backoff**, so you don't hammer a struggling server. **Never retry** 4xx errors: a wrong key or URL won't fix itself.
2. If retries run out, **fail the whole run** (non-zero exit code, `ERROR` log line). Never write a golden file built from 14 of 20 pages.
3. Because the run is a **full reload** (Step 6), re-running from page 1 later is safe. For very large exports you could save each page to `data\raw\crm\page_015.json` and resume from the last saved page, which is itself a small idempotency design.

✅ **Checkpoint:** you can explain why 503 is retried but 401 is not.

### Step 5 — Secrets and the optional AI step

**Secrets.** `CONFIG["crm_api_key"]` reads `os.environ.get("CRM_API_KEY", "")`. The key lives in the **environment**, not in `pipeline.py`, so it never ends up in Git, screenshots or code reviews. Try it (the check prints only whether a key is set, never the key itself):

```powershell
python -c "import pipeline; print(bool(pipeline.CONFIG['crm_api_key']))"
$env:CRM_API_KEY = "test-key-123"
python -c "import pipeline; print(bool(pipeline.CONFIG['crm_api_key']))"
Remove-Item Env:CRM_API_KEY
```

**Expected output:**

```
False
True
```

`$env:NAME = "..."` sets a variable for the current PowerShell window only. In production, a scheduler or a secrets manager (AWS Secrets Manager in SK-04) provides it. Never `logger.info` a secret: logs are copied to many places.

**The AI bonus (guided lab Part 6).** Paste guided lab **Step 6.1** (`infer_region_with_llm`) into `pipeline.py` **above** `generate_eda_report`. Leave the call in `run_pipeline` commented out unless you have an Anthropic API key. Read it for its error-handling design:
- `import anthropic` is **inside** the function. The package is optional, so the rest of the pipeline still runs without it installed. If you uncomment the call without installing it, the run stops with `ModuleNotFoundError: No module named 'anthropic'`, the right loud failure for a step you explicitly turned on.
- Each batch of 10 records is wrapped in `try/except Exception` with `logger.warning(...)`. One failed batch (network blip, unexpected reply) is logged and skipped; the others still get regions. This is a *deliberate* catch: the step is an optional improvement, and records it can't fill simply stay `NaN`, which the quality report already counts.
- Only answers in `CONFIG["valid_regions"]` are accepted, and a `region_inferred` flag marks every guessed value. Never mix guesses with facts without a label.
- The key comes from the `ANTHROPIC_API_KEY` environment variable, the same rule as above.

To try it (optional): `python -m pip install anthropic`, set `$env:ANTHROPIC_API_KEY`, uncomment `deduped = infer_region_with_llm(deduped)` in `run_pipeline`, and run. The `NOT NULL: region` check should improve. Then think about guided lab Reflection Q3: you just sent customer emails to an external service.

✅ **Checkpoint:** `python pipeline.py` still runs without `anthropic` installed.

### Step 6 — Prove idempotency

**What:** Run the pipeline twice and compare the output files byte by byte:

```powershell
python pipeline.py
Get-FileHash data\processed\golden_customers.csv, data\processed\golden_customers.parquet, data\processed\quality_report.csv | Select-Object Hash, Path
python pipeline.py
Get-FileHash data\processed\golden_customers.csv, data\processed\golden_customers.parquet, data\processed\quality_report.csv | Select-Object Hash, Path
```

**Expected:** the 3 hashes are **identical** between the two runs. (A hash is a fingerprint of a file's bytes. One changed byte gives a completely different hash.) Only `logs\pipeline.log` changes, because each run adds its own entries.

**Why it's idempotent:** the pipeline is a **full reload**. Every run reads all of `data\raw` from scratch and **overwrites** each output (`to_parquet`/`to_csv` replace the file). Nothing depends on the previous run's output.

**How it would break:** imagine a "small optimisation", `deduped.to_csv(csv_path, mode="a")` (append). Run it twice and the golden file holds every customer twice (1,490 rows, plus a second header line in the middle), so every customer gets emailed twice. Appending, `+=` counters stored in files and "insert without checking" are the usual ways idempotency is lost.

**Reflection Q4, incremental loads.** At 500K customers, a nightly full reload might become too slow, so you'd process only new or changed records. That means you need: a **watermark** (e.g. the latest `registration_date` or `updated_at` processed, saved after each successful run); **upserts** (update the customer if the email already exists, insert if not) instead of overwrites; dedup that compares new records against the *existing* golden records, not just against each other; and an opt-out rule that can switch an existing customer to opted out. Each of these adds ways to break idempotency, which is why full reloads are the right default until size forces a change.

✅ **Checkpoint:** identical hashes on two runs, and you can name 2 changes that would break idempotency.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No `logs\pipeline.log`, or it's empty | Run from another folder (relative `logs/` path) | Always run from `C:\DataEngineering\sk01` |
| `UnicodeEncodeError` writing ✓/✗ to the log | `FileHandler` without `encoding="utf-8"` | Re-copy guided lab Step 1.1 |
| ✓ shows as `\u2713` or `?` in the terminal | The console's code page can't display it | Cosmetic only; the log file is correct. Windows Terminal displays it fine. |
| `AttributeError: module 'pipeline' has no attribute 'requests'` in Step 3 | `import requests` missing from `pipeline.py` | It must be in the Step 1.1 imports |
| The mock test makes a real network call or hangs | Patched `requests.get` instead of `pipeline.requests.get` | Patch the name **where it's used**: `"pipeline.requests.get"` |
| Hashes differ between runs | Something non-deterministic: data regenerated, `seed` removed, a timestamp written into the output | Compare the two CSVs (`Compare-Object (Get-Content a) (Get-Content b)`) to find the difference |
| `$LASTEXITCODE` is 0 after a failure | The exception was caught and hidden | Remove the broad `except`; let unrecoverable errors propagate |

---

## Key takeaways

- Log with levels and timestamps to the screen and a UTF-8 file. The log is the only witness of a 2 a.m. run.
- Stop the run when there's no correct way to continue (missing source, failed API). Catch errors only for optional steps, and always log what was skipped.
- A failed run must exit non-zero and must not leave output that looks fresh. Watch out for stale files from earlier runs.
- Test network code offline with mocks. Retry 5xx errors and timeouts with backoff; never retry 4xx errors.
- Secrets come from environment variables and never appear in code or logs.
- Full reload + overwrite = idempotent. Append, counters and blind inserts break it.

---

## Next

[Lab 06 — Testing and Packaging a Production Pipeline](Lab_06_Testing_and_Packaging_a_Production_Pipeline.md): turn today's manual checks (the GDPR opt-out rule, the dedup counts, the API mock) into automated tests that run in seconds, then package the project for handover.

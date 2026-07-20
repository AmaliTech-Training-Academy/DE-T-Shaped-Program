# Lab 05 — Logging, Error Handling, Configuration, and Idempotency

> **Module:** SK-01 Python Data Engineering Practitioner
> **Estimated time:** 5–7 hours
> **Prerequisites:** Lab 04 — your modular pipeline runs end to end.

---

## 1. Environment Setup

One new package, plus a config folder:

```powershell
cd $HOME\Downloads\Upskilling\SK-01_Python_Data_Engineering_Practitioner
.\.venv\Scripts\Activate.ps1
pip install pyyaml python-dotenv
mkdir pipeline\config, pipeline\logs -Force
```

| Package | Purpose |
|---|---|
| `pyyaml` | Reads YAML configuration files (human-friendly settings format). |
| `python-dotenv` | Loads secrets from a `.env` file into environment variables. |

**Verify:**

```powershell
python -c "import yaml, dotenv; print('config tooling OK')"
```

**Common problem:** `pip install yaml` fails — the package name is `pyyaml` even though you `import yaml`. A classic.

---

## 2. Business Context

Your Lab 04 pipeline works — *when everything goes right, run by you, on your machine, once.* Production is none of those things:

- It runs **unattended** at 2 a.m. When it breaks, nobody is watching. The only witness is whatever it wrote down. Right now, it writes `print` statements that vanish when the terminal closes.
- **Things fail routinely**: source systems export late, files arrive corrupt, disks fill. A production pipeline's quality is measured less by its happy path than by how it behaves when things go wrong.
- It will run on **multiple environments** — your laptop, a teammate's, a server — with different paths and credentials. Hardcoded settings mean editing code per machine, which means bugs.
- Failed runs get **re-run**. If re-running duplicates data — every employee appearing twice in the golden dataset — the "fix" causes a worse incident than the failure.

**Who cares about this?** Everyone who depends on the pipeline. Operations teams triage from logs. Clients ask "what's your retry strategy?" in vendor reviews. Auditors ask for evidence of controls. These four topics — logging, error handling, configuration, idempotency — are what separates a portfolio project from a production system, and they are exactly what senior engineers probe for in interviews.

---

## 3. Concept Explanation

### 3.1 Logging (vs printing)

**What:** A log is a permanent, timestamped, leveled record of what a program did. Python's built-in `logging` module writes to files and console simultaneously, stamps every line with time and severity, and can be tuned without code changes.

**Why print isn't enough:** prints have no timestamps (when did it fail?), no levels (is this line routine or an emergency?), no destinations (terminal closes → evidence gone), and no way to silence debug noise in production.

**Levels** — the shared vocabulary of the industry:

| Level | Meaning | Example |
|---|---|---|
| `DEBUG` | Diagnostic detail, normally off in prod | "Parsed 14 date formats in column X" |
| `INFO` | Normal milestones | "Extract complete: 15,000 rows" |
| `WARNING` | Odd but not fatal; a human should skim these | "312 salaries could not be converted" |
| `ERROR` | An operation failed | "Could not read payroll_data.xlsx" |
| `CRITICAL` | The pipeline cannot continue | "Quality gate failed — halting" |

### 3.2 Error-handling strategy (not just try/except)

`try/except` is syntax; **strategy** is deciding, per failure type, among:

- **Fail the run** — data would be wrong otherwise (quality gate breach, missing critical source).
- **Retry** — the cause is transient (network blip, file locked by antivirus). Retries must be **bounded** (e.g., 3 attempts) with **backoff** (wait longer each try), or you hammer a struggling system.
- **Degrade gracefully** — continue without a non-critical piece, loudly (e.g., benefits feed missing → produce golden dataset anyway, log WARNING, mark the output as partial).
- **Quarantine** — bad *records* (not whole runs) go to the rejected zone with a reason; the rest proceed.

**The cardinal sin:** `except Exception: pass` — swallowing errors silently. Every failure must leave evidence.

### 3.3 Configuration and secrets

**Config** is every value that varies by environment or evolves without logic changes: paths, thresholds, exchange rates, feature toggles. It belongs in a file (we'll use YAML), not in code. The rule of thumb: *code defines behavior; config defines context.*

**Secrets** (passwords, API keys) are config too dangerous for a config file that gets committed to version control. They live in **environment variables**, loaded locally from a `.env` file that is **never committed** (`.gitignore` it). In the cloud, dedicated secret stores take over (AWS Secrets Manager — SK-04), but the code pattern — *read from the environment, never hardcode* — is identical.

A real-world cautionary tale: thousands of AWS keys are scraped from public GitHub repos every year by bots within *minutes* of being pushed; crypto-mining bills in the tens of thousands of dollars follow. `.env` + `.gitignore` is the habit that prevents it.

### 3.4 Idempotency

**Definition:** an operation is **idempotent** if running it twice produces the same result as running it once. `x = 5` is idempotent; `x = x + 5` is not. Appending to a file is not; overwriting a file deterministically is.

**Why it's the most important word in this lab:** re-runs are a fact of life — failures, backfills, "just to be sure" reruns by operators. If your pipeline *appends* to outputs, every re-run doubles the data. Design targets:

- **Overwrite, don't append**: write outputs to deterministic paths and replace them wholesale.
- **Write atomically**: write to a temp name, then rename over the target — so a crash mid-write never leaves a half-file that looks complete.
- **Partition by run scope**: in date-partitioned systems, "re-run for 2026-07-01" rewrites exactly that partition (you'll live this in SK-03/04).

Interviewers ask about idempotency *constantly*. After this lab you'll have implemented it, not just defined it.

---

## 4. Step-by-Step Implementation

### Step 1 — Add real logging

**What:** Create `pipeline/hr_pipeline/logging_setup.py`:

```python
"""Central logging configuration — call once from the entry point."""
import logging
from datetime import datetime
from pathlib import Path


def setup_logging(log_dir: Path, level: str = "INFO") -> logging.Logger:
    log_dir.mkdir(parents=True, exist_ok=True)
    log_file = log_dir / f"pipeline_{datetime.now():%Y%m%d_%H%M%S}.log"

    fmt = logging.Formatter(
        "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s"
    )

    root = logging.getLogger()
    root.setLevel(level)

    file_handler = logging.FileHandler(log_file, encoding="utf-8")
    file_handler.setFormatter(fmt)
    root.addHandler(file_handler)

    console = logging.StreamHandler()
    console.setFormatter(fmt)
    root.addHandler(console)

    logging.getLogger(__name__).info("Logging to %s", log_file)
    return root
```

Then, in **every** module, replace prints:

```python
import logging
logger = logging.getLogger(__name__)   # top of each module

# print(f"Exact dedup: removed {n} rows")        # before
logger.info("Exact dedup: removed %d rows", n)   # after
logger.warning("Salary conversion failed for %d of %d rows", bad, total)
```

And in `run_pipeline.py`, first thing in `main()`:

```python
from hr_pipeline.logging_setup import setup_logging
setup_logging(BASE_DIR / "logs")
```

**Why the specifics:**
- `logging.getLogger(__name__)` names each logger after its module, so every line says *which file* produced it — free fault localization.
- **Two handlers**: console for humans watching, file for the permanent record. Same events, two destinations, zero extra code at call sites.
- Timestamped log filenames keep one file per run — run history you can diff.

**Run and verify:** run the pipeline, then `Get-Content pipeline\logs\pipeline_*.log -TotalCount 10`. Every line should carry timestamp, level, module, message.

**Common mistakes:**
- Calling `setup_logging()` in more than one place → duplicated lines (handlers added twice). Configure once, at the entry point only.
- Leaving `print`s behind. Sweep them: `Select-String -Path pipeline\hr_pipeline\*.py -Pattern "print\("` should return nothing.

### Step 2 — Externalize configuration

**What:** Create `pipeline/config/pipeline.yaml`:

```yaml
paths:
  raw: data/raw
  processed: data/processed
  rejected: data/rejected
  output: output
  logs: logs

quality_gate:
  max_failed_checks: 2

dedup:
  fuzzy_threshold: 88
  hire_date_window_days: 30

exchange_rates_to_usd:
  USD: 1.00
  EUR: 1.08
  GBP: 1.27

salary_bounds:
  min: 15000
  max: 2000000
```

And `pipeline/hr_pipeline/config.py`:

```python
"""Typed access to pipeline configuration."""
from pathlib import Path
import yaml


def load_config(path: Path) -> dict:
    with open(path, encoding="utf-8") as f:
        cfg = yaml.safe_load(f)
    # Fail fast if a required section is missing
    for key in ("paths", "quality_gate", "dedup", "exchange_rates_to_usd"):
        if key not in cfg:
            raise KeyError(f"Config missing required section: '{key}'")
    return cfg
```

Now thread `cfg` through the orchestrator and pass values into the stage functions (e.g., `find_fuzzy_candidates(df, threshold=cfg["dedup"]["fuzzy_threshold"])`), deleting the constants that were hardcoded in modules.

**Why:**
- The fuzzy threshold, the gate tolerance, the exchange rates — these are *business decisions*, and they now change via a one-line config edit reviewed by the data owner, not a code change.
- `yaml.safe_load` (never plain `load`) prevents YAML's arbitrary-code-execution foot-gun.
- **Validating config at startup** turns "mysterious KeyError at stage 4" into "missing section 'dedup'" in the first second.

**Verify:** change `fuzzy_threshold` to `95`, re-run, and confirm fewer review candidates appear in the log — behavior changed with zero code edits. Change it back.

**Common mistake:** loading the config file in *every* module. Load once in the orchestrator; pass values (or the cfg dict) down as function parameters. Modules that secretly read global config are untestable.

### Step 3 — Secrets via environment variables

**What:** Imagine the exchange-rate feed requires an API key (in the capstone you'll simulate exactly this). Create `pipeline/.env`:

```
EXCHANGE_API_KEY=demo-key-12345
```

Create `pipeline/.gitignore` (yes, even though this folder isn't a repo yet — the habit is the point):

```
.env
logs/
data/processed/
output/
```

Read the secret in code:

```python
import os
from dotenv import load_dotenv

load_dotenv()  # reads .env into environment variables (no-op if absent)

api_key = os.environ.get("EXCHANGE_API_KEY")
if not api_key:
    raise RuntimeError(
        "EXCHANGE_API_KEY is not set. Copy .env.example to .env and fill it in."
    )
```

Also create `.env.example` with the variable name but a placeholder value — this *is* committed, documenting what secrets the pipeline needs without revealing them.

**Why:** secrets in code end up in version control, chat logs, and screenshots. Environment variables keep them out of all three, and the pattern scales directly to cloud secret managers later. The helpful error message ("copy .env.example...") is a professional courtesy that saves every future teammate an hour.

**Verify:** temporarily rename `.env`, run, and confirm you get the clear RuntimeError, not a cryptic `None` failure somewhere deep. Restore it.

### Step 4 — Bounded retries with backoff

**What:** Create `pipeline/hr_pipeline/retry.py`:

```python
"""Retry decorator for transient failures."""
import functools
import logging
import time

logger = logging.getLogger(__name__)


def retry(times: int = 3, delay_seconds: float = 2.0, backoff: float = 2.0,
          exceptions: tuple = (IOError, OSError)):
    """Retry a function on transient errors with exponential backoff.

    delay pattern with defaults: 2s, 4s, 8s — then give up and re-raise.
    """
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            wait = delay_seconds
            for attempt in range(1, times + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions as exc:
                    if attempt == times:
                        logger.error("%s failed after %d attempts: %s",
                                     fn.__name__, times, exc)
                        raise
                    logger.warning("%s attempt %d/%d failed (%s); retrying in %.0fs",
                                   fn.__name__, attempt, times, exc, wait)
                    time.sleep(wait)
                    wait *= backoff
        return wrapper
    return decorator
```

Apply it to the ingestion functions:

```python
from hr_pipeline.retry import retry

@retry(times=3)
def ingest_payroll_excel(path: Path) -> pd.DataFrame:
    ...
```

**Why the design decisions:**
- **Only transient exception types are retried.** Retrying a `ValueError` (bad data) three times gets you the same bad data three times, slower. Retry I/O; fail fast on logic.
- **Exponential backoff** (2s → 4s → 8s) gives a struggling system room to recover instead of hammering it — this is how every production HTTP client behaves.
- **The last failure re-raises.** Retries must never convert a failure into silence.

**Verify by simulating:** open the payroll file in Excel (which locks it on Windows), run the pipeline, and watch WARNING lines as attempts fail — then close Excel mid-retries and watch the run succeed. You have just rehearsed a real production incident.

### Step 5 — Make the pipeline idempotent

**What:** three changes.

**(a) Overwrite outputs deterministically.** Audit every write in `load.py`: each deliverable has a fixed name and `to_parquet`/`to_csv` replaces it. No `mode="a"`, no timestamped *output* names (timestamps belong on *logs*, which are events, not on *datasets*, which are states).

**(b) Write atomically.** Add to `load.py`:

```python
def _atomic_write_parquet(df: pd.DataFrame, target: Path) -> Path:
    """Write to a temp file, then rename — readers never see a half-written file."""
    tmp = target.with_suffix(".parquet.tmp")
    df.to_parquet(tmp, index=False)
    tmp.replace(target)          # atomic on the same filesystem
    return target
```

**(c) Prove it.** Run the pipeline twice in a row, then:

```powershell
python -c "import pandas as pd; df = pd.read_parquet('output/golden_employees.parquet'); print(len(df), df['employee_id'].duplicated().sum())"
```

**Expected output:** the same row count as a single run, and `0` duplicates. That measurement — *run twice, output identical* — is the idempotency test, and it belongs in your capstone evidence.

**Why atomic writes matter:** a crash (or Ctrl+C) halfway through a 500 MB write leaves a truncated file that *looks* valid and loads partially. Downstream reads it, reports go out with half the employees, nobody notices for a week. Temp-then-rename makes that impossible: the target either doesn't change or changes completely.

**Common mistakes:**
- "Idempotent" by deleting the whole output folder at start — works locally, but in shared storage you've just destroyed deliverables when the run then fails. Overwrite per-artifact, atomically, instead.
- Appending run metadata rows to a shared CSV "history" file inside the pipeline — that's non-idempotent state. Run history belongs in logs.

### Step 6 — Top-level error boundary

**What:** Wrap the orchestrator's `main()` so every failure is logged and classified:

```python
def main() -> int:
    logger = logging.getLogger("run_pipeline")
    try:
        run(cfg)          # the five stages, moved into run()
        logger.info("Pipeline succeeded")
        return 0
    except QualityGateError as exc:      # define this in validate.py
        logger.critical("Quality gate failed: %s", exc)
        return 2
    except FileNotFoundError as exc:
        logger.critical("Missing input: %s", exc)
        return 3
    except Exception:
        logger.critical("Unexpected failure", exc_info=True)
        return 1
```

**Why:** distinct exit codes per failure class let a scheduler (or a human reading the run history) distinguish "data quality problem — call the data owner" from "missing file — call the source system team" without opening logs. `exc_info=True` records the full stack trace *into the log file* — the difference between diagnosing in minutes and reproducing for hours.

**Verify:** trigger each failure (rename an input; poison one employment type) and confirm the right exit code via `echo $LASTEXITCODE` and a CRITICAL log line each time.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Leveled, file-persisted logging** | The only witness at 2 a.m. WARNING vs ERROR triages incidents. |
| **One logging setup at the entry point; `getLogger(__name__)` everywhere** | Every log line names its module; no duplicate handlers. |
| **Config in YAML; validated at startup** | Business knobs change without code changes; misconfiguration fails in second one. |
| **Secrets in env vars / `.env` + `.gitignore` + `.env.example`** | Keeps credentials out of code and version control; scales to cloud secret stores. |
| **Bounded retries with exponential backoff, transient errors only** | Rides out blips without hammering systems or masking real bugs. |
| **Idempotent, atomic writes** | Re-runs are safe; crashes can't leave plausible-looking half-files. |
| **Top-level error boundary with classified exit codes** | Failures are always logged, always classified, never silent. |

---

## 6. Reflection

### What you learned
You converted a working pipeline into an *operable* one: it explains itself (logs), survives transient failures (retries), adapts per environment (config, secrets), tolerates re-runs (idempotency), and fails with classified, logged, non-zero exits.

### Why it matters
These are the exact criteria engineering managers use to distinguish "wrote a script" from "built a system", and the exact topics that dominate mid-level data engineering interviews.

### Interview questions (with model answers)

1. **"What does idempotency mean and why does it matter for pipelines?"**
   Running an operation twice yields the same result as once. Pipelines get re-run — after failures, for backfills — so writes must overwrite deterministic targets (or rewrite partitions), never blindly append. I verify it by running twice and diffing outputs.

2. **"How do you handle transient vs permanent failures?"**
   Transient (network, file locks): bounded retries with exponential backoff, then fail loudly. Permanent (bad data, missing config): fail fast, no retries — retrying a logic error just delays the alert.

3. **"Where do you keep configuration and secrets?"**
   Config in versioned files (YAML), validated at startup, passed into functions. Secrets in environment variables — `.env` locally (gitignored, with a committed `.env.example`), a secrets manager in the cloud. Never in code.

4. **"What logging levels would you use in a pipeline and who reads them?"**
   INFO for milestones and row counts (operators skim these), WARNING for anomalies that didn't stop the run (reviewed daily), ERROR/CRITICAL for failures (trigger alerts). DEBUG stays off in production but is switchable via config.

5. **"What's an atomic write and when do you need one?"**
   Write to a temp path, then rename over the target, so readers see either the old complete file or the new complete file — never a torn intermediate. Needed whenever a consumer might read while you write, or a crash mid-write is possible (i.e., always).

6. **"Your nightly run failed at 2 a.m. Walk me through the morning."**
   Check exit code / alert classification, open the run's log file, find the last INFO banner to localize the stage, read the CRITICAL entry with stack trace, fix the cause, re-run — safely, because the pipeline is idempotent.

### Common interview traps
- Defining idempotency correctly but having *no example of implementing it* — this lab's run-twice test is your example.
- "I'd add try/except everywhere" — blanket catching is an anti-pattern; strategy per failure class is the answer.
- Forgetting that retries need *bounds and backoff*. Infinite retry is an outage generator.

### Key takeaways
1. Logs are for the person debugging at 2 a.m. — usually future you.
2. Code defines behavior; config defines context; env vars hold secrets.
3. Retry the transient, fail-fast the permanent, and always leave evidence.
4. Run it twice: if the output changes, it isn't done.

**Next:** [Lab 06](Lab_06_Testing_and_Packaging_a_Production_Pipeline.md) — proof. Automated tests that catch regressions before users do, and final packaging for handover.

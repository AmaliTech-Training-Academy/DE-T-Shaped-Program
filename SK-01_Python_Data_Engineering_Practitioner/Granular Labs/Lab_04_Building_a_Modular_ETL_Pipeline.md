# Lab 04 — Building a Modular ETL Pipeline

> **Module:** SK-01 Python Data Engineering Practitioner
> **Estimated time:** 5–7 hours
> **Prerequisites:** Labs 02 and 03 — you have working ingestion, cleaning, dedup, and validation code.

---

## 1. Environment Setup

No new libraries. What changes is the **project layout** — we graduate from lab scripts to a real pipeline package:

```powershell
cd $HOME\Downloads\Upskilling\SK-01_Python_Data_Engineering_Practitioner
.\.venv\Scripts\Activate.ps1
mkdir pipeline, pipeline\hr_pipeline, pipeline\data\raw, pipeline\data\processed, pipeline\data\rejected, pipeline\output -Force
copy "Project\Data\*" "pipeline\data\raw\"
```

Target structure:

```
pipeline/
├── hr_pipeline/            # the Python package (importable code)
│   ├── __init__.py
│   ├── ingest.py           # Extract
│   ├── clean.py            # Transform
│   ├── dedup.py            # Transform
│   ├── validate.py         # Quality gate
│   └── load.py             # Load
├── run_pipeline.py         # the entry point that orchestrates everything
├── data/
│   ├── raw/                # inputs — immutable
│   ├── processed/          # standardized intermediates
│   └── rejected/           # dead-letter records with reasons
└── output/                 # final deliverables
```

Create the package marker (an empty file that tells Python "this folder is a package"):

```powershell
New-Item pipeline\hr_pipeline\__init__.py -ItemType File
```

**Verify:**

```powershell
cd pipeline
python -c "import hr_pipeline; print('package OK')"
```

**Expected output:** `package OK`. If you get `ModuleNotFoundError`, you're not running from inside `pipeline\` — the folder containing the package must be your working directory (we make this robust later).

---

## 2. Business Context

**The problem:** Your Lab 02–03 code works, but it lives in scattered scripts you run by hand in the right order, editing paths as you go. Now imagine handing that to a colleague, or running it every night at 2 a.m. It would collapse immediately. Businesses don't buy scripts; they buy **pipelines** — repeatable, ordered, hands-off processes that turn raw inputs into trusted outputs.

**Where you'll see it in industry:** every scheduled data process is a pipeline. Nightly warehouse loads, hourly marketing syncs, monthly regulatory extracts. The orchestration tools change (cron, Airflow — SK-03, cloud schedulers — SK-04), but the *shape* you build today — modular stages, one entry point, clear data zones — is universal.

**Who consumes it?** Operations teams run it, fellow engineers maintain it, schedulers invoke it. Note the shift: in earlier labs your audience was *you*; from now on your audience is *other people and machines*.

**What happens if pipelines are built badly?** The "4,000-line script" is a real industry pathology: one file, no functions, paths hardcoded, only the original author (long since departed) understands it. Companies pay consultants large sums to untangle these. You're learning to never create one.

---

## 3. Concept Explanation

### 3.1 ETL: the shape of almost everything

**ETL = Extract, Transform, Load.** Extract data from sources, transform it into trustworthy shape, load it where consumers can use it. You already wrote all three — you just haven't named them:

| Stage | Your Lab code | This lab's module |
|---|---|---|
| **E**xtract | Lab 02 readers | `ingest.py` |
| **T**ransform | Lab 03 cleaning + dedup | `clean.py`, `dedup.py` |
| (Quality gate) | Lab 03 validator | `validate.py` |
| **L**oad | Parquet/CSV writes | `load.py` |

**ELT** is the modern variant where you load raw data first and transform inside the warehouse (you'll do this with dbt in SK-03). The concepts transfer completely.

### 3.2 Why modular? The case against the big script

A **module** is a file of related functions with a single responsibility. Splitting the pipeline into modules buys you:

- **Testability** — you can test `clean.parse_salary` without touching real files (Lab 06).
- **Parallel work** — two engineers can edit `ingest.py` and `dedup.py` without collisions.
- **Fault isolation** — when the run fails in `validate`, you know which file to open.
- **Reuse** — the next project imports your `DataQualityValidator` instead of rewriting it.

**The trade-off:** more files means more navigation, and over-abstracting tiny projects wastes time. The professional heuristic: *one module per pipeline stage* is the sweet spot for a pipeline this size.

### 3.3 Data zones: raw → processed → output (+ rejected)

Notice the `data/` sub-folders. This mirrors the **multi-zone / medallion** pattern used in real platforms (you'll meet it as Bronze/Silver/Gold in SK-03 and SK-07):

| Zone | Contents | Rule |
|---|---|---|
| `raw/` | Files exactly as received | **Immutable.** Never edited, never deleted by the pipeline. |
| `processed/` | Standardized, typed intermediates (Parquet) | Regenerable — safe to delete and rebuild from raw. |
| `rejected/` | Dead-letter records + reason codes | Evidence. Reviewed by humans. |
| `output/` | Final deliverables | The only zone consumers touch. |

**Why it exists:** when (not if) a bug is found, you rebuild everything from `raw/`. If your pipeline had modified raw files, there would be nothing to rebuild from.

### 3.4 The orchestrator

One entry point — `run_pipeline.py` — calls the stages in order and handles what happens between them. Later, schedulers (cron, Airflow) will call this one file. The rule: **modules do the work; the orchestrator does the ordering.** Business logic in the orchestrator is a smell.

---

## 4. Step-by-Step Implementation

### Step 1 — Move your code into the package

**What:** Copy the functions you wrote in Labs 02–03 into the package modules:

- `hr_pipeline/ingest.py` ← the four `ingest_*` functions from Lab 02's `ingest.py`
- `hr_pipeline/clean.py` ← `standardize_text`, `standardize_employment_type`, `parse_salary`, `to_annual_usd`, `parse_dates_multiformat`, `flag_implausible_dates`, `namespace_ids`
- `hr_pipeline/dedup.py` ← `dedup_exact`, `find_fuzzy_candidates`
- `hr_pipeline/validate.py` ← `DataQualityValidator`, `run_validation`

**Why:** This is deliberate practice at **refactoring** — restructuring working code without changing behavior. You'll do this constantly in real jobs. Key discipline: move code *unchanged* first, confirm it still works, and only then improve it. Moving and changing simultaneously makes failures undebuggable.

**One change you must make:** remove any module-level code that *runs* on import (top-level `print`s, file reads). Modules should define functions; only the orchestrator executes. Importing a module should never have side effects — otherwise `import hr_pipeline.ingest` in a test suddenly reads 15,000 rows.

**Verify:**

```powershell
python -c "from hr_pipeline import ingest, clean, dedup, validate; print('all modules import cleanly')"
```

### Step 2 — Write the load stage

**What:** Create `hr_pipeline/load.py`:

```python
"""Load stage: writes final deliverables to the output zone."""
from pathlib import Path
import pandas as pd


def write_golden_dataset(df: pd.DataFrame, output_dir: Path) -> Path:
    """The unified, cleaned, deduplicated employee dataset."""
    out = output_dir / "golden_employees.parquet"
    df.to_parquet(out, index=False)
    return out


def write_review_file(candidates: pd.DataFrame, output_dir: Path) -> Path:
    """Fuzzy-match pairs for HR to confirm or reject."""
    out = output_dir / "probable_matches_review.csv"
    candidates.to_csv(out, index=False)
    return out


def write_quality_report(report: pd.DataFrame, output_dir: Path) -> Path:
    out = output_dir / "quality_report.csv"
    report.to_csv(out, index=False)
    return out
```

**Why a whole module for three writes?** Because "where do outputs go and in what format" is a *contract with consumers*. When HR asks for Excel instead of CSV, or the golden dataset needs partitioning by company, you change one module. Loads also grow: in later modules of this program, "load" means databases and cloud storage — same seam, different destination.

### Step 3 — Build the orchestrator

**What:** Create `pipeline/run_pipeline.py`:

```python
"""HR Integration Pipeline — entry point.

Usage:  python run_pipeline.py
"""
from pathlib import Path
import sys
import pandas as pd

from hr_pipeline import ingest, clean, dedup, validate, load

BASE_DIR = Path(__file__).parent
RAW = BASE_DIR / "data" / "raw"
PROCESSED = BASE_DIR / "data" / "processed"
REJECTED = BASE_DIR / "data" / "rejected"
OUTPUT = BASE_DIR / "output"


def main() -> int:
    for d in (PROCESSED, REJECTED, OUTPUT):
        d.mkdir(parents=True, exist_ok=True)

    # ---- EXTRACT -------------------------------------------------
    print("[1/5] Extract")
    hris = ingest.ingest_hris_csv(RAW / "globaltech_hris.csv")
    acq = ingest.ingest_acquiredco_json(RAW / "acquiredco_api.json")
    payroll = ingest.ingest_payroll_excel(RAW / "payroll_data.xlsx")
    benefits = ingest.ingest_benefits_xml(RAW / "benefits_enrollment.xml")
    print(f"      hris={len(hris):,}  acquiredco={len(acq):,}  "
          f"payroll={len(payroll):,}  benefits={len(benefits):,}")

    # ---- TRANSFORM: standardize each source ----------------------
    print("[2/5] Clean & standardize")
    hris = clean.namespace_ids(hris, "GT")
    acq = clean.namespace_ids(acq.rename(columns={"employee_identifier": "employee_id"}), "AC")
    # ... apply your Lab 03 cleaning: text, employment types, dates, salaries
    # (exercise: wire in every cleaning function you built)

    # persist standardized intermediates
    hris.to_parquet(PROCESSED / "hris.parquet", index=False)
    acq.to_parquet(PROCESSED / "acquiredco.parquet", index=False)

    # ---- TRANSFORM: unify and dedup -------------------------------
    print("[3/5] Unify & deduplicate")
    employees = pd.concat([hris, acq], ignore_index=True)
    employees = dedup.dedup_exact(employees)
    review = dedup.find_fuzzy_candidates(employees)

    # ---- VALIDATE (quality gate) ----------------------------------
    print("[4/5] Validate")
    report = validate.run_validation(employees)   # raises if gate fails

    # ---- LOAD ------------------------------------------------------
    print("[5/5] Load")
    golden = load.write_golden_dataset(employees, OUTPUT)
    load.write_review_file(review, OUTPUT)
    load.write_quality_report(report, OUTPUT)
    print(f"Done. Golden dataset: {golden}  ({len(employees):,} employees)")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

**Why the details matter:**
- **Numbered stage banners** (`[1/5] Extract`) — when this runs unattended and dies, the last banner in the log tells you instantly where. You are designing for the person debugging at 2 a.m. (usually future you).
- **`sys.exit(main())`** — the pipeline returns an **exit code**: `0` means success, anything else means failure. Schedulers and shells use this to know whether to alert. `echo $LASTEXITCODE` in PowerShell shows it.
- **Row counts printed at every stage** — the single cheapest observability tool that exists. 15,000 in, 14,200 out of dedup: plausible. 15,000 in, 300 out: a join bug just announced itself.
- **The validation gate sits *before* load** — bad data physically cannot reach consumers.

**Run and verify:**

```powershell
cd pipeline
python run_pipeline.py
echo $LASTEXITCODE
```

**Expected output:** five stage banners with counts, a validation table, a final "Done" line, and exit code `0`. Check `output\` contains the three deliverables.

**Common mistakes:**
- Doing transformations inside `run_pipeline.py` "just this once" — the orchestrator grows into the 4,000-line script you swore never to write. Ordering only.
- Rebuilding `processed/` intermediates but reading stale ones later. Keep a strict downstream flow: each stage reads only from the previous stage's output.

**Troubleshooting:** `ModuleNotFoundError: hr_pipeline` → run from inside `pipeline\`. Column-name errors → your real data's column names differ from the examples; adapt (this is intended — engineers reconcile code to data, not the reverse).

### Step 4 — Handle a missing input like a professional

**What:** Add explicit precondition checks to the top of `main()`:

```python
    expected_files = [
        RAW / "globaltech_hris.csv",
        RAW / "acquiredco_api.json",
        RAW / "payroll_data.xlsx",
        RAW / "benefits_enrollment.xml",
    ]
    missing = [f.name for f in expected_files if not f.exists()]
    if missing:
        print(f"FATAL: missing input files: {missing}", file=sys.stderr)
        return 1
```

**Why:** Without this, a missing file crashes the run *midway* with a stack trace — after wasting minutes of processing and possibly leaving half-written outputs. Checking preconditions **up front** fails in one second with a clear message and a non-zero exit code. This pattern is called *fail fast at the boundary*.

**Verify by breaking it:** rename one raw file, run, confirm you get the FATAL message and `echo $LASTEXITCODE` prints `1`. Rename it back, confirm exit `0`. **Always rehearse your failure paths** — untested error handling is decoration.

### Step 5 — Write the README

**What:** Create `pipeline/README.md` documenting: purpose (2 sentences), inputs (files, formats, where they come from), outputs (each deliverable and who consumes it), how to run (exact commands), and known limitations.

**Why:** The README is part of the pipeline, not an optional extra. The test of done-ness: *a colleague who has never seen this project can run it successfully using only the README.* That's also precisely what the capstone project will demand.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Modular package, one module per stage** | Testable, maintainable, team-workable. The antidote to the 4,000-line script. |
| **Data zones (raw/processed/rejected/output), raw immutable** | Any bug is recoverable by rebuilding from raw. Mirrors medallion architecture you'll use in SK-03/SK-07. |
| **Single orchestrator entry point** | Machines (schedulers) and humans have exactly one way to run it. |
| **Exit codes** | The universal contract between your pipeline and every scheduler on earth. |
| **Stage banners + row counts** | Minimum viable observability; debugging goes from hours to seconds. |
| **Fail fast at the boundary (precondition checks)** | Clear errors in one second beat stack traces after ten minutes. |
| **Import-safe modules (no side effects on import)** | Prerequisite for testing (Lab 06) and reuse. |
| **README as part of the deliverable** | If only you can run it, it isn't done. |

---

## 6. Reflection

### What you learned
How to convert working scripts into a structured ETL pipeline: package layout, data zones, a load stage as a consumer contract, an orchestrator with stage banners, precondition checks, and exit codes.

### Why it matters
This shape — extract, transform, validate, load, orchestrated by one entry point over zoned storage — is the skeleton of virtually every batch data system you will ever touch. Airflow DAGs (SK-03), Glue jobs (SK-04), and Fabric pipelines (SK-07) are this same shape with bigger engines.

### Interview questions (with model answers)

1. **"Walk me through the structure of a pipeline you've built."**
   Describe this one: modular package with one module per stage, raw/processed/rejected/output zones, a validation gate before load, an orchestrator returning proper exit codes, README for operability. Naming *why* each part exists is what distinguishes you.

2. **"What's the difference between ETL and ELT?"**
   ETL transforms before loading to the destination; ELT loads raw data first and transforms inside the destination (warehouse compute, e.g., dbt). ELT dominates modern cloud stacks because warehouses scale transformation well and keeping raw data enables reprocessing.

3. **"Why should raw data be immutable?"**
   It's the audit trail and the only basis for reprocessing after bugs. Any zone downstream of raw is regenerable; raw is not.

4. **"How does a scheduler know your pipeline failed?"**
   Exit codes — 0 success, non-zero failure — plus logs and, in mature setups, metrics/alerts. If your process exits 0 on failure, monitoring is blind.

5. **"Where should data validation sit in a pipeline?"**
   After transformation and before load, as a gate. Optionally also at ingestion (schema checks on inputs). The principle: bad data must not be publishable.

6. **"Your pipeline processes 15,000 records in and outputs 300. What do you do?"**
   Investigate immediately — stage-by-stage row counts localize the drop (likely a bad join or over-aggressive filter). This is why we log counts at every stage.

### Common interview traps
- Describing a pipeline as "a script that does everything" — always narrate *stages*.
- Not knowing what an exit code is. It's one sentence; know it.
- Claiming validation happens "in the transform code" — the gate must be a distinct, evidence-producing step.

### Key takeaways
1. Modules do work; the orchestrator does ordering.
2. Zones: raw is immutable; everything else is regenerable or evidence.
3. Fail fast at the boundary; exit non-zero on failure.
4. Row counts at every stage are the cheapest observability you'll ever buy.

**Next:** [Lab 05](Lab_05_Logging_Error_Handling_and_Idempotency.md) — the pipeline runs, but it *prints* instead of logging, its settings are hardcoded, and re-running it can double-write. We fix all three.

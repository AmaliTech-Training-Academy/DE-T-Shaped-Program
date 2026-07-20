# Lab 03 — Data Cleaning and Validation: Making Data Trustworthy

> **Module:** SK-01 Python Data Engineering Practitioner
> **Estimated time:** 5–7 hours
> **Prerequisites:** Lab 02 completed — you have the four sources ingested and saved as Parquet in `labs/lab02/output/`.

---

## 1. Environment Setup

Activate your environment and add one new library:

```powershell
cd $HOME\Downloads\Upskilling\SK-01_Python_Data_Engineering_Practitioner
.\.venv\Scripts\Activate.ps1
pip install rapidfuzz
```

`rapidfuzz` computes **string similarity** — how alike two strings are on a 0–100 scale. We'll use it to catch duplicates like "Jon Smith" vs "John Smith" that exact matching misses.

Create the lab folder:

```powershell
mkdir labs\lab03\output -Force
```

**Verify setup:**

```powershell
python -c "from rapidfuzz import fuzz; print(fuzz.ratio('John Smith', 'Jon Smith'))"
```

**Expected output:** a number around `95` (very similar strings).

**Common problems:** if the import fails, you installed outside the venv — activate first, reinstall. On some corporate machines antivirus quarantines compiled wheels; `pip install --no-cache-dir rapidfuzz` usually resolves it.

---

## 2. Business Context

**The problem:** Raw data is dirty. Not occasionally — *always*. Industry studies consistently estimate knowledge workers lose 20–50% of their time to data quality issues, and IBM famously put the cost of bad data to the US economy at $3.1 trillion a year. Dirty data is not an edge case; it is the normal state of the world your pipelines live in.

**Concretely, in our GlobalTech/AcquiredCo merger scenario:**
- Payroll exports salaries as `"$85,000"` strings in three currencies. Sum them naively and your compensation report is fiction.
- The two companies' employee IDs overlap — employee #1042 exists in *both*. Merge naively and two different humans become one record.
- Dates arrive in three formats. Sort them as strings and "02/01/2025" lands before "13/12/2024".
- Some payroll rows have no matching employee — possibly terminated staff still being paid ("ghost employees"), which is a fraud and compliance issue.

**Who consumes cleaned data?** HR leadership planning the merger, finance running payroll migration, compliance filing regulatory headcounts. **If cleaning fails:** people get paid wrongly, regulators get wrong filings, and the merger integration decisions are made on false numbers.

**Where you'll see this in industry:** every single pipeline. "Data cleaning" is not a junior chore before the real work — deciding *what counts as clean, what to fix automatically, what to reject, and what to flag for humans* is core data engineering judgment.

---

## 3. Concept Explanation

### 3.1 The data quality dimensions

Data quality is usually assessed on six dimensions. Learn these terms — they appear in interviews and client conversations constantly:

| Dimension | Question it answers | Example failure |
|---|---|---|
| **Completeness** | Are required values present? | 400 employees with no email |
| **Uniqueness** | Is each real-world thing represented once? | Same employee twice in payroll |
| **Validity** | Do values match the expected format/domain? | `employment_type = "FT?"` |
| **Consistency** | Does the same fact agree across sources? | HRIS says "Engineering", payroll says "ENG-01" |
| **Accuracy** | Does the data reflect reality? | Salary of $8 (typo for $80,000) |
| **Timeliness** | Is the data current enough? | Benefits feed is 3 months stale |

### 3.2 Clean, reject, or flag? The triage decision

For every problem you find, there are exactly three professional responses:

1. **Fix automatically** — when the correction is unambiguous (trim whitespace, parse `"$85,000"` → `85000.0`, standardize `"FT"` → `"Full-Time"`).
2. **Reject to a dead-letter file** — when the record is unusable (no employee ID at all). *Reject* never means *delete*: rejected records go to a separate file with a reason, so nothing silently disappears.
3. **Flag for human review** — when a machine can't decide (two records 90% similar: same person or not?). Producing a review file for a business owner is a deliverable, not an admission of failure.

**The anti-pattern** is the fourth option juniors take by default: fix nothing, reject nothing, hope for the best.

### 3.3 Validation vs cleaning

- **Cleaning** changes data (standardize, convert, dedupe).
- **Validation** measures data and renders a verdict (pass/fail per rule) *without* changing it.

They're separate steps because you validate *after* cleaning to prove the cleaning worked — and you keep the validation report as evidence. In mature teams, validation gates the pipeline: too many failures and the run halts rather than publishing garbage. Tools like **Great Expectations** and **Pandera** industrialize this; today we build a small validator by hand so you understand what those tools do.

---

## 4. Step-by-Step Implementation

All code goes in `labs\lab03\`. We work on the Parquet files from Lab 02.

### Step 1 — Profile before you clean

**What:** Create `labs\lab03\profile.py`:

```python
from pathlib import Path
import pandas as pd

LAB02_OUT = Path(__file__).parent.parent / "lab02" / "output"

for name in ["hris", "acquiredco", "payroll", "benefits"]:
    df = pd.read_parquet(LAB02_OUT / f"{name}.parquet")
    print(f"\n===== {name} ({len(df):,} rows) =====")
    print("Nulls:\n", df.isna().sum()[lambda s: s > 0])
    print("Exact duplicate rows:", df.duplicated().sum())
    for col in df.select_dtypes("object").columns[:6]:
        uniques = df[col].nunique()
        if uniques <= 15:
            print(f"{col}: {sorted(df[col].dropna().unique().tolist())}")
```

**Why:** You cannot clean what you haven't measured. Profiling first tells you *which* problems exist and *how big* they are, so your cleaning code targets reality instead of guesses. Professionals always profile; amateurs start writing fixes immediately.

**Run it** and write down what you see: null counts, duplicate counts, and the actual category values (you'll likely find variants like `FT` / `Full-Time` / `full time`).

**Expected output:** a per-source report. **Verify:** every anomaly you note here should be addressed by a cleaning step below — that's your checklist.

**Common mistake:** profiling only the first source and assuming the rest look the same. Each source system has its own pathologies.

### Step 2 — Standardize text and categories

**What:** Create `labs\lab03\clean.py` and start with reusable helpers:

```python
"""Cleaning module for the HR integration pipeline."""
from pathlib import Path
import pandas as pd

OUTPUT_DIR = Path(__file__).parent / "output"

EMPLOYMENT_TYPE_MAP = {
    "ft": "Full-Time", "full-time": "Full-Time", "full time": "Full-Time",
    "pt": "Part-Time", "part-time": "Part-Time", "part time": "Part-Time",
    "contractor": "Contractor", "contract": "Contractor",
}


def standardize_text(s: pd.Series) -> pd.Series:
    """Trim whitespace and collapse internal runs of spaces."""
    return s.astype("string").str.strip().str.replace(r"\s+", " ", regex=True)


def standardize_employment_type(s: pd.Series) -> pd.Series:
    """Map the many observed spellings onto the canonical three values."""
    cleaned = standardize_text(s).str.lower()
    result = cleaned.map(EMPLOYMENT_TYPE_MAP)
    unmapped = cleaned[result.isna() & cleaned.notna()].unique()
    if len(unmapped) > 0:
        print(f"WARNING: unmapped employment types: {list(unmapped)}")
    return result
```

**Why:**
- **Mapping tables beat if/else chains.** When a new variant appears, you add one dictionary line — no logic changes. In production this map often lives in a config file so non-engineers can maintain it.
- **We *warn* on unmapped values instead of silently leaving them null.** Silence is the enemy: every transformation should tell you what it couldn't handle.

**Verify:** in a Python session, `standardize_employment_type(pd.Series(["FT", " full time ", "??"]))` should return `Full-Time`, `Full-Time`, `<NA>` and print a warning about `??`.

**Common mistake:** calling `.str.lower()` on a column containing actual `NaN` floats without `astype("string")` first — you get errors or object soup. Convert to the pandas string dtype before string operations.

### Step 3 — Fix the money

**What:** Add to `clean.py`:

```python
EXCHANGE_RATES_TO_USD = {"USD": 1.00, "EUR": 1.08, "GBP": 1.27}
PAY_FREQUENCY_TO_ANNUAL = {"Annual": 1, "Monthly": 12, "Bi-Weekly": 26}


def parse_salary(s: pd.Series) -> pd.Series:
    """'$85,000.00' / '€64.000' / 85000 -> float. Unparseable -> NaN."""
    txt = s.astype("string").str.replace(r"[^\d.\-]", "", regex=True)
    return pd.to_numeric(txt, errors="coerce")


def to_annual_usd(df: pd.DataFrame) -> pd.DataFrame:
    """Adds salary_usd_annual; keeps original columns for auditability."""
    df = df.copy()
    amount = parse_salary(df["salary"])
    rate = df["currency"].map(EXCHANGE_RATES_TO_USD)
    freq = df["pay_frequency"].map(PAY_FREQUENCY_TO_ANNUAL)
    df["salary_usd_annual"] = (amount * rate * freq).round(2)

    bad = df["salary_usd_annual"].isna() & df["salary"].notna()
    print(f"Salary conversion: {bad.sum()} of {len(df)} rows could not be converted")
    return df
```

(Adapt the column names to what your profiling in Step 1 actually found — that's part of the exercise.)

**Why:**
- **Add a new column; never overwrite the original.** If the conversion has a bug, the raw value is still there to re-derive from. Auditors will ask "where did this number come from?" — the answer must be visible in the data.
- **The regex strips symbols and thousands separators** in one pass, handling `$`, `€`, `£`, and commas.
- **Report the failure count.** A 0.1% conversion failure is a data note; a 40% failure means your assumption about the format is wrong.

**Expected output when run on payroll:** a conversion report line, and a numeric `salary_usd_annual` column.

**Common mistakes:**
- Multiplying by exchange rate but forgetting pay frequency (a €5,000 *monthly* salary is not a €5,000 *annual* salary).
- Hardcoding today's exchange rates deep in the logic. We put them in a named constant at the top — in Lab 05 they move to a config file. Rates change; code shouldn't have to.

### Step 4 — Standardize dates from three formats

**What:**

```python
def parse_dates_multiformat(s: pd.Series) -> pd.Series:
    """Handle YYYY-MM-DD, MM/DD/YYYY and DD-Mon-YYYY in one column."""
    s = s.astype("string").str.strip()
    result = pd.to_datetime(s, format="%Y-%m-%d", errors="coerce")
    for fmt in ("%m/%d/%Y", "%d-%b-%Y"):
        mask = result.isna() & s.notna()
        result.loc[mask] = pd.to_datetime(s[mask], format=fmt, errors="coerce")
    still_bad = result.isna() & s.notna()
    print(f"Date parsing: {still_bad.sum()} unparseable values")
    return result


def flag_implausible_dates(s: pd.Series) -> pd.Series:
    """True where a date is before 1970 or in the future."""
    return (s < pd.Timestamp("1970-01-01")) | (s > pd.Timestamp.now())
```

**Why:** Trying formats **in order, explicitly** is deterministic. The tempting shortcut — `pd.to_datetime(s)` with no format, letting pandas guess row by row — is dangerous: `03/04/2025` is March 4 in the US and April 3 in Europe, and guessing can interpret different rows differently. *You* must know which format each source uses (Step 1's profiling tells you) and declare it.

Plausibility flags matter because a date can parse perfectly and still be wrong: a hire date of `1899-12-31` usually means an Excel epoch bug, and a hire date next year means a typo. Parseable ≠ correct.

**Verify:** feed the function `pd.Series(["2024-01-15", "01/15/2024", "15-Jan-2024", "garbage"])` — the first three should produce the same date; the fourth becomes `NaT` and the counter reports 1.

### Step 5 — Namespace the overlapping IDs

**What:**

```python
def namespace_ids(df: pd.DataFrame, prefix: str, id_col: str = "employee_id") -> pd.DataFrame:
    """1042 -> 'GT-001042' so IDs from different companies can never collide."""
    df = df.copy()
    digits = df[id_col].astype("string").str.extract(r"(\d+)")[0]
    df[id_col] = prefix + "-" + digits.str.zfill(6)
    return df
```

Apply `namespace_ids(hris, "GT")` and `namespace_ids(acquiredco, "AC")`.

**Why:** The two companies both have an employee #1042 — different people. Any merge on raw IDs would fuse them. Prefixing with a company namespace makes collisions *structurally impossible* instead of merely unlikely. This is the same idea behind composite keys in databases and is a very common integration-project pattern.

**Common mistake:** doing the merge first "to see what happens" and deduping later. Once two people's records are fused, no code can reliably un-fuse them. Prevent, don't repair.

### Step 6 — Deduplicate in passes: exact → fuzzy → flag

**What:** Create `labs\lab03\dedup.py`:

```python
"""Multi-pass deduplication."""
from pathlib import Path
import pandas as pd
from rapidfuzz import fuzz

OUTPUT_DIR = Path(__file__).parent / "output"


def dedup_exact(df: pd.DataFrame) -> pd.DataFrame:
    """Pass 1: identical employee_id -> keep first, count the rest."""
    before = len(df)
    df = df.drop_duplicates(subset=["employee_id"], keep="first")
    print(f"Exact dedup: removed {before - len(df)} rows")
    return df


def find_fuzzy_candidates(df: pd.DataFrame, threshold: int = 88) -> pd.DataFrame:
    """Pass 2: near-identical names with hire dates within 30 days.

    We block by hire month so we compare ~dozens of rows at a time,
    not all pairs (which would be O(n^2) — 15,000 rows = 112 million pairs).
    """
    df = df.assign(_block=df["hire_date"].dt.to_period("M"))
    pairs = []
    for _, group in df.groupby("_block"):
        recs = group.to_dict("records")
        for i in range(len(recs)):
            for j in range(i + 1, len(recs)):
                a, b = recs[i], recs[j]
                score = fuzz.ratio(
                    f"{a['first_name']} {a['last_name']}".lower(),
                    f"{b['first_name']} {b['last_name']}".lower(),
                )
                if score >= threshold:
                    pairs.append({
                        "record_1_id": a["employee_id"],
                        "record_2_id": b["employee_id"],
                        "name_1": f"{a['first_name']} {a['last_name']}",
                        "name_2": f"{b['first_name']} {b['last_name']}",
                        "similarity": score,
                        "recommended_action": "HR review",
                    })
    return pd.DataFrame(pairs)
```

**Why the design:**
- **Passes go from certain to uncertain.** Exact ID duplicates are mechanically removable. Fuzzy name matches are *candidates* — the code flags them into a review file; a human decides. Auto-merging on 88% similarity would eventually merge two real people named "Ana Silva" and "Ana Silva" who are genuinely different humans.
- **Blocking** (grouping by hire month before comparing) is the standard technique that makes fuzzy matching feasible at scale. Comparing every row to every row grows quadratically; blocking keeps comparisons local.

**Run it and verify:** the exact pass prints a removal count; the fuzzy pass produces a candidates DataFrame you save as `output/probable_matches_review.csv`. Open it — every row should be a *plausible* same-person pair.

**Troubleshooting:** if the fuzzy pass takes minutes, your blocking column isn't working (check `hire_date` parsed as datetime, not string).

### Step 7 — Build the validator and gate the pipeline

**What:** Create `labs\lab03\validate.py`:

```python
"""Rule-based data quality validation with a pass/fail report."""
import re
import pandas as pd


class DataQualityValidator:
    def __init__(self, df: pd.DataFrame, name: str):
        self.df, self.name, self.results = df, name, []

    def _record(self, check: str, description: str, failed_mask: pd.Series):
        failed = int(failed_mask.sum())
        total = len(self.df)
        self.results.append({
            "check": check, "description": description,
            "total": total, "passed": total - failed, "failed": failed,
            "pass_rate": round(100 * (total - failed) / total, 2) if total else 100.0,
            "status": "PASS" if failed == 0 else "FAIL",
        })

    def not_null(self, col: str):
        self._record(f"not_null:{col}", f"{col} must be present", self.df[col].isna())
        return self

    def unique(self, col: str):
        self._record(f"unique:{col}", f"{col} must be unique",
                     self.df[col].duplicated(keep=False))
        return self

    def in_set(self, col: str, allowed: set):
        self._record(f"in_set:{col}", f"{col} in {sorted(allowed)}",
                     ~self.df[col].isin(allowed) & self.df[col].notna())
        return self

    def matches(self, col: str, pattern: str):
        ok = self.df[col].astype("string").str.match(pattern, na=False)
        self._record(f"regex:{col}", f"{col} matches {pattern}", ~ok)
        return self

    def in_range(self, col: str, lo, hi):
        s = self.df[col]
        self._record(f"range:{col}", f"{lo} <= {col} <= {hi}",
                     ((s < lo) | (s > hi)) & s.notna())
        return self

    def report(self) -> pd.DataFrame:
        return pd.DataFrame(self.results)


def run_validation(df: pd.DataFrame) -> pd.DataFrame:
    v = (DataQualityValidator(df, "employees")
         .not_null("employee_id").not_null("email")
         .unique("employee_id")
         .in_set("employment_type", {"Full-Time", "Part-Time", "Contractor"})
         .matches("employee_id", r"^(GT|AC)-\d{6}$")
         .matches("email", r"^[\w.+-]+@[\w-]+\.[\w.]+$")
         .in_range("salary_usd_annual", 15_000, 2_000_000))
    report = v.report()
    failures = (report["status"] == "FAIL").sum()
    print(report.to_string(index=False))
    if failures > 2:
        raise RuntimeError(f"QUALITY GATE FAILED: {failures} checks failing — halting pipeline")
    return report
```

**Why:**
- The validator **measures without mutating** — the report is your evidence that cleaning worked.
- The **quality gate** (`raise` if >2 checks fail) encodes a business decision: better no report than a wrong report. The threshold belongs to the data's business owner; your job is to make the gate exist.
- The **fluent chained API** (`.not_null(...).unique(...)`) mirrors how Great Expectations and Pandera feel, so those tools will be familiar later.

**Run it on your cleaned data and verify:** a printed table with one row per check. Save it: `report.to_csv(OUTPUT_DIR / "quality_report.csv", index=False)`. Deliberately break something (set one employment type to `"???"`) and confirm the check flips to FAIL — *always test that your alarms actually fire.*

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Profile before cleaning** | Fixes target measured reality, not assumptions. |
| **Never overwrite originals; add derived columns** | Auditability — every number traceable to its raw source. |
| **Every transformation reports what it couldn't handle** | Silent data loss is the worst failure mode in the profession. |
| **Triage: fix / reject-to-dead-letter / flag-for-review** | Machines fix the unambiguous; humans decide the ambiguous; nothing vanishes. |
| **Mapping tables over branching logic** | Maintainable by config change, eventually by non-engineers. |
| **Deterministic date parsing (explicit formats, in order)** | Guessing formats row-by-row corrupts data across US/EU conventions. |
| **Blocking for fuzzy matching** | The difference between an overnight job and a 10-second one. |
| **Validation as a separate, evidence-producing step with a gate** | Publishing is a privilege the data must earn on every run. |

---

## 6. Reflection

### What you learned
Profiling, principled cleaning (text, money, dates, IDs), multi-pass deduplication with human-review escalation, and rule-based validation that gates the pipeline and produces an evidence report.

### Why it matters
Cleaning and validation are where data engineering earns trust. Every practice here — auditability, loud failures, quality gates — is something clients and auditors explicitly look for.

### Interview questions (with model answers)

1. **"How do you approach a new, messy dataset?"**
   Profile first: row counts, null rates, duplicates, distinct values of categoricals, min/max of numerics and dates. Then design targeted fixes, and validate afterward to prove they worked.

2. **"Exact duplicates are easy. How do you find non-exact duplicates?"**
   Fuzzy string similarity (e.g., rapidfuzz/Levenshtein) on normalized names, combined with a second signal like hire date proximity, using blocking to avoid O(n²). High-confidence matches can auto-merge per agreed rules; borderline ones go to a human review file.

3. **"What do you do with records that fail validation?"**
   Depends on severity and business rules: auto-fix unambiguous issues, route unusable records to a dead-letter file with a reason code, and gate the pipeline if failures exceed an agreed threshold. Never silently drop.

4. **"Why validate after cleaning if you wrote the cleaning code yourself?"**
   Because cleaning code has bugs and data changes over time. Validation is the independent measurement that catches both, and its report is the evidence trail.

5. **"How do you handle dates arriving in multiple formats?"**
   Identify each source's format from profiling, parse with explicit format strings in order, coerce failures to NaT with a counted report, and add plausibility checks (not before 1970, not in the future).

6. **"What is a data quality gate and where would you set the threshold?"**
   An automated halt when validation failures exceed a threshold. The threshold is a business decision made with the data owner — engineering's job is to implement and enforce it.

### Common interview traps
- "I'd just drop rows with nulls" — instant red flag; interviewers want triage reasoning.
- Auto-merging fuzzy matches — shows you haven't been burned yet. Say "flag for review".
- Not mentioning that rejected records are *kept* somewhere. Dead-letter, not delete.

### Key takeaways
1. Profile → clean → validate → gate. In that order, every time.
2. Add columns, keep originals, report every coercion.
3. Duplicates: exact passes merge, fuzzy passes flag.
4. A quality gate turns "we hope it's right" into "it can't publish unless it's right."

**Next:** [Lab 04](Lab_04_Building_a_Modular_ETL_Pipeline.md) — you have ingestion functions and cleaning functions. Now we assemble them into one coherent, runnable pipeline.

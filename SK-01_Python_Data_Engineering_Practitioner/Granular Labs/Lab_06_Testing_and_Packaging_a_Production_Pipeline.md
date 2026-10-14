# Lab 06 — Testing and Packaging a Production Pipeline

> **Goal:** Protect the ShopStream pipeline with automated tests: unit tests for each cleaning rule, tests that lock in the GDPR opt-out rule, offline tests for the CRM API, and one end-to-end test that runs the whole pipeline in a temporary folder. Then package the project so another engineer can set it up and run it from the README alone.
>
> **Time:** 3–4 hours | **Difficulty:** Intermediate | **Prerequisites:** [Lab 05](Lab_05_Logging_Error_Handling_and_Idempotency.md) completed: `pipeline.py` contains the complete guided lab code and `python pipeline.py` gives 745 records and 7/7 checks.
>
> **Guided lab coverage:** [SK-01 End-to-End Guided Lab](../Labs/SK-01_End_to_End_Guided_Lab.md): tests for **Parts 2–5 and 8** (cleaning, deduplication, validation, CRM API, full run) and the handover of the finished project. The guided lab has no testing part; this lab adds it on top of the same `pipeline.py`.

---

## The business problem

In 3 weeks the campaign goes out. Before then, someone will "just tweak" the pipeline: a new region spelling, a faster dedup, a new source. In Lab 03 you saw that commenting out 2 lines silently breaks the GDPR opt-out rule. Nothing crashes and the summary still says 7/7. Only an EU customer's complaint would reveal it.

**Automated tests** are the safety net. They're small programs that check your code gives the expected answers, run in seconds, and fail loudly the moment a change breaks a rule. Packaging (`requirements.txt`, `README.md`, `.gitignore`) is the other half of a handover: the next engineer can rebuild your environment and run the pipeline without you.

---

## Learning objectives

By the end of this lab you can:

1. Write and run pytest tests: plain `assert`, `@pytest.mark.parametrize`, `pytest.raises`, and the `tmp_path`/`monkeypatch` fixtures.
2. Choose what to test: business rules (GDPR opt-out, source priority), edge cases (blanks, junk) and the contract between stages.
3. Test code that calls an API, offline, with `unittest.mock.patch`.
4. Write an end-to-end test that runs the whole pipeline in isolation.
5. Package a project for handover: pinned requirements, a README and a `.gitignore`.

---

## Key terms

- **Unit test:** checks one function with small, hand-made inputs, e.g. `standardize_regions(["usa"]) == ["US"]`.
- **End-to-end (integration) test:** runs the whole pipeline and checks the final outputs.
- **Regression:** a bug in something that used to work. Tests exist mainly to catch regressions.
- **Fixture:** something pytest prepares for a test. `tmp_path` is a fresh empty folder; `monkeypatch` changes a setting for one test and undoes it afterwards.
- **Parametrize:** run the same test with several inputs: one test function, many cases.

---

## Steps

### Step 1 — How pytest finds and runs tests

pytest is already installed (Lab 00). Its rules are simple:
- It looks in files named `test_*.py` (ours: `tests\test_pipeline.py`).
- It runs every function whose name starts with `test_`.
- A test **passes** if it finishes, and **fails** if an `assert` is false or an error is raised. `assert x == y` is all you need; pytest shows both values when it fails.

Always run it from the project root as **`python -m pytest`**. Like `python -m practice.x` in Lab 02, the `-m` form puts the project root on Python's import path, so `import pipeline` works from `tests\`.

### Step 2 — Write the test file

Create `tests\test_pipeline.py` with this content. The sections below explain it part by part.

```python
"""Unit tests for the ShopStream pipeline (run from the project root: python -m pytest -v)."""
from unittest.mock import MagicMock, patch

import numpy as np
import pandas as pd
import pytest
import requests

import pipeline
from pipeline import (
    DataQualityValidator,
    align_schema,
    deduplicate_customers,
    ingest_crm_api,
    standardize_emails,
    standardize_names,
    standardize_phone_numbers,
    standardize_regions,
    validate_emails,
)


# ── Cleaning functions ────────────────────────────────────────────────────────
def test_standardize_emails_trims_lowercases_and_removes_spaces():
    raw = pd.Series(["  Maria@GMAIL.com ", "jo hn@yahoo.com"])
    assert standardize_emails(raw).tolist() == ["maria@gmail.com", "john@yahoo.com"]


def test_standardize_emails_turns_blanks_into_missing():
    result = standardize_emails(pd.Series(["a@x.com", "", None, np.nan]))
    assert result.iloc[0] == "a@x.com"
    assert result.iloc[1:].isna().all()


@pytest.mark.parametrize("email, expected", [
    ("customer1@yahoo.com", True),
    ("missing@", False),
    ("@nodomain.com", False),
    ("double@@sign.com", False),
    ("not-an-email", False),
])
def test_validate_emails(email, expected):
    assert validate_emails(pd.Series([email])).iloc[0] == expected


def test_validate_emails_treats_missing_as_invalid():
    emails = standardize_emails(pd.Series(["a@x.com", None]))
    assert validate_emails(emails).tolist() == [True, False]


@pytest.mark.parametrize("raw, expected", [
    ("+1 (555) 123-4567", "+15551234567"),
    ("555.123.4567", "5551234567"),
    ("+44 20 7946 0958", "+442079460958"),
])
def test_standardize_phone_numbers(raw, expected):
    assert standardize_phone_numbers(pd.Series([raw])).iloc[0] == expected


@pytest.mark.parametrize("raw", ["invalid-phone", "12345", None, ""])
def test_standardize_phone_numbers_rejects_junk(raw):
    assert pd.isna(standardize_phone_numbers(pd.Series([raw])).iloc[0])


def test_standardize_names():
    raw = pd.Series(["  maria  ", "o'brien", "anne   marie", None])
    assert standardize_names(raw).tolist()[:3] == ["Maria", "O'Brien", "Anne Marie"]
    assert pd.isna(standardize_names(raw).iloc[3])


def test_standardize_regions_maps_variants_and_drops_unknowns():
    raw = pd.Series(["usa", " Europe ", "EMEA", "Asia Pacific", "AP", "N/A", "mars"])
    result = standardize_regions(raw)
    assert result.tolist()[:5] == ["US", "EU", "EU", "APAC", "APAC"]
    assert result.iloc[5:].isna().all()


def test_align_schema_adds_missing_and_drops_extra_columns():
    df = pd.DataFrame({"email": ["a@x.com"], "lifetime_value": [99.0], "source": ["crm"]})
    result = align_schema(df, "crm")
    assert list(result.columns) == ["email", "first_name", "last_name", "phone", "region",
                                    "registration_date", "opt_out", "source"]
    assert pd.isna(result.loc[0, "phone"])


# ── Deduplication ─────────────────────────────────────────────────────────────
def make_cleaned(rows):
    """Build a small DataFrame shaped like clean_dataframe()'s output."""
    df = pd.DataFrame(rows)
    df["email_valid"] = validate_emails(df["email"])
    return df


def test_dedup_keeps_one_row_per_email_and_prefers_crm():
    df = make_cleaned([
        {"email": "a@x.com", "first_name": "Web", "phone": np.nan, "region": "EU",
         "registration_date": pd.NaT, "opt_out": False, "source": "website"},
        {"email": "a@x.com", "first_name": "Crm", "phone": np.nan, "region": np.nan,
         "registration_date": pd.NaT, "opt_out": False, "source": "crm"},
    ])
    result = deduplicate_customers(df)
    assert len(result) == 1
    row = result.iloc[0]
    assert row["first_name"] == "Crm"        # CRM has the highest priority...
    assert row["region"] == "EU"             # ...but blanks are filled from other sources
    assert row["source_count"] == 2


def test_dedup_opt_out_from_any_source_wins():
    """GDPR: one opt-out anywhere means the customer must not be emailed."""
    df = make_cleaned([
        {"email": "a@x.com", "first_name": "A", "opt_out": False, "source": "crm"},
        {"email": "a@x.com", "first_name": "A", "opt_out": True, "source": "marketing"},
    ])
    assert bool(deduplicate_customers(df).iloc[0]["opt_out"]) is True


def test_dedup_keeps_invalid_email_rows():
    df = make_cleaned([
        {"email": "a@x.com", "first_name": "A", "opt_out": False, "source": "crm"},
        {"email": "missing@", "first_name": "B", "opt_out": False, "source": "website"},
        {"email": "missing@", "first_name": "C", "opt_out": False, "source": "erp"},
    ])
    result = deduplicate_customers(df)
    assert len(result) == 3                  # invalid emails are never merged
    assert result["email"].tolist() == ["a@x.com", "missing@", "missing@"]


def test_dedup_does_not_modify_its_input():
    df = make_cleaned([{"email": "a@x.com", "first_name": "A", "opt_out": False, "source": "crm"}])
    before = df.copy()
    deduplicate_customers(df)
    pd.testing.assert_frame_equal(df, before)


# ── Validator ─────────────────────────────────────────────────────────────────
def test_validator_pass_and_fail_against_threshold():
    df = pd.DataFrame({"email": ["a@x.com"] * 19 + [None]})   # 95% complete
    v = DataQualityValidator(df, threshold=0.95)
    assert v.check_not_null("email", "")["status"] == "PASS"
    v = DataQualityValidator(df, threshold=0.99)
    assert v.check_not_null("email", "")["status"] == "FAIL"


def test_validator_unique_counts_only_extra_copies():
    df = pd.DataFrame({"email": ["a@x.com", "a@x.com", "a@x.com", "b@x.com", None]})
    result = DataQualityValidator(df).check_unique("email", "")
    assert (result["failed"], result["total"]) == (2, 4)


# ── CRM API (no network: requests.get is replaced by a fake) ──────────────────
def fake_page(customers, total_pages):
    response = MagicMock()
    response.json.return_value = {"customers": customers, "total_pages": total_pages}
    return response


def test_crm_api_reads_every_page_and_renames_columns():
    pages = [fake_page([{"email": "a@x.com", "profile": {"first_name": "Ana"}}], 2),
             fake_page([{"email": "b@x.com", "profile": {"first_name": "Ben"}}], 2)]
    with patch("pipeline.requests.get", side_effect=pages) as get:
        df = ingest_crm_api("https://crm.example/v2/customers", api_key="test-key")
    assert get.call_count == 2
    assert df["first_name"].tolist() == ["Ana", "Ben"]
    assert (df["source"] == "crm").all()


def test_crm_api_stops_on_server_error():
    broken = MagicMock()
    broken.raise_for_status.side_effect = requests.HTTPError("503 Server Error")
    with patch("pipeline.requests.get", side_effect=[fake_page([{"email": "a@x.com"}], 2), broken]):
        with pytest.raises(requests.HTTPError):
            ingest_crm_api("https://crm.example/v2/customers", api_key="test-key")


# ── End to end: the whole pipeline in a temporary folder ──────────────────────
def test_full_pipeline_end_to_end(tmp_path, monkeypatch):
    monkeypatch.setitem(pipeline.CONFIG, "input_dir", tmp_path / "raw")
    monkeypatch.setitem(pipeline.CONFIG, "output_dir", tmp_path / "processed")
    for folder in ["raw", "processed"]:
        (tmp_path / folder).mkdir()

    pipeline.generate_synthetic_data()
    pipeline.run_pipeline()

    golden = pd.read_parquet(tmp_path / "processed" / "golden_customers.parquet")
    assert len(golden) == 745
    assert golden["opt_out"].dtype == bool
    valid_emails = golden.loc[golden["email_valid"], "email"]
    assert valid_emails.is_unique                    # dedup worked
    report = pd.read_csv(tmp_path / "processed" / "quality_report.csv")
    assert (report["status"] == "PASS").all()
    assert (tmp_path / "processed" / "customer_quality_report.png").exists()
```

**What each section tests, and why:**

| Section | Tests | Why it matters |
|---|---|---|
| Cleaning functions | The exact examples from the guided lab docstrings and your Lab 03 practice: `+1 (555) 123-4567` → `+15551234567`, `"usa"` → `US`, `"missing@"` invalid... | These rules decide who gets emailed. `@pytest.mark.parametrize` runs one test function for each case, so adding a case is one line. |
| `align_schema` | Missing columns are added and extra ones dropped | The contract between ingestion and cleaning: every source must arrive with the same 8 columns. |
| Deduplication | One row per email; CRM wins; blanks filled from other sources; **an opt-out anywhere wins**; invalid emails never merged; input not modified | The business rules from Lab 03. `make_cleaned` builds a tiny DataFrame shaped like `clean_dataframe`'s output, small enough to check by eye. |
| Validator | 95% passes at a 0.95 threshold and fails at 0.99; `UNIQUE` counts only the extra copies | The quality gate must be exactly as strict as documented. |
| CRM API | Pages are followed and columns renamed; a 503 raises an error | Lab 05's offline checks, now automatic. `pytest.raises(requests.HTTPError)` *passes* only if that error is raised. |
| End to end | The whole pipeline in a temporary folder gives 745 records, unique valid emails, all checks PASS, and a PNG | Proves the stages work **together**. |

**How the end-to-end test stays isolated:** `monkeypatch.setitem(pipeline.CONFIG, "input_dir", tmp_path / "raw")` points the pipeline at a fresh temporary folder for this one test, and is undone afterwards. Your real `data\raw` and `data\processed` are never touched. This works only because every function reads its folders from `CONFIG` instead of hard-coding them, one more reason for the config dictionary from Lab 02.

### Step 3 — Run the tests

```powershell
python -m pytest -v
```

**Expected output** (about 30–60 seconds; the end-to-end test runs the whole pipeline, and its log lines are hidden unless a test fails):

```
============================= test session starts =============================
platform win32 -- Python 3.12.x, pytest-8.x.x, pluggy-1.x.x -- C:\DataEngineering\sk01\.venv\Scripts\python.exe
rootdir: C:\DataEngineering\sk01
collected 27 items

tests/test_pipeline.py::test_standardize_emails_trims_lowercases_and_removes_spaces PASSED [  3%]
tests/test_pipeline.py::test_standardize_emails_turns_blanks_into_missing PASSED [  7%]
tests/test_pipeline.py::test_validate_emails[customer1@yahoo.com-True] PASSED [ 11%]
tests/test_pipeline.py::test_validate_emails[missing@-False] PASSED      [ 14%]
tests/test_pipeline.py::test_validate_emails[@nodomain.com-False] PASSED [ 18%]
tests/test_pipeline.py::test_validate_emails[double@@sign.com-False] PASSED [ 22%]
tests/test_pipeline.py::test_validate_emails[not-an-email-False] PASSED  [ 25%]
tests/test_pipeline.py::test_validate_emails_treats_missing_as_invalid PASSED [ 29%]
tests/test_pipeline.py::test_standardize_phone_numbers[+1 (555) 123-4567-+15551234567] PASSED [ 33%]
tests/test_pipeline.py::test_standardize_phone_numbers[555.123.4567-5551234567] PASSED [ 37%]
tests/test_pipeline.py::test_standardize_phone_numbers[+44 20 7946 0958-+442079460958] PASSED [ 40%]
tests/test_pipeline.py::test_standardize_phone_numbers_rejects_junk[invalid-phone] PASSED [ 44%]
tests/test_pipeline.py::test_standardize_phone_numbers_rejects_junk[12345] PASSED [ 48%]
tests/test_pipeline.py::test_standardize_phone_numbers_rejects_junk[None] PASSED [ 51%]
tests/test_pipeline.py::test_standardize_phone_numbers_rejects_junk[] PASSED [ 55%]
tests/test_pipeline.py::test_standardize_names PASSED                    [ 59%]
tests/test_pipeline.py::test_standardize_regions_maps_variants_and_drops_unknowns PASSED [ 62%]
tests/test_pipeline.py::test_align_schema_adds_missing_and_drops_extra_columns PASSED [ 66%]
tests/test_pipeline.py::test_dedup_keeps_one_row_per_email_and_prefers_crm PASSED [ 70%]
tests/test_pipeline.py::test_dedup_opt_out_from_any_source_wins PASSED   [ 74%]
tests/test_pipeline.py::test_dedup_keeps_invalid_email_rows PASSED       [ 77%]
tests/test_pipeline.py::test_dedup_does_not_modify_its_input PASSED      [ 81%]
tests/test_pipeline.py::test_validator_pass_and_fail_against_threshold PASSED [ 85%]
tests/test_pipeline.py::test_validator_unique_counts_only_extra_copies PASSED [ 88%]
tests/test_pipeline.py::test_crm_api_reads_every_page_and_renames_columns PASSED [ 92%]
tests/test_pipeline.py::test_crm_api_stops_on_server_error PASSED        [ 96%]
tests/test_pipeline.py::test_full_pipeline_end_to_end PASSED             [100%]

============================= 27 passed in 50.53s =============================
```

`-v` (verbose) lists every test. Each parametrized case appears separately with its input in brackets, e.g. `[missing@-False]`. Without `-v` you just get one dot per passing test.

✅ **Checkpoint:** `27 passed`.

### Step 4 — Watch a test catch the GDPR bug

Repeat the Lab 03 experiment, but this time with a safety net. In `pipeline.py`, inside `merge_group`, comment out the 2 opt-out lines:

```python
        # if "opt_out" in group.columns:
        #     best["opt_out"] = group["opt_out"].fillna(0).astype(bool).any()
```

Run the tests, skipping the slow end-to-end test with `-k` (only tests whose names match the expression run):

```powershell
python -m pytest -q -k "not end_to_end"
```

**Expected output:**

```
...................F......                                               [100%]
================================== FAILURES ===================================
___________________ test_dedup_opt_out_from_any_source_wins ___________________

    def test_dedup_opt_out_from_any_source_wins():
        """GDPR: one opt-out anywhere means the customer must not be emailed."""
        df = make_cleaned([
            {"email": "a@x.com", "first_name": "A", "opt_out": False, "source": "crm"},
            {"email": "a@x.com", "first_name": "A", "opt_out": True, "source": "marketing"},
        ])
>       assert bool(deduplicate_customers(df).iloc[0]["opt_out"]) is True
E       assert False is True
E        +  where False = bool(np.False_)

tests\test_pipeline.py:115: AssertionError
=========================== short test summary info ===========================
FAILED tests/test_pipeline.py::test_dedup_opt_out_from_any_source_wins - asse...
1 failed, 25 passed, 1 deselected in 30.62s
```

The failure names the rule (the test's name and docstring), shows the exact line, and shows the value it got. In Lab 03 this bug was silent. Now it can't reach production unnoticed. **Undo the change** and confirm `python -m pytest -q` is all green again.

> **The habit:** whenever you fix a bug, first write a test that fails because of it, then fix the code so the test passes. The bug can never quietly come back.

✅ **Checkpoint:** you saw the failure, restored the code, and all 27 tests pass.

### Step 5 — Package the project for handover

**1. Pin the environment.** Re-create `requirements.txt` so it lists exactly what the pipeline was tested with:

```powershell
python -m pip freeze > requirements.txt
```

**2. Ignore what shouldn't be shared.** Create `.gitignore` in the project root:

```
.venv/
__pycache__/
.pytest_cache/
logs/
data/raw/
data/processed/
practice/
```

Generated data and outputs can be re-created by running the pipeline, the venv by `requirements.txt`, and logs are per-machine. Real customer data must **never** go into Git.

**3. Write `README.md`** in the project root. It's for the engineer who takes over at 2 a.m.:

````markdown
# ShopStream Customer Data Quality Pipeline

Ingests customers from 4 sources (website CSV, CRM JSON/API, ERP fixed-width, marketing Excel),
cleans and deduplicates them, runs 7 quality checks, and delivers golden customer files.

## Setup (Windows, Python 3.12)
    python -m venv .venv
    .\.venv\Scripts\Activate.ps1
    python -m pip install -r requirements.txt

## Run
    python pipeline.py
First run generates test data in data\raw\. Outputs in data\processed\:
golden_customers.parquet, golden_customers.csv, quality_report.csv, customer_quality_report.png.
Log: logs\pipeline.log. Exit code 0 = success.

## Configuration
Settings are in CONFIG at the top of pipeline.py.
Secrets come from environment variables: CRM_API_KEY (CRM API), ANTHROPIC_API_KEY (optional AI step).

## Tests
    python -m pytest -v

## Key rules
- Source priority: CRM > website > ERP > marketing.
- An opt-out in ANY source makes the golden record opted out (GDPR).
- Each quality check must pass for at least 95% of records.
````

**4. Final check, as the next engineer would.** Close every window, open a **new** PowerShell, and follow your README word for word:

```powershell
cd C:\DataEngineering\sk01
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python -m pytest -q
python pipeline.py
```

**Expected:** pip reports `Requirement already satisfied` for everything, the tests show `27 passed`, and the pipeline ends with `PIPELINE COMPLETE`, 745 golden records and 7/7 checks.

✅ **Checkpoint:** a fresh window plus only the README is enough to set up, test and run.

### Step 6 — Final project check against the guided lab

Your project should now look like this:

```
C:\DataEngineering\sk01\
├── pipeline.py                 # every code block of the guided lab, Parts 1–8
├── tests\test_pipeline.py      # 27 tests
├── requirements.txt
├── README.md
├── .gitignore
├── practice\                   # your Lab 01–05 learning scripts
├── data\raw\                   # 4 generated source files
├── data\processed\             # golden_customers.parquet/.csv, quality_report.csv, customer_quality_report.png
└── logs\pipeline.log
```

You have now built everything in the [SK-01 End-to-End Guided Lab](../Labs/SK-01_End_to_End_Guided_Lab.md). Work through its **Reflection Questions**. Labs 03 and 05 covered Q1–Q2 and Q4–Q5, and Q3 (sending customer data to an LLM) is worth discussing with your mentor.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'pipeline'` | Ran `pytest` instead of `python -m pytest`, or from inside `tests\` | `cd C:\DataEngineering\sk01`, then `python -m pytest` |
| `collected 0 items` | File or functions not named `test_*` | `tests\test_pipeline.py`, functions start with `test_` |
| End-to-end test gives a count other than 745 | `generate_synthetic_data` differs from the guided lab, or numpy's seed was changed | Re-copy guided lab Step 1.2 |
| End-to-end test overwrote your real outputs | `monkeypatch.setitem` lines missing | The test must point `input_dir`/`output_dir` at `tmp_path` |
| `KeyError: 'email_valid'` in a dedup test | The test DataFrame is missing a column `deduplicate_customers` needs | Use `make_cleaned`, which adds `email_valid` |
| Tests pass but the pipeline fails | Tests don't cover the broken part | Add a test that reproduces the failure, then fix it |
| `.ps1` activation blocked in the new window | Execution policy | See Lab 00 troubleshooting |

---

## Key takeaways

- Test the business rules (opt-out wins, CRM priority, the 95% threshold), not just "it runs". The most dangerous bugs don't crash.
- Keep unit tests small and readable; use `parametrize` for many cases, mocks for APIs, and `tmp_path` + `monkeypatch` to isolate end-to-end runs.
- Every bug fix starts with a test that fails because of it.
- Handover = pinned `requirements.txt` + a README that a stranger can follow + a `.gitignore` that keeps data and secrets out of Git.
- Settings in one `CONFIG` dictionary made the pipeline testable. Good design and testability go together.

---

## Next

Back to the [SK-01 End-to-End Guided Lab](../Labs/SK-01_End_to_End_Guided_Lab.md): you've built and tested the complete ShopStream pipeline. Compare your `pipeline.py` with the guided lab once more and answer its reflection questions. Then continue to the SK-01 capstone project in `Project\`.

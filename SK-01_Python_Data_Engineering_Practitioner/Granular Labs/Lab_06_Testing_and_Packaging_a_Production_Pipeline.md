# Lab 06 — Testing and Packaging a Production Pipeline

> **Module:** SK-01 Python Data Engineering Practitioner
> **Estimated time:** 4–6 hours
> **Prerequisites:** Lab 05 — your pipeline logs, retries, reads config, and is idempotent.

---

## 1. Environment Setup

```powershell
cd $HOME\Downloads\Upskilling\SK-01_Python_Data_Engineering_Practitioner
.\.venv\Scripts\Activate.ps1
pip install pytest
mkdir pipeline\tests -Force
New-Item pipeline\tests\__init__.py -ItemType File
```

**pytest** is the de-facto standard Python test runner: it discovers files named `test_*.py`, runs every function named `test_*`, and reports pass/fail.

**Verify:**

```powershell
cd pipeline
pytest --version
```

**Expected output:** a version line like `pytest 8.x.x`.

**Common problems:** `pytest` not recognized → venv not active. Tests not discovered later → files/functions must be named `test_*`; a file named `tests.py` or a function named `check_salary` is silently ignored.

---

## 2. Business Context

**The problem:** Next month, someone (probably you) will change `parse_salary` to handle a new currency. How will you know the change didn't break the *existing* currencies? Today the answer is "run the whole pipeline and eyeball the output" — slow, unreliable, and it gets skipped under deadline pressure. Then a regression ships, payroll numbers are wrong, and the client finds out before you do.

**Automated tests** are the answer the industry converged on: small programs that verify your functions against known inputs and expected outputs, runnable in seconds, on every change.

**Why businesses insist on tests:**
- **Regression protection** — the #1 value. Most bugs are introduced by *changes to working code*.
- **Change velocity** — with tests, engineers refactor confidently; without, codebases fossilize because everyone fears touching them.
- **Handover** — a consultancy delivering this pipeline to a client *will be asked for testing evidence*. No tests, no acceptance. (Your capstone replicates this exactly.)
- **Documentation that can't lie** — tests show how functions are meant to be called, and unlike comments, they fail when they go stale.

**What happens without them?** The industry term is "change paralysis": systems nobody dares modify, patched around instead of fixed, until a rewrite is cheaper. Tests are how codebases stay alive.

---

## 3. Concept Explanation

### 3.1 The testing pyramid (data engineering edition)

| Layer | What it checks | Speed | How many |
|---|---|---|---|
| **Unit tests** | One function, in isolation, with tiny hand-made data | milliseconds | many |
| **Integration tests** | Stages working together (ingest → clean → validate) on small realistic files | seconds | some |
| **End-to-end tests** | The whole pipeline on a sample dataset, checking final outputs | seconds–minutes | few |
| **Data quality checks** | The *data* each production run (your Lab 03 validator) | per run | per run |

The last row is the data-engineering twist: **tests check code once per change; data quality checks check data on every run.** You need both — a bug-free pipeline fed garbage still produces garbage.

### 3.2 Anatomy of a test: Arrange–Act–Assert

Every good test has three visible parts:

```python
def test_parse_salary_strips_symbols():
    s = pd.Series(["$85,000.00"])        # Arrange: build tiny input
    result = parse_salary(s)             # Act:     call the function
    assert result.iloc[0] == 85000.0     # Assert:  verify the outcome
```

**Tiny, hand-made inputs are the point.** A test on 3 rows you constructed proves exactly one behavior and fails with an obvious cause. A test on the full 15,000-row file proves nothing specific and fails mysteriously.

### 3.3 What to test (and what not to)

Prioritize, in order:
1. **Pure transformation functions** (`parse_salary`, `namespace_ids`, date parsing) — highest value per line of test.
2. **Edge cases and past bugs** — empty input, all-null column, the weird value that once broke production. Every bug you fix gets a test so it can never return ("regression test").
3. **The validator itself** — your quality gate must provably fire. An alarm you never tested is decoration.

Don't bother testing: pandas itself (`read_csv` works), trivial getters, or exact log message wording. Test *your* logic.

### 3.4 Fixtures and temporary directories

Tests must be **independent and repeatable**: no test may depend on another's leftovers, on your real data folder, or on network access. pytest gives you `tmp_path` — a fresh temporary folder per test — so file-writing tests never touch real zones and clean up automatically.

---

## 4. Step-by-Step Implementation

All tests live in `pipeline/tests/`. Run everything from inside `pipeline\`.

### Step 1 — First unit tests: the salary parser

**What:** Create `tests/test_clean.py`:

```python
import pandas as pd
import pytest

from hr_pipeline.clean import parse_salary, namespace_ids, parse_dates_multiformat


class TestParseSalary:
    def test_plain_number_passes_through(self):
        assert parse_salary(pd.Series(["85000"])).iloc[0] == 85000.0

    def test_dollar_sign_and_commas_stripped(self):
        assert parse_salary(pd.Series(["$85,000.00"])).iloc[0] == 85000.0

    def test_garbage_becomes_nan(self):
        assert pd.isna(parse_salary(pd.Series(["N/A"])).iloc[0])

    def test_empty_series_returns_empty(self):
        assert len(parse_salary(pd.Series([], dtype="string"))) == 0

    def test_negative_salary_preserved_for_validation_to_catch(self):
        # cleaning parses; the VALIDATOR judges plausibility — separation of concerns
        assert parse_salary(pd.Series(["-5000"])).iloc[0] == -5000.0
```

**Why these five:** one happy path, one realistic mess, one garbage input, one empty input, one *policy* case documenting a design decision (cleaning doesn't judge — validation does). Five tests, five distinct behaviors, each with a name that reads as a specification.

**Run:**

```powershell
pytest tests\test_clean.py -v
```

**Expected output:** five green `PASSED` lines. `-v` shows each test name.

**Common mistakes:**
- Multiple unrelated asserts crammed into one test — when it fails you can't tell which behavior broke. One behavior per test.
- Vague names (`test_1`, `test_salary`). The name should state the expected behavior; a failing test's name is the first diagnostic you read.

**Troubleshooting:** `ModuleNotFoundError: hr_pipeline` → run pytest from inside `pipeline\`. If it persists, create `pipeline/pytest.ini` with:

```ini
[pytest]
pythonpath = .
```

### Step 2 — Test IDs and dates

**What:** Extend `tests/test_clean.py`:

```python
class TestNamespaceIds:
    def test_prefix_and_zero_padding(self):
        df = pd.DataFrame({"employee_id": ["1042"]})
        assert namespace_ids(df, "GT")["employee_id"].iloc[0] == "GT-001042"

    def test_original_dataframe_not_mutated(self):
        df = pd.DataFrame({"employee_id": ["7"]})
        namespace_ids(df, "GT")
        assert df["employee_id"].iloc[0] == "7"   # caller's data untouched


class TestParseDatesMultiformat:
    def test_three_formats_agree(self):
        s = pd.Series(["2024-01-15", "01/15/2024", "15-Jan-2024"])
        result = parse_dates_multiformat(s)
        assert result.nunique() == 1
        assert result.iloc[0] == pd.Timestamp("2024-01-15")

    def test_unparseable_becomes_nat(self):
        assert pd.isna(parse_dates_multiformat(pd.Series(["not a date"])).iloc[0])
```

**Why `test_original_dataframe_not_mutated` deserves special attention:** functions that silently modify their *input* cause the nastiest pandas bugs — a caller's DataFrame changes behind their back, and the symptom appears three modules away. This test pins the contract: our functions `copy()` and return. If someone later "optimizes" the copy away, this test catches it the same minute.

**Run and verify:** `pytest -v` — all green.

### Step 3 — Prove the quality gate fires

**What:** Create `tests/test_validate.py`:

```python
import pandas as pd
import pytest

from hr_pipeline.validate import DataQualityValidator


def make_employees(**overrides) -> pd.DataFrame:
    """Small helper: one valid employee row, tweakable per test."""
    base = {
        "employee_id": ["GT-000001"], "email": ["a@globaltech.com"],
        "employment_type": ["Full-Time"], "salary_usd_annual": [90_000.0],
    }
    base.update(overrides)
    return pd.DataFrame(base)


def test_valid_data_passes_all_checks():
    report = (DataQualityValidator(make_employees(), "t")
              .not_null("email").unique("employee_id")
              .in_set("employment_type", {"Full-Time", "Part-Time", "Contractor"})
              .report())
    assert (report["status"] == "PASS").all()


def test_null_email_fails_not_null_check():
    report = (DataQualityValidator(make_employees(email=[None]), "t")
              .not_null("email").report())
    assert report.loc[report["check"] == "not_null:email", "status"].iloc[0] == "FAIL"


def test_duplicate_ids_fail_unique_check():
    df = pd.concat([make_employees(), make_employees()], ignore_index=True)
    report = DataQualityValidator(df, "t").unique("employee_id").report()
    assert (report["status"] == "FAIL").any()
```

**Why:** the validator is your last line of defense — the code that decides whether data may be published. A bug here (a check that never fails) means garbage ships with a green checkmark, the worst possible outcome. So we test *both directions*: good data passes AND bad data fails. Testing only the happy path leaves your alarm untested.

The `make_employees` helper is a **test factory** — one place defines a valid record; each test overrides only what it's probing. When the schema grows a column, you update one function, not thirty tests.

### Step 4 — Integration test with `tmp_path`

**What:** Create `tests/test_integration.py`:

```python
"""Ingest -> clean -> validate on a tiny file, in a temp folder."""
import pandas as pd

from hr_pipeline import ingest, clean


def test_csv_roundtrip_through_cleaning(tmp_path):
    # Arrange: write a small raw file into an isolated temp folder
    raw = tmp_path / "hris.csv"
    raw.write_text(
        "employee_id,first_name,last_name,email,employment_type,hire_date\n"
        "1042,Ada,Lovelace,ada@globaltech.com,FT,2023-05-01\n"
        "7,Grace,Hopper,grace@globaltech.com,full time,2022-11-15\n",
        encoding="utf-8",
    )

    # Act: run the real stages
    df = ingest.ingest_hris_csv(raw)
    df = clean.namespace_ids(df, "GT")
    df["employment_type"] = clean.standardize_employment_type(df["employment_type"])

    # Assert: end state is what the next stage expects
    assert list(df["employee_id"]) == ["GT-001042", "GT-000007"]
    assert set(df["employment_type"]) == {"Full-Time"}
    assert str(df["hire_date"].dtype) == "datetime64[ns]"
```

**Why:** unit tests prove each part; integration tests prove the *joints* — that ingest's output shape is what clean expects. Most real breakage lives in the joints (a renamed column, a dtype change). `tmp_path` keeps the test hermetic: no dependence on your real `data/raw`, no cleanup code, runs identically on any machine — which is what makes it CI-ready.

**Run the whole suite:**

```powershell
pytest -v
```

**Expected output:** every test green, total runtime well under 10 seconds. That speed is a feature: fast suites get run; slow ones get skipped.

### Step 5 — Break something on purpose

**What:** In `clean.py`, "accidentally" change the zero-padding from 6 to 5 (`zfill(5)`). Run `pytest`. Watch `test_prefix_and_zero_padding` fail with a precise diff. Revert. Green again.

**Why:** this rehearsal is the entire value proposition of testing compressed into one minute — a change that would have silently corrupted every downstream join was caught in seconds, named by a test whose title told you exactly what broke. Also verify your suite *can* fail; a suite that passes when the code is wrong is worse than no suite.

### Step 6 — Package for handover

**What:** Final packaging checklist — make the `pipeline/` folder deliverable:

1. **`requirements.txt`** — pin what the pipeline needs:

   ```powershell
   pip freeze | Select-String "pandas|openpyxl|lxml|pyarrow|rapidfuzz|pyyaml|python-dotenv|pytest" > requirements.txt
   ```

   A new machine then reproduces your environment with `pip install -r requirements.txt`. Without this file, "works on my machine" is the best anyone can say.

2. **README.md** — update Lab 04's README with: prerequisites (Python version), setup (`python -m venv`, activate, `pip install -r requirements.txt`, copy `.env.example` → `.env`), how to run, **how to run the tests**, outputs and their consumers, known limitations.

3. **Structure sweep** — the deliverable now looks like:

   ```
   pipeline/
   ├── hr_pipeline/          # source package
   ├── tests/                # test suite
   ├── config/pipeline.yaml  # behavior knobs
   ├── data/raw/             # inputs (immutable)
   ├── logs/                 # run history (gitignored)
   ├── output/               # deliverables (gitignored)
   ├── run_pipeline.py       # entry point
   ├── requirements.txt      # reproducible environment
   ├── .env.example          # documented secrets (committed)
   ├── .gitignore
   └── README.md
   ```

4. **The clean-machine test (thought experiment made real):** delete your venv (`Remove-Item -Recurse .venv` from the module root — it's fully regenerable), recreate it, `pip install -r requirements.txt`, run `pytest`, run the pipeline. If all three succeed, your handover is real. This is exactly what the client's engineer will do on day one.

**Verify:** all tests pass and the pipeline runs from the freshly rebuilt environment. That's the definition of done.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Unit tests on pure transforms, tiny hand-made inputs** | Regressions caught in milliseconds, failures self-explanatory. |
| **Testing the alarm (validator fails on bad data)** | An untested quality gate is decoration; test both directions. |
| **Non-mutation tests** | Pins the "copy, don't mutate" contract that prevents pandas' nastiest bug class. |
| **Test factories** | Schema changes touch one helper, not every test. |
| **Hermetic integration tests via `tmp_path`** | No shared state, no cleanup, CI-ready. |
| **Break-it rehearsal** | Proves the suite can fail — and shows you its value viscerally. |
| **`requirements.txt` + clean-machine rebuild** | Reproducibility: the difference between a deliverable and a demo. |
| **Tests documented in the README** | Testing evidence is a contractual deliverable in client work. |

---

## 6. Reflection

### What you learned
The data engineering testing pyramid; Arrange–Act–Assert; unit tests for transforms, both-direction tests for the quality gate, hermetic integration tests; and packaging (requirements, README, clean-machine verification) that makes the pipeline a genuine handover artifact.

### Why it matters
Tests are what let code change safely for years, and testing evidence is what clients and employers demand before accepting delivery. You now have both — plus a complete, production-shaped pipeline: modular, zoned, logged, configured, secret-safe, retrying, idempotent, validated, tested, documented. That is the exact standard the capstone will hold you to.

### Interview questions (with model answers)

1. **"How do you test a data pipeline?"**
   Two complementary layers: *code tests* (pytest — units on transformation functions, integration across stages with temp dirs, run per change) and *data quality checks* (rule-based validation with a gate, run on every pipeline execution). Code tests catch regressions; data checks catch bad inputs.

2. **"What makes a good unit test?"**
   Tests one behavior, on tiny constructed input, with a name that states the expectation, independent of other tests and external state, and fast enough that nobody hesitates to run the suite.

3. **"How do you test error paths?"**
   Deliberately construct the failure — null columns, duplicate IDs, malformed values — and assert the system responds as designed (check FAILs, exception raised, record quarantined). Also `pytest.raises` for exception contracts.

4. **"A bug made it to production. What do you do after fixing it?"**
   Write a regression test that reproduces the bug and fails on the old code, so it can never silently return. Over time the suite becomes a museum of everything that ever went wrong — and a guarantee none of it recurs.

5. **"What's in your requirements.txt and why pin versions?"**
   Direct dependencies with versions. Pinning makes environments reproducible — without it, a fresh install six months later pulls newer libraries and behavior drifts. (Mention lock files / `pip-tools` as the next maturity step.)

6. **"How would you know your validation logic itself is correct?"**
   Test it in both directions: known-good data must pass every check, and each specific defect must trip its specific check. The gate is code; code needs tests.

### Common interview traps
- "I test by running it and checking the output" — that's manual testing; the question is about automation.
- Conflating code tests with data quality checks — naming both, and when each runs, marks you as experienced.
- Claiming 100% coverage as the goal. Coverage of *critical transforms and the gate* is the goal; chasing a number produces junk tests.

### Key takeaways
1. Test the transforms, test the joints, and above all test the alarm.
2. Tiny hand-made inputs; one behavior per test; names that read as specs.
3. Every fixed bug becomes a regression test.
4. Reproducibility (requirements + README + clean-machine run) turns your project into a deliverable.

**Next:** you're ready for the [capstone project](../Project/01_Business_Scenario.md) — a full client engagement that uses everything from Labs 00–06.

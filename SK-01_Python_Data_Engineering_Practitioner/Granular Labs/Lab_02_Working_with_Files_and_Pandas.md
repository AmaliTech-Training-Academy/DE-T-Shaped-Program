# Lab 02 — Working with Files and Pandas: Ingesting Real-World Data

> **Module:** SK-01 Python Data Engineering Practitioner
> **Estimated time:** 4–6 hours
> **Prerequisites:** Lab 00 (environment setup) and Lab 01 (Python foundations) completed.

---

## 1. Environment Setup

You already installed Python and created a virtual environment in Lab 00. In this lab we add the libraries needed to read the four file formats you will meet constantly as a data engineer: **CSV, JSON, Excel, and XML**.

### 1.1 Activate your environment and install packages

Open PowerShell in your project folder:

```powershell
cd $HOME\Downloads\Upskilling\SK-01_Python_Data_Engineering_Practitioner
.\.venv\Scripts\Activate.ps1
pip install pandas openpyxl lxml pyarrow
```

What each package does:

| Package | Why we need it |
|---|---|
| `pandas` | The core data manipulation library. Think of it as "Excel in code" — tables you can program against. |
| `openpyxl` | Pandas needs this behind the scenes to read `.xlsx` Excel files. |
| `lxml` | A fast XML parser. Pandas and Python's standard library can use it to read XML. |
| `pyarrow` | Lets pandas read/write **Parquet**, the compressed columnar format used in real data platforms. |

### 1.2 Create the lab folder structure

```powershell
mkdir labs\lab02\data, labs\lab02\output -Force
```

A predictable folder structure matters in production: pipelines read from known input locations and write to known output locations. Guessing paths is how pipelines break at 2 a.m.

```
labs/lab02/
├── data/      # raw input files (never edited by hand)
└── output/    # everything your code produces
```

### 1.3 Verify setup works

Run this in PowerShell:

```powershell
python -c "import pandas, openpyxl, lxml, pyarrow; print('pandas', pandas.__version__, '- all imports OK')"
```

**Expected output:** something like `pandas 2.2.3 - all imports OK` (your version may differ — any 2.x is fine).

### 1.4 Common installation problems

| Symptom | Cause | Fix |
|---|---|---|
| `pip` not recognized | venv not activated | Run `.\.venv\Scripts\Activate.ps1`; you should see `(.venv)` in your prompt |
| `Activate.ps1 cannot be loaded` | PowerShell execution policy | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`, then retry |
| `ImportError: Missing optional dependency 'openpyxl'` | Installed outside the venv | Activate venv first, reinstall |
| Install extremely slow / times out | Corporate proxy | Ask IT for the proxy URL and use `pip install --proxy http://proxy:port ...` |

---

## 2. Business Context

**The problem:** Companies never have their data in one tidy place. A typical business runs a CRM (customer data as JSON from an API), an ERP (finance data as CSV exports), spreadsheets maintained by the finance team (Excel), and legacy systems that only speak XML. Before anyone can analyze anything, someone must **ingest** all of it — read it reliably into a common structure.

**Who does this?** You. Ingestion is the first stage of every data pipeline, and data engineers own it.

**Where you'll see it in industry:**
- A retailer pulling daily sales CSVs from 500 store point-of-sale systems.
- A bank receiving XML payment files from the SWIFT network.
- An HR consolidation project (exactly like this module's capstone) merging Workday CSV exports, a BambooHR JSON API, ADP Excel payroll files, and an XML benefits feed.

**Who consumes the output?** Analysts, data scientists, finance teams, regulators — everyone downstream. If ingestion is wrong, *everything* downstream is wrong.

**What happens if it fails?** Reports show stale or missing data, dashboards go blank, month-end close is delayed, and in regulated industries you can miss legal filing deadlines. Ingestion failures are the most common cause of data incidents — which is why we practice doing it properly.

---

## 3. Concept Explanation

### 3.1 What is a DataFrame?

A **DataFrame** is pandas' core object: a table with named columns and typed values, like a spreadsheet held in memory. Every file format we read today ends up as a DataFrame, which gives us one consistent way to inspect, clean, and transform data regardless of where it came from.

**Why does it exist?** Before pandas, Python programmers processed tabular data with nested lists and dictionaries — slow to write and easy to get wrong. Pandas gives you vectorized operations (operating on whole columns at once), which is both faster and more readable.

**Alternatives and when to use them:**

| Tool | Best for | Trade-off |
|---|---|---|
| pandas | Datasets that fit in one machine's memory (up to a few GB) | Single machine only |
| Polars | Same niche, faster, more modern API | Smaller ecosystem |
| PySpark | Datasets too big for one machine (covered in SK-03) | Heavy setup, cluster needed |
| Plain `csv` module | Tiny scripts, no dependencies | You write all the logic yourself |

For this module (files under ~1 GB, local machine), pandas is the industry-standard choice.

### 3.2 The four formats you will read

| Format | Structure | Typical source | Gotchas |
|---|---|---|---|
| **CSV** | Flat rows of text separated by commas | System exports, legacy tools | Encodings, delimiters, quotes, no types |
| **JSON** | Nested objects and arrays | APIs, web services | Nesting must be flattened into columns |
| **Excel** | Sheets, formatting, formulas | Humans! Finance teams | Multiple sheets, merged cells, header rows |
| **XML** | Nested tagged elements | Banking, healthcare, government, B2B feeds | Verbose; requires parsing element trees |

A key mental model: **CSV and Excel are already tables; JSON and XML are trees.** Reading tables is easy. Reading trees means deciding how to flatten them into rows and columns — and that decision is yours, the engineer's.

---

## 4. Step-by-Step Implementation

We'll build one small ingestion script per format, then combine them. Copy the module's project data into your lab folder so we practice on realistic files:

```powershell
copy "Project\Data\*" "labs\lab02\data\"
dir labs\lab02\data
```

**Expected output:** four files — `globaltech_hris.csv`, `acquiredco_api.json`, `payroll_data.xlsx`, `benefits_enrollment.xml`.

### Step 1 — Read a CSV and *interrogate* it

**What:** Create `labs\lab02\read_csv_demo.py`:

```python
from pathlib import Path
import pandas as pd

DATA_DIR = Path(__file__).parent / "data"

df = pd.read_csv(DATA_DIR / "globaltech_hris.csv")

print("Shape (rows, columns):", df.shape)
print("\nColumn types:")
print(df.dtypes)
print("\nFirst 5 rows:")
print(df.head())
print("\nNull counts per column:")
print(df.isna().sum())
```

**Why:** `read_csv` is the single most-used function in data engineering. But reading is never enough — a professional immediately *interrogates* the data: how many rows? what types did pandas guess? where are the nulls? This habit catches problems before they poison your pipeline.

**Run it:**

```powershell
python labs\lab02\read_csv_demo.py
```

**Expected output:** a shape like `(15000, 10)`, a dtype listing, five sample rows, and null counts. Note that `hire_date` is read as `object` (text), not a date — pandas does **not** parse dates unless you ask.

**Verify success:** the shape's row count should match your expectations (thousands of employee rows). If you get `FileNotFoundError`, check Step 0's copy command ran.

**Common mistakes:**
- Using a hardcoded absolute path like `C:\Users\you\...` — breaks on any other machine. We use `Path(__file__).parent` so the script finds data relative to itself.
- Trusting inferred types. `employee_id` may be read as `int64` here, but in files where IDs have leading zeros ("00042"), integer parsing silently destroys them. When IDs matter, pass `dtype={"employee_id": str}`.

**Troubleshooting:** `UnicodeDecodeError` → the file isn't UTF-8. Try `pd.read_csv(path, encoding="latin-1")`. Encoding issues are extremely common with exports from older Windows systems.

### Step 2 — Parse dates and types explicitly

**What:** Update the read to be explicit:

```python
df = pd.read_csv(
    DATA_DIR / "globaltech_hris.csv",
    dtype={"employee_id": str, "manager_id": str},
    parse_dates=["hire_date"],
)
print(df.dtypes)
```

**Why:** *Explicit beats inferred.* In production, a pipeline that relies on pandas guessing types will eventually guess differently when the data changes (e.g., a column that was all integers gets one blank and becomes `float64`, turning `42` into `42.0`). Declaring types makes your pipeline deterministic — same input, same behavior, every run.

**Expected output:** `hire_date` now shows `datetime64[ns]`; the ID columns show `object` (string).

**Common mistake:** parsing dates *after* loading with `pd.to_datetime(df["hire_date"])` but forgetting `errors="coerce"`. One malformed date then crashes the whole load. `errors="coerce"` turns bad dates into `NaT` (Not-a-Time) which you can count and report instead of crashing.

### Step 3 — Read nested JSON and flatten it

**What:** Create `labs\lab02\read_json_demo.py`:

```python
import json
from pathlib import Path
import pandas as pd

DATA_DIR = Path(__file__).parent / "data"

with open(DATA_DIR / "acquiredco_api.json", encoding="utf-8") as f:
    payload = json.load(f)

print("Top-level keys:", list(payload.keys()))
print("Records reported by API:", payload["total_records"])

# The employee list is nested — json_normalize flattens nested dicts into columns
df = pd.json_normalize(payload["employees"])
print("\nFlattened columns:", list(df.columns))
print(df.head(3))
```

**Why:** API responses wrap the real data in an envelope (`status`, `timestamp`, `total_records`, then the records). You must (a) navigate to the records, and (b) flatten nesting like `{"name": {"first": ...}}` into columns like `name.first`. `pd.json_normalize` does the flattening; the navigation is your judgment.

**Expected output:** columns such as `employee_identifier`, `name.first`, `name.last`, `contact.email`, `assignment.department`, `assignment.hire_timestamp`.

**Verify success:** compare `len(df)` to `payload["total_records"]`. This is your first **reconciliation check** — the source told you how many records to expect, so confirm you got them all. Real pipelines log this comparison on every run.

**Common mistakes:**
- `pd.read_json(path)` directly on an envelope-wrapped file — it errors or produces a one-row mess. Read the JSON manually first, then normalize the records list.
- Forgetting that flattened column names contain dots (`name.first`), which can be awkward later; many teams immediately rename them: `df.columns = [c.replace(".", "_") for c in df.columns]`.

### Step 4 — Read Excel (the human factor)

**What:** Create `labs\lab02\read_excel_demo.py`:

```python
from pathlib import Path
import pandas as pd

DATA_DIR = Path(__file__).parent / "data"
xlsx = DATA_DIR / "payroll_data.xlsx"

# Always inspect the workbook before reading blindly
sheets = pd.ExcelFile(xlsx).sheet_names
print("Sheets found:", sheets)

df = pd.read_excel(xlsx, sheet_name=sheets[0])
print("Shape:", df.shape)
print(df.head())
print(df.dtypes)
```

**Why:** Excel files are made *by humans for humans*: multiple sheets, title rows above the header, totals at the bottom, merged cells. Listing sheet names first, then reading deliberately, avoids silently reading the wrong sheet. If a file has junk rows above the header, use `skiprows=N`; if it has trailing totals, filter them out after loading.

**Expected output:** the sheet list and a payroll table (employee IDs, salary figures, currency, pay frequency).

**Common mistakes:**
- Reading `sheet_name=None` returns a *dictionary* of DataFrames (one per sheet), not a DataFrame — a classic confusion.
- Currency columns like `"$85,000"` load as text. Don't convert yet — we do cleaning properly in Lab 03. For now just *notice* it: `df.dtypes` showing `object` for a salary column is a red flag to record.

### Step 5 — Parse XML

**What:** Create `labs\lab02\read_xml_demo.py`:

```python
from pathlib import Path
import xml.etree.ElementTree as ET
import pandas as pd

DATA_DIR = Path(__file__).parent / "data"

tree = ET.parse(DATA_DIR / "benefits_enrollment.xml")
root = tree.getroot()
print("Root tag:", root.tag, "| attributes:", root.attrib)

records = []
for enrollment in root.findall("enrollment"):
    records.append({child.tag: child.text for child in enrollment})

df = pd.DataFrame(records)
print("Shape:", df.shape)
print(df.head())
```

**Why:** XML is a tree of tagged elements. The pattern is always the same: parse the tree, find the repeating element (here `<enrollment>`), turn each one into a dictionary, and build a DataFrame from the list. Note the root element's attributes advertise `total_records` — another reconciliation check for free.

**Expected output:** a DataFrame with columns `employee_id`, `plan_type`, `coverage_level`, `enrollment_date`, `premium_employee`, `premium_employer`.

**Verify success:** `df.shape[0]` should equal the `total_records` attribute on the root: `int(root.attrib["total_records"])`.

**Common mistakes:**
- Everything from XML arrives as **text** — premiums are strings like `"54.5"`. Convert numerics explicitly: `df["premium_employee"] = pd.to_numeric(df["premium_employee"], errors="coerce")`.
- Real-world XML often uses namespaces (`<ns:enrollment>`); `findall` then needs the namespace map. Ours doesn't, but expect it in banking/health feeds.

### Step 6 — Combine into one ingestion module and write Parquet

**What:** Create `labs\lab02\ingest.py` that wraps each reader in a function and writes outputs:

```python
"""Ingestion module: reads all four HR sources into standardized DataFrames."""
from pathlib import Path
import json
import xml.etree.ElementTree as ET
import pandas as pd

DATA_DIR = Path(__file__).parent / "data"
OUTPUT_DIR = Path(__file__).parent / "output"


def ingest_hris_csv(path: Path) -> pd.DataFrame:
    """GlobalTech HRIS export (CSV)."""
    return pd.read_csv(
        path,
        dtype={"employee_id": str, "manager_id": str},
        parse_dates=["hire_date"],
    )


def ingest_acquiredco_json(path: Path) -> pd.DataFrame:
    """AcquiredCo API dump (nested JSON)."""
    with open(path, encoding="utf-8") as f:
        payload = json.load(f)
    df = pd.json_normalize(payload["employees"])
    df.columns = [c.replace(".", "_") for c in df.columns]
    expected = payload["total_records"]
    if len(df) != expected:
        raise ValueError(f"Record count mismatch: got {len(df)}, API said {expected}")
    return df


def ingest_payroll_excel(path: Path) -> pd.DataFrame:
    """Combined payroll export (Excel)."""
    return pd.read_excel(path, sheet_name=0)


def ingest_benefits_xml(path: Path) -> pd.DataFrame:
    """Benefits provider feed (XML)."""
    root = ET.parse(path).getroot()
    records = [{c.tag: c.text for c in e} for e in root.findall("enrollment")]
    df = pd.DataFrame(records)
    for col in ("premium_employee", "premium_employer"):
        df[col] = pd.to_numeric(df[col], errors="coerce")
    return df


def main() -> None:
    OUTPUT_DIR.mkdir(exist_ok=True)
    sources = {
        "hris": ingest_hris_csv(DATA_DIR / "globaltech_hris.csv"),
        "acquiredco": ingest_acquiredco_json(DATA_DIR / "acquiredco_api.json"),
        "payroll": ingest_payroll_excel(DATA_DIR / "payroll_data.xlsx"),
        "benefits": ingest_benefits_xml(DATA_DIR / "benefits_enrollment.xml"),
    }
    for name, df in sources.items():
        out = OUTPUT_DIR / f"{name}.parquet"
        df.to_parquet(out, index=False)
        print(f"{name:12s} {df.shape[0]:>7,} rows, {df.shape[1]:>2} cols -> {out.name}")


if __name__ == "__main__":
    main()
```

**Why each design choice matters:**
- **One function per source** — each source can change independently; each function can be tested independently (Lab 06 does exactly that).
- **Reconciliation check raises an error** — a pipeline that ingests 2,900 of 3,200 records and continues silently is worse than one that stops loudly. *Fail fast, fail loud.*
- **Parquet output** — Parquet stores column types (dates stay dates), compresses ~5–10× smaller than CSV, and is the lingua franca of data platforms. Writing your standardized layer to Parquet is exactly what production lakes do.
- **`if __name__ == "__main__"`** — allows the file to be *imported* (for tests) without running everything.

**Run and verify:**

```powershell
python labs\lab02\ingest.py
dir labs\lab02\output
```

**Expected output:** one summary line per source and four `.parquet` files in `output/`.

**Prove Parquet round-trips faithfully:**

```powershell
python -c "import pandas as pd; df = pd.read_parquet('labs/lab02/output/hris.parquet'); print(df.dtypes)"
```

`hire_date` should still be `datetime64[ns]` — unlike CSV, Parquet remembers types.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Where you used it | Why it matters |
|---|---|---|
| **Explicit schemas** | `dtype=` and `parse_dates=` in every reader | Pipelines must be deterministic; inferred types drift as data changes. A real bank lost leading zeros on account numbers this way. |
| **Reconciliation checks** | Comparing loaded counts to source-declared counts | Detects truncated files and partial API responses — the #1 silent data loss mode. |
| **Fail fast** | Raising `ValueError` on count mismatch | A loud failure at ingestion costs minutes; a silent one discovered in a board report costs credibility. |
| **Relative paths via `pathlib`** | `Path(__file__).parent` | Code runs identically on your laptop, a teammate's, and a server. |
| **Separation of raw and output** | `data/` vs `output/` | Raw data is immutable evidence. Never overwrite inputs — you can't debug what you destroyed. |
| **Columnar output format** | Parquet | Type preservation, compression, and speed. Industry default for the "cleaned" layer. |
| **Modular functions** | One ingest function per source | Testability and independent evolution — the foundation for Lab 04's full pipeline. |

---

## 6. Reflection

### What you learned
You can now ingest the four dominant file formats into pandas, interrogate what you loaded, enforce explicit types, reconcile record counts against source claims, and persist a typed, compressed standardized layer in Parquet.

### Why it matters
Ingestion is the front door of every data platform. Engineers who read files "optimistically" ship pipelines that silently corrupt data; engineers who interrogate, declare, and reconcile ship pipelines people trust.

### Interview questions (with model answers)

1. **"What's the difference between CSV and Parquet, and when do you use each?"**
   CSV is human-readable text with no type information; Parquet is a compressed, columnar, typed binary format. Use CSV at system boundaries (humans, legacy exports); use Parquet for anything machines read downstream — it's smaller, faster, and preserves schema.

2. **"How do you handle a CSV where pandas infers the wrong types?"**
   Pass explicit `dtype=` and `parse_dates=` arguments. For messy values, load as string and convert with `pd.to_numeric`/`pd.to_datetime` using `errors="coerce"`, then count and report the coerced nulls.

3. **"An API returns paginated JSON. How do you make sure you ingested everything?"**
   Loop pages until the API signals the end, accumulate records, then reconcile the total against any count the API declares (and/or against yesterday's volume). Log the counts on every run.

4. **"Why shouldn't your pipeline overwrite its input files?"**
   Raw inputs are your audit trail and your only means of reprocessing after a bug. Treat raw as immutable; write outputs elsewhere.

5. **"What is `errors='coerce'` and what's the risk of using it?"**
   It converts unparseable values to `NaN`/`NaT` instead of raising. The risk is silently swallowing bad data — so always pair it with a count of how many values were coerced, and alert if that count is abnormal.

6. **"How would you read a 50 GB CSV that doesn't fit in memory?"**
   Stream it in chunks (`pd.read_csv(chunksize=...)`), processing and writing each chunk; or use a tool built for scale — Polars lazy mode, DuckDB, or Spark (SK-03).

### Common interview traps
- Saying "pandas handles types automatically" — interviewers want to hear *explicit schemas* and *determinism*.
- Forgetting encodings exist. Mentioning `UnicodeDecodeError` and `encoding="latin-1"` signals real-world experience.
- Treating Excel as "just another CSV" — mention sheets, header offsets, and human-created chaos.

### Key takeaways
1. Every format becomes a DataFrame — but *how* it becomes one is your responsibility.
2. Explicit beats inferred: declare types, parse dates deliberately.
3. Reconcile counts on every load; fail loudly on mismatch.
4. Raw data is immutable; outputs are typed Parquet.
5. One function per source = testable, maintainable ingestion.

**Next:** [Lab 03](Lab_03_Data_Cleaning_and_Validation.md) — the data you just loaded is full of problems (mixed currencies, inconsistent dates, duplicates). Now we clean it, properly.

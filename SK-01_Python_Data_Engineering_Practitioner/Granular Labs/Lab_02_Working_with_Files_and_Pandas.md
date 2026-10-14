# Lab 02 — Working with Files and pandas

> **Goal:** Start the ShopStream pipeline. You create `pipeline.py`, generate the 4 source files, and load each one (a CSV, a nested JSON, a headerless fixed-width text file and an Excel workbook) into a pandas DataFrame. Then you combine all 2,000 records into one table with a single, consistent set of columns.
>
> **Time:** 3–4 hours | **Difficulty:** Beginner | **Prerequisites:** [Lab 01](Lab_01_Python_Foundations_for_Data_Engineering.md) completed; venv active in `C:\DataEngineering\sk01`.
>
> **Guided lab coverage:** [SK-01 End-to-End Guided Lab](../Labs/SK-01_End_to_End_Guided_Lab.md) **Part 1** (Steps 1.1–1.2: setup and test data) and **Part 2** (Steps 2.1–2.5: ingestion).

---

## The business problem

ShopStream's customer data lives in 4 systems that have never been joined up:

| Source | File in `data\raw\` | Format and problems |
|---|---|---|
| Website | `website_customers.csv` | CSV in the old **ISO-8859-1** encoding, spaces in column names, 20 internal test accounts |
| CRM | `crm_export.json` | **Nested JSON** (`profile.first_name`); the source of truth for opt-outs |
| ERP | `erp_customers.txt` | **Fixed-width** text with **no header row**: columns are character positions |
| Marketing | `marketing_engagement.xlsx` | **Excel**, `Unsubscribed` as `Y`/`N` instead of 0/1 |

Before anything can be cleaned, all 4 must be loaded *correctly*. Most bad data starts here: a wrong encoding garbles every accented name, and a phone number read as a number loses its `+` and leading zeros.

---

## Learning objectives

By the end of this lab you can:

1. Explain what a pandas DataFrame and Series are, and inspect one (`shape`, `head`, `dtypes`, `value_counts`, `isna`).
2. Read CSV, JSON, fixed-width and Excel files with the right options (`encoding`, `dtype`, `na_values`, `colspecs`, `sheet_name`).
3. Flatten nested JSON with `pd.json_normalize`.
4. Rename columns to one standard schema and combine sources with `pd.concat`.
5. Explain why Parquet is used for analytics instead of CSV.

---

## Key terms

- **DataFrame:** a table in memory: rows, named columns, and an **index** (row labels, shown on the left).
- **Series:** one column of a DataFrame. Most cleaning functions in the guided lab take a Series and return a Series.
- **dtype:** a column's data type: `int64`, `float64`, `bool`, `datetime64[ns]`, or `object` (usually text, or a mix of types).
- **NaN:** "not a number". This is how pandas shows a missing value (dates use `NaT`). `isna()` finds them.
- **Schema:** the agreed list of column names and types. The guided lab's is `STANDARD_SCHEMA`.

---

## Steps

### Step 1 — Start `pipeline.py` with the setup code

**What:** Create `C:\DataEngineering\sk01\pipeline.py` and paste in the code block from guided lab **Step 1.1** (from the docstring down to the `for d in [...]: d.mkdir(...)` loop).

Then run it:

```powershell
python pipeline.py
```

**Expected output:** nothing on screen. The script only defines settings and creates folders. Check that `logs\pipeline.log` (empty for now), `data\raw` and `data\processed` exist.

**Why**, part by part:

| Part | What it does |
|---|---|
| `import pandas as pd`, `numpy as np`, `re`, `json`, `requests`, `datetime`, `Path`, `logging` | Every library the pipeline uses. `pd` and `np` are the universal short names. |
| `logging.basicConfig(...)` | Sends every message to the screen **and** to `logs\pipeline.log`, with a timestamp and level. From now on the pipeline uses `logger.info(...)` instead of `print`. Lab 05 explains logging in detail. |
| `CONFIG = {...}` | All settings in one dictionary (Lab 01, Step 4): folders, the CRM URL, the email regex, the 95% quality threshold and source priorities. Functions read from `CONFIG` instead of hard-coding values. |
| `os.environ.get("CRM_API_KEY", "")` | The API key comes from an environment variable, never from code. Secrets in code end up in Git. More in Lab 05. |
| `d.mkdir(parents=True, exist_ok=True)` | Creates the data folders. It's safe to run again. |

✅ **Checkpoint:** `python pipeline.py` runs without errors and `logs\pipeline.log` exists.

### Step 2 — Generate the 4 source files

**What:** Paste the guided lab **Step 1.2** code block (`def generate_synthetic_data(): ...`) at the end of `pipeline.py`. Keep the final line `# generate_synthetic_data()` commented out. You'll call the function once from the command line instead:

```powershell
python -c "import pipeline; pipeline.generate_synthetic_data()"
```

**Expected output** (each line starts with a date and time, shown here as `<timestamp>`):

```
<timestamp> [INFO] Generated website CSV: 1020 records
<timestamp> [INFO] Generated CRM JSON: 500 records
<timestamp> [INFO] Generated ERP fixed-width: 250 records
<timestamp> [INFO] Generated marketing Excel: 250 records
<timestamp> [INFO] Synthetic data generation complete.
```

`python -c "import pipeline; ..."` *imports* your file as a module and calls one function. Importing runs the top-level setup code (logging, `CONFIG`, folders) but not any function unless you call it.

**Why the generator matters:** it builds realistic, deliberately messy data. Read through it and notice where each audit problem comes from:
- `np.random.seed(42)` makes the "random" data **the same on every machine**. That's why your counts will match the expected outputs in these labs exactly.
- `emails_pool` holds 800 real-looking emails plus 5 bad ones (`"not-an-email"`, `"missing@"`, `""`, ...). Each source picks emails from the same pool, so the **same customer appears in several sources**. Those are the duplicates you'll remove in Lab 03.
- `regions_messy` has 15 spellings (`"us"`, `"USA"`, `"EMEA"`, `"Asia Pacific"`, ...) plus blanks; `phones_messy` has 7 formats plus `None`.
- The website file is saved with `encoding="iso-8859-1"`, as a legacy system would. The ERP lines are padded to exact widths with f-strings like `f"{name:<50}"` (Lab 01, Step 1).

Look at the raw files:

```powershell
Get-Content data\raw\website_customers.csv -TotalCount 3
Get-Content data\raw\erp_customers.txt -TotalCount 2
```

**Expected output:**

```
CustomerEmail,First Name,Last Name,Phone,Region,Registration Date,OptOut
customer92@company.com,Jane,Díaz,+1 (555) 123-4567,North America,2020-01-01,0
customer620@company.com,Sarah,Martínez,+44 20 7946 0958,us,2020-01-01,0
         0Maria Smith                                       customer606@gmail.com                                       +1 (555) 123-4567   North America  2019-03-01ACTIV
         1José Johnson                                      customer455@company.com                                     +44 20 7946 0958    us             2019-12-01ACTIV
```

Notice the ERP file has no header and no commas. Without the system's specification, you couldn't tell where the email ends and the phone begins.

> In **PowerShell 7** (`pwsh`) the CSV shows `D�az` instead of `Díaz`, because PowerShell 7 assumes UTF-8 and the file is ISO-8859-1. That's the encoding problem in action. Windows PowerShell 5.1 happens to display it correctly.

✅ **Checkpoint:** `data\raw` contains the 4 files.

### Step 3 — First look with pandas, before writing ingestion code

**What:** Explore the website file to see what `ingest_website_csv` has to deal with. Professionals always look at the data before writing code for it.

Create `practice\lab02_first_look.py`:

```python
"""Lab 02 - first look at the website CSV, before writing any ingestion code."""
import pandas as pd

pd.set_option("display.width", 120)          # show more columns per line
pd.set_option("display.max_columns", 10)

path = "data/raw/website_customers.csv"

try:
    pd.read_csv(path)                      # pandas assumes UTF-8 by default...
except UnicodeDecodeError as e:
    print("UTF-8 failed:", e)

df = pd.read_csv(path, encoding="iso-8859-1")
print(df.shape)
print(df.head(3))
print(df.dtypes)
print(df["Region"].value_counts(dropna=False).head(8))
print("test accounts:", df["CustomerEmail"].str.endswith("@test.shopstream.com").sum())
```

```powershell
python practice\lab02_first_look.py
```

**Expected output:**

```
UTF-8 failed: 'utf-8' codec can't decode byte 0xed in position 103: invalid continuation byte
(1020, 7)
             CustomerEmail First Name Last Name              Phone         Region Registration Date  OptOut
0   customer92@company.com       Jane      Díaz  +1 (555) 123-4567  North America        2020-01-01       0
1  customer620@company.com      Sarah  Martínez   +44 20 7946 0958             us        2020-01-01       0
2  customer752@company.com      André       Kim   +44 20 7946 0958            NaN        2020-01-01       0
CustomerEmail        object
First Name           object
Last Name            object
Phone                object
Region               object
Registration Date    object
OptOut                int64
dtype: object
Region
NaN              111
US                92
us                71
united states     71
eu                69
Europe            67
europe            66
EMEA              66
Name: count, dtype: int64
test accounts: 20
```

**What this tells you:**
- **Encoding:** byte `0xed` is `í` in ISO-8859-1 (the `í` in `Díaz`), but it isn't valid UTF-8. So reading needs `encoding="iso-8859-1"`. Never "fix" this by deleting the bad characters: that turns `Díaz` into `Daz`.
- **`shape` (1020, 7):** 1,000 customers plus 20 test accounts.
- **`Registration Date` is `object` (text), not a date.** Without `parse_dates`, sorting and date-range checks won't work properly.
- **`Phone` is `object` here** only because some values contain `+`. A file of digits-only phones would be read as numbers: `05551234567` would lose its leading zero and become `5551234567.0`. That's why the guided lab forces `dtype={"Phone": str}`.
- **`value_counts(dropna=False)`** shows US written as `US`, `us`, `united states` and more, and 111 blanks. You'll fix this in Lab 03.
- `df.head(3)`, `df.dtypes` and `value_counts()` are the first 3 commands to run on any new file.

✅ **Checkpoint:** you can name 4 problems in this file without looking at the guided lab.

### Step 4 — Add the website ingestion function

**What:** Paste guided lab **Step 2.1** (`def ingest_website_csv(...)`) at the end of `pipeline.py`.

**Why** each `read_csv` option is there:

| Option | Problem it solves |
|---|---|
| `encoding="iso-8859-1"` | The `UnicodeDecodeError` you just saw |
| `dtype={"Phone": str}` | Phone numbers must stay text: keep `+` and leading zeros |
| `parse_dates=["Registration Date"]` | Turns text into real `datetime64` dates |
| `na_values=["", "N/A", "null", ...]` | Treats every "empty" spelling as `NaN`, so missing really means missing |

Then the function:
1. **Renames columns to snake_case.** `.str.lower()`, then `.str.replace(r"[^\w]", "_", regex=True)` turns `First Name` into `first_name` (regex from Lab 01). `rename` maps the rest (`customeremail` → `email`, `optout` → `opt_out`).
2. **Removes test accounts** with a **boolean mask**. `test_mask` is a Series of True/False, one value per row. `df[~test_mask]` keeps the rows where it is False (`~` means "not"). `.copy()` makes an independent DataFrame, so later changes don't affect the original.
3. **Tags each row** with `df["source"] = "website"`. Lab 03's deduplication needs to know where each row came from.

### Step 5 — Add the CRM, ERP and marketing ingestion functions

**What:** Paste guided lab **Steps 2.2, 2.3 and 2.4** at the end of `pipeline.py`: `ingest_crm_json`, `ingest_crm_api`, `ingest_erp_fixed_width` and `ingest_marketing_excel`.

**Why:**

- **CRM, `pd.json_normalize(raw["customers"], sep="_")`:** each customer is a dict with a nested `profile` dict (you saw this shape in Lab 01, Step 8). `json_normalize` flattens it into columns, so `profile.first_name` becomes `profile_first_name`, which is then renamed to `first_name`. `ingest_crm_api` does the same for the live API, one page at a time. You'll test it without a real server in Lab 05.
- **ERP, `pd.read_fwf(colspecs=..., names=..., dtype=str)`:** `colspecs` lists each column's character positions, `(start, end)`, from the ERP specification document. `names` supplies the missing header. `dtype=str` reads everything as text first. Values are padded with spaces, so every column is `.str.strip()`ped. `full_name` is split once on the first space (`str.split(n=1, expand=True)`) into `first_name` and `last_name`.
- **Marketing, `pd.read_excel(filepath, sheet_name="Engagement")`:** needs `openpyxl` (Lab 00). `Unsubscribed` uses `Y`/`N`. `.map({"Y": 1, "N": 0})` converts it to the same `opt_out` meaning the other sources use. Without this, a customer who unsubscribed in the marketing tool could still be emailed.

Now check all 4 functions. Create `practice\lab02_check_ingest.py`:

```python
"""Lab 02 - call each ingestion function and inspect what it returns."""
import pandas as pd
from pipeline import (CONFIG, ingest_website_csv, ingest_crm_json,
                      ingest_erp_fixed_width, ingest_marketing_excel)

pd.set_option("display.width", 120)
pd.set_option("display.max_columns", 10)
raw = CONFIG["input_dir"]

web = ingest_website_csv(raw / "website_customers.csv")
print(web.columns.tolist())
print(web.dtypes["registration_date"], web.dtypes["phone"])

crm = ingest_crm_json(raw / "crm_export.json")
print(crm.columns.tolist())

erp = ingest_erp_fixed_width(raw / "erp_customers.txt")
print(erp[["customer_id", "first_name", "last_name", "email", "region"]].head(3))

mkt = ingest_marketing_excel(raw / "marketing_engagement.xlsx")
print(mkt[["email", "unsubscribed", "opt_out"]].head(3))
```

Run it **as a module** from the project root:

```powershell
python -m practice.lab02_check_ingest
```

> **Why `-m practice.lab02_check_ingest`?** The script imports `pipeline`, which is in the project root. With `python -m`, Python searches the current folder (the project root) for imports. With `python practice\lab02_check_ingest.py` it searches only `practice\`, and you'd get `ModuleNotFoundError: No module named 'pipeline'`. Use `-m` (no `.py`, dots instead of `\`) for every practice script that imports `pipeline`.

**Expected output** (`<timestamp>` stands for the date and time at the start of each log line):

```
<timestamp> [INFO] Ingesting website CSV: data\raw\website_customers.csv
<timestamp> [INFO]   Removed 20 test accounts
<timestamp> [INFO]   Ingested 1000 records from website CSV
['email', 'first_name', 'last_name', 'phone', 'region', 'registration_date', 'opt_out', 'source']
datetime64[ns] object
<timestamp> [INFO] Ingesting CRM JSON: data\raw\crm_export.json
<timestamp> [INFO]   Ingested 500 records from CRM JSON
['id', 'email', 'phone', 'region', 'registration_date', 'opt_out', 'lifetime_value', 'first_name', 'last_name', 'source']
<timestamp> [INFO] Ingesting ERP fixed-width: data\raw\erp_customers.txt
<timestamp> [INFO]   Ingested 250 records from ERP
  customer_id first_name last_name                    email         region
0           0      Maria     Smith    customer606@gmail.com  North America
1           1       José   Johnson  customer455@company.com             us
2           2      André  Williams    customer501@gmail.com            NaN
<timestamp> [INFO] Ingesting marketing Excel: data\raw\marketing_engagement.xlsx
<timestamp> [INFO]   Ingested 250 records from marketing Excel
                   email unsubscribed  opt_out
0  customer343@yahoo.com            N        0
1  customer276@gmail.com            N        0
2  customer672@gmail.com            N        0
```

Notice that the 4 sources now share names like `email` and `first_name`, but each still has extra columns (`id`, `lifetime_value`, `customer_id`, `unsubscribed`...). The next step lines them up.

✅ **Checkpoint:** 1000 website, 500 CRM, 250 ERP and 250 marketing records, and `José` is spelled correctly.

### Step 6 — Combine all sources into one schema

**What:** Paste guided lab **Step 2.5** (`STANDARD_SCHEMA`, `align_schema`, `ingest_all_sources`) at the end of `pipeline.py`.

**Why:**
- `STANDARD_SCHEMA` is the 8 columns every source must have. `align_schema` adds any missing column as `NaN` (the ERP file has no opt-out; marketing has no phone or date), drops extra columns, and puts the columns in the same order.
- `.astype(object)` gives every source the same generic column types, so `pd.concat` never has to guess a type from a column that is entirely empty in one source. Real types (dates, booleans) are set during cleaning in Lab 03.
- `pd.concat(frames, ignore_index=True)` stacks the 4 tables vertically and numbers the rows 0–1999. Without `ignore_index=True`, the row labels 0, 1, 2... would repeat once per source.

Create `practice\lab02_combined.py`:

```python
"""Lab 02 - combine the 4 sources and look at the result."""
import pandas as pd
from pipeline import CONFIG, ingest_all_sources, ingest_website_csv

pd.set_option("display.width", 120)
pd.set_option("display.max_columns", 10)

combined = ingest_all_sources()
print(combined.shape)
print(combined.groupby("source").size())
print(combined.isna().sum())
print(combined.sample(4, random_state=1)[["email", "region", "registration_date", "opt_out", "source"]])

# CSV vs Parquet: write the website data both ways and read it back
web = ingest_website_csv(CONFIG["input_dir"] / "website_customers.csv")
out = CONFIG["output_dir"]
web.to_csv(out / "website_sample.csv", index=False)
web.to_parquet(out / "website_sample.parquet", index=False)
for name in ["website_sample.csv", "website_sample.parquet"]:
    print(f"{name:<24} {(out / name).stat().st_size / 1024:6.1f} KB")
print(pd.read_csv(out / "website_sample.csv").dtypes["registration_date"])
print(pd.read_parquet(out / "website_sample.parquet").dtypes["registration_date"])
```

```powershell
python -m practice.lab02_combined
```

**Expected output** (log lines shortened to `<timestamp>`):

```
<timestamp> [INFO] ============================================================
<timestamp> [INFO] STEP 1: Data Ingestion
<timestamp> [INFO] Ingesting website CSV: data\raw\website_customers.csv
<timestamp> [INFO]   Removed 20 test accounts
<timestamp> [INFO]   Ingested 1000 records from website CSV
<timestamp> [INFO] Ingesting CRM JSON: data\raw\crm_export.json
<timestamp> [INFO]   Ingested 500 records from CRM JSON
<timestamp> [INFO] Ingesting ERP fixed-width: data\raw\erp_customers.txt
<timestamp> [INFO]   Ingested 250 records from ERP
<timestamp> [INFO] Ingesting marketing Excel: data\raw\marketing_engagement.xlsx
<timestamp> [INFO]   Ingested 250 records from marketing Excel
<timestamp> [INFO] Total records combined: 2000
<timestamp> [INFO]   website: 1000 records
<timestamp> [INFO]   crm: 500 records
<timestamp> [INFO]   erp: 250 records
<timestamp> [INFO]   marketing: 250 records
(2000, 8)
source
crm           500
erp           250
marketing     250
website      1000
dtype: int64
email                  2
first_name             0
last_name              0
phone                461
region               188
registration_date    250
opt_out              250
source                 0
dtype: int64
                        email         region    registration_date opt_out   source
674    customer44@company.com             eu  2020-04-22 00:00:00       0  website
1699    customer390@gmail.com         Europe  2019-05-01 00:00:00     NaN      erp
1282  customer329@company.com  North America  2022-10-01 00:00:00   False      crm
1315    customer256@yahoo.com  North America  2023-08-01 00:00:00   False      crm
<timestamp> [INFO] Ingesting website CSV: data\raw\website_customers.csv
<timestamp> [INFO]   Removed 20 test accounts
<timestamp> [INFO]   Ingested 1000 records from website CSV
website_sample.csv         74.5 KB
website_sample.parquet     14.5 KB
object
datetime64[ns]
```

**Read the output like an engineer:**
- **2,000 rows, 8 columns.** Every source now fits one schema.
- **`isna().sum()`** shows the gaps: 250 missing dates and 250 missing opt-outs. Those are exactly the marketing rows (no date) and the ERP rows (no opt-out). Missing values that line up with one source mean a *structural* gap, not bad data.
- **`opt_out` is a mix:** `0` (website), `False` (CRM) and `NaN` (ERP). One column, three meanings of "not opted out". Lab 03 turns this into a single True/False column.
- **CSV vs Parquet:** the same data is 74.5 KB as CSV and 14.5 KB as Parquet, and Parquet **keeps the column types**. The date comes back as `datetime64` from Parquet, but as text (`object`) from CSV. That's why the pipeline delivers `golden_customers.parquet` for analytics, plus a CSV only because the marketing team opens files in Excel.

✅ **Checkpoint:** 2,000 combined rows, and you can explain the 250 missing dates and 250 missing opt-outs.

> You can delete `data\processed\website_sample.*` now. They were only for the comparison.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `UnicodeDecodeError: 'utf-8' codec can't decode byte 0xed` | Reading the website CSV without its encoding | `encoding="iso-8859-1"` (Step 4) |
| Names show `JosÃ©` or `D�az` | Wrong encoding when reading or displaying | Read with the encoding the file was written in; write outputs as UTF-8 |
| `ModuleNotFoundError: No module named 'pipeline'` | Practice script run as `python practice\x.py` | Run from the project root with `python -m practice.x` |
| `ImportError: Missing optional dependency 'openpyxl'` | Excel support not installed | `python -m pip install openpyxl` (venv active) |
| `ImportError: Unable to find a usable engine ... pyarrow` | Parquet support not installed | `python -m pip install pyarrow` |
| `FileNotFoundError: data\raw\...` | Data not generated, or running from another folder | `cd C:\DataEngineering\sk01`; run Step 2 again |
| `KeyError: 'Registration Date'` in `read_csv` | Column name typo in `parse_dates` | Compare with `df.columns` from Step 3 |
| ERP names cut off or columns shifted | Wrong `colspecs` | Positions must match the spec exactly: `(10, 60)` is characters 10 to 59 |
| Your counts differ from 1000/500/250/250 | `generate_synthetic_data` was edited, or `seed(42)` removed | Re-copy Step 1.2 from the guided lab, delete `data\raw\*` and regenerate |

---

## Key takeaways

- Look before you code: `shape`, `head()`, `dtypes` and `value_counts(dropna=False)` on every new file.
- Read options are data-quality decisions: `encoding`, `dtype=str` for phone numbers and IDs, `na_values`, `parse_dates`.
- Each file format needs its own reader: `read_csv`, `json_normalize`, `read_fwf` (+ `colspecs`), `read_excel` (+ `sheet_name`).
- Map every source to one standard schema **before** combining, and tag each row with its `source`.
- Parquet keeps types and is much smaller. CSV is for people and Excel.

---

## Next

[Lab 03 — Data Cleaning and Validation](Lab_03_Data_Cleaning_and_Validation.md): the 2,000 rows have 15 region spellings, phone numbers in many formats, three kinds of opt-out and many duplicate customers. You'll clean them, merge duplicates and prove the quality with 7 checks.

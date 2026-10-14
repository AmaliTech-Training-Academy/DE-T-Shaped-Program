# Lab 04 — Building a Modular ETL Pipeline

> **Goal:** Turn your collection of functions into a pipeline that runs end to end with one command, `python pipeline.py`. It ingests, cleans, deduplicates, validates, draws a 6-chart quality report and delivers ShopStream's golden customer files. You'll also check the deliverables the way the marketing team will use them.
>
> **Time:** 3 hours | **Difficulty:** Intermediate | **Prerequisites:** [Lab 03](Lab_03_Data_Cleaning_and_Validation.md) completed: `pipeline.py` contains guided lab Parts 1–5.
>
> **Guided lab coverage:** [SK-01 End-to-End Guided Lab](../Labs/SK-01_End_to_End_Guided_Lab.md) **Part 7** (EDA visualisation) and **Part 8** (orchestration and delivery: `run_pipeline` and the `__main__` block).

---

## The business problem

So far you've run each stage by hand from practice scripts. The marketing team doesn't want scripts. They want 4 files, delivered the same way every time:

| Deliverable | For whom | Why this format |
|---|---|---|
| `golden_customers.parquet` | Analysts and downstream pipelines | Small, keeps types (Lab 02) |
| `golden_customers.csv` | Marketing team's email tool / Excel | Opens anywhere |
| `quality_report.csv` | Data governance | Evidence of how clean the data is |
| `customer_quality_report.png` | Managers, the campaign kick-off meeting | The data's health at a glance |

A pipeline that only runs when its author clicks through it isn't a pipeline. **ETL** (Extract, Transform, Load) means a repeatable sequence that someone, or a scheduler at 2 a.m., can start with one command.

---

## Learning objectives

By the end of this lab you can:

1. Explain the "modular" design: small single-purpose functions plus one orchestrator function.
2. Use `if __name__ == "__main__":` so a file can be both run and imported.
3. Build a multi-chart figure with matplotlib and save it as an image.
4. Export to Parquet and CSV with the right options (`index=False`, compression, `utf-8-sig`).
5. Check a pipeline's deliverables independently of the code that produced them.

---

## Key terms

- **Orchestrator:** the function that calls the stages in order and passes data between them. Here, `run_pipeline()`. It contains no cleaning logic of its own.
- **Modular:** each stage is its own function with clear inputs and outputs, so it can be tested, reused or replaced alone.
- **Entry point:** where execution starts when the file is run: the `if __name__ == "__main__":` block.
- **Deliverable:** an output someone else depends on. Its name, format and columns are a promise.

---

## Steps

### Step 1 — Add the EDA chart

**What:** Paste guided lab **Step 7.1** at the end of `pipeline.py`: the two imports (`matplotlib.pyplot as plt`, `seaborn as sns`) and `generate_eda_report`.

**How it's built:**
- `fig, axes = plt.subplots(2, 3, figsize=(18, 10))` creates one figure with a 2×3 grid of charts. `axes[row, col]` selects one; e.g. `axes[0, 0]` is top-left.
- `sns.set_theme(style="whitegrid")` applies seaborn's cleaner style to every matplotlib chart.
- Each chart is pandas plus one plotting call:

| Position | Chart | pandas behind it |
|---|---|---|
| `[0, 0]` | Customers by region (bar) | `df["region"].value_counts()` |
| `[0, 1]` | Monthly registrations (line) | `.set_index("registration_date").resample("ME").size()`: counts per month-end (`"ME"` needs pandas 2.2+) |
| `[0, 2]` | Records by source (pie) | `df["sources"].str.split(",").explode().value_counts()`: `"crm,website"` counts once for each source |
| `[1, 0]` | Field completeness (horizontal bar) | `df[fields].notna().mean()`: the share of non-missing values per column. Red if below 90%. |
| `[1, 1]` | Email validity (pie) | `df["email_valid"].value_counts()` |
| `[1, 2]` | Opt-out rate by region (bar) | `df.groupby("region")["opt_out"].mean()`: the mean of True/False *is* the rate |

- `plt.savefig(..., dpi=150, bbox_inches="tight")` writes the PNG; `plt.close()` frees the memory. A pipeline that runs every day must close its figures.

> The two `import` lines sit in the middle of the file because the guided lab adds code part by part. In your own projects, put all imports at the top.

### Step 2 — Add the orchestrator and entry point

**What:** Paste guided lab **Step 8.1** at the end of `pipeline.py`: `run_pipeline()` and the `if __name__ == "__main__":` block.

**Read `run_pipeline` top to bottom.** It's a to-do list, and every line calls a function you already know:

```
combined = ingest_all_sources()          # Lab 02
cleaned  = clean_dataframe(combined)     # Lab 03
deduped  = deduplicate_customers(cleaned)
# deduped = infer_region_with_llm(deduped)   (optional AI bonus, Lab 05)
quality_report = run_quality_checks(deduped)
generate_eda_report(deduped, ...)        # Step 1
to_parquet / to_csv                      # export
summary log lines                        # counts and duration
```

That's the modular design: **`run_pipeline` decides the order; the functions do the work.** Want to swap the CSV source for an API? Change one line. Want to test deduplication? Call that one function (Lab 06).

**The export options:**
- `index=False`: don't write pandas' row numbers as an extra column.
- `to_parquet(..., engine="pyarrow", compression="gzip")`: compressed Parquet.
- `to_csv(..., encoding="utf-8-sig")`: UTF-8 with a "byte order mark" at the start, so **Excel** shows `Martínez` correctly instead of `MartÃ­nez`.

**The entry point:**

```python
if __name__ == "__main__":
    if not (CONFIG["input_dir"] / "website_customers.csv").exists():
        generate_synthetic_data()      # first run only: create the 4 source files
    run_pipeline()
```

Python sets `__name__` to `"__main__"` only for the file you run directly. When a practice script or test does `import pipeline`, `__name__` is `"pipeline"`, so the pipeline does **not** start. That's why your Lab 02–03 practice scripts could import functions without triggering a full run.

✅ **Checkpoint:** `python -c "import pipeline; print('imported, nothing ran')"` prints only `imported, nothing ran`.

### Step 3 — Run the whole pipeline

```powershell
python pipeline.py
```

**Expected output** (takes 5–20 seconds; `<timestamp>` = date and time; your `Run started` time and `Duration` will differ):

```
<timestamp> [INFO] ============================================================
<timestamp> [INFO] SHOPSTREAM CUSTOMER DATA QUALITY PIPELINE
<timestamp> [INFO] Run started: 2026-10-14T06:16:17.679610
<timestamp> [INFO] ============================================================
<timestamp> [INFO] ============================================================
<timestamp> [INFO] STEP 1: Data Ingestion
  ... 15 ingestion lines, as in Lab 02 ...
<timestamp> [INFO]   marketing: 250 records
<timestamp> [INFO] STEP 2: Cleaning & Standardization
<timestamp> [INFO]   Invalid emails: 10
<timestamp> [INFO]   Null regions after standardization: 219
<timestamp> [INFO]   Records after cleaning: 2000
<timestamp> [INFO] STEP 3: Deduplication
<timestamp> [INFO]   Records before deduplication: 2000
<timestamp> [INFO]   Duplicate records removed: 1255
<timestamp> [INFO]   Records after deduplication: 745
<timestamp> [INFO] STEP 4: Quality Validation
<timestamp> [INFO] ============================================================
<timestamp> [INFO] DATA QUALITY REPORT
  ... the 7 checks, as in Lab 03 ...
<timestamp> [INFO]   OVERALL: ALL CHECKS PASSED ✓
<timestamp> [INFO] ============================================================
<timestamp> [INFO] STEP 5: Generating EDA visualization...
<timestamp> [INFO]   EDA report saved: data\processed\customer_quality_report.png
<timestamp> [INFO] STEP 6: Exporting Results
<timestamp> [INFO]   Parquet export: data\processed\golden_customers.parquet (18.2 KB)
<timestamp> [INFO]   CSV export: data\processed\golden_customers.csv
<timestamp> [INFO]   Quality report: data\processed\quality_report.csv
<timestamp> [INFO] ============================================================
<timestamp> [INFO] PIPELINE COMPLETE
<timestamp> [INFO]   Input records:          2,000
<timestamp> [INFO]   Output (golden) records:745
<timestamp> [INFO]   Duplicates removed:     1,255
<timestamp> [INFO]   Quality checks passed:  7/7
<timestamp> [INFO]   Duration:               12.7s
<timestamp> [INFO] ============================================================
```

> **Step numbers:** the log says `STEP 4: Quality Validation` while the code comment says `# Step 5`. The log counts the steps that ran; the optional AI step (commented out) is the code's step 4. Log messages are what operators read, so they should match what actually ran.

✅ **Checkpoint:** `PIPELINE COMPLETE` with 2,000 in, 745 out and 7/7 checks passed.

### Step 4 — Read the quality chart

Open `data\processed\customer_quality_report.png` (double-click it in File Explorer, or click it in VS Code). You should see 6 charts under the title *ShopStream Customer Data Quality Report*:

- **Customers by region:** EU 275, US 274, APAC 175. The 21 golden records with no region aren't shown. Would marketing want to see them?
- **Monthly registrations:** a big spike in early 2020. That's the test data's website rows (one every 4 hours from 2020-01-01), not a real marketing success. Always ask whether a spike is real or produced by the way the data was made.
- **Records by source:** website about 41%, CRM about 28%, ERP and marketing about 15% each.
- **Field completeness:** phone is about 87%, below the dashed 95% line, and drawn **red**. Phone isn't one of the 7 quality checks, so the pipeline passes, but an SMS campaign could not use this data.
- **Email validity:** about 98.7% valid.
- **Opt-out rate by region:** all 3 regions are above the dashed 20% line (27–29%). The test data gives each source about a 15% opt-out chance, and merging takes an opt-out from *any* source, so merged customers are opted out more often. This chart is the dedup rule made visible.

✅ **Checkpoint:** you can explain why phone completeness is red but the pipeline still passed.

### Step 5 — Check the deliverables as a consumer would

Your pipeline says it succeeded. Now check its outputs **without using its code**, as the marketing team will. Create `practice\lab04_outputs.py`:

```python
"""Lab 04 - inspect what the pipeline delivered."""
import pandas as pd

pd.set_option("display.width", 120)
pd.set_option("display.max_columns", 12)

golden = pd.read_parquet("data/processed/golden_customers.parquet")
print(golden.shape)
print(golden.dtypes)
print(golden["sources"].value_counts().head(5))

campaign = golden[golden["email_valid"] & ~golden["opt_out"] & golden["region"].notna()]
print("emailable customers:", len(campaign))
print(campaign.groupby("region").size())

report = pd.read_csv("data/processed/quality_report.csv")
print(report[["check", "pass_rate", "status"]])
```

```powershell
python practice\lab04_outputs.py
```

(It doesn't import `pipeline`, so it can run as a normal script.)

**Expected output:**

```
(745, 13)
email                        object
first_name                   object
last_name                    object
phone                        object
region                       object
registration_date    datetime64[ns]
opt_out                        bool
source                       object
email_raw                    object
email_valid                    bool
source_priority               int64
sources                      object
source_count                  int64
dtype: object
sources
website                  172
crm,website              144
crm                       62
website,erp               62
crm,website,marketing     55
Name: count, dtype: int64
emailable customers: 515
region
APAC    125
EU      192
US      198
dtype: int64
                           check  pass_rate status
0                NOT NULL: email     0.9946   PASS
1           NOT NULL: first_name     1.0000   PASS
2               NOT NULL: region     0.9718   PASS
3                  UNIQUE: email     0.9960   PASS
4                   REGEX: email     0.9919   PASS
5          VALUES IN SET: region     1.0000   PASS
6  DATE RANGE: registration_date     1.0000   PASS
```

**What this proves:**
- Parquet kept the real types: `registration_date` is a date, `opt_out` and `email_valid` are booleans. A consumer can filter on them directly.
- **515 customers can actually be emailed:** valid email, **not** opted out and with a region. That's the number marketing really needs, and it is much smaller than 745. Delivering 745 rows without the `opt_out` flag would have meant emailing 200+ people who said no.
- `sources` shows the cross-system overlap. `crm,website` (144) means customers known to both systems.
- Columns like `email_raw`, `source_priority` and `source_count` are for lineage and debugging. A future version might deliver a slimmer file for marketing (a reflection question in the guided lab).

✅ **Checkpoint:** you get 515 emailable customers and can explain each of the 3 filter conditions.

### Step 6 — Prove it runs from nothing

A pipeline is only reproducible if it runs from an empty state. Delete all generated data and outputs:

```powershell
Remove-Item data\raw\*, data\processed\*
python pipeline.py
```

**Expected:** the same output as Step 3, but 5 lines come first, because the entry point found no website CSV and generated the data:

```
<timestamp> [INFO] Generated website CSV: 1020 records
<timestamp> [INFO] Generated CRM JSON: 500 records
<timestamp> [INFO] Generated ERP fixed-width: 250 records
<timestamp> [INFO] Generated marketing Excel: 250 records
<timestamp> [INFO] Synthetic data generation complete.
<timestamp> [INFO] ============================================================
<timestamp> [INFO] SHOPSTREAM CUSTOMER DATA QUALITY PIPELINE
...
<timestamp> [INFO]   Output (golden) records:745
```

The counts are identical (745, 7/7) because the generator uses `seed(42)`.

✅ **Checkpoint:** a run from an empty `data\` folder gives exactly the same summary.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Nothing happens when running `python pipeline.py` | `if __name__ == "__main__":` block missing or mis-indented | It must start at column 0, at the very end of the file |
| `NameError: name 'generate_eda_report' is not defined` | A guided-lab part wasn't pasted | Check `pipeline.py` has every Step 1.1–5.1, 7.1 and 8.1 block |
| `ValueError: Invalid frequency: ME` | pandas older than 2.2 | `python -m pip install "pandas>=2.2,<3"` |
| `PermissionError` writing `golden_customers.csv` | The file is open in Excel | Close it in Excel and re-run |
| `ArrowTypeError` / `ArrowInvalid` in `to_parquet` | A column mixes types (e.g. `opt_out` not converted) | `clean_dataframe` must convert `opt_out` to bool (Lab 03, Step 2) |
| PNG is blank or squashed | `plt.tight_layout()` or `savefig` arguments changed | Re-copy Step 7.1 |
| `UserWarning: ... FigureCanvasAgg is non-interactive` | Something called `plt.show()` | Pipelines save figures; they never `show()` them |
| Excel shows `MartÃ­nez` | CSV written without the BOM | `encoding="utf-8-sig"` in `to_csv` |

---

## Key takeaways

- Modular = small functions that do the work + one orchestrator that decides the order.
- `if __name__ == "__main__":` lets the same file be run as a pipeline and imported by scripts and tests.
- Deliverables are promises: fixed names, a typed Parquet file for machines, CSV (`utf-8-sig`) for Excel users, and a quality report as evidence.
- Check outputs as a consumer would, without the producing code. "745 rows" and "515 emailable customers" are different answers.
- Re-run from an empty state to prove the pipeline is reproducible.

---

## Next

[Lab 05 — Logging, Error Handling and Idempotency](Lab_05_Logging_Error_Handling_and_Idempotency.md): what happens when a file is missing, the CRM API returns a 503, or someone runs the pipeline twice? You'll make the pipeline's behaviour in each case deliberate.

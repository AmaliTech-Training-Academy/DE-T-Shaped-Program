# Lab 03 — Data Cleaning and Validation

> **Goal:** Turn ShopStream's 2,000 combined rows into 745 trustworthy customer records. You standardise emails, phones, names, regions, dates and opt-outs; merge duplicate customers across the 4 sources so that an opt-out from *any* source wins; and prove the result with 7 quality checks.
>
> **Time:** 4–5 hours | **Difficulty:** Beginner–Intermediate | **Prerequisites:** [Lab 02](Lab_02_Working_with_Files_and_Pandas.md) completed: `pipeline.py` contains guided lab Parts 1–2 and `data\raw` has the 4 source files.
>
> **Guided lab coverage:** [SK-01 End-to-End Guided Lab](../Labs/SK-01_End_to_End_Guided_Lab.md) **Part 3** (cleaning), **Part 4** (deduplication) and **Part 5** (quality validation).

---

## The business problem

The audit said 23% of emails are invalid or duplicated, regions come in 15+ spellings, phone numbers come in many formats and 8% of records have no region. In Lab 02 you saw all of this in the combined table. The marketing campaign launches in 3 weeks. If this data goes out as is:

- Duplicate sends annoy customers and damage the brand.
- Unreachable addresses waste about $460K.
- **EU customers who opted out in one system get emailed anyway.** That is a GDPR violation, the most serious risk of all.

This lab is the core of the job. As the guided lab puts it: *"Ingestion is 10 lines; cleaning is 300."*

---

## Learning objectives

By the end of this lab you can:

1. Clean a whole column at once with vectorised `.str` methods, `.map()` and `.apply()`, and say when to use each.
2. Represent missing data consistently as `NaN`, and turn mixed flags (`0`/`1`, `True`/`False`, `NaN`) into one boolean column.
3. Deduplicate with `groupby` + a merge function, using source priority and a "most protective wins" rule for opt-outs.
4. Build a reusable validator class that reports pass rates against a threshold.
5. Read a quality report and explain every failure in it.

---

## Key terms

- **Vectorised operation:** one instruction applied to a whole column, e.g. `series.str.lower()`. It's faster and clearer than a Python loop over the rows.
- **Standardisation:** making equal things look equal (`"USA"`, `"us"`, `"North America"` → `"US"`).
- **Validation:** checking whether a value is acceptable (is it a valid email?). Standardise first, then validate.
- **Deduplication:** merging records that describe the same real customer into one **golden record**.
- **Source priority:** which system to trust when sources disagree. The guided lab uses CRM (1) > Website (2) > ERP (3) > Marketing (4).

---

## Steps

### Step 1 — Add the column-cleaning functions

**What:** Paste guided lab **Steps 3.1, 3.2 and 3.3** at the end of `pipeline.py`: `standardize_emails`, `validate_emails`, `standardize_phone_numbers`, `REGION_MAP`, `standardize_names`, `standardize_regions` and `clean_dataframe`.

Before using them on 2,000 rows, try them on a few values you can check by eye. Create `practice\lab03_cleaning.py`:

```python
"""Lab 03 - try each cleaning function on a handful of messy values."""
import numpy as np
import pandas as pd
from pipeline import (standardize_emails, validate_emails, standardize_phone_numbers,
                      standardize_names, standardize_regions)

emails = pd.Series(["  Maria@GMAIL.com ", "jo hn@yahoo.com", "missing@", "", None, np.nan])
clean = standardize_emails(emails)
print(pd.DataFrame({"raw": emails, "clean": clean, "valid": validate_emails(clean)}))

phones = pd.Series(["+1 (555) 123-4567", "555.123.4567", "+44 20 7946 0958", "invalid-phone", None])
print(pd.DataFrame({"raw": phones, "clean": standardize_phone_numbers(phones)}))

names = pd.Series(["  maria  ", "JOSÉ", "o'brien", "anne   marie", None])
print(pd.DataFrame({"raw": names, "clean": standardize_names(names)}))

regions = pd.Series(["usa", " Europe ", "EMEA", "Asia Pacific", "AP", "N/A", "", None])
print(pd.DataFrame({"raw": regions, "clean": standardize_regions(regions)}))
```

```powershell
python -m practice.lab03_cleaning
```

**Expected output:**

```
                  raw            clean  valid
0    Maria@GMAIL.com   maria@gmail.com   True
1     jo hn@yahoo.com   john@yahoo.com   True
2            missing@         missing@  False
3                                  NaN  False
4                None              NaN  False
5                 NaN              NaN  False
                 raw          clean
0  +1 (555) 123-4567   +15551234567
1       555.123.4567     5551234567
2   +44 20 7946 0958  +442079460958
3      invalid-phone            NaN
4               None            NaN
            raw       clean
0       maria         Maria
1          JOSÉ        José
2       o'brien     O'Brien
3  anne   marie  Anne Marie
4          None         NaN
            raw clean
0           usa    US
1       Europe     EU
2          EMEA    EU
3  Asia Pacific  APAC
4            AP  APAC
5           N/A   NaN
6                 NaN
7          None   NaN
```

**Why each function is written the way it is:**

| Function | Technique | Watch out for |
|---|---|---|
| `standardize_emails` | `.astype(str)` → `.str.strip()` → `.str.lower()` → remove internal spaces | `astype(str)` turns a missing value into the *text* `"nan"`, so the last line `.replace({"nan": np.nan, "none": np.nan, "": np.nan})` turns those back into real `NaN`. Without it, `"nan"` would be counted as an email. |
| `validate_emails` | `.str.match(regex, na=False)` | Validate **after** standardising. `na=False` means "missing = invalid" instead of returning `NaN`. |
| `standardize_phone_numbers` | `.apply(clean_phone)`: runs a normal Python function on each value | Use `.apply` when the logic needs `if` statements (keep the `+`, reject fewer than 7 digits). It's slower than `.str`, which is fine at this size. This is your Lab 01 `clean_phone`. |
| `standardize_names` | `\s+` → one space, `.str.title()` | `.title()` fails on some names: `"mcdonald"` → `"Mcdonald"`. Production systems keep a list of exceptions. |
| `standardize_regions` | lowercase, then `.map(REGION_MAP)` | Anything not in the map becomes `NaN`, so `"N/A"` is reported as missing instead of being guessed. Add new spellings to `REGION_MAP`, never to the code. |

✅ **Checkpoint:** you can explain why `"jo hn@yahoo.com"` becomes valid, but `"missing@"` stays invalid.

### Step 2 — Understand `clean_dataframe`

**What:** Read `clean_dataframe` line by line. It applies all the functions above to the combined table:

- `df = df.copy()`: never change the caller's DataFrame. A function that quietly changes its input causes bugs far away from where it was called.
- `df["email_raw"] = df["email"].copy()`: keep the **original** value next to the cleaned one. When someone asks "why was this customer rejected?", you can show them what arrived.
- `df["email_valid"] = validate_emails(df["email"])`: a **flag column** instead of deleting rows. Invalid emails stay visible and are counted in the report, not silently dropped.
- `df["opt_out"].astype(float).fillna(0).astype(bool)`: Lab 02 showed `opt_out` mixes `0`/`1` (website, marketing), `True`/`False` (CRM) and `NaN` (ERP). Converting to float makes all of them numbers (`True` → `1.0`). `fillna(0)` treats "no information" as "not opted out", and `astype(bool)` gives one True/False column.

> **Design question:** is `fillna(0)` (missing = *not* opted out) safe for GDPR? It's acceptable here because the ERP has no opt-out field at all, and the dedup step (next) takes an opt-out from *any* other source. If the ERP were a customer's only source, you'd want to discuss this with the compliance team.

### Step 3 — Add deduplication

**What:** Paste guided lab **Step 4.1** (`deduplicate_customers`) at the end of `pipeline.py`.

**How it works,** step by step:

1. **Rank the rows.** `df["source"].map(CONFIG["source_priority"])` gives CRM 1, website 2, ERP 3, marketing 4. Sorting by it puts the most trusted row first in every group.
2. **Split valid from invalid emails.** You can only match customers on an email that is real. The 10 invalid-email rows skip deduplication and are added back at the end, with `sources` set to their own source and `source_count = 1`.
3. **Group by email.** `groupby("email")` gathers all rows that share an email, and `.apply(merge_group, include_groups=False)` calls `merge_group` once per group. `include_groups=False` means the group passed in doesn't contain the `email` column (pandas 2.2+ warns otherwise), and `.reset_index()` brings `email` back as a normal column.
4. **Merge each group** (`merge_group`):
   - Start from the highest-priority row: `best = group.iloc[0].copy()`.
   - For phone, region, names and date: if `best` has a blank, take the first non-blank value from the other rows, in priority order.
   - **Opt-out:** `group["opt_out"].any()`. If *any* source says opted out, the golden record is opted out. This is the GDPR rule. Notice it ignores priority on purpose: the CRM is trusted most for names, but for consent, the most protective answer wins.
   - **Provenance:** `sources` (e.g. `crm,website,erp`) and `source_count` record where each golden record came from. When marketing asks "where did this customer come from?", you can answer.

### Step 4 — Add the quality validator

**What:** Paste guided lab **Step 5.1** (`class DataQualityValidator` and `run_quality_checks`) at the end of `pipeline.py`.

**Why a class?** The validator needs to keep state (the DataFrame, the threshold and the growing list of results) across several check calls. A class bundles that state with the methods that use it:

| Piece | Role |
|---|---|
| `__init__(self, df, threshold=0.95)` | Runs when you create `DataQualityValidator(df)`. Stores what every check needs. |
| `_record(...)` | Shared helper: works out `pass_rate = 1 - failed/total` and marks PASS if it is at least the threshold. The leading `_` means "internal, not for callers". |
| `check_not_null`, `check_unique`, `check_regex`, `check_values_in_set`, `check_date_range` | One reusable rule each. Notice that `check_unique`, `check_regex` and others first `dropna()`: a missing value is the not-null check's job, so it isn't counted twice. |
| `generate_report()` | Logs every result with ✓/✗ and an overall verdict, and returns the results as a DataFrame. |

`run_quality_checks` lists ShopStream's 7 rules. To reuse the validator on another project, keep the class and swap only these lines.

### Step 5 — Run clean → dedup → validate and follow one customer

Create `practice\lab03_dedup.py`:

```python
"""Lab 03 - clean, deduplicate and validate the combined data."""
import pandas as pd
from pipeline import ingest_all_sources, clean_dataframe, deduplicate_customers, run_quality_checks

pd.set_option("display.width", 120)
pd.set_option("display.max_columns", 10)

cleaned = clean_dataframe(ingest_all_sources())
print(cleaned["region"].value_counts(dropna=False))
print(cleaned["opt_out"].value_counts())

# How many rows share an email before deduplication?
valid = cleaned[cleaned["email_valid"]]
print("valid-email rows:", len(valid), "| distinct emails:", valid["email"].nunique())

# Follow one customer through deduplication
email = valid["email"].value_counts().index[0]
print(cleaned.loc[cleaned["email"] == email, ["email", "first_name", "phone", "region", "opt_out", "source"]])

deduped = deduplicate_customers(cleaned)
print(deduped.loc[deduped["email"] == email, ["email", "first_name", "phone", "region", "opt_out", "sources", "source_count"]])

report = run_quality_checks(deduped)
print(report[["check", "failed", "total", "pass_rate", "status"]])
```

```powershell
python -m practice.lab03_dedup
```

**Expected output** (the 16 ingestion log lines from Lab 02 come first and are not repeated here):

```
<timestamp> [INFO] STEP 2: Cleaning & Standardization
<timestamp> [INFO]   Invalid emails: 10
<timestamp> [INFO]   Null regions after standardization: 219
<timestamp> [INFO]   Records after cleaning: 2000
region
US      688
EU      665
APAC    428
NaN     219
Name: count, dtype: int64
opt_out
False    1770
True      230
Name: count, dtype: int64
valid-email rows: 1990 | distinct emails: 735
                      email first_name          phone region  opt_out   source
358   customer546@gmail.com      Sarah            NaN     EU    False  website
390   customer546@gmail.com       Jane  +442079460958     US    False  website
1002  customer546@gmail.com       Jane  +442079460958    NaN    False      crm
1062  customer546@gmail.com     Müller            NaN     US    False      crm
1257  customer546@gmail.com       Alex     5551234567   APAC     True      crm
1369  customer546@gmail.com        Pat            NaN     EU    False      crm
1585  customer546@gmail.com      Sarah   +15551234567     US    False      erp
1597  customer546@gmail.com       John            NaN    NaN    False      erp
1703  customer546@gmail.com       Jane   +15551234567     US    False      erp
<timestamp> [INFO] STEP 3: Deduplication
<timestamp> [INFO]   Records before deduplication: 2000
<timestamp> [INFO]   Duplicate records removed: 1255
<timestamp> [INFO]   Records after deduplication: 745
                   email first_name          phone region  opt_out          sources  source_count
1  customer546@gmail.com       Jane  +442079460958     EU     True  crm,website,erp             3
<timestamp> [INFO] ============================================================
<timestamp> [INFO] DATA QUALITY REPORT
<timestamp> [INFO] ============================================================
<timestamp> [INFO]   [✓] NOT NULL: email: 99.5% (4 failed of 745)
<timestamp> [INFO]   [✓] NOT NULL: first_name: 100.0% (0 failed of 745)
<timestamp> [INFO]   [✓] NOT NULL: region: 97.2% (21 failed of 745)
<timestamp> [INFO]   [✓] UNIQUE: email: 99.6% (3 failed of 741)
<timestamp> [INFO]   [✓] REGEX: email: 99.2% (6 failed of 741)
<timestamp> [INFO]   [✓] VALUES IN SET: region: 100.0% (0 failed of 724)
<timestamp> [INFO]   [✓] DATE RANGE: registration_date: 100.0% (0 failed of 718)
<timestamp> [INFO] ============================================================
<timestamp> [INFO]   OVERALL: ALL CHECKS PASSED ✓
<timestamp> [INFO] ============================================================
                           check  failed  total  pass_rate status
0                NOT NULL: email       4    745     0.9946   PASS
1           NOT NULL: first_name       0    745     1.0000   PASS
2               NOT NULL: region      21    745     0.9718   PASS
3                  UNIQUE: email       3    741     0.9960   PASS
4                   REGEX: email       6    741     0.9919   PASS
5          VALUES IN SET: region       0    724     1.0000   PASS
6  DATE RANGE: registration_date       0    718     1.0000   PASS
```

**Read it like an engineer:**

- **Regions:** 15+ spellings are now 3 values. There are 219 blanks, more than the 188 in Lab 02, because strings like `"N/A"` in the CRM, ERP and marketing sources were text, not `NaN`, until `REGION_MAP` reported them as unknown.
- **1,990 valid-email rows but only 735 distinct emails:** most customers exist in several sources. Deduplication turns 1,990 rows into 735 golden records. Add the 10 invalid-email rows and you get the 745 total (2,000 − 1,255 removed).
- **Following `customer546@gmail.com`:** 9 rows from 3 sources merge into one. The merged row starts from a CRM row (`Jane`, phone `+442079460958`). Its blank region is filled from the next row in priority order. And **`opt_out` is True**, because one CRM row (index 1257) said so, even though 8 rows said False. That one row is the GDPR rule working. (The names disagree because the test data picks names at random. Real duplicates usually agree more.)
- **The quality report:**
  - `NOT NULL: email` (4 failed): rows that arrived with no email.
  - `UNIQUE: email` (3 failed of 741) and `REGEX: email` (6 failed): the invalid-email rows weren't deduplicated (step 2 of Step 3), so repeats of `missing@` and similar remain. These are known and explained, not a dedup bug.
  - `NOT NULL: region` (21 failed, 97.2%): golden records with no region from any source. That still passes the 95% threshold. The guided lab's Part 6 bonus (Lab 05) tries to infer these with an LLM.
  - `total` changes between checks because each format check ignores missing values: 745 rows, 741 with an email, 724 with a region.

✅ **Checkpoint:** you get 745 golden records and 7/7 checks passed, and you can explain why `customer546@gmail.com` ends up opted out.

### Step 6 — Break it on purpose

Understanding comes from seeing failures. Try each change, re-run `python -m practice.lab03_dedup`, then **undo it**:

1. In `CONFIG`, change `"quality_threshold": 0.95` to `0.99`. `NOT NULL: region` now shows `[✗]` and the overall line says `SOME CHECKS FAILED ✗`. Thresholds are business decisions: who decides 95%?
2. In `merge_group`, comment out the 2 opt-out lines (`if "opt_out" in group.columns:` and the line below it). The merged `customer546@gmail.com` row now shows `opt_out False`. This is the GDPR bug the scenario warns about, and nothing crashes. Lab 06 writes a test so this mistake can never reach production unnoticed.
3. Remove `"emea": "EU"` from `REGION_MAP`. The NaN region count jumps. Every unmapped spelling silently becomes "missing".

✅ **Checkpoint:** all 3 changes are undone and you get 745 / 7/7 again.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `AttributeError: Can only use .str accessor with string values` | `.str` used on a column that isn't text (e.g. all `NaN`, or numbers) | Convert first (`.astype(str)` then restore `NaN`, as `standardize_emails` does), or check the column's dtype |
| `"nan"` appears as a real email or name | `astype(str)` turned `NaN` into text | Keep the `.replace({"nan": np.nan, ...})` line |
| `DeprecationWarning: DataFrameGroupBy.apply operated on the grouping columns` | `include_groups=False` missing | Copy Step 4.1 exactly; needs pandas 2.2+ |
| `KeyError: 'email'` after deduplication | `.reset_index()` missing, so `email` is still the index | Keep `.reset_index()` after `.apply(...)` |
| `TypeError` comparing dates in `check_date_range` | `registration_date` is still text | `clean_dataframe` must run `pd.to_datetime(..., errors="coerce")` |
| Count is not 745 | Data regenerated with different code, or a function changed | Compare each function with the guided lab, delete `data\raw\*` and regenerate (Lab 02, Step 2) |
| Opt-outs disappear after dedup | `merge_group` opt-out lines missing | `best["opt_out"] = group["opt_out"].fillna(0).astype(bool).any()` |

---

## Key takeaways

- Standardise, then validate. Keep the raw value (`email_raw`) and a flag (`email_valid`) instead of deleting rows.
- Prefer vectorised `.str` and `.map()`. Use `.apply()` when the rule needs real logic.
- Unknown values become `NaN` on purpose. A visible gap is better than a wrong guess.
- Deduplicate with `groupby` + a merge rule. Source priority decides most fields, but consent (opt-out) always takes the most protective answer.
- A validator class with a threshold turns "is the data clean?" into a report with numbers.

---

## Next

[Lab 04 — Building a Modular ETL Pipeline](Lab_04_Building_a_Modular_ETL_Pipeline.md): you'll connect ingest → clean → dedup → validate → chart → export into one command, `python pipeline.py`, and deliver ShopStream's golden customer files.

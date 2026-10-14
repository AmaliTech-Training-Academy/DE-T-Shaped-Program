# Lab 01 — Python Foundations for Data Engineering

> **Goal:** Learn the core Python a data engineer uses every day by working on real ShopStream problems: messy emails, inconsistent region labels, phone numbers in many formats, duplicate customers and nested JSON. You finish by cleaning a handful of customer records with plain Python, which is exactly what the guided lab's pipeline does with pandas at scale.
>
> **Time:** 3–4 hours | **Difficulty:** Beginner | **Prerequisites:** [Lab 00](Lab_00_Environment_Setup.md) completed (venv active, project at `C:\DataEngineering\sk01`).
>
> **Guided lab coverage:** the Python the [guided lab](../Labs/SK-01_End_to_End_Guided_Lab.md) relies on: f-strings and logging messages, dictionaries (`CONFIG`, `REGION_MAP`), functions with docstrings, `re` (email regex, phone cleaning), `json` and `pathlib` (CRM export, file paths), `try/except` and `datetime`.

---

## The business problem

ShopStream's marketing team is about to email 500,000 customers. The audit found that 23% of emails are invalid or duplicated, regions come in 15+ spellings, and phone numbers come in many formats. Before using pandas (Lab 02), you'll solve these problems on a few values at a time with plain Python. Every cleaning rule in the guided lab is one of the small ideas you practise here.

---

## Learning objectives

By the end of this lab you can:

1. Use variables, the core types and f-strings to build readable log messages.
2. Clean text with string methods (`strip`, `lower`, `title`, `replace`, `split`).
3. Store and look up data with lists, dictionaries and sets, and pick the right one.
4. Write small, documented functions, like the guided lab's `standardize_*` helpers.
5. Validate and clean text with regular expressions.
6. Read and write JSON with `pathlib`, and handle errors with `try/except`.

---

## Before you start

```powershell
cd C:\DataEngineering\sk01
.\.venv\Scripts\Activate.ps1
```

Each step below has you create a small script in `practice\` and run it from the project root with `python practice\<name>.py`. Create the files in VS Code.

---

## Steps

### Step 1 — Variables, types and f-strings

**What:** Store one customer's values and print them in a log-style message.

Create `practice\lab01_basics.py`:

```python
customer_id = 42
email = "maria@gmail.com"
lifetime_value = 1234.5
opt_out = False
region = None

print(f"Customer {customer_id}: {email}, LTV={lifetime_value:,.2f}, opted out? {opt_out}")
print(type(customer_id), type(email), type(lifetime_value), type(opt_out), type(region))
print(f"[{'US':<6}] [{'EU':>6}] id={7:06d} rate={0.0812:.1%} total={1255000:,}")
print("=" * 30)
```

```powershell
python practice\lab01_basics.py
```

**Expected output:**

```
Customer 42: maria@gmail.com, LTV=1,234.50, opted out? False
<class 'int'> <class 'str'> <class 'float'> <class 'bool'> <class 'NoneType'>
[US    ] [    EU] id=000007 rate=8.1% total=1,255,000
==============================
```

**Why:**
- **`None`** is Python's "no value". A missing region is `None`, not `""` or `"N/A"`. In pandas (Lab 02), missing values are shown as `NaN`.
- **f-strings** (`f"...{value}..."`) are how the guided lab writes every log line. The part after `:` formats the value: `:,` adds thousands separators, `.2f` gives 2 decimals, `.1%` shows a percentage, `<6`/`>6` pad left/right. The guided lab's quality report uses `{r['pass_rate']:.1%}` and the summary uses `{input_count:,}`.
- `"=" * 30` repeats a string. The guided lab uses `"=" * 60` for its log section dividers.

✅ **Checkpoint:** you can explain what `:.1%` and `:,` do.

### Step 2 — String methods: the cleaning toolkit

**What:** Fix the kind of messy text ShopStream's sources contain.

Create `practice\lab01_strings.py`:

```python
raw = "  Maria.Garcia@GMAIL.com  "
print(repr(raw.strip().lower()))
print("josé o'brien".title())
print("555.123.4567".replace(".", ""))
print("Maria Garcia".split(" ", 1))
print("test7@test.shopstream.com".endswith("@test.shopstream.com"))
print("@" in "not-an-email")
```

**Expected output:**

```
'maria.garcia@gmail.com'
José O'Brien
5551234567
['Maria', 'Garcia']
True
False
```

**Why:** each line matches a guided-lab cleaning rule:

| Method | Guided lab use |
|---|---|
| `.strip().lower()` | `standardize_emails`: `Maria@GMAIL.com ` and `maria@gmail.com` must match |
| `.title()` | `standardize_names`: `JOSÉ` → `José` |
| `.replace()` | removing formatting from phone numbers |
| `.split(" ", 1)` | the ERP file gives `full_name`; the guided lab splits it into first and last name (`str.split(n=1)`) |
| `.endswith(...)` | removing website test accounts (`@test.shopstream.com`) |

`repr()` shows the quotes, so you can see that the spaces are gone. String methods always return a **new** string. Strings never change in place, so you must assign the result: `email = email.strip()`.

✅ **Checkpoint:** you can say why `"Maria@GMAIL.com "` and `"maria@gmail.com"` would be counted as two customers without `.strip().lower()`.

### Step 3 — Lists and comprehensions

**What:** Hold many values in order and transform them in one line.

Create `practice\lab01_lists.py`:

```python
regions = ["US", "eu", "Asia", None]
for i, r in enumerate(regions):
    print(i, r)
print(["US", "EU"] * 3)
print(list(range(3)))
cleaned = [r.upper() for r in regions if r is not None]
print(cleaned)
```

**Expected output:**

```
0 US
1 eu
2 Asia
3 None
['US', 'EU', 'US', 'EU', 'US', 'EU']
[0, 1, 2]
['US', 'EU', 'ASIA']
```

**Why:**
- `enumerate` gives you the position and the value together. The guided lab's chart code uses `for i, (r, c) in enumerate(region_counts.items())` to place a label above each bar.
- `["US", "EU"] * 3` repeats a list. The guided lab builds its messy test data this way (`[...] * 150`).
- The **list comprehension** `[r.upper() for r in regions if r is not None]` is a filtered loop in one line. The `if` skips `None`, because `None.upper()` would crash.

Notice that `"Asia".upper()` gives `ASIA`, not `APAC`. Changing the case doesn't fix a label. You need a mapping (next step).

✅ **Checkpoint:** you can rewrite the comprehension as a normal `for` loop.

### Step 4 — Dictionaries: lookups and configuration

**What:** Use a dictionary to map 15+ messy region labels to 3 valid ones.

Create `practice\lab01_dicts.py`:

```python
customer = {"email": "maria@gmail.com", "first_name": "Maria", "region": "usa"}
print(customer["first_name"])
print(customer.get("phone"))
print(customer.get("phone", "no phone"))
customer["opt_out"] = True
for key, value in customer.items():
    print(f"  {key} = {value}")

REGION_MAP = {"us": "US", "usa": "US", "europe": "EU", "asia": "APAC"}
for raw in ["usa", "Europe", "mars"]:
    print(raw, "->", REGION_MAP.get(raw.lower()))
```

**Expected output:**

```
Maria
None
no phone
  email = maria@gmail.com
  first_name = Maria
  region = usa
  opt_out = True
usa -> US
Europe -> EU
mars -> None
```

**Why:** dictionaries are everywhere in the guided lab:
- **`CONFIG`** holds every setting (paths, regex, thresholds) in one place, so nothing is hard-coded inside functions.
- **`REGION_MAP`** turns `"usa"`, `"United States"` and `"North America"` into `"US"`. You lowercase the key first so one entry covers every capitalisation.
- `.get(key)` returns `None` for a missing key instead of crashing like `customer["phone"]` would. An unknown region (`"mars"`) becomes `None`. That's what we want: it's reported as missing, not silently guessed. In pandas, `.map(REGION_MAP)` does the same and gives `NaN`.
- `CONFIG["source_priority"] = {"crm": 1, "website": 2, ...}` is a dictionary used as a ranking.

✅ **Checkpoint:** add `"emea": "EU"` to `REGION_MAP` and check that `"EMEA"` now maps to `EU`.

### Step 5 — Sets: finding duplicates

**What:** Find repeated emails. Removing duplicates is half the guided lab.

Create `practice\lab01_sets.py`:

```python
emails = ["a@x.com", "b@x.com", "a@x.com", "c@x.com", "b@x.com"]
print(len(emails), len(set(emails)))
seen = set()
for e in emails:
    if e in seen:
        print("duplicate:", e)
    seen.add(e)
```

**Expected output:**

```
5 3
duplicate: a@x.com
duplicate: b@x.com
```

**Why:** a **set** holds each value once, and `in` checks are instant even for millions of items. In a list, `in` gets slower as the list grows. `len(emails) - len(set(emails))` tells you how many duplicates there are. The guided lab's "UNIQUE: email" check counts duplicates the same way, using pandas' `.duplicated()`.

| Use a… | When you need… |
|---|---|
| list | order, and repeats allowed (rows of a file) |
| dict | lookup by key (config, mappings, one record) |
| set | uniqueness and fast "have I seen this?" checks |

✅ **Checkpoint:** you can say which structure you would use to track emails already loaded.

### Step 6 — Functions with docstrings

**What:** Wrap a cleaning rule in a reusable, documented function. The guided lab is built from about 20 functions like this.

Create `practice\lab01_functions.py`:

```python
def standardize_region(value: str) -> str:
    """Return US/EU/APAC for a messy region label, or None if unknown."""
    region_map = {"us": "US", "usa": "US", "eu": "EU", "europe": "EU", "apac": "APAC", "asia": "APAC"}
    if value is None:
        return None
    return region_map.get(value.strip().lower())

print(standardize_region(" USA "), standardize_region("Europe"), standardize_region("mars"), standardize_region(None))


def count_valid(values: list) -> int:
    """Count values that a nested helper says are valid."""
    def is_valid(v):
        return v is not None and v != ""
    return sum(1 for v in values if is_valid(v))

print(count_valid(["a", "", None, "b"]))
```

**Expected output:**

```
US EU None None
2
```

**Why:**
- **Docstrings** (`"""..."""`) describe what a function does. Every guided-lab function has one, so the next engineer knows what it expects and returns.
- **Type hints** (`value: str -> str`) are documentation. Python doesn't enforce them, but VS Code uses them to warn you.
- **Check for `None` first.** Calling `.strip()` on `None` crashes. Real data always has missing values.
- **Nested functions:** `is_valid` lives inside `count_valid`. The guided lab uses this pattern: `clean_phone` is defined inside `standardize_phone_numbers`, and `merge_group` inside `deduplicate_customers`.

✅ **Checkpoint:** `standardize_region("")` returns `None`. Can you explain why?

### Step 7 — Regular expressions

**What:** Validate emails and strip phone formatting with patterns.

Create `practice\lab01_regex.py`:

```python
import re

phone = "+1 (555) 123-4567"
digits = re.sub(r"[^\d]", "", phone)
print(digits, phone.startswith("+"))

EMAIL_REGEX = r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
for e in ["customer1@yahoo.com", "missing@", "double@@sign.com"]:
    print(e, bool(re.match(EMAIL_REGEX, e)))

print(re.sub(r"\s+", " ", "Maria    del   Carmen"))
```

**Expected output:**

```
15551234567 True
customer1@yahoo.com True
missing@ False
double@@sign.com False
Maria del Carmen
```

**Why:** a **regular expression** (regex) is a pattern for text.

| Pattern | Means |
|---|---|
| `\d` | any digit; `[^\d]` = anything that is *not* a digit |
| `\s+` | one or more whitespace characters |
| `^` … `$` | start … end of the text (the whole value must match) |
| `[a-zA-Z0-9._%+-]+` | one or more of these characters |
| `{2,}` | 2 or more of the previous item |

- `re.sub(r"[^\d]", "", phone)` is the core of the guided lab's `clean_phone`. It also keeps a leading `+` and rejects anything shorter than 7 digits.
- `EMAIL_REGEX` is exactly `CONFIG["email_regex"]` in the guided lab: "something @ something . at least 2 letters".
- `re.sub(r"\s+", " ", ...)` turns runs of spaces into one space, as `standardize_names` does.
- The `r"..."` prefix makes a **raw string**, so `\d` reaches the regex engine unchanged. Always use it for regex.

✅ **Checkpoint:** add `"@nodomain.com"` to the email list. It should print `False`. Why?

### Step 8 — JSON files and `pathlib`

**What:** Write and read nested JSON, like the CRM export.

Create `practice\lab01_json.py`:

```python
import json
from pathlib import Path

out_dir = Path("data") / "processed"
out_dir.mkdir(parents=True, exist_ok=True)
record = {"id": "CRM-000001", "email": "maria@gmail.com", "profile": {"first_name": "Maria", "last_name": "García"}}
text = json.dumps({"customers": [record]})
print(text)
path = out_dir / "lab01_sample.json"
path.write_text(text, encoding="utf-8")
loaded = json.loads(path.read_text(encoding="utf-8"))
print(loaded["customers"][0]["profile"]["last_name"])
print(path.exists(), path.name, path.suffix)
```

**Expected output:**

```
{"customers": [{"id": "CRM-000001", "email": "maria@gmail.com", "profile": {"first_name": "Maria", "last_name": "Garc\u00eda"}}]}
García
True lab01_sample.json .json
```

**Why:**
- **JSON** is how APIs and many systems exchange data. In Python it becomes dicts and lists. The CRM export has exactly this shape: `{"customers": [ {..., "profile": {...}} ]}`. `profile` is *nested*, which is why the guided lab flattens it with `pd.json_normalize` (Lab 02).
- `json.dumps` writes `í` as `\u00ed`. That's normal JSON escaping, and `json.loads` turns it back into `García`.
- **`pathlib.Path`** builds file paths with `/`, which works on Windows, macOS and Linux. The guided lab writes `CONFIG["input_dir"] / "crm_export.json"`.
- `mkdir(parents=True, exist_ok=True)` creates the folder if needed and doesn't fail if it already exists. Running it twice is safe.
- **Always pass `encoding="utf-8"`.** On Windows the default encoding is often not UTF-8, and accented names (`García`, `Müller`) get garbled. Garbled names are one of the problems in the ShopStream audit.

✅ **Checkpoint:** `data\processed\lab01_sample.json` exists. Open it in VS Code and find `profile`.

### Step 9 — Errors and dates

**What:** Handle bad values without crashing, and work with dates.

Create `practice\lab01_errors.py`:

```python
from datetime import datetime

start = datetime.now()
for value in ["85000", "N/A"]:
    try:
        print(int(value) * 2)
    except ValueError as e:
        print(f"Could not convert {value!r}: {e}")
duration = (datetime.now() - start).total_seconds()
print(f"Duration: {duration:.1f}s")
print(datetime(2024, 1, 15).strftime("%Y-%m-%d"), datetime(2024, 1, 15).isoformat())
```

**Expected output:**

```
170000
Could not convert 'N/A': invalid literal for int() with base 10: 'N/A'
Duration: 0.0s
2024-01-15 2024-01-15T00:00:00
```

**Why:**
- **`try/except`** catches a specific error (`ValueError`) and carries on. Catch the narrowest error you expect. A bare `except:` also hides real bugs. The guided lab uses `try/except` around each AI request, so one failed batch doesn't stop the run. Lab 05 covers when to catch an error and when to stop.
- `{value!r}` prints the value with quotes, so empty strings and spaces are visible in error messages.
- **`datetime.now()`** at the start and end is how the guided lab measures `Duration` and stamps `Run started: ...isoformat()`. `strftime("%Y-%m-%d")` produces `2024-01-15`, the date format all ShopStream sources are standardised to.

✅ **Checkpoint:** change `"N/A"` to `"12a"`. The except branch still handles it.

### Step 10 — Mini-pipeline: clean 5 customer records

**What:** Combine everything above into one script that does, in plain Python, what the guided lab does with pandas: standardise, validate, drop test accounts, deduplicate (an opt-out anywhere wins) and write the result.

Create `practice\lab01_mini_pipeline.py`:

```python
"""Lab 01 mini-pipeline: clean a few ShopStream customer records with plain Python."""
import json
import re
from pathlib import Path

EMAIL_REGEX = r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
REGION_MAP = {
    "us": "US", "usa": "US", "united states": "US", "north america": "US",
    "eu": "EU", "europe": "EU", "emea": "EU",
    "apac": "APAC", "asia": "APAC", "asia pacific": "APAC", "ap": "APAC",
}

raw_records = [
    {"email": "  Maria@GMAIL.com ", "first_name": "maria", "phone": "+1 (555) 123-4567", "region": "USA", "opt_out": 0},
    {"email": "maria@gmail.com", "first_name": "MARIA", "phone": None, "region": "", "opt_out": 1},
    {"email": "jose@yahoo.com", "first_name": "josé", "phone": "555.123.4567", "region": "Europe", "opt_out": 0},
    {"email": "missing@", "first_name": "pat", "phone": "invalid-phone", "region": "Asia", "opt_out": 0},
    {"email": "test3@test.shopstream.com", "first_name": "Test", "phone": None, "region": "US", "opt_out": 0},
]


def standardize_email(value):
    """Trim, lowercase and remove spaces; return None for blanks."""
    if value is None:
        return None
    value = re.sub(r"\s+", "", value).lower()
    return value or None


def is_valid_email(value):
    return value is not None and re.match(EMAIL_REGEX, value) is not None


def standardize_region(value):
    if value is None:
        return None
    return REGION_MAP.get(value.strip().lower())


def clean_phone(value):
    """Keep digits (and a leading +); None if fewer than 7 digits."""
    if value is None:
        return None
    digits = re.sub(r"[^\d]", "", value)
    if len(digits) < 7:
        return None
    return "+" + digits if value.strip().startswith("+") else digits


golden = {}          # email -> merged record
rejected = []
for record in raw_records:
    email = standardize_email(record["email"])
    if email is not None and email.endswith("@test.shopstream.com"):
        continue                                   # drop internal test accounts
    if not is_valid_email(email):
        rejected.append(record)
        continue
    cleaned = {
        "email": email,
        "first_name": record["first_name"].strip().title(),
        "phone": clean_phone(record["phone"]),
        "region": standardize_region(record["region"]),
        "opt_out": bool(record["opt_out"]),
    }
    if email in golden:                            # duplicate: merge into the first record
        first = golden[email]
        for field in ["phone", "region"]:
            if first[field] is None:
                first[field] = cleaned[field]
        first["opt_out"] = first["opt_out"] or cleaned["opt_out"]   # an opt-out anywhere wins
    else:
        golden[email] = cleaned

print(f"Input: {len(raw_records)}  golden: {len(golden)}  rejected: {len(rejected)}")
for customer in golden.values():
    print(customer)

out = Path("data") / "processed" / "lab01_golden.json"
out.parent.mkdir(parents=True, exist_ok=True)
out.write_text(json.dumps(list(golden.values()), indent=2), encoding="utf-8")
print(f"Wrote {out}")
```

```powershell
python practice\lab01_mini_pipeline.py
```

**Expected output:**

```
Input: 5  golden: 2  rejected: 1
{'email': 'maria@gmail.com', 'first_name': 'Maria', 'phone': '+15551234567', 'region': 'US', 'opt_out': True}
{'email': 'jose@yahoo.com', 'first_name': 'José', 'phone': '5551234567', 'region': 'EU', 'opt_out': False}
Wrote data\processed\lab01_golden.json
```

**Why:** follow Maria. She appears twice: once with a messy email and a US region, and once with no region but `opt_out = 1`. After cleaning, both rows have the same email, so they merge. The merged record keeps her phone and region, **and** is opted out, because one opt-out in any source is final. That is the GDPR rule in the guided lab's `deduplicate_customers`. `missing@` is rejected and the test account is dropped.

The guided lab does the same work with pandas, on 2,000 rows from 4 files instead of 5 hand-typed dicts:

| Mini-pipeline (plain Python) | Guided lab (pandas) |
|---|---|
| `standardize_email`, `is_valid_email` | `standardize_emails`, `validate_emails` (Part 3.1) |
| `clean_phone` | `standardize_phone_numbers` (Part 3.2) |
| `standardize_region` + `REGION_MAP` | `standardize_regions` + `REGION_MAP` (Part 3.3) |
| `endswith("@test.shopstream.com")` | test-account filter in `ingest_website_csv` (Part 2.1) |
| `golden` dict keyed by email | `groupby("email")` + `merge_group` (Part 4) |

✅ **Checkpoint:** the output matches, and `data\processed\lab01_golden.json` lists 2 customers. Try adding a sixth record with email `JOSE@yahoo.com ` and `"opt_out": 1`, then re-run. José should become opted out.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `python: can't open file '...\practice\lab01_basics.py'` | Not in the project root, or the file is saved elsewhere | `cd C:\DataEngineering\sk01`; check the file is in `practice\` |
| `AttributeError: 'NoneType' object has no attribute 'strip'` | Calling a string method on `None` | Check `if value is None` first (Step 6) |
| `KeyError: 'phone'` | `record["phone"]` on a dict without that key | Use `record.get("phone")` |
| `IndentationError` | Mixed tabs and spaces, or a missing indent after `:` | Use 4 spaces. VS Code: bottom bar → "Spaces: 4" |
| Accented names print as `JosÃ©` | File read or written without UTF-8 | Pass `encoding="utf-8"` to `read_text` / `write_text` |
| Regex never matches | Forgot the `r` prefix, or `^`/`$` misplaced | Use `r"..."`; test one value at a time |

---

## Key takeaways

- Clean text with `strip()`, `lower()` and `title()` before comparing values. Otherwise one customer looks like two.
- Use dicts for mappings and config, and sets for "have I seen this?". `.get()` handles missing keys without crashing.
- Small, documented functions that check for `None` first are what the guided lab is built from.
- Regex validates formats (emails) and removes formatting (phones). Always use raw strings.
- Use `pathlib` for paths and always pass `encoding="utf-8"`.
- Catch specific exceptions. Deciding what may fail quietly and what must stop the run is a design decision (Lab 05).

---

## Next

[Lab 02 — Working with Files and pandas](Lab_02_Working_with_Files_and_Pandas.md): you start `pipeline.py`, generate ShopStream's 4 source files and load each one into a pandas DataFrame.

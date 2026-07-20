# Lab 01 — Python Foundations for Data Engineering

> **Goal:** learn the core Python you will use every day as a data engineer — variables, the data
> types that model records, loops, functions, file I/O with `pathlib`, and modules — by building a
> small program that reads a raw CSV file *without pandas*, so that when pandas arrives in Lab 02
> you understand exactly what it's saving you from.
>
> **Time:** 4–6 hours. **Prerequisites:** [Lab 00](Lab_00_Environment_Setup.md) completed.

---

## 1. Environment Setup

Nothing new to install today. Verify the Lab 00 setup:

```powershell
cd C:\DataEngineering\sk01
.\.venv\Scripts\Activate.ps1
python --version
```

Expected: the `(.venv)` prefix and your Python version. If activation fails, revisit Lab 00 Step 5.

Create today's working file:

```powershell
New-Item src\lab01_foundations.py
```

Open the folder in VS Code (`code .` from the project directory opens VS Code right there — try it).

We also need a small raw data file to practice on. Create `data\raw\employees_sample.csv` in
VS Code with exactly this content (type it — noticing structure is the point):

```csv
employee_id,first_name,last_name,department,salary,hire_date
101,Ama,Mensah,Engineering,85000,2021-03-15
102,Kofi,Boateng,Marketing,62000,2019-11-02
103,Esi,Owusu,Engineering,91000,2022-07-30
104,Yaw,Asante,,58000,2020-01-20
105,Ama,Mensah,Engineering,85000,2021-03-15
```

Note two deliberate defects: row 104 has an **empty department**, and row 105 **duplicates** row
101. Real data always has both; we'll detect them today by hand.

**Verification:** `Get-Content data\raw\employees_sample.csv` in PowerShell prints the 6 lines.

---

## 2. Business Context

Why learn "plain Python" when pandas exists?

Because pipelines are mostly *not* DataFrame operations. Real pipeline code is: looping over files
in a folder, building dictionaries of configuration, writing functions that decide whether a file
is valid, parsing a filename to extract a date, constructing output paths. That's all core Python.
Engineers who only know pandas write pipelines that are un-testable one-thousand-line scripts;
engineers who know Python structure them into small functions and modules.

Industry reality check: in the capstone (and at any employer), you'll receive JSON from an API —
which arrives in Python as **dictionaries and lists**, exactly what you learn today. If nested
dicts confuse you, every API integration will too. Who consumes this skill? *You*, in every
subsequent lab. What happens if it's weak? You'll copy-paste pandas snippets you can't adapt, and
the first non-tabular problem will stop you cold.

---

## 3. Concept Explanation

### Values, types, and variables

A **value** is a piece of data (`42`, `"Ama"`, `3.5`). Every value has a **type** — the kind of
thing it is, which determines what you can do with it. A **variable** is a name attached to a
value so you can refer to it later.

The types you'll use constantly:

| Type | Example | Data engineering use |
|---|---|---|
| `int` | `85000` | counts, IDs, whole-number amounts |
| `float` | `85000.0` | measurements, ratios (beware: floats are approximate — never use for money math in finance; use `decimal` there) |
| `str` (string) | `"Engineering"` | names, codes, anything textual — most raw data arrives as strings |
| `bool` | `True` / `False` | flags: is_valid, is_duplicate |
| `None` | `None` | "no value" — Python's version of NULL/missing |
| `list` | `[101, 102, 103]` | ordered collection — a column of values, lines of a file |
| `dict` (dictionary) | `{"employee_id": 101, "name": "Ama"}` | key→value mapping — **one record**; also configs, lookups |
| `tuple` | `(101, "Ama")` | like a list but immutable — fixed-shape pairs |
| `set` | `{101, 102}` | unordered, unique values — perfect for "have I seen this ID before?" |

The mental model that makes data engineering click: **a table is a list of dicts.** Each dict is a
row; the keys are column names. pandas will later give you a much faster, richer version of the
same idea.

### Why alternatives exist

Could you do this lab in Excel? For 5 rows, yes. For 15,000 rows arriving nightly, needing
deduplication rules and an audit trail — no. Could you use another language (Java, Scala, SQL)?
All are used in data engineering, but Python won the general-purpose slot because it's readable,
has pandas/PySpark, and every orchestration tool (Airflow, SK-03) speaks it. SQL (SK-02)
complements Python; it doesn't replace it for file wrangling and glue logic.

### Functions and modules

A **function** is a named, reusable block of code with inputs (**parameters**) and an output
(**return value**). A **module** is simply a `.py` file whose functions you can `import` into
another file. These two ideas are the whole basis of "modular pipeline code" in Lab 04.

---

## 4. Step-by-Step Implementation

Work in `src\lab01_foundations.py`. After each step, run it with
`python src\lab01_foundations.py` and compare with the expected output. Delete or comment out
previous experiment lines as you go if output gets noisy (`#` starts a comment — Python ignores
the rest of the line).

### Step 1 — Variables, types, and f-strings

**What/why:** get comfortable creating values and printing readable messages — logging readable
messages is 20% of pipeline code.

```python
employee_id = 101              # int
first_name = "Ama"             # str
salary = 85000.0               # float
is_active = True               # bool
department = None              # None: we don't know it (yet)

print(f"Employee {employee_id}: {first_name}, salary={salary:,.2f}, active={is_active}")
print(type(employee_id), type(first_name), type(department))
```

An **f-string** (`f"..."`) embeds variables into text with `{}`. The `:,.2f` is a format spec —
thousands separators, 2 decimals.

**Expected output:**

```
Employee 101: Ama, salary=85,000.00, active=True
<class 'int'> <class 'str'> <class 'NoneType'>
```

**Common mistake:** forgetting the `f` prefix — you'll literally print `{employee_id}`.
**Troubleshooting:** `SyntaxError` usually means a missing quote or parenthesis on the *line
above* the one reported.

### Step 2 — Strings are your parsing toolkit

**What/why:** raw data arrives as text. These four methods do most cleaning work in the wild:

```python
raw = "  ENGINEERING  "
print(raw.strip())                 # remove surrounding whitespace -> "ENGINEERING"
print(raw.strip().title())         # -> "Engineering"
line = "101,Ama,Mensah,Engineering,85000,2021-03-15"
fields = line.split(",")           # str -> list of str
print(fields)
print("-".join(["2021", "03", "15"]))   # list of str -> one str
print("salary" in "salary,hire_date")   # substring test -> True
```

**Expected output:**

```
ENGINEERING
Engineering
['101', 'Ama', 'Mensah', 'Engineering', '85000', '2021-03-15']
2021-03-15
True
```

Notice: after `split`, **everything is a string** — `'85000'` is text, not a number. Converting
(**casting**) is explicit: `int('85000')` → `85000`. This distinction — *representation vs value* —
is the root of half of all data quality bugs, so burn it in now.

**Common mistake:** `int('85000.0')` raises `ValueError` (int() won't parse a decimal string);
use `float(...)` first or `int(float(...))`.

### Step 3 — Lists and loops

**What/why:** processing a file = looping over its lines. Processing a table = looping over records.

```python
salaries = [85000, 62000, 91000, 58000]

total = 0
for s in salaries:          # "for each item s in the list..."
    total += s              # shorthand for total = total + s
print(f"count={len(salaries)}, total={total}, avg={total/len(salaries):.0f}")

# Pythonic shortcuts you'll see everywhere:
print(sum(salaries), max(salaries), min(salaries))

# A list comprehension: build a new list from an old one in one line
high_earners = [s for s in salaries if s > 80000]
print(high_earners)
```

**Expected output:**

```
count=4, total=296000, avg=74000
296000 91000 58000
[85000, 91000]
```

**Common mistake:** indentation. Python uses indentation (4 spaces) instead of braces to mark what
is "inside" a loop/if/function. `IndentationError` means inconsistent spacing — configure VS Code
to insert 4 spaces per Tab (it does by default for Python).

### Step 4 — Dictionaries: the shape of a record

**What/why:** dicts model records, JSON, and configuration — the three most common data shapes
you'll ever touch.

```python
employee = {
    "employee_id": 101,
    "first_name": "Ama",
    "last_name": "Mensah",
    "department": "Engineering",
    "salary": 85000,
}

print(employee["first_name"])                 # access by key
print(employee.get("email"))                  # missing key -> None (no crash)
print(employee.get("email", "MISSING"))       # ...or a default
employee["email"] = "ama.mensah@corp.com"     # add/overwrite a key

for key, value in employee.items():          # iterate over key/value pairs
    print(f"  {key} = {value}")
```

**Expected output:**

```
Ama
None
MISSING
  employee_id = 101
  first_name = Ama
  last_name = Mensah
  department = Engineering
  salary = 85000
  email = ama.mensah@corp.com
```

**Key distinction:** `employee["email"]` on a missing key raises `KeyError` (crash);
`employee.get("email")` returns `None`. In pipelines, `.get()` with a default is how you survive
records with missing fields — you'll use it constantly on API JSON in the capstone.

### Step 5 — Conditionals: encoding business rules

```python
salary = 58000

if salary is None:
    category = "unknown"
elif salary >= 80000:
    category = "senior band"
elif salary >= 60000:
    category = "mid band"
else:
    category = "entry band"

print(category)     # -> entry band
```

Note `is None` (identity check) is the idiomatic way to test for missing values — not `== None`.

### Step 6 — Functions: name your logic

**What/why:** any logic you might reuse or test gets a function. This is the habit that makes
Lab 06 (testing) possible.

```python
def categorize_salary(salary):
    """Return the salary band label for a numeric salary (None-safe)."""
    if salary is None:
        return "unknown"
    if salary >= 80000:
        return "senior band"
    if salary >= 60000:
        return "mid band"
    return "entry band"

print(categorize_salary(85000))   # senior band
print(categorize_salary(None))    # unknown
```

The triple-quoted string is a **docstring** — documentation living inside the function. Write one
for every function from now on; tools, editors, and teammates all read them.

**Common mistake:** forgetting `return` — the function then returns `None` silently, and the bug
shows up far away from its cause.

### Step 7 — Reading a file safely with `pathlib` and `with`

**What/why:** now we read our sample CSV, the raw way. Two tools matter:

- **`pathlib.Path`** — the modern way to handle file paths. It handles Windows `\` vs Linux `/`
  automatically, so your code is portable.
- **`with open(...) as f:`** — a **context manager**: it guarantees the file is closed even if an
  error occurs mid-read. Unclosed files cause locked-file errors on Windows and resource leaks on
  servers. Always `with`, never bare `open()`.

```python
from pathlib import Path

RAW_FILE = Path("data") / "raw" / "employees_sample.csv"   # "/" joins path parts

def read_employees(path):
    """Read the sample CSV into a list of dicts (one dict per row)."""
    records = []
    with open(path, "r", encoding="utf-8") as f:
        header = f.readline().strip().split(",")     # first line = column names
        for line in f:                               # remaining lines = data
            values = line.strip().split(",")
            record = dict(zip(header, values))       # pair headers with values
            records.append(record)
    return records

employees = read_employees(RAW_FILE)
print(f"Loaded {len(employees)} records")
print(employees[0])
```

Run it **from the project root** (`C:\DataEngineering\sk01`) — the relative path `data\raw\...`
is resolved from your *current directory*, not the script's location.

**Expected output:**

```
Loaded 5 records
{'employee_id': '101', 'first_name': 'Ama', 'last_name': 'Mensah', 'department': 'Engineering', 'salary': '85000', 'hire_date': '2021-03-15'}
```

Observe: every value is a **string** — even salary. And `zip(header, values)` pairs
`('employee_id', '101'), ('first_name', 'Ama'), ...` which `dict()` turns into a record. This tiny
function is a hand-made `pd.read_csv`.

**Troubleshooting**

| Symptom | Cause | Fix |
|---|---|---|
| `FileNotFoundError` | Running from the wrong directory | `cd C:\DataEngineering\sk01` first; print `Path.cwd()` to see where you are |
| `UnicodeDecodeError` | File saved in a non-UTF-8 encoding | In VS Code, bottom-right shows encoding — click it → "Save with Encoding" → UTF-8 |
| Last record looks wrong | File missing trailing newline or has a blank last line | Guard with `if not line.strip(): continue` inside the loop |

### Step 8 — Detect the defects: nulls and duplicates by hand

**What/why:** you knew rows 104 and 105 were bad. Prove code can find them — this is the seed of
Lab 03's validator.

```python
def find_missing_departments(records):
    """Return employee_ids whose department field is empty."""
    return [r["employee_id"] for r in records if r["department"] == ""]

def find_duplicates(records):
    """Return employee_ids seen more than once (by full-field match except id)."""
    seen = set()
    dupes = []
    for r in records:
        key = (r["first_name"], r["last_name"], r["hire_date"])  # tuple as a composite key
        if key in seen:
            dupes.append(r["employee_id"])
        else:
            seen.add(key)
    return dupes

print("Missing department:", find_missing_departments(employees))
print("Duplicate rows    :", find_duplicates(employees))
```

**Expected output:**

```
Missing department: ['104']
Duplicate rows    : ['105']
```

That `set` + tuple-key pattern is a real deduplication technique — sets test membership in
constant time, so this scales to millions of rows.

### Step 9 — Write an output file and split into a module

**What/why:** pipelines end by *writing* results. And to prepare for Lab 04's modular structure,
we'll move our reusable functions into their own module.

1. Create `src\filetools.py` and move `read_employees`, `find_missing_departments`, and
   `find_duplicates` into it (with their `from pathlib import Path` import).
2. Reduce `src\lab01_foundations.py` to:

```python
"""Lab 01 driver script: read raw employees, report quality issues, write a summary."""
from pathlib import Path
from filetools import read_employees, find_missing_departments, find_duplicates

RAW_FILE = Path("data") / "raw" / "employees_sample.csv"
OUT_FILE = Path("data") / "processed" / "lab01_quality_summary.txt"

employees = read_employees(RAW_FILE)
missing = find_missing_departments(employees)
dupes = find_duplicates(employees)

summary = (
    f"records_read={len(employees)}\n"
    f"missing_department_ids={missing}\n"
    f"duplicate_ids={dupes}\n"
)
OUT_FILE.write_text(summary, encoding="utf-8")
print(f"Wrote summary to {OUT_FILE}")
print(summary)
```

3. Run from the project root — **note:** run as `python src\lab01_foundations.py` *from* `src`'s
   parent won't find the import unless Python can see `src`. Simplest fix for now: run from inside
   `src` with data paths adjusted — or better, run this way from the project root:

```powershell
python -m src.lab01_foundations
```

Wait — that needs `src` to be a **package**. Make it one by creating an empty file:

```powershell
New-Item src\__init__.py
```

and change the import line to `from src.filetools import ...`. Now `python -m src.lab01_foundations`
runs from the project root and all relative data paths work.

**Expected output:**

```
Wrote summary to data\processed\lab01_quality_summary.txt
records_read=5
missing_department_ids=['104']
duplicate_ids=['105']
```

You just built the skeleton of every pipeline you'll ever write: **read → check → transform →
write**, split across a reusable module and a thin driver script. The `python -m` +
`__init__.py` dance felt fussy — remember it; it's exactly how Lab 04 structures the full ETL
project, and there we'll explain the mechanics properly.

---

## 5. Production Engineering Practices

- **Small named functions, one job each.** *Failure story:* a 900-line script at a logistics firm
  mixed parsing, cleaning, and writing. A currency bug took three days to isolate because nothing
  could be run — or reasoned about — in isolation. Your `find_duplicates` can be tested in five
  lines; that's the standard to hold.
- **Docstrings and comments explain *why*, code shows *how*.** Six months from now, the comment
  `# ADP export duplicates header on page breaks` will save you an afternoon.
- **`pathlib` over string paths.** Hard-coded `"C:\\Users\\me\\data.csv"` breaks on every other
  machine and OS. Relative `Path` objects + a defined working directory travel well; Lab 05 will
  move paths into config entirely.
- **Context managers (`with`) for every resource.** Files today; database connections in SK-02 —
  the pattern is identical, and leaked connections take down real systems.
- **Never trust field values.** Everything read from a text file is a string until *you* cast it,
  and casting is where errors surface — which is good: loud early failures beat silent wrong numbers.

---

## 6. Reflection

### What you learned
Python's core types and how they model records; strings as a parsing toolkit; loops, dicts, sets,
and functions; safe file I/O with `pathlib` and `with`; splitting code into an importable module;
and you hand-built CSV reading, null detection, and deduplication — the three most common
operations in data engineering — before letting any library do it for you.

### Why it matters
Lab 02's pandas will do all of this in one line each. Because you built the manual version, you'll
know what those one-liners actually do, what can go wrong inside them, and how to work when the
data *isn't* a neat table.

### Interview questions

1. **What's the difference between a list and a tuple?**
   *Lists are mutable (can change), tuples immutable. Tuples can therefore be dict keys/set
   members — handy for composite keys in deduplication.*
2. **When would you use a set in a data pipeline?**
   *Fast uniqueness/membership checks — e.g., "seen this ID before?" is O(1) with a set vs O(n)
   with a list.*
3. **`dict["key"]` vs `dict.get("key")`?**
   *Bracket access raises `KeyError` on missing keys; `.get` returns None or a default. Use `.get`
   for optional fields in messy records, brackets when absence is a genuine error.*
4. **What does a context manager (`with`) do?**
   *Guarantees setup/teardown — e.g., closing a file — even if an exception occurs inside the block.*
5. **Why is `salary = "85000"` dangerous in a pipeline?**
   *It's a string; comparisons and arithmetic on it are wrong or crash ("100" < "85" as strings!).
   Types must be cast explicitly and validated.*
6. **How would you deduplicate a million records on (name, hire_date)?**
   *Build a set of composite-key tuples while iterating; membership check per record is constant
   time — linear overall. (In pandas: `drop_duplicates(subset=...)`.)*
7. **What is `None` and how do you test for it?**
   *Python's null/missing marker; test with `is None`, not `== None`.*

**Interview trap:** "What does `[s for s in salaries if s > 80000]` return if `salaries` contains
a `None`?" — It crashes with `TypeError` (can't compare None with int). The trap tests whether you
reflexively think about missing values. Correct habit: filter or default Nones first.

### Real-world applications
Config dicts in Airflow, JSON API payloads, filename parsing for date-partitioned drops,
line-by-line processing of files too large for memory — all pure today-Python.

### Key takeaways
- A table is a list of dicts; a record is a dict; JSON is dicts and lists.
- Everything from a text file is a string until you cast it.
- Functions with docstrings, one job each — that's how testable pipelines are made.
- `with open(...)` and `pathlib.Path`, always.

**Next:** [Lab 02 — Working with Files and pandas](Lab_02_Working_with_Files_and_Pandas.md), where
your 15-line CSV reader becomes `pd.read_csv(path)` — and you learn the DataFrame.

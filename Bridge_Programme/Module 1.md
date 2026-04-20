# BRIDGE PROGRAMME: Layer 0
## From Career Changer to Data Engineering Ready

---

> **Who this is for:** You have SQL and Python basics. You understand what a table is, you can write a SELECT statement, and you can write a Python function. That is enough to start here. By the end of this programme, you will be ready for SK-01 — the first skillset in the library.
>
> **What you will have at the end:** A working local development environment, comfort with the command line, confidence with pandas and SQL beyond basics, a mental model of how cloud infrastructure works, and a clear picture of what data engineers actually do and what tools they use.

---

# MODULE 1: Python for Data Engineering

## Why This Module Exists

You know Python basics. You can write a loop, define a function, and maybe work with a list. That is genuinely useful. But data engineering Python is a different flavour of the language. Data engineers use Python to read and write files, manipulate tables in memory, connect to databases, and process millions of rows. This module closes the gap between "Python basics" and "Python as a data tool."

---

## Module 1 Glossary

**DataFrame** — A table held in memory inside Python. Think of it as a spreadsheet that Python can manipulate at high speed. The most common tool for this is a library called pandas.

**Library / Package** — A collection of code that someone else wrote, which you can use in your own programmes. You install libraries with pip (e.g. `pip install pandas`). Libraries save you from having to write everything from scratch.

**pandas** — The most important Python library for data engineers working with structured data. It gives you the DataFrame object and hundreds of functions for filtering, grouping, merging, and reshaping data.

**pip** — Python's package installer. You use it in the terminal to install libraries: `pip install pandas`.

**Virtual environment** — An isolated Python installation for a specific project. Prevents conflicts between projects that need different versions of the same library. Think of it as a clean room for each project.

**CSV** — Comma-Separated Values. A plain text file where each row is a line and each column is separated by a comma. The most common format for sharing tabular data.

**JSON** — JavaScript Object Notation. A text format that stores data as key-value pairs, like a Python dictionary. Very common in APIs and web services.

**Exception / Error handling** — Code that anticipates something going wrong (a missing file, a network timeout, a bad value) and responds gracefully instead of crashing. In Python, this is the `try / except` pattern.

**Type** — The category of a value. `"hello"` is a string (str). `42` is an integer (int). `3.14` is a float. `True` is a boolean (bool). Knowing types matters in data engineering because mixing them causes errors.

**Method** — A function that belongs to an object. `df.head()` is a method of a DataFrame. You call it with a dot after the object.

**f-string** — A Python string that can contain expressions evaluated at runtime. `f"There are {len(df)} rows"` will substitute the actual count into the string.

**None** — Python's way of representing "nothing" or "missing value". The equivalent of NULL in SQL.

---

## Concept Explainer 1.1: What is a DataFrame and Why Does It Matter?

Imagine a spreadsheet. It has rows and columns. Each column has a name. Each cell has a value. You can filter rows, sort columns, add new columns based on calculations, and merge two sheets together.

A **DataFrame** is exactly this — but in Python memory, which makes it dramatically faster than a spreadsheet for large datasets, and programmable in ways a spreadsheet cannot be.

When a data engineer says "I loaded the data into a DataFrame," they mean: "I read a file (CSV, Excel, database table, API response) and now I have it as a table in memory that I can manipulate with code."

The library that gives us DataFrames in Python is called **pandas**. Every data engineering team that works with Python uses pandas. It is not optional knowledge — it is foundational.

```python
# This is what working with a DataFrame looks like:

import pandas as pd

# Read a CSV file into a DataFrame
df = pd.read_csv("orders.csv")

# Look at the first 5 rows
print(df.head())

# How many rows and columns?
print(df.shape)   # e.g. (10000, 8) means 10,000 rows, 8 columns

# What are the column names and their types?
print(df.dtypes)

# Filter: only rows where the order status is "completed"
completed = df[df["status"] == "completed"]

# Add a new column: revenue per order
df["revenue"] = df["quantity"] * df["unit_price"]

# Sort by revenue, highest first
df = df.sort_values("revenue", ascending=False)
```

None of this requires a database. None of it requires SQL. It is all Python, all in memory, and it works on files with millions of rows.

---

## Concept Explainer 1.2: The try / except Pattern — Defensive Programming

In data engineering, things go wrong constantly. A file is missing. A value that should be a number is actually the string "N/A". A database connection times out. A learner who has never written error handling will write code that crashes when any of these happen.

The `try / except` pattern is Python's way of saying: "Try to do this. If something goes wrong, do THIS instead of crashing."

```python
# WITHOUT error handling — crashes if the file doesn't exist:
df = pd.read_csv("orders.csv")

# WITH error handling — responds gracefully:
try:
    df = pd.read_csv("orders.csv")
    print(f"Loaded {len(df)} rows successfully")
except FileNotFoundError:
    print("ERROR: orders.csv not found. Check the file path.")
except Exception as e:
    print(f"Something went wrong: {e}")
```

In a pipeline that runs automatically at 2 AM, you cannot be there to fix a crash. Error handling lets the code report what went wrong and either continue (if it can) or fail cleanly with a useful message.

---

## Worked Example 1.1: Reading and Cleaning a Real-World CSV

This is the kind of task data engineers do constantly: load a messy file and produce a clean version. Read every line carefully — each step is explained.

```python
import pandas as pd

# ── STEP 1: Load the raw data ──────────────────────────────────────────────
# pd.read_csv() reads a CSV file and returns a DataFrame.
# We will work with a fictional customer orders file.

df = pd.read_csv("raw_orders.csv")

print("Raw data:")
print(df.head(3))
print(f"Shape: {df.shape}")
print()

# Imagine the output looks like this:
#    order_id  customer_name   amount   status     date
# 0      1001    John Smith    150.5  Completed  2026-01-15
# 1      1002           NaN     -20   pending    2026-01-16
# 2      1003    Mary Jones    300.0  completed  2026-01-16

# Problems we can already see:
# - Row 1001: "Completed" vs Row 1003: "completed" — inconsistent capitalisation
# - Row 1002: NaN in customer_name (missing value) and -20 amount (impossible)
# - Date is stored as a string, not a date type


# ── STEP 2: Understand the data ────────────────────────────────────────────
# dtypes shows us the data type of each column.
print("Column types:")
print(df.dtypes)

# null values
print("\nMissing values per column:")
print(df.isnull().sum())


# ── STEP 3: Fix the types ──────────────────────────────────────────────────
# Convert 'date' from a string to an actual date type.
# pd.to_datetime() understands most common date formats automatically.
df["date"] = pd.to_datetime(df["date"])

# Convert 'amount' to a number (it might have come in as text in some files)
df["amount"] = pd.to_numeric(df["amount"], errors="coerce")
# errors="coerce" means: if a value can't be converted, replace it with NaN
# instead of raising an error


# ── STEP 4: Standardise values ────────────────────────────────────────────
# Make all status values lowercase so "Completed" == "completed"
df["status"] = df["status"].str.lower().str.strip()
# .strip() removes any accidental spaces at the start or end


# ── STEP 5: Handle missing and invalid values ─────────────────────────────
# Drop rows where customer_name is missing — we cannot process nameless orders
df = df.dropna(subset=["customer_name"])

# Remove orders with negative or zero amounts — these are clearly data errors
df = df[df["amount"] > 0]


# ── STEP 6: Add derived columns ───────────────────────────────────────────
# Add a month column for reporting purposes
df["month"] = df["date"].dt.to_period("M")


# ── STEP 7: Check the result ───────────────────────────────────────────────
print("\nCleaned data:")
print(df.head())
print(f"\nRows remaining after cleaning: {len(df)}")

# Save the cleaned version
df.to_csv("clean_orders.csv", index=False)
# index=False means: don't write the row numbers (0, 1, 2...) as a column
print("Saved to clean_orders.csv")
```

This is a realistic data cleaning task. The six steps — load, understand, fix types, standardise, handle bad values, add derived columns — are the same six steps in almost every Bronze-to-Silver transformation in the skillsets that follow.

---

## Worked Example 1.2: Grouping and Aggregating Data

The second most common data engineering task after cleaning is summarising. "How much revenue per month?" "How many orders per customer?" "What is the average order value by region?"

```python
import pandas as pd

df = pd.read_csv("clean_orders.csv")
df["date"] = pd.to_datetime(df["date"])
df["month"] = df["date"].dt.to_period("M")

# ── Total revenue per month ────────────────────────────────────────────────
monthly_revenue = (
    df.groupby("month")["amount"]    # Group by month, look at amount column
    .sum()                            # Sum all amounts in each group
    .reset_index()                    # Turn the result back into a normal DataFrame
    .rename(columns={"amount": "total_revenue"})
)
print("Monthly revenue:")
print(monthly_revenue)


# ── Order count and average order value per status ────────────────────────
by_status = df.groupby("status").agg(
    order_count=("order_id", "count"),       # Count orders
    total_revenue=("amount", "sum"),          # Sum amounts
    avg_order_value=("amount", "mean"),       # Average amount
).reset_index()
print("\nBy status:")
print(by_status)


# ── Top 5 customers by total spend ────────────────────────────────────────
top_customers = (
    df.groupby("customer_name")["amount"]
    .sum()
    .sort_values(ascending=False)
    .head(5)
    .reset_index()
    .rename(columns={"amount": "total_spend"})
)
print("\nTop 5 customers:")
print(top_customers)
```

Notice the pattern: `groupby()` → aggregation function (`sum`, `mean`, `count`) → `reset_index()`. This is a pattern you will use hundreds of times.

---

## Worked Example 1.3: Merging Two DataFrames (The JOIN Equivalent)

Data engineers often need to combine data from two sources. In SQL you write a JOIN. In pandas you use `merge()`.

```python
import pandas as pd

# Two DataFrames — imagine these came from two different files or tables
orders = pd.DataFrame({
    "order_id":    [1001, 1002, 1003, 1004],
    "customer_id": ["C01", "C02", "C01", "C03"],
    "amount":      [150.0, 75.0, 300.0, 50.0],
})

customers = pd.DataFrame({
    "customer_id": ["C01", "C02", "C04"],   # Note: C03 is missing
    "name":        ["Alice", "Bob", "Carol"],
    "region":      ["North", "South", "East"],
})

# ── INNER JOIN: only orders that have a matching customer ─────────────────
inner = pd.merge(orders, customers, on="customer_id", how="inner")
print("Inner join (orders with known customers):")
print(inner)
# Order 1004 (customer C03) is dropped — C03 not in the customers table

# ── LEFT JOIN: keep ALL orders, add customer info where available ─────────
left = pd.merge(orders, customers, on="customer_id", how="left")
print("\nLeft join (all orders, NaN where customer unknown):")
print(left)
# Order 1004 appears with NaN for name and region

# ── Key insight: "how" controls the same behaviour as SQL JOIN types ──────
# how="inner" → INNER JOIN
# how="left"  → LEFT JOIN
# how="right" → RIGHT JOIN
# how="outer" → FULL OUTER JOIN
```

---

## Worked Example 1.4: Reading and Writing Different File Formats

Data engineers work with many file formats. Here is how to handle the most common ones:

```python
import pandas as pd
import json

# ── Reading CSV ────────────────────────────────────────────────────────────
df = pd.read_csv("data.csv")

# Common options:
df = pd.read_csv(
    "data.csv",
    sep="|",              # Some CSVs use | or tab instead of comma
    encoding="utf-8",     # Always specify encoding to avoid garbled text
    parse_dates=["date"], # Tell pandas to parse this column as a date
    dtype={"zip_code": str},  # Read this column as text (not a number)
)

# ── Writing CSV ────────────────────────────────────────────────────────────
df.to_csv("output.csv", index=False)


# ── Reading JSON ───────────────────────────────────────────────────────────
# JSON can be structured in different ways. The two most common:

# Option 1: Array of objects (most common from APIs)
# [ {"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"} ]
df = pd.read_json("data.json")

# Option 2: Read with Python's json library first (more control)
with open("data.json", "r") as f:
    raw = json.load(f)          # raw is now a Python dict or list
df = pd.DataFrame(raw["orders"])  # Navigate to the part you need


# ── Reading Parquet ────────────────────────────────────────────────────────
# Parquet is the standard file format in data engineering — compressed,
# fast to read, preserves column types automatically.
# You will see it constantly in SK-03 onwards.
df = pd.read_parquet("data.parquet")
df.to_parquet("output.parquet", index=False)

# Parquet requires pyarrow: pip install pyarrow


# ── Reading from a SQL database ───────────────────────────────────────────
import sqlalchemy

# Create a connection to a PostgreSQL database
engine = sqlalchemy.create_engine(
    "postgresql://username:password@host:5432/database_name"
)

# Read a whole table
df = pd.read_sql_table("orders", engine)

# Or run a specific query
df = pd.read_sql("SELECT * FROM orders WHERE status = 'completed'", engine)
```

---

## Module 1 Self-Check

Before moving to Module 2, answer these questions. If you cannot answer them confidently, re-read the relevant section.

1. You have a DataFrame with 100,000 rows. The `amount` column contains some rows with the string "N/A" instead of a number. What pandas function do you use to convert this column to numbers, and what argument prevents the function from crashing on the "N/A" values?

2. You have two DataFrames: `orders` and `customers`. Every order has a `customer_id`. Some customers in the `customers` table have never placed an order. You want a result that contains EVERY customer, with their orders if they have any. What `how=` argument do you use in `pd.merge()`?

3. What is the difference between `dropna()` and `fillna()`? When would you use each?

4. Write a one-line pandas statement that: takes a DataFrame called `df`, groups by the `region` column, and returns the sum of the `revenue` column for each region.

5. What does `index=False` mean in `df.to_csv("output.csv", index=False)`? What happens if you leave it out?

---

## Module 1 Resources

### Read First (Free, Currently Live)
1. **pandas Getting Started — Official Documentation** (pandas.pydata.org/getting-started.html)
   Read "10 minutes to pandas" in the official docs. It is genuinely 10 minutes, well-written, and covers exactly what you need. Stop at the "Plotting" section — that is not relevant here.

2. **Real Python: pandas DataFrames 101** (realpython.com/pandas-dataframe)
   A beginner-friendly walkthrough that covers creating DataFrames, accessing columns and rows, filtering, and the most common operations. Better written than the official docs for first-time readers.

3. **Python File I/O — Real Python** (realpython.com/working-with-files-in-python)
   Covers reading and writing text files, the `with open()` pattern, and how paths work. Short and practical.

### Watch (Free, Currently Live)
1. **pandas Tutorial — Corey Schafer (YouTube)** — Search "Corey Schafer pandas tutorial" on YouTube. His playlist has about 10 short episodes (15–20 min each). Watch episodes 1–5. Corey is widely regarded as the clearest Python teacher on YouTube. Each episode is focused and practical.

2. **Python for Beginners: File Handling** — Search "Mosh Hamedani Python file handling" on YouTube. A clear 15-minute explanation of reading and writing files.

### Practice (Free, Interactive)
1. **Kaggle Learn: pandas** (kaggle.com/learn/pandas) — 6 short lessons with exercises you run in the browser. No setup required. Completes in about 3 hours. This is the best hands-on practice for pandas fundamentals.

---


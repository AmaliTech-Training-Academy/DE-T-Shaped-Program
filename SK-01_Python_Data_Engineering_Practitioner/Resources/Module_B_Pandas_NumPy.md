# Module B: Data Manipulation with Pandas & NumPy

**Time:** 6-8 hours | **Focus:** DataFrames, aggregation, joins, multi-source ingestion

---

## B.1 Pandas Fundamentals

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Modern Pandas Series (Parts 1-3)](https://tomaugspurger.github.io/posts/modern-1-intro/) | 90 min | The definitive guide to idiomatic Pandas |
| [Pandas User Guide - Indexing](https://pandas.pydata.org/docs/user_guide/indexing.html) | 30 min | Official reference for slicing data |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [Complete Pandas Tutorial - Corey Schafer](https://www.youtube.com/watch?v=ZyhVh-qRZPA&list=PL-osiE80TeTsWmV9i9c58mdDCSskIFdDS) | 4 hrs | Best free Pandas tutorial (watch at 1.5x) |

---

## B.2 Advanced Operations

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Pandas GroupBy](https://realpython.com/pandas-groupby/) | 45 min | Split-apply-combine mastery |
| [Merge, Join, Concatenate](https://pandas.pydata.org/docs/user_guide/merging.html) | 30 min | Required before any merge() call |

### Quick Reference: Essential Operations

```python
# ── LOADING ──────────────────────────────────────────────
df = pd.read_csv("file.csv", dtype={"id": str}, parse_dates=["date"])
df = pd.read_parquet("file.parquet")
df = pd.read_json("file.json")

# ── INSPECTING ───────────────────────────────────────────
df.shape              # (rows, cols)
df.dtypes             # column types
df.isna().sum()       # null counts
df["col"].value_counts()

# ── FILTERING ────────────────────────────────────────────
df[df["col"] > 100]
df[df["col"].isin(["A", "B"])]
df.query("col > 100 and status == 'active'")

# ── TRANSFORMING ─────────────────────────────────────────
df["new"] = df["col"].str.strip().str.lower()
df["date"] = pd.to_datetime(df["date"], errors="coerce")
df["num"] = pd.to_numeric(df["num"], errors="coerce")

# ── AGGREGATING ──────────────────────────────────────────
df.groupby("region")["revenue"].agg(["sum", "mean", "count"])
df.groupby(["region", "month"]).agg(
    total=("revenue", "sum"),
    avg=("revenue", "mean")
)

# ── MERGING ──────────────────────────────────────────────
pd.merge(left, right, on="id", how="left", validate="m:1")

# ── EXPORTING ────────────────────────────────────────────
df.to_parquet("out.parquet", index=False, compression="gzip")
df.to_csv("out.csv", index=False, encoding="utf-8-sig")
```

### Quick Reference: Join Types

| Type | Use When | Risk |
|------|----------|------|
| `inner` | Only matching records needed | Silently drops non-matches |
| `left` | Keep all left, matched from right | Nulls on right side |
| `outer` | Keep all from both | Nulls on both sides |

**Always validate joins:**
```python
pd.merge(orders, customers, on="customer_id", how="left", validate="m:1")
# Raises error if duplicate customer_ids cause row explosion
```

---

## B.3 NumPy Essentials

### Read

| Resource | Time | Why |
|----------|------|-----|
| [NumPy Quickstart](https://numpy.org/doc/stable/user/quickstart.html) | 30 min | Arrays, operations, broadcasting |

NumPy powers Pandas under the hood. Focus on:
- Array creation and indexing
- Vectorized operations (no loops!)
- Boolean indexing

---

## B.4 Multi-Source Ingestion

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Requests Library Docs](https://requests.readthedocs.io/en/latest/) | 30 min | API ingestion essentials |
| [API Pagination Patterns](https://nordicapis.com/everything-you-need-to-know-about-api-pagination/) | 20 min | Prevents incomplete data |

### Pattern: Ingestion Functions

```python
def ingest_csv(path: Path, encoding: str = "utf-8") -> pd.DataFrame:
    """Ingest CSV with explicit encoding and type handling."""
    df = pd.read_csv(
        path,
        encoding=encoding,
        dtype={"phone": str, "id": str},  # Prevent numeric coercion
        na_values=["", "N/A", "null"],
    )
    df["source"] = "csv"
    logger.info(f"Ingested {len(df)} records from {path}")
    return df

def ingest_json_api(url: str, api_key: str) -> pd.DataFrame:
    """Ingest from paginated API."""
    all_records = []
    page = 1

    while True:
        response = requests.get(
            url,
            headers={"Authorization": f"Bearer {api_key}"},
            params={"page": page, "per_page": 100},
            timeout=30,
        )
        response.raise_for_status()
        data = response.json()

        if not data.get("items"):
            break

        all_records.extend(data["items"])
        page += 1

    return pd.json_normalize(all_records)
```

---

## Checkpoint

Before moving to Module C, you should be able to:

- [ ] Load data from CSV, JSON, and APIs
- [ ] Filter, transform, and aggregate with Pandas
- [ ] Merge DataFrames with proper validation
- [ ] Handle pagination in API ingestion

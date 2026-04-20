# SK-01 Quick Reference

One-page reference for Python data engineering essentials. Print this or keep it open while working.

---

## Project Structure
```
my_pipeline/
├── pipeline.py          # Entry point
├── config.py            # Configuration
├── ingest.py            # Data ingestion
├── clean.py             # Cleaning functions
├── validate.py          # Quality checks
├── export.py            # Output writing
├── tests/               # Unit tests
├── data/raw/            # Source files (never modify)
├── data/processed/      # Pipeline outputs
└── logs/                # Auto-generated logs
```

---

## Essential Pandas Operations

```python
# LOADING
df = pd.read_csv("file.csv", dtype={"id": str}, parse_dates=["date"])
df = pd.read_parquet("file.parquet")

# INSPECTING
df.shape                    # (rows, cols)
df.dtypes                   # column types
df.isna().sum()             # null counts
df["col"].value_counts()    # frequency

# FILTERING
df[df["col"] > 100]
df[df["col"].isin(["A", "B"])]
df.query("col > 100 and status == 'active'")

# TRANSFORMING
df["new"] = df["col"].str.strip().str.lower()
df["date"] = pd.to_datetime(df["date"], errors="coerce")

# AGGREGATING
df.groupby("region")["revenue"].agg(["sum", "mean", "count"])

# MERGING
pd.merge(left, right, on="id", how="left", validate="m:1")

# EXPORTING
df.to_parquet("out.parquet", index=False, compression="gzip")
df.to_csv("out.csv", index=False, encoding="utf-8-sig")
```

---

## Format Selection

| Use Case | Format | Why |
|----------|--------|-----|
| Analytics/BI | Parquet | 70-90% smaller, columnar |
| Business users | CSV (UTF-8 BOM) | Excel compatible |
| APIs | JSON/JSONL | Web native |
| Archival | Parquet + gzip | Self-describing |

---

## Data Cleaning Decision Tree

```
NULL VALUES
├── > 50% null → Drop column
├── Meaningful null → Keep as NaN + indicator column
└── Data entry error → Impute or infer

WRONG FORMAT
├── Numbers as strings → pd.to_numeric(errors='coerce')
├── Dates wrong format → pd.to_datetime(format=..., errors='coerce')
└── Low-cardinality → .astype("category")

DUPLICATES
├── Exact → df.drop_duplicates()
├── Near-duplicate → rapidfuzz
└── Logical (same entity) → Multi-key match
```

---

## Logging Template

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("pipeline.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)
```

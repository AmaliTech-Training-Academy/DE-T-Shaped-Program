# Module D: EDA, Visualization & Pipeline Delivery

**Time:** 4-5 hours | **Focus:** Exploratory analysis, visualization, production patterns

---

## D.1 Exploratory Data Analysis

### Read

| Resource | Time | Why |
|----------|------|-----|
| [EDA Workflow](https://towardsdatascience.com/exploratory-data-analysis-8fc1cb20fd15) | 30 min | Systematic EDA process |

### EDA Checklist

```python
# Shape and types
print(f"Rows: {len(df):,}, Columns: {df.shape[1]}")
print(df.dtypes)

# Nulls
print(df.isna().sum())

# Distributions (numeric)
print(df.describe())

# Cardinality (categorical)
for col in df.select_dtypes(include="object").columns:
    print(f"{col}: {df[col].nunique()} unique values")

# Duplicates
print(f"Duplicate rows: {df.duplicated().sum()}")
```

---

## D.2 Visualization

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Matplotlib Tutorials](https://matplotlib.org/stable/tutorials/index.html) | 20 min | Figure/Axes model |
| [Choosing the Right Chart](https://www.storytellingwithdata.com/chart-guide) | 20 min | Visual selection guide |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [Matplotlib Tutorial - Corey Schafer](https://www.youtube.com/playlist?list=PL-osiE80TeTvipOqomVEeZ1HRrcEvtZB_) | 3 hrs | Comprehensive tutorial |

### Chart Selection

| Question | Chart Type |
|----------|------------|
| Compare across groups | Horizontal bar |
| Change over time | Line chart |
| Distribution | Histogram |
| Compare distributions | Box/violin plot |
| Relationship X and Y | Scatter plot |
| Field completeness | Horizontal bar (%) |

### EDA Report Template

```python
import matplotlib.pyplot as plt

def generate_eda_report(df: pd.DataFrame, output_path: Path):
    fig, axes = plt.subplots(2, 3, figsize=(15, 10))
    fig.suptitle("Data Quality Report", fontsize=14)

    # 1. Records by category
    df["category"].value_counts().plot.barh(ax=axes[0, 0])
    axes[0, 0].set_title("Records by Category")

    # 2. Trend over time
    monthly = df.set_index("date").resample("M").size()
    monthly.plot(ax=axes[0, 1])
    axes[0, 1].set_title("Monthly Trend")

    # 3. Field completeness
    completeness = df.notna().mean().sort_values()
    completeness.plot.barh(ax=axes[1, 0])
    axes[1, 0].set_title("Field Completeness")
    axes[1, 0].axvline(x=0.95, color="red", linestyle="--")

    plt.tight_layout()
    plt.savefig(output_path, dpi=150)
    plt.close()
```

---

## D.3 Production Pipeline Patterns

### Read

| Resource | Time | Why |
|----------|------|-----|
| [ETL Pipeline Best Practices](https://towardsdatascience.com/etl-pipeline-best-practices-cc2cf5721c80) | 25 min | Idempotency, error handling |
| [Testing with pytest](https://realpython.com/pytest-python-testing/) | 35 min | Unit testing pipelines |

### Idempotency Patterns

```python
# PATTERN 1: Watermark-based incremental loading
from datetime import datetime
from pathlib import Path

def get_watermark(path: Path) -> datetime:
    if path.exists():
        return datetime.fromisoformat(path.read_text().strip())
    return datetime(2000, 1, 1)

def save_watermark(path: Path, ts: datetime):
    path.write_text(ts.isoformat())

# PATTERN 2: Output existence check
output_path = Path(f"data/processed/{datetime.now().date()}.parquet")
if output_path.exists():
    logger.info("Output exists, skipping")
else:
    run_pipeline()

# PATTERN 3: Upsert
existing = pd.read_parquet("data/customers.parquet")
new = ingest_and_clean()
merged = pd.concat([existing, new])
merged = merged.sort_values("updated_at").drop_duplicates("email", keep="last")
```

### Pipeline Testing

```python
# tests/test_clean.py
import pytest
import pandas as pd
from clean import standardize_emails, standardize_regions

class TestEmailStandardization:
    def test_lowercases(self):
        s = pd.Series(["TEST@GMAIL.COM"])
        assert standardize_emails(s).iloc[0] == "test@gmail.com"

    def test_handles_null(self):
        s = pd.Series([None, ""])
        result = standardize_emails(s)
        assert result.isna().all()

class TestRegionStandardization:
    def test_maps_variants(self):
        s = pd.Series(["US", "usa", "united states"])
        result = standardize_regions(s)
        assert (result == "US").all()
```

---

## D.4 Main Pipeline Structure

```python
def run_pipeline():
    """Main pipeline: Ingest → Clean → Validate → Export."""
    start = datetime.now()
    logger.info("Pipeline started")

    # Ingest
    raw = ingest_all_sources()
    logger.info(f"Ingested {len(raw)} records")

    # Clean
    cleaned = clean_dataframe(raw)

    # Deduplicate
    deduped = deduplicate(cleaned)

    # Validate
    report = run_quality_checks(deduped)

    # Export
    deduped.to_parquet("data/processed/golden.parquet", index=False)
    report.to_csv("data/processed/quality_report.csv", index=False)

    # Visualize
    generate_eda_report(deduped, Path("data/processed/report.png"))

    duration = (datetime.now() - start).total_seconds()
    logger.info(f"Pipeline complete in {duration:.1f}s")

if __name__ == "__main__":
    run_pipeline()
```

---

## Checkpoint

You are ready for the Lab when you can:

- [ ] Build a 6-chart EDA visualization
- [ ] Write idempotent pipeline code
- [ ] Test cleaning functions with pytest
- [ ] Structure a complete pipeline with logging

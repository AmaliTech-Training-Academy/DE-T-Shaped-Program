# Module C: Data Cleaning, Quality & Validation

**Time:** 5-6 hours | **Focus:** Cleaning patterns, deduplication, validation frameworks

---

## C.1 Data Cleaning Fundamentals

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Ultimate Guide to Data Cleaning](https://towardsdatascience.com/the-ultimate-guide-to-data-cleaning-3969843991d4) | 40 min | Taxonomy of problems with solutions |
| [String Operations in Pandas](https://pandas.pydata.org/docs/user_guide/text.html) | 30 min | The `.str` accessor reference |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [Data Cleaning with Pandas - Keith Galli](https://www.youtube.com/watch?v=iYie42M1ZyU) | 45 min | Real messy dataset walkthrough |

---

## C.2 Cleaning Decision Tree

```
Column has issues? Identify the type:

├── NULL VALUES
│   ├── > 50% null → Drop column
│   ├── Meaningful null → Keep as NaN + indicator column
│   └── Data entry error → Impute or infer

├── WRONG FORMAT / TYPE
│   ├── Numbers as strings → pd.to_numeric(errors='coerce')
│   ├── Dates wrong format → pd.to_datetime(format=..., errors='coerce')
│   └── Low-cardinality → .astype("category")

├── INCONSISTENT VALUES
│   ├── Casing → .str.lower() then .map(lookup_dict)
│   ├── Whitespace → .str.strip()
│   └── Abbreviations → Build canonical mapping

├── OUTLIERS
│   ├── Data entry errors → df["col"].clip(lower, upper)
│   └── Legitimate extremes → Keep but flag

└── DUPLICATES
    ├── Exact → df.drop_duplicates()
    ├── Near-duplicates → rapidfuzz
    └── Logical duplicates → Multi-key match + merge
```

---

## C.3 Cleaning Functions

```python
def standardize_emails(series: pd.Series) -> pd.Series:
    """Lowercase, strip whitespace, handle nulls."""
    return (
        series
        .astype(str)
        .str.strip()
        .str.lower()
        .replace({"nan": np.nan, "": np.nan})
    )

def validate_emails(series: pd.Series, pattern: str) -> pd.Series:
    """Return boolean Series: True = valid format."""
    return series.str.match(pattern, na=False)

def standardize_phone(phone: str) -> str:
    """Normalize phone: remove formatting, preserve + prefix."""
    if pd.isna(phone) or str(phone).strip() in ("", "nan"):
        return np.nan
    phone = str(phone).strip()
    has_plus = phone.startswith("+")
    digits = re.sub(r"[^\d]", "", phone)
    if len(digits) < 7:
        return np.nan
    return f"+{digits}" if has_plus else digits

REGION_MAP = {
    "us": "US", "usa": "US", "united states": "US",
    "eu": "EU", "europe": "EU", "emea": "EU",
    "apac": "APAC", "asia": "APAC", "asia pacific": "APAC",
}

def standardize_regions(series: pd.Series) -> pd.Series:
    """Map region variants to canonical values."""
    return series.str.strip().str.lower().map(REGION_MAP)
```

---

## C.4 Deduplication Strategy

```python
def deduplicate(df: pd.DataFrame, source_priority: dict) -> pd.DataFrame:
    """
    Multi-pass deduplication with source priority.

    Strategy:
    1. Sort by source priority (most trusted first)
    2. Group by email (primary key)
    3. Take first non-null value from highest priority source
    """
    df["source_priority"] = df["source"].map(source_priority)
    df = df.sort_values("source_priority")

    def merge_group(group):
        best = group.iloc[0].copy()
        for col in ["phone", "region", "name"]:
            if pd.isna(best.get(col)):
                non_null = group[col].dropna()
                if len(non_null) > 0:
                    best[col] = non_null.iloc[0]
        return best

    return df.groupby("email").apply(merge_group).reset_index(drop=True)
```

---

## C.5 Data Quality Validation

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Great Expectations Quickstart](https://docs.greatexpectations.io/docs/tutorials/quickstart/) | 30 min | Industry-standard validation |

### Simple Validator Pattern

```python
class DataQualityValidator:
    def __init__(self, df: pd.DataFrame, threshold: float = 0.95):
        self.df = df
        self.threshold = threshold
        self.results = []

    def check_not_null(self, column: str, description: str):
        failed = int(self.df[column].isna().sum())
        total = len(self.df)
        pass_rate = 1 - (failed / total)
        self.results.append({
            "check": f"NOT NULL: {column}",
            "pass_rate": pass_rate,
            "status": "PASS" if pass_rate >= self.threshold else "FAIL"
        })

    def check_unique(self, column: str, description: str):
        non_null = self.df[column].dropna()
        failed = int(non_null.duplicated().sum())
        pass_rate = 1 - (failed / len(non_null))
        self.results.append({
            "check": f"UNIQUE: {column}",
            "pass_rate": pass_rate,
            "status": "PASS" if pass_rate >= self.threshold else "FAIL"
        })

    def generate_report(self) -> pd.DataFrame:
        return pd.DataFrame(self.results)
```

---

## C.6 Memory Optimization

```python
def optimize_dtypes(df: pd.DataFrame) -> pd.DataFrame:
    """Reduce DataFrame memory by 50-90%."""
    before = df.memory_usage(deep=True).sum() / 1024**2

    # Downcast integers
    for col in df.select_dtypes(include="integer").columns:
        df[col] = pd.to_numeric(df[col], downcast="integer")

    # Downcast floats
    for col in df.select_dtypes(include="float").columns:
        df[col] = pd.to_numeric(df[col], downcast="float")

    # Convert low-cardinality strings to category
    for col in df.select_dtypes(include="object").columns:
        if df[col].nunique() / len(df) < 0.5:
            df[col] = df[col].astype("category")

    after = df.memory_usage(deep=True).sum() / 1024**2
    print(f"Memory: {before:.1f} MB → {after:.1f} MB")
    return df
```

---

## Checkpoint

Before moving to Module D, you should be able to:

- [ ] Clean messy strings (emails, phones, regions)
- [ ] Deduplicate across multiple sources with priority
- [ ] Build a validation report with pass/fail checks
- [ ] Optimize DataFrame memory usage

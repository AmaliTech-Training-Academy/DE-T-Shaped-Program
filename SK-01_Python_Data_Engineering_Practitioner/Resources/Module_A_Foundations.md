# Module A: Python & Data Engineering Foundations

**Time:** 5-6 hours | **Focus:** Project structure, logging, data formats, AI skills

---

## A.1 The Data Engineering Landscape

### Read (Choose ONE)

| Resource | Time | Why |
|----------|------|-----|
| [Fundamentals of Data Engineering, Ch 1-2](https://www.oreilly.com/library/view/fundamentals-of-data/9781098108298/) | 45 min | Clearest mapping of where Python fits in the modern data stack |
| [What is Data Engineering? - YouTube](https://www.youtube.com/watch?v=qWru-b6m030) | 20 min | Quick video overview if you prefer watching |

---

## A.2 Python Project Structure

### Read

| Resource | Time | Why |
|----------|------|-----|
| [The Hitchhiker's Guide - Project Structure](https://docs.python-guide.org/writing/structure/) | 20 min | Definitive guide to `src/`, `tests/`, `config/` organization |
| [Python Logging Deep Dive](https://realpython.com/python-logging-source-code/) | 30 min | Logging is the first thing senior engineers check |

### Template: Pipeline Project Structure

```
my_pipeline/
├── pipeline.py              # Entry point
├── config.py                # All configuration
├── ingest.py                # Source-specific ingestion
├── clean.py                 # Cleaning functions
├── validate.py              # Data quality checks
├── export.py                # Output writing
├── utils.py                 # Shared helpers
├── tests/
│   ├── test_clean.py
│   └── test_validate.py
├── data/
│   ├── raw/                 # Never modify
│   └── processed/           # Pipeline outputs
├── logs/
├── requirements.txt
└── README.md
```

### Template: Logging Setup

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

# Usage
logger.info(f"Loaded {len(df)} records")
logger.warning("Missing values detected in email column")
logger.error(f"Failed to parse file: {e}")
```

---

## A.3 Data Formats & Serialization

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Apache Parquet Format](https://parquet.apache.org/documentation/latest/) | 20 min | Industry standard for analytics |
| [Parquet vs CSV - YouTube](https://www.youtube.com/watch?v=9LYYOdIwQXg) | 12 min | Side-by-side size/speed comparison |

### Quick Reference: Format Selection

| Use Case | Format | Why |
|----------|--------|-----|
| Analytics/BI | **Parquet** (gzip) | 70-90% smaller, columnar |
| Business users | **CSV** (UTF-8 BOM) | Opens in Excel |
| APIs | **JSON/JSONL** | Web native |
| Archival | **Parquet** + gzip | Self-describing |

---

## A.4 AI Skills for Data Engineers

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) | 30 min | Official guide for effective prompts |

### Quick Reference: When to Use AI

| Task | Use AI? | Why |
|------|---------|-----|
| Classify free-text into categories | Yes | LLM with structured output |
| Standardize phone numbers | No | Regex is faster, cheaper, reliable |
| Validate email format | No | Regex |
| Infer missing country from signals | Yes | With caution, flag as inferred |
| Generate docstrings | Yes | Code generation + review |

### Pattern: Privacy-Safe AI Enrichment

```python
# WRONG: Sending PII
prompt = f"What region is {row['email']} from?"

# RIGHT: Send only signals
prompt = f"""
Infer region (US, EU, APAC) from:
- Email domain: {email_domain}
- Phone prefix: {phone_prefix}
- Currency: {currency}

Respond as JSON: {{"region": "EU", "confidence": "high"}}
"""
```

---

## Checkpoint

Before moving to Module B, you should be able to:

- [ ] Structure a Python project with proper directories
- [ ] Set up logging that writes to both file and console
- [ ] Explain why Parquet beats CSV for analytics
- [ ] Know when to use AI vs deterministic rules

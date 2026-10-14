# Lab 00 — Environment Setup

> **Goal:** Set up a professional Python data engineering workstation on Windows 11: Python, a project folder, a virtual environment with every library the ShopStream pipeline needs, and VS Code. You will also learn *why* each piece exists.
>
> **Time:** 1.5–2 hours | **Difficulty:** Beginner | **Prerequisites:** A Windows 11 laptop where you can install software. No coding experience needed.
>
> **Guided lab coverage:** [SK-01 End-to-End Guided Lab](../Labs/SK-01_End_to_End_Guided_Lab.md) — *Prerequisites* and the project folder used from *Part 1* onward.

---

## How the SK-01 Granular Labs fit together

The SK-01 guided lab has you build a customer data pipeline for **ShopStream**, an e-commerce company. The pipeline reads customer data from 4 systems, cleans it, removes duplicates, checks quality and delivers a "golden" customer file for a marketing campaign. It is one Python file, `pipeline.py`, of about 1,000 lines.

That is a lot for a beginner. These granular labs take you there in small steps:

| Lab | You learn | You add to `pipeline.py` (guided lab part) |
|---|---|---|
| 00 (this lab) | Python, venv, packages, VS Code | — (setup) |
| 01 | Core Python on ShopStream values: strings, lists, dicts, sets, functions, regex, JSON | — (practice scripts only) |
| 02 | pandas: reading CSV, JSON, fixed-width and Excel files | Parts 1–2 (setup, test data, ingestion) |
| 03 | Cleaning, deduplication and quality checks | Parts 3–5 |
| 04 | Turning the functions into a pipeline that runs end to end | Parts 7–8 (chart, export, `run_pipeline`) |
| 05 | Logging, errors, secrets, re-runs, the CRM API | Part 6 and reflection Q4–Q5 |
| 06 | Automated tests and handover | Tests for everything above |

When you finish Lab 06, your `pipeline.py` is the guided lab's pipeline, and you will understand every line of it.

---

## Learning objectives

By the end of this lab you can:

1. Install Python 3.12 and check that the right version runs.
2. Explain what a virtual environment is, and create and activate one.
3. Install packages with `python -m pip` and record them in `requirements.txt`.
4. Create the SK-01 project folder layout.
5. Point VS Code at your virtual environment and run a script from its terminal.

---

## What we're installing, and why

| Tool | What it is | Why a data engineer needs it |
|---|---|---|
| **Python 3.12** (3.11+ is fine) | A programming language, and the interpreter program that runs it | The standard language for data pipelines |
| **pip** | Python's package installer (comes with Python) | Installs libraries like pandas |
| **venv** | Python's built-in virtual environment tool | Keeps each project's libraries separate, so projects don't break each other |
| **pandas** and friends | Libraries for tables, charts, files and web requests | The tools the ShopStream pipeline is built with |
| **VS Code** | A free code editor from Microsoft | Where you write and debug code |
| **PowerShell** | Windows' command line | Pipelines are run and scheduled from the command line, not by clicking |

**Term: command line / terminal / shell.** A text window where you type commands instead of clicking. PowerShell is Windows' built-in shell. Anything you can type can be *automated*, and automation is the core of the job.

---

## Steps

### Step 1 — Open PowerShell

Press the **Windows key**, type `powershell`, press **Enter**. You'll see a prompt like:

```
PS C:\Users\YourName>
```

`PS C:\...>` is the **prompt**: type a command here. The path is your **current working directory** (the folder commands work in by default).

> **macOS/Linux:** use the Terminal app. Use `python3` where these labs say `python`, `source .venv/bin/activate` to activate, and `/` instead of `\` in paths.

### Step 2 — Check whether Python is already installed

```powershell
python --version
```

**Expected output (good):** `Python 3.12.x` (any 3.11 or newer is fine). Skip to Step 4.

**Other outcomes:**
- The **Microsoft Store opens**, or nothing prints: Windows has a placeholder `python` command. Go to Step 3.
- `python : The term 'python' is not recognized...`: Python isn't installed. Go to Step 3.
- A version older than 3.11 (e.g. `Python 3.8.10`): install a current one in Step 3.

### Step 3 — Install Python

1. Go to <https://www.python.org/downloads/windows/> and download the latest **Python 3.12** "Windows installer (64-bit)". (3.13 also works; 3.12 is what these labs were tested on.)
2. Run the installer. **On the first screen, tick "Add python.exe to PATH".** Skipping this box is the most common setup mistake.
3. Click **Install Now** and wait.
4. If the last screen offers **"Disable path length limit"**, click it. It lets Windows handle long file paths.

**Term: PATH.** A list of folders Windows searches when you type a command. "Add to PATH" means you can type `python` from any folder.

Close PowerShell completely (PATH changes only reach new windows), open a new one, and run:

```powershell
python --version
python -m pip --version
```

**Expected output:**

```
Python 3.12.x
pip 24.x from C:\Users\YourName\AppData\Local\Programs\Python\Python312\Lib\site-packages\pip (python 3.12)
```

✅ **Checkpoint:** both commands print a version, and Python is 3.11 or newer.

### Step 4 — Create the project folder

All SK-01 work lives in one folder: `C:\DataEngineering\sk01`. This is the project root used by the guided lab.

```powershell
mkdir C:\DataEngineering\sk01
cd C:\DataEngineering\sk01
mkdir data\raw, data\processed, logs, practice, tests
```

`mkdir` creates folders; `cd` ("change directory") moves into one. Your prompt should now read `PS C:\DataEngineering\sk01>`.

> **Why not a OneDrive or Desktop folder?** OneDrive syncs files while your pipeline is writing them. That can lock a file mid-write and crash the run with `PermissionError`. Keep pipeline projects in a plain local folder like `C:\DataEngineering`.

What each folder is for:

```
C:\DataEngineering\sk01\
├── data\
│   ├── raw\          # the 4 ShopStream source files (Lab 02 generates them). Never edited by hand.
│   └── processed\    # pipeline outputs: golden_customers.parquet/.csv, quality report, chart
├── logs\             # pipeline.log, written by the pipeline (Lab 05)
├── practice\         # small learning scripts from Labs 01–05
├── tests\            # automated tests (Lab 06)
├── pipeline.py       # the ShopStream pipeline (you start it in Lab 02)
└── .venv\            # the virtual environment (next step)
```

The golden rule for the whole module: **raw data is read-only.** You never change source files. You read them and write new outputs. If the pipeline has a bug, you fix the code and run it again on the untouched raw files.

✅ **Checkpoint:** `tree` shows `data\raw`, `data\processed`, `logs`, `practice` and `tests`.

### Step 5 — Create and activate a virtual environment

**Term: virtual environment (venv).** A private copy of Python with its own package folder, inside your project. Without one, every project on your laptop shares one set of packages. The day one project needs pandas 2.x and an older one needs 1.x, one of them breaks.

From `C:\DataEngineering\sk01`:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

The first line creates a `.venv` folder (about 10 seconds). The second line activates it.

**Expected:** your prompt gains a prefix:

```
(.venv) PS C:\DataEngineering\sk01>
```

`(.venv)` means `python` now refers to *this project's* environment.

**You must activate in every new PowerShell window.** If a package "suddenly isn't installed", check for `(.venv)` first. `deactivate` turns it off.

✅ **Checkpoint:** `python -c "import sys; print(sys.prefix)"` prints a path ending in `sk01\.venv`.

### Step 6 — Install the pipeline's libraries

With the venv **active**:

```powershell
python -m pip install --upgrade pip
python -m pip install "pandas>=2.2,<3" numpy matplotlib seaborn requests openpyxl pyarrow pytest
```

| Package | Used for | First used |
|---|---|---|
| `pandas` | Tables (DataFrames): read, clean, combine, write | Lab 02 |
| `numpy` | Numbers and the `np.nan` "missing" value pandas uses | Lab 02 |
| `openpyxl` | Lets pandas read Excel `.xlsx` files (the marketing source) | Lab 02 |
| `pyarrow` | Lets pandas write **Parquet**, a compressed file format for analytics | Lab 02 |
| `matplotlib`, `seaborn` | The 6-chart quality report image | Lab 04 |
| `requests` | Calling web APIs (the CRM API) | Lab 05 |
| `pytest` | Automated tests | Lab 06 |

`"pandas>=2.2,<3"` asks for pandas 2.2 or newer, but not 3.x. The guided lab needs 2.2+ features, and these labs were tested on pandas 2.2 and 2.3. The quotes stop PowerShell from treating `>` and `<` as redirection.

**Why `python -m pip` and not `pip`?** It makes sure pip installs into the Python you are running, which is your venv's.

**Expected:** several `Successfully installed ...` lines. Yellow warnings are fine; red `ERROR` lines are not.

Verify:

```powershell
python -c "import pandas, matplotlib, seaborn, openpyxl, pyarrow, requests; print('pandas', pandas.__version__)"
```

**Expected output:** `pandas 2.x.x` (2.2 or newer).

Record exactly what you installed:

```powershell
python -m pip freeze > requirements.txt
```

`requirements.txt` lists every package with its exact version. Anyone, including you on a new laptop, can rebuild the same environment with `python -m pip install -r requirements.txt`.

> **Optional, for the guided lab's AI bonus (Part 6) only:** `python -m pip install anthropic`. It also needs an API key, so skip it if you don't have one. The pipeline runs fine without it.

✅ **Checkpoint:** the import check prints a pandas version of 2.2 or newer, and `requirements.txt` exists.

### Step 7 — Install and configure VS Code

1. Download VS Code from <https://code.visualstudio.com/> and install it with the default options.
2. Open VS Code. Click **Extensions** in the left bar (four squares), search **Python** and install the one published by **Microsoft**.
3. **File → Open Folder → `C:\DataEngineering\sk01`**. Click "Yes, I trust the authors".
4. Press **Ctrl+Shift+P**, type `Python: Select Interpreter`, and choose the entry containing `.venv`.
5. Open a terminal inside VS Code with **Ctrl+`** (backtick). Look for `(.venv)` in the prompt. If it's missing, run `.\.venv\Scripts\Activate.ps1`.

### Step 8 — Prove the toolchain works

In VS Code create `practice\hello_pipeline.py`:

```python
"""First script: checks the toolchain and shows a tiny ShopStream table."""
import sys
import pandas as pd

print(f"Python : {sys.version.split()[0]}")
print(f"pandas : {pd.__version__}")

customers = pd.DataFrame({
    "email": ["maria@gmail.com", "jose@yahoo.com", "lea@company.com"],
    "region": ["US", "EU", "APAC"],
})
print(customers)
```

Run it from the project root:

```powershell
python practice\hello_pipeline.py
```

**Expected output** (your version numbers may differ):

```
Python : 3.12.x
pandas : 2.2.x
             email region
0  maria@gmail.com     US
1   jose@yahoo.com     EU
2  lea@company.com   APAC
```

✅ **Checkpoint:** you see the 3-row table. Your workstation is ready.

### Step 9 — Prove the environment is isolated

1. Open a **new** PowerShell window (not VS Code) and run `cd C:\DataEngineering\sk01`.
2. Run `python practice\hello_pipeline.py` **without** activating. If pandas isn't installed globally you get `ModuleNotFoundError: No module named 'pandas'`. That is the isolation working.
3. Activate (`.\.venv\Scripts\Activate.ps1`) and run it again. It works.

✅ **Checkpoint:** you can explain why the same command failed and then worked.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Microsoft Store opens when you type `python` | Windows "app execution alias" placeholder | Settings → Apps → Advanced app settings → App execution aliases → turn **off** `python.exe` and `python3.exe`. Open a new PowerShell. |
| `'python' is not recognized` after installing | "Add to PATH" not ticked, or an old window | Re-run the installer → Modify → tick "Add Python to environment variables". Open a **new** PowerShell. |
| `Activate.ps1 cannot be loaded because running scripts is disabled` | PowerShell blocks scripts by default | Run once: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`, answer `Y`, then activate again. |
| `ModuleNotFoundError: No module named 'pandas'` | venv not active, or VS Code uses another interpreter | Check for `(.venv)`. In VS Code, click the interpreter in the bottom-right and choose `.venv`. |
| `pip install` fails with a proxy or SSL error | Corporate network | Ask IT for the proxy address, then `python -m pip install --proxy http://proxy:port ...`. |
| `PermissionError` when a file is written later | Project inside OneDrive, or the file is open in Excel | Use `C:\DataEngineering\sk01`; close the file in Excel. |

---

## Key takeaways

- Work inside an activated venv. Look for `(.venv)` in the prompt, and re-activate in every new window.
- Install with `python -m pip`, and record versions in `requirements.txt` from day one.
- The project root is `C:\DataEngineering\sk01`. `data\raw` is read-only; outputs go to `data\processed`.
- Keep pipeline projects out of OneDrive-synced folders.

**Interview check:** *Why use a virtual environment?* Each project gets its own isolated packages, so dependencies don't clash and the environment can be rebuilt exactly from `requirements.txt`.

---

## Next

[Lab 01 — Python Foundations for Data Engineering](Lab_01_Python_Foundations_for_Data_Engineering.md): you write Python against real ShopStream problems, like messy emails, region labels and phone numbers.

# Lab 00 — Environment Setup

> **Goal:** by the end of this lab you will have a complete, professional Python data engineering
> workstation on Windows 11: Python installed, a project folder, a virtual environment with pandas,
> and VS Code configured — and you will understand *why* each piece exists.
>
> **Time:** 2–3 hours. **Prerequisites:** none. A Windows 11 laptop where you can install software.

---

## 1. Environment Setup

This entire lab *is* environment setup, so this section is the main event. Later labs will only
verify this setup and add one or two new packages.

### 1.1 What we're installing, and why

| Tool | What it is | Why a data engineer needs it |
|---|---|---|
| **Python 3.12+** | A programming language and its interpreter (the program that runs your code) | The de facto standard language for data pipelines |
| **pip** | Python's package installer (comes with Python) | Installs libraries like pandas |
| **venv** | Python's built-in virtual environment tool | Keeps each project's libraries isolated so projects don't break each other |
| **pandas** | A Python library for tabular data | The workhorse for reading, transforming, and writing data files |
| **VS Code** | A free code editor from Microsoft | Where you'll write and debug code; the most common editor in industry |
| **Windows Terminal / PowerShell** | Your command line | Pipelines are run and automated from the command line, not by clicking |

**Term: command line / terminal / shell.** A text window where you type commands instead of
clicking. PowerShell is Windows' built-in shell. Data engineers live here because anything you can
type can be *automated* — and automation is the whole job.

### 1.2 Step-by-step installation

#### Step 1 — Open PowerShell

Press the **Windows key**, type `powershell`, press **Enter**. You'll see a blue/black window with
a prompt like:

```
PS C:\Users\YourName>
```

That `PS C:\...>` is the **prompt** — it means "type a command here." The path shows your
**current working directory** (the folder commands operate in by default).

> **macOS/Linux note:** use the built-in Terminal app; commands differ slightly and are noted
> where relevant.

#### Step 2 — Check whether Python is already installed

Type this and press Enter:

```powershell
python --version
```

**Expected output (good):** something like `Python 3.12.4` (any 3.11+ is fine).

**Possible bad outcomes:**
- The **Microsoft Store opens** or you see nothing: Windows ships a fake `python` alias.
  Continue to Step 3 to install the real thing.
- `python : The term 'python' is not recognized...`: Python isn't installed. Continue to Step 3.
- A version older than 3.11 (e.g., `Python 3.8.x`): install a current version in Step 3.

#### Step 3 — Install Python

1. Go to <https://www.python.org/downloads/> in your browser.
2. Click the big yellow **Download Python 3.x.x** button (latest stable — the site always offers
   the current recommended version; if unsure which is "latest stable", it's the one that button gives you).
3. Run the downloaded installer. **On the first screen, CHECK the box "Add python.exe to PATH."**
   This is the single most common setup mistake — if you skip it, PowerShell won't find Python.
4. Click **Install Now** and wait.
5. At the end, if you see **"Disable path length limit"**, click it (it lets Windows handle long
   file paths; harmless and helpful).

**Term: PATH.** An environment variable — a list of folders Windows searches when you type a
command. "Add to PATH" means "let me type `python` from anywhere."

**Verify.** Close PowerShell completely (PATH changes only apply to new windows), open a fresh
one, and run:

```powershell
python --version
pip --version
```

Expected:

```
Python 3.12.4
pip 24.x from C:\Users\YourName\AppData\Local\Programs\Python\Python312\Lib\site-packages\pip (python 3.12)
```

> **macOS:** `brew install python` (or python.org installer) then use `python3` / `pip3`.
> **Linux:** usually preinstalled; `sudo apt install python3 python3-venv python3-pip`.

**Troubleshooting**

| Symptom | Cause | Fix |
|---|---|---|
| Microsoft Store opens when you type `python` | Windows "app execution alias" shadowing real Python | Settings → Apps → Advanced app settings → App execution aliases → turn **off** both `python.exe` and `python3.exe` entries, reopen PowerShell |
| `'python' is not recognized` after install | Forgot "Add to PATH", or old PowerShell window | Re-run installer → Modify → check "Add python to environment variables"; open a NEW PowerShell |
| `pip` not recognized but `python` works | pip not on PATH | Use `python -m pip --version` instead — `python -m pip` always works and is the professional habit anyway |

#### Step 4 — Create your module project folder

We'll keep all SK-01 work in one predictable place. In PowerShell:

```powershell
mkdir C:\DataEngineering\sk01
cd C:\DataEngineering\sk01
```

`mkdir` creates a folder; `cd` ("change directory") moves you into it. Your prompt should now read
`PS C:\DataEngineering\sk01>`.

Now create the standard subfolders a data project uses:

```powershell
mkdir data\raw, data\processed, data\rejected, logs, src, tests, notebooks
```

**Why this structure?** This is a scaled-down version of what real pipeline repositories look like:

```
C:\DataEngineering\sk01\
├── data\
│   ├── raw\          # incoming source files — NEVER edited by hand
│   ├── processed\    # pipeline outputs
│   └── rejected\     # bad records quarantined for review ("dead letters")
├── logs\             # log files the pipeline writes
├── src\              # your Python source code
├── tests\            # automated tests (Lab 06)
├── notebooks\        # exploratory scratch work
└── .venv\            # virtual environment (created next)
```

The golden rule you'll hear all module: **raw data is immutable** — you never modify source files,
you only read them and write new outputs. If your pipeline has a bug, you fix the code and re-run
against the untouched raw files.

**Verify:**

```powershell
tree /F
```

Expected: the folder tree above (minus `.venv`, which comes next).

#### Step 5 — Create and activate a virtual environment

**Term: virtual environment (venv).** A private copy of Python + its own package folder, living
inside your project. Without one, every project shares one global package set — and the day
Project A needs pandas 2.x while old Project B needs 1.x, one of them breaks. Every professional
Python project uses an isolated environment. Alternatives exist (conda, Poetry, uv) but `venv`
ships with Python and is all we need.

From inside `C:\DataEngineering\sk01`:

```powershell
python -m venv .venv
```

This takes ~10 seconds and creates a `.venv` folder. (`python -m venv` means "run the venv module";
`.venv` is the conventional name.)

**Activate it:**

```powershell
.\.venv\Scripts\Activate.ps1
```

**Expected:** your prompt gains a green prefix:

```
(.venv) PS C:\DataEngineering\sk01>
```

That `(.venv)` means: `python` and `pip` now refer to *this project's* private environment.

> **macOS/Linux:** `source .venv/bin/activate`

**Troubleshooting**

| Symptom | Cause | Fix |
|---|---|---|
| `...Activate.ps1 cannot be loaded because running scripts is disabled` | PowerShell's default execution policy blocks scripts | Run once: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`, answer `Y`, then activate again. This allows locally created scripts — a standard, safe developer setting. |
| No `(.venv)` prefix appears | Activation silently failed or wrong path | Confirm you're in `C:\DataEngineering\sk01` (`pwd`), confirm `.venv\Scripts\Activate.ps1` exists (`ls .venv\Scripts`) |
| `python -m venv .venv` errors about missing venv | Rare on Windows; common on Linux | Linux: `sudo apt install python3-venv` |

**Deactivating:** type `deactivate` any time. **Re-activating:** you must run the `Activate.ps1`
line in every new PowerShell window before working. Forgetting to activate is the #1 beginner
mistake — if pandas "suddenly isn't installed," check for the `(.venv)` prefix first.

#### Step 6 — Install pandas and friends

With the venv **active**:

```powershell
python -m pip install --upgrade pip
python -m pip install pandas pyarrow openpyxl
```

What each package is for:
- **pandas** — tabular data manipulation (the module's core tool)
- **pyarrow** — lets pandas read/write **Parquet**, the compressed columnar format used in
  production (explained properly in Lab 02)
- **openpyxl** — lets pandas read Excel `.xlsx` files (you'll need it for the capstone's payroll file)

**Expected output:** several `Successfully installed ...` lines. Warnings in yellow are fine;
red `ERROR` lines are not.

**Verify:**

```powershell
python -c "import pandas as pd; print(pd.__version__)"
```

Expected: a version like `2.2.3`. (`python -c "..."` runs a one-line program without a file —
handy for quick checks.)

**Record your dependencies:**

```powershell
python -m pip freeze > requirements.txt
```

This writes every installed package + exact version to `requirements.txt`. Anyone (including
future-you on a new laptop) can now recreate your environment with
`python -m pip install -r requirements.txt`. We'll revisit this in Lab 06.

#### Step 7 — Install and configure VS Code

1. Download from <https://code.visualstudio.com/> → run installer → accept defaults, but **check
   "Add 'Open with Code' action"** boxes (handy right-click integration).
2. Open VS Code → **Extensions** icon in the left bar (four squares) → search **Python** → install
   the one published by **Microsoft** (installs Pylance too).
3. Open your project: **File → Open Folder → `C:\DataEngineering\sk01`**. Click "Yes, I trust the authors."
4. Tell VS Code about your venv: press **Ctrl+Shift+P** → type `Python: Select Interpreter` →
   choose the entry containing `.venv` (e.g., `Python 3.12.4 ('.venv': venv)`).
5. Open a terminal inside VS Code with **Ctrl+`** (backtick). It should auto-activate the venv —
   look for `(.venv)` in the prompt.

**Verify end-to-end.** In VS Code, create a file `src\hello_pipeline.py`:

```python
"""My first data engineering script — verifies the whole toolchain."""
import sys
import pandas as pd

print(f"Python : {sys.version.split()[0]}")
print(f"pandas : {pd.__version__}")

df = pd.DataFrame({"tool": ["python", "venv", "pandas", "vscode"], "status": ["ok"] * 4})
print(df)
```

Run it from the VS Code terminal:

```powershell
python src\hello_pipeline.py
```

**Expected output:**

```
Python : 3.12.4
pandas : 2.2.3
     tool status
0  python     ok
1    venv     ok
2  pandas     ok
3  vscode     ok
```

If you see that table, **your workstation is ready**. 🎉

**Common problems at this stage**

| Symptom | Fix |
|---|---|
| `ModuleNotFoundError: No module named 'pandas'` | venv not active (no `(.venv)` prefix) or wrong interpreter selected in VS Code — redo Step 7.4 |
| VS Code runs the wrong Python | Bottom-right of VS Code shows the interpreter — click it and pick the `.venv` one |
| Squiggly underline `Import "pandas" could not be resolved` but script runs | Pylance is looking at the wrong interpreter — same fix as above |

---

## 2. Business Context

Why does a *setup* lab deserve two hours of your life?

Because in industry, **environment problems are the most common reason pipelines fail when moved
between machines.** "It works on my laptop" is a running joke precisely because it's a daily
reality: a developer builds a pipeline against pandas 2.2, the production server has 1.5, and a
function behaves differently — the nightly job silently produces wrong numbers, and Finance makes
a decision on them.

Every company running Python pipelines — banks reconciling transactions, retailers computing
inventory, hospitals reporting to regulators — depends on **reproducible environments**: the
guarantee that the same code + the same declared dependencies produce the same behavior anywhere.
The tools you just set up (`venv`, `requirements.txt`) are the entry-level version of that
guarantee; Docker and cloud environments (SK-03, SK-04) are the industrial versions, and they'll
make far more sense because you learned this first.

**Who consumes this?** Your future teammates. A repo with a clear folder structure and a
`requirements.txt` can be picked up by a new hire in ten minutes. One without can burn a day.
**What happens if it fails?** Onboarding drags, "works on my machine" bugs ship, and — worst case —
production runs different library versions than were tested.

---

## 3. Concept Explanation

### The interpreter, scripts, and the REPL

Python is an **interpreted language**: a program called the interpreter (`python.exe`) reads your
`.py` file top to bottom and executes it. You can also run the interpreter interactively — type
`python` alone and you get the **REPL** (Read-Eval-Print Loop), a prompt (`>>>`) where each line
runs immediately. Great for experiments; exit with `exit()`. Real pipelines are always scripts,
because scripts can be scheduled and version-controlled.

### Why virtual environments, really

Python packages install into a folder called `site-packages`. One global folder = one version of
each package for *everything* on your machine. Virtual environments give each project its own
`site-packages`. Pros: isolation, reproducibility, easy cleanup (delete `.venv`, done). Cons:
you must remember to activate; disk space (tens of MB — trivial). Alternatives:

| Tool | When you'd see it |
|---|---|
| `venv` (ours) | Built-in, ubiquitous, perfect for this module |
| conda | Data science teams needing non-Python binaries (C libraries, CUDA) |
| Poetry / uv | Teams wanting lockfiles + packaging in one tool; uv is a fast modern favorite |

Learn `venv` first — the concept transfers to all of them.

### Why the command line and not clicking?

Pipelines run at 2 a.m. on servers with no screen. Anything that requires a mouse cannot be
scheduled. Every skill in this module funnels toward: *a command that can run unattended*.

---

## 4. Step-by-Step Implementation

Done above (Section 1) — this lab's implementation *is* the setup. Before moving on, complete this
consolidation exercise to prove to yourself the environment is genuinely reproducible:

1. Open a **brand-new** PowerShell window (not VS Code).
2. `cd C:\DataEngineering\sk01`
3. Activate the venv: `.\.venv\Scripts\Activate.ps1`
4. Run `python src\hello_pipeline.py` — same output as before.
5. Deactivate (`deactivate`) and run `python -m pip show pandas`:
   - If pandas is **not** installed globally you'll get a warning/`Package(s) not found` — perfect,
     that *proves* your isolation works.
   - If it is found globally (maybe from an old install), notice the different `Location:` path —
     that's two separate environments coexisting.

If all five steps behave as described, you understand activation better than many working developers.

---

## 5. Production Engineering Practices

Each lab ends by connecting what you did to production habits. Today's are foundational:

- **Reproducibility (`requirements.txt`).** *Failure story:* a team's ML pipeline produced
  different forecasts on the prod server than in tests. Cause: unpinned pandas versions — a minor
  release had changed default sorting in `groupby`. A one-line `requirements.txt` pin would have
  prevented a week of investigation. From today, every project you create gets one.
- **Immutable raw data (`data\raw` is read-only by convention).** *Failure story:* an analyst
  "quickly fixed" a header directly in a source CSV. The next day's automated drop of the same file
  overwrote it and the pipeline broke again — but now nobody remembered what the manual fix was.
  Fix data in code, never in files.
- **Convention over cleverness (standard folder layout).** New engineers should recognize your
  project's shape instantly. Novel layouts cost every future reader minutes; standard ones cost nothing.
- **Isolation (venv per project).** Never `pip install` outside a venv again. Global installs are
  how machines rot.

We'll add logging, idempotency, config, and testing in Labs 04–06 — one layer at a time.

---

## 6. Reflection

### What you learned
You built a professional Python workstation: interpreter, isolated environment, package manager,
standard project layout, and an editor wired to all of it — and you can verify each layer
independently, which is exactly how you'll debug environment issues for the rest of your career.

### Why it matters
Every subsequent lab assumes this setup. More importantly, *environment fluency* is a hiring
signal: candidates who can't explain venvs get filtered out of junior DE interviews fast.

### Interview questions (with model answers)

1. **What is a virtual environment and why use one?**
   *An isolated Python environment with its own installed packages, so each project's dependencies
   don't conflict with others and the environment can be reproduced from a requirements file.*
2. **What does `pip freeze > requirements.txt` do, and what's it for?**
   *Writes all installed packages with exact versions to a file; anyone can recreate the
   environment with `pip install -r requirements.txt` — the basis of reproducible deployments.*
3. **What is PATH?**
   *An environment variable listing directories the OS searches for executables — it's why typing
   `python` works from any folder.*
4. **Why `python -m pip install` rather than plain `pip install`?**
   *It guarantees pip runs under the exact interpreter you invoked, avoiding the classic bug of
   installing packages into a different Python than the one running your code.*
5. **Why should raw source data be treated as immutable?**
   *So every pipeline run is reproducible from the original inputs; hand-edits are invisible,
   unrepeatable, and get overwritten by the next delivery.*
6. **How would you debug `ModuleNotFoundError` for a package you're sure you installed?**
   *Check which interpreter is running (`python -c "import sys; print(sys.executable)"`) and
   whether the venv is activated — nine times out of ten the package is installed in a different
   environment than the one executing.*

**Common interview trap:** "Is Python compiled or interpreted?" — Naively saying "interpreted" is
fine at junior level, but the sharp answer notes Python compiles to bytecode (`.pyc`) which the
interpreter executes; the practical point is you don't run a separate build step.

### Real-world applications
Everything: every pandas pipeline, Airflow DAG (SK-03), and AWS Lambda (SK-04) starts with a
declared, isolated, reproducible Python environment.

### Key takeaways
- Always work inside an activated venv; check for the `(.venv)` prefix.
- `python -m pip`, always.
- `requirements.txt` from day one.
- Raw data folder = read-only.
- New PowerShell window = re-activate.

**Next:** [Lab 01 — Python Foundations for Data Engineering](Lab_01_Python_Foundations_for_Data_Engineering.md),
where you actually start writing Python.

# MODULE 4: The Command Line, Git and Your Environment

## Why This Module Exists

Data engineers work in terminals, not graphical interfaces. Code lives in Git repositories, not in email attachments. Your Python environment is managed carefully to avoid version conflicts. This module gives you those foundational working habits.

---

## Module 4 Glossary

**Terminal / Command line** — A text interface for your computer. Instead of clicking on icons, you type commands. Also called: shell, bash, command prompt, CLI (Command Line Interface).

**Working directory** — The folder your terminal is currently "inside." Commands you run apply to this folder unless you specify otherwise.

**Path** — The location of a file or folder. Absolute path: starts from the root (`/Users/yourname/projects/`). Relative path: relative to the current working directory (`../data/orders.csv`).

**Git** — A version control system. It tracks every change made to your code files over time, allows multiple people to work on the same codebase, and lets you roll back to any previous version.

**Repository (repo)** — A folder tracked by Git. Contains all the code files and the complete history of changes.

**Commit** — A saved snapshot of your changes. Like a save point in a video game. Each commit has a message explaining what changed.

**Branch** — A parallel version of the codebase. You create a branch to work on a new feature without affecting the main codebase. When ready, you merge it back.

**main (or master)** — The primary branch of a repository. Production code lives here.

**Pull request (PR)** — A request to merge code from one branch into another. In a team, this triggers a code review before the merge happens.

**Clone** — Downloading a repository from a remote server (GitHub) to your local computer.

**Push** — Uploading your local commits to the remote repository (GitHub).

**Pull** — Downloading new commits from the remote repository to your local copy.

**Virtual environment** — An isolated Python installation for a specific project. The packages installed in a virtual environment do not affect your system Python or other projects.

**pip** — Python's package manager. Used to install Python libraries.

**requirements.txt** — A text file listing all the Python packages a project needs, with their versions. Allows anyone to recreate the exact environment.

**Environment variable** — A value stored in the operating system that programmes can read. Used to pass configuration (database passwords, API keys) to applications without hardcoding them in the code.

**.env file** — A file containing environment variables for local development. Never committed to Git (contains secrets).

---

## Concept Explainer 4.1: The Terminal — Navigating Without a Mouse

The first time you open a terminal, it looks empty and threatening. After a week of use, it feels faster than any graphical interface.

**The essential commands you will use every day:**

```bash
# Show where you are
pwd
# Output: /Users/yourname/projects

# List files in the current folder
ls
ls -la    # Show all files including hidden ones, with details

# Move into a folder
cd projects
cd ..         # Go up one level
cd ~          # Go to your home directory

# Create a folder
mkdir my-project

# Create an empty file
touch notes.txt

# Print the contents of a file
cat requirements.txt

# Run a Python file
python my_script.py

# Install a Python package
pip install pandas

# Show installed packages
pip list

# Find out the Python version
python --version
```

**The mental model:** Every command you type is a small programme. `ls` is a programme that lists files. `cd` is a programme that changes the working directory. You are just running programmes by name.

---

## Concept Explainer 4.2: Virtual Environments — Clean Rooms for Projects

Imagine you have two projects:
- Project A needs pandas version 1.5
- Project B needs pandas version 2.1

If you install both on the same Python, they conflict. The solution is virtual environments — a separate Python installation per project that keeps packages isolated.

```bash
# ── Create a virtual environment ──────────────────────────────────────────
# Navigate to your project folder first
cd my-project

# Create the virtual environment (a folder called .venv)
python -m venv .venv

# ── Activate it ───────────────────────────────────────────────────────────
# Mac/Linux:
source .venv/bin/activate

# Windows:
.venv\Scripts\activate

# Your terminal prompt will change to show (.venv) at the start.
# This means the virtual environment is active.

# ── Install packages ───────────────────────────────────────────────────────
pip install pandas sqlalchemy great-expectations

# ── Save the packages list ────────────────────────────────────────────────
pip freeze > requirements.txt
# This creates a file listing every installed package and its version

# ── Install from requirements.txt (on a new machine or for a colleague) ──
pip install -r requirements.txt

# ── Deactivate ────────────────────────────────────────────────────────────
deactivate
```

**The rule:** Create a new virtual environment for every project. Activate it every time you work on that project. This is standard practice and the cause of many "it works on my machine" issues when ignored.

---

## Concept Explainer 4.3: Git — Tracking Changes to Your Code

Git is version control. It records every change you make to your code, who made it, when, and why. It is also how teams collaborate without overwriting each other's work.

**The four commands you use 90% of the time:**

```bash
# ── The daily workflow ─────────────────────────────────────────────────────

# 1. Check what has changed
git status
# Shows: which files are new, which are modified, which are ready to commit

# 2. Stage your changes (mark which changes to include in the next commit)
git add my_script.py                  # Stage one file
git add .                              # Stage all changed files

# 3. Commit (save a snapshot with a message explaining what you did)
git commit -m "Add data cleaning function for order amounts"
# Good commit message: clear, specific, explains WHY not just WHAT

# 4. Push (send your commits to GitHub)
git push origin main
```

**The branching workflow (how teams work):**

```bash
# ── Working on a new feature ───────────────────────────────────────────────

# Start from the latest version of main
git checkout main
git pull origin main

# Create a new branch for your feature
git checkout -b feature/add-revenue-calculation

# ... make your changes ...
git add .
git commit -m "Add monthly revenue calculation to customer summary"

# Push your branch to GitHub
git push origin feature/add-revenue-calculation

# On GitHub, open a Pull Request to merge into main
# A colleague reviews it, approves, and it gets merged

# When done, come back to main and pull the latest
git checkout main
git pull origin main
```

**Why branching matters:** You never commit untested code directly to `main`. Your feature branch is where you experiment. Main is always working, always deployable.

---

## Concept Explainer 4.4: Environment Variables — Keeping Secrets Out of Code

A cardinal rule in data engineering: **never hardcode passwords, API keys, or database credentials in your code.**

If you write `password = "MySecret123"` in a Python file and push it to GitHub, that password is now public forever (even if you delete it later — Git remembers).

The correct approach: store secrets as environment variables.

```bash
# Set an environment variable in the terminal
export DB_PASSWORD="MySecret123"
export API_KEY="abcdef1234"

# These are now available to any programme running in this terminal session
```

```python
# In Python, read the environment variable — NEVER the actual value
import os

db_password = os.environ.get("DB_PASSWORD")
api_key     = os.environ.get("API_KEY")

if db_password is None:
    raise ValueError("DB_PASSWORD environment variable is not set")
```

**For local development, use a `.env` file:**

```bash
# Create a file called .env in your project folder
# Contents of .env:
DB_HOST=localhost
DB_PASSWORD=MyLocalPassword
API_KEY=my-dev-api-key
```

```python
# In Python, use python-dotenv to load the .env file automatically
# pip install python-dotenv
from dotenv import load_dotenv
import os

load_dotenv()   # Reads the .env file and sets environment variables

db_password = os.environ.get("DB_PASSWORD")
```

**Critical:** Add `.env` to your `.gitignore` file so it is never committed to Git.

```bash
# .gitignore — list of files Git should never track
.env
.venv/
__pycache__/
*.pyc
.DS_Store
```

---

## Module 4 Self-Check

1. You open a terminal. You type `pwd` and see `/Users/yourname`. You want to navigate to a folder called `projects` which is inside your home directory. What command do you type?

2. You are working on two Python projects simultaneously: `project-a` (needs pandas 1.5) and `project-b` (needs pandas 2.2). How do you manage this so neither project breaks the other?

3. You made changes to `transform.py` and `tests/test_transform.py`. You want to commit ONLY the changes to `transform.py` (not the test file yet). What two commands do you run?

4. A colleague asks you to use a private API key in your Python script. The key is `sk-abc123`. You need to use it in your code as `api_key = "sk-abc123"`. Why is this wrong, and what is the correct approach?

5. What does `.gitignore` do and what should always be in it for a data engineering project?

---

## Module 4 Resources

### Read First (Free, Currently Live)
1. **The Missing Semester of Your CS Education — MIT** (missing.csail.mit.edu) — Lecture notes and videos from an MIT course specifically about the tools that CS education skips: the command line, Git, text editors, and scripting. The Shell lecture and Git lecture are exactly what this module covers. Freely available online.

2. **Git Handbook — GitHub Docs** (docs.github.com/en/get-started/using-git/about-git) — GitHub's official introduction to Git concepts. Covers the core workflow (clone, commit, push, pull, branch, merge) with clear diagrams.

3. **Python Virtual Environments — Real Python** (realpython.com/python-virtual-environments-a-primer) — The most thorough beginner explanation of virtual environments available. Covers venv, pip, and requirements.txt.

### Watch (Free, Currently Live)
1. **Git and GitHub for Beginners — freeCodeCamp** — Search "freeCodeCamp git and github for beginners" on YouTube. A comprehensive 1-hour tutorial that covers everything in this module. One of the most-watched Git tutorials available.

2. **Command Line Crash Course — Traversy Media** — Search "Traversy Media command line crash course" on YouTube. A practical 30-minute introduction to the terminal that covers the commands data engineers use most.


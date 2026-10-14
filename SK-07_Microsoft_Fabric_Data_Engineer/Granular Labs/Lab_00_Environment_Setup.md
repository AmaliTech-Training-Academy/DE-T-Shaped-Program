# Lab 00 — Environment Setup: Fabric Trial, MedVista Workspaces and Sample Data

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Goal:** Start a Microsoft Fabric trial, create the two MedVista workspaces, set up `C:\DataEngineering\sk07`, and generate the three days of MedVista source data that every later lab uses.
> **Time:** 2–3 hours | **Difficulty:** Beginner | **Prerequisites:** A Microsoft Entra work or school account; Windows with PowerShell; Python 3.12 (3.11+ is fine)
> **Guided lab coverage:** "Before you start: account, capacity and cost", Part 1 Steps 1.1 (workspaces) and 1.2 (sample data)
> **Cost:** $0 on the Fabric trial. Read Section 3.3 before you create anything — it explains where Fabric can cost money and how to avoid it.

---

## Learning objectives

By the end of this lab you can:

- Explain what Microsoft Fabric is, and what a **capacity**, a **capacity unit (CU)**, a **workspace** and an **item** are.
- Start (or join) a Fabric trial and check how many days are left.
- Create workspaces and assign them to the trial capacity.
- Explain the cost model of the trial and of paid F-SKUs, and the three habits that keep both under control.
- Generate the seeded MedVista source extracts and name the data-quality problems hidden in them.

---

## 1. Environment Setup

### 1.1 What you need

| Item | Why | Check |
|---|---|---|
| Work or school account (Microsoft Entra ID) | Fabric doesn't accept personal Microsoft accounts (outlook.com, hotmail.com, gmail.com) | Sign in at [app.fabric.microsoft.com](https://app.fabric.microsoft.com) |
| A browser (Edge or Chrome) | Fabric is SaaS — everything except the two Python scripts runs in the browser | — |
| Python 3.12 | Runs the data generator now and the bed-event simulator in Lab_07 | `py -3.12 --version` or `python --version` |
| PowerShell 5.1 or 7 | All local commands in SK-07 are PowerShell | `$PSVersionTable.PSVersion` |

### 1.2 Folder layout

Everything local for SK-07 lives under one root. Avoid OneDrive-synced folders (Desktop, Documents): sync clients lock files while you work.

```
C:\DataEngineering\sk07\
├── .venv\                        Python virtual environment
├── generate_medvista_data.py     this lab
├── bed_event_simulator.py        Lab_07
└── data\source_drops\            generated CSVs (uploaded to OneLake in Lab_01)
    ├── epic\        encounters_2026-03-17.csv ...
    ├── workday\     schedules_2026-03-17.csv ...
    └── sap\         supply_orders_2026-03-17.csv ...
```

---

## 2. Business Context

### The problem this lab solves

MedVista Health runs 12 hospitals. Their patient encounters live in Epic, staff schedules in Workday, supply orders in SAP, and bed status in a home-grown system. Nobody can join these together, so CMS readmission reports are late, beds are tracked on whiteboards, and nobody can explain a 23% rise in supply costs.

Your job across SK-07 is to build the platform described in the [guided lab](../Labs/SK-07_End_to_End_Guided_Lab.md): Bronze/Silver/Gold lakehouses, a daily pipeline, a T-SQL Warehouse for analysts, and a real-time bed-occupancy view. This lab gets the ground ready: a Fabric capacity to run on, two workspaces to build in, and realistic source data to build with.

### Why environment setup matters in industry

On a real engagement the first week is often spent here: getting a capacity approved, agreeing workspace names and boundaries, and getting a representative extract of source data. Mistakes made now (wrong region, one workspace for everything, no cost guardrails) are expensive to undo later — moving a workspace with Fabric items to another region means deleting the items first.

### Who consumes the output of this lab?

Every later lab. Lab_01 uploads the CSVs you generate here; Labs 03–04 have expected row counts that only match if your data matches; Lab_06 puts the Warehouse in `medvista-analytics`.

---

## 3. Concept Explanation

### 3.1 Fabric in one paragraph

**Microsoft Fabric** is a software-as-a-service analytics platform. Instead of provisioning separate services for storage, data movement, Spark, SQL warehousing, real-time analytics and BI, you get them as **experiences** inside one product, sharing one storage layer (**OneLake**) and one pool of compute (a **capacity**). You sign in, create items in a workspace, and Fabric runs them.

### 3.2 The vocabulary you need today

| Term | Meaning | MedVista example |
|---|---|---|
| **Tenant** | Your organisation's Microsoft Entra directory | `yourcompany.onmicrosoft.com` |
| **Capacity** | A pool of compute you (or your company) pay for, sized as an F-SKU: F2, F4 … F2048 | The trial capacity |
| **Capacity unit (CU)** | The unit of capacity size. F4 = 4 CUs, F64 = 64 CUs. Every job (Spark, SQL, pipeline, KQL) consumes CU-seconds | A notebook run consumes CUs while its Spark session is alive |
| **Workspace** | A folder of items with its own access control, assigned to one capacity | `medvista-data-engineering` |
| **Item** | Anything you create: lakehouse, notebook, pipeline, warehouse, eventstream, report | `medvista_bronze` |
| **OneLake** | The single data lake under every workspace. Lakehouse tables are Delta Lake files in OneLake | Lab_01 |

**Why two workspaces?** Access is granted per workspace. Engineers need the raw (PHI-containing) Bronze data; analysts should only ever see curated Gold data. Putting the Warehouse in `medvista-analytics` means analysts get a role there and nowhere else (Lab_06).

### 3.3 Cost: what's free, what isn't, and how to stay safe

**The trial.** A Fabric trial gives you a capacity of 4 or 64 CUs for **60 days**, free, with up to 1 TB of OneLake storage. Copilot and AI features aren't included (SK-07 doesn't need them). When it ends, workspaces drop back to Power BI Pro, Fabric items become unusable, and content is kept for 7 days in case you move the workspace to a paid capacity.

**What "free" doesn't protect you from: throttling.** A capacity has a fixed size. Fabric lets short jobs burst above it and smooths usage over time, but if you keep it busy (Spark sessions left open, a pipeline on a tight schedule, a stream running all night) it starts delaying and then rejecting work — for you and anyone sharing your trial.

**Paid capacity (if your organisation gives you one).** Pay-as-you-go F-SKUs bill **per hour while the capacity is running**, busy or idle. An F2 is roughly US$0.36 per hour, about US$263 per month if never paused, in US regions at the time of writing — check the [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/?service=microsoft-fabric) for your region. **Pausing** the capacity in the Azure portal stops compute billing; OneLake storage (a few MB for this module) is still billed.

**Three habits for all of SK-07:**

1. **Stop Spark sessions** when you leave a notebook (toolbar → **Stop session**). An idle session keeps consuming CUs until it times out.
2. **Turn schedules and streams off** when a lab ends. Labs 05 and 07 tell you exactly what to switch off.
3. **Paid capacity only: pause it** whenever you stop for the day.

### 3.4 Seeded synthetic data

You don't have access to MedVista's EHR, so a Python script generates extracts that behave like real ones — including the mess. It uses a fixed random seed, so everyone in your cohort gets byte-identical files and the expected row counts in Labs 02–07 match exactly. The defects are deliberate; each one is fixed by a specific line of code in Lab_03.

---

## 4. Step-by-Step Implementation

### Step 1 — Confirm you can sign in to Fabric

1. Open [app.fabric.microsoft.com](https://app.fabric.microsoft.com) and sign in with your work or school account.
2. Select your profile picture (top right) to open the **Account manager**.

**Expected output:** the Fabric home page loads and the Account manager shows your name and a license line.

✅ **Checkpoint:** You are signed in to Fabric with a work or school account.

### Step 2 — Start (or join) the Fabric trial

1. In the Account manager, select **Start trial** (or **Free trial**).
2. In **Activate your 60-day free Fabric trial capacity**, check the **Trial capacity region**. Accept your home region unless your instructor says otherwise — you can't move Fabric items between regions later without deleting them.
3. Select **Activate**.

If you don't see **Start trial**, see the troubleshooting table. Your organisation may have disabled trials, or a colleague may share an existing trial with you (they assign your workspace to their trial capacity).

**Expected output:** a confirmation message; the Account manager now shows **Trial status** with the days remaining.

✅ **Checkpoint:** **Trial status** shows about 60 days left. (You can also see it under **OneLake catalog → Govern → Capacities → Trial**.)

### Step 3 — Create the two MedVista workspaces

1. Left navigation → **Workspaces** → **+ New workspace**.
2. Name: `medvista-data-engineering`. Description: `MedVista medallion lakehouses, notebooks, pipeline, real-time`.
3. Expand **Advanced** and choose the **Trial** license mode (workspace type). If your organisation gave you a paid capacity, choose **Fabric capacity** and select it instead.
4. Select **Apply**.
5. Repeat for `medvista-analytics` (description: `Analyst-facing Warehouse and reports`).

If you created a workspace without choosing the trial: **Workspace settings → License info → Edit → Trial → Select license**.

**Expected output:** both workspaces appear in the Workspaces list; each shows a diamond (premium/Fabric capacity) icon next to its name.

✅ **Checkpoint:** In each workspace, **+ New item** offers **Lakehouse**, **Notebook**, **Data pipeline**, **Warehouse** and **Eventhouse**. If you only see Power BI items, the workspace isn't on a Fabric capacity.

### Step 4 — Create the local project folder and virtual environment

```powershell
New-Item -ItemType Directory -Force C:\DataEngineering\sk07 | Out-Null
cd C:\DataEngineering\sk07
py -3.12 -m venv .venv            # or: python -m venv .venv
.\.venv\Scripts\Activate.ps1
python --version
```

**Expected output:** `Python 3.12.x` (3.11 or later is fine), and your prompt starts with `(.venv)`.

The generator uses only the standard library, so there's nothing to install yet. Lab_07 adds one package (`azure-eventhub`).

✅ **Checkpoint:** `python -c "import sys; print(sys.prefix)"` prints a path ending in `sk07\.venv`.

### Step 5 — Save and run the data generator

Save this as `C:\DataEngineering\sk07\generate_medvista_data.py` (it's the same script as the guided lab, Step 1.2):

```python
"""generate_medvista_data.py - MedVista sample source extracts for SK-07.

Creates three days (2026-03-17 .. 2026-03-19) of deliberately imperfect extracts
that simulate the three MedVista source systems:

  source_drops/epic/encounters_YYYY-MM-DD.csv        (stands in for the Epic EHR SFTP export)
  source_drops/workday/schedules_YYYY-MM-DD.csv      (stands in for the Workday REST API)
  source_drops/sap/supply_orders_YYYY-MM-DD.csv      (stands in for the SAP OData feed)

Standard library only. Deterministic: the same seed always produces the same files.
Usage:  python generate_medvista_data.py [output_root]
"""
import csv
import random
import sys
from datetime import datetime, timedelta
from pathlib import Path

SEED = 2026
DATES = ["2026-03-17", "2026-03-18", "2026-03-19"]
ROOT = Path(sys.argv[1]) if len(sys.argv) > 1 else Path(r"C:\DataEngineering\sk07\data")
TS = "%Y-%m-%d %H:%M:%S"

rng = random.Random(SEED)
HOSPITALS = [f"H{i:03d}" for i in range(1, 13)]
HOSP_WEIGHTS = [18, 12, 10, 6, 14, 6, 3, 3, 6, 8, 7, 7]
DEPARTMENTS = ["D-ER", "D-CARD", "D-ORTH", "D-ONC", "D-PED", "D-MED", "D-SURG", "D-ICU"]
ENC_TYPES = {  # raw spelling variants -> base type (Silver standardises them)
    "ER": "E", "ED": "E", "Emergency": "E", "EMERGENCY": "E",
    "IP": "I", "Inpatient": "I", "OP": "O", "outpatient": "O",
    "OBS": "B", "Observation": "B", " Telehealth ": "T",
}
TYPE_WEIGHTS = [10, 8, 6, 4, 9, 7, 12, 8, 4, 3, 2]
STAY_HOURS = {"E": (2, 10), "I": (24, 110), "O": (1, 4), "B": (12, 40), "T": (0.25, 1)}
CHARGES = {"E": (800, 6000), "I": (9000, 60000), "O": (150, 2500), "B": (2500, 12000), "T": (60, 300)}
DIAGNOSES = ["I21.9", "I50.9", "J18.9", "E11.9", "N39.0", "S72.001A", "K35.80", "J44.1",
             "A41.9", "R07.9", "I10", "F32.9", "O80", "M54.5", "C34.90"]
INSURANCE = ["Medicare", "Medicaid", "Commercial", "Self-Pay"]


def ts(dt):
    return dt.strftime(TS)


def write_csv(path, header, rows):
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("w", newline="", encoding="utf-8") as fh:
        w = csv.writer(fh)
        w.writerow(header)
        w.writerows(rows)


# ---------------------------------------------------------------- encounters
ENC_HEADER = ["encounter_id", "patient_id", "encounter_type", "admission_datetime",
              "discharge_datetime", "department_id", "attending_physician",
              "primary_diagnosis", "secondary_diagnoses", "hospital_id",
              "insurance_type", "total_charges"]
still_admitted = []  # encounters exported without discharge, re-sent once discharged


def new_encounter(day, seq, patient_id=None, admit=None):
    raw_type = rng.choices(list(ENC_TYPES), weights=TYPE_WEIGHTS)[0]
    base = ENC_TYPES[raw_type]
    admit = admit or day + timedelta(minutes=rng.randint(0, 23 * 60 + 59))
    lo, hi = STAY_HOURS[base]
    discharge = admit + timedelta(hours=rng.uniform(lo, hi))
    charges = round(rng.uniform(*CHARGES[base]), 2)
    if rng.random() < 0.01:
        charges = -charges                      # data-entry error -> clamped to 0 in Silver
    insurance = rng.choice(INSURANCE) if rng.random() > 0.05 else ""   # ~5% unknown
    secondary = "|".join(rng.sample(DIAGNOSES, rng.randint(0, 3)))
    return {
        "encounter_id": f"E{day:%Y%m%d}{seq:05d}",
        "patient_id": patient_id or f"P{rng.randint(1, 40000):06d}",
        "encounter_type": raw_type,
        "admit": admit, "discharge": discharge,
        "department_id": "D-ER" if base == "E" else rng.choice(DEPARTMENTS),
        "attending_physician": f"DR{rng.randint(1, 600):04d}",
        "primary_diagnosis": rng.choice(DIAGNOSES),
        "secondary_diagnoses": secondary,
        "hospital_id": rng.choices(HOSPITALS, weights=HOSP_WEIGHTS)[0],
        "insurance_type": insurance, "total_charges": charges,
    }


def enc_row(e, extract_cutoff, swap=False):
    discharge = e["discharge"] if e["discharge"] <= extract_cutoff else None
    admit = e["admit"]
    if swap:                                     # invalid: discharge before admission
        admit, discharge = e["discharge"], e["admit"]
    return [e["encounter_id"], e["patient_id"], e["encounter_type"], ts(admit),
            ts(discharge) if discharge else "", e["department_id"],
            e["attending_physician"], e["primary_diagnosis"], e["secondary_diagnoses"],
            e["hospital_id"], e["insurance_type"], f'{e["total_charges"]:.2f}']


for d in DATES:
    day = datetime.strptime(d, "%Y-%m-%d")
    cutoff = day + timedelta(days=1, hours=6)    # extract runs 06:00 next morning
    rows, seq = [], 0
    # 1) late updates: earlier encounters now discharged -> re-sent with discharge time
    for e in list(still_admitted):
        if e["discharge"] <= cutoff:
            rows.append(enc_row(e, cutoff))
            still_admitted.remove(e)
    # 2) today's admissions
    todays = []
    for _ in range(1000):
        seq += 1
        e = new_encounter(day, seq)
        todays.append(e)
    # 3) same-day readmissions (~5%): ER visit, discharged, back again later that day
    for _ in range(50):
        seq += 1
        first = rng.choice([t for t in todays if t["discharge"] < day + timedelta(hours=20)])
        admit = first["discharge"] + timedelta(hours=rng.uniform(0.5, 3))
        todays.append(new_encounter(day, seq, first["patient_id"], admit))
    for e in todays:
        swap = rng.random() < 0.01 and e["discharge"] <= cutoff
        rows.append(enc_row(e, cutoff, swap))
        if e["discharge"] > cutoff:
            still_admitted.append(e)
    rows += rng.sample(rows, 20)                 # exact duplicate rows (re-sent records)
    rng.shuffle(rows)
    write_csv(ROOT / "source_drops" / "epic" / f"encounters_{d}.csv", ENC_HEADER, rows)
    print(f"encounters_{d}.csv     rows={len(rows)}")

# ---------------------------------------------------------------- schedules
ROLES = ["Physician", "Hospitalist MD", "Emergency Doctor", "Registered Nurse (RN)",
         "Nurse Practitioner", "LPN", "Charge Nurse", "Patient Care Assistant",
         "Lab Technician", "Radiology Tech", "Nursing Aide", "Pharmacist",
         "Environmental Services", "Unit Clerk", "Medical Intern", "Endoscopy Tech"]
ROLE_WEIGHTS = [8, 4, 4, 22, 5, 6, 4, 10, 6, 5, 8, 4, 8, 6, 3, 2]
SCHED_HEADER = ["schedule_id", "employee_id", "role", "department_id", "hospital_id",
                "shift_date", "shift_start", "shift_end", "is_overtime"]
for d in DATES:
    day = datetime.strptime(d, "%Y-%m-%d")
    rows = []
    for i in range(1, 601):
        start = day + timedelta(hours=rng.choice([7, 7, 7, 11, 15, 19, 19, 23]))
        r = rng.random()
        hours = 17 if r < 0.015 else (1.5 if r < 0.025 else rng.choice([8, 8, 10, 12, 12]))
        rows.append([f"S{day:%Y%m%d}{i:04d}", f"EMP{rng.randint(1, 3000):05d}",
                     rng.choices(ROLES, weights=ROLE_WEIGHTS)[0], rng.choice(DEPARTMENTS),
                     rng.choices(HOSPITALS, weights=HOSP_WEIGHTS)[0], d, ts(start),
                     ts(start + timedelta(hours=hours)),
                     "true" if rng.random() < 0.12 else "false"])
    rows += rng.sample(rows, 6)
    rng.shuffle(rows)
    write_csv(ROOT / "source_drops" / "workday" / f"schedules_{d}.csv", SCHED_HEADER, rows)
    print(f"schedules_{d}.csv      rows={len(rows)}")

# ---------------------------------------------------------------- supply orders
CATEGORIES = ["Pharmaceuticals", "Drug - Generic", "Medication Cart", "Surgical Supplies",
              "Orthopedic Implant", "PPE - Gloves", "Mask N95", "Isolation Gown",
              "Lab Reagent", "Rapid Test Kit", "Linen", "Office Supplies"]
ITEMS = [(f"ITM{i:04d}", f"Item {i:04d}", rng.choice(CATEGORIES), round(rng.uniform(0.5, 400), 2))
         for i in range(1, 121)]
SUPPLY_HEADER = ["order_id", "item_id", "item_name", "category", "quantity", "unit_cost",
                 "total_cost", "department_id", "hospital_id", "order_date", "vendor_id"]
for d in DATES:
    day = datetime.strptime(d, "%Y-%m-%d")
    rows = []
    for i in range(1, 401):
        item_id, name, cat, cost = rng.choice(ITEMS)
        qty = 0 if rng.random() < 0.01 else rng.randint(1, 40)
        total = "" if rng.random() < 0.05 else f"{qty * cost:.2f}"    # ~5% missing total
        rows.append([f"PO{day:%Y%m%d}{i:04d}", item_id, name, cat, qty, f"{cost:.2f}", total,
                     rng.choice(DEPARTMENTS), rng.choices(HOSPITALS, weights=HOSP_WEIGHTS)[0],
                     d, f"V{rng.randint(1, 15):02d}"])
    rows += rng.sample(rows, 4)
    rng.shuffle(rows)
    write_csv(ROOT / "source_drops" / "sap" / f"supply_orders_{d}.csv", SUPPLY_HEADER, rows)
    print(f"supply_orders_{d}.csv  rows={len(rows)}")

print(f"Done. Files written under {ROOT / 'source_drops'}")
```

Run it:

```powershell
cd C:\DataEngineering\sk07
python generate_medvista_data.py
```

**Expected output:**

```
encounters_2026-03-17.csv     rows=1070
encounters_2026-03-18.csv     rows=1202
encounters_2026-03-19.csv     rows=1281
schedules_2026-03-17.csv      rows=606
schedules_2026-03-18.csv      rows=606
schedules_2026-03-19.csv      rows=606
supply_orders_2026-03-17.csv  rows=404
supply_orders_2026-03-18.csv  rows=404
supply_orders_2026-03-19.csv  rows=404
Done. Files written under C:\DataEngineering\sk07\data\source_drops
```

✅ **Checkpoint:** `Get-ChildItem -Recurse C:\DataEngineering\sk07\data\source_drops -File | Measure-Object` reports `Count : 9`. The three encounters files are roughly 120–150 KB each.

### Step 6 — Meet the defects

Look at day one of the Epic extract the way a data engineer would before writing any transformation:

```powershell
$enc = Import-Csv C:\DataEngineering\sk07\data\source_drops\epic\encounters_2026-03-17.csv
$enc.Count                                                                  # 1070

# 1. Encounter types: eleven spellings of five concepts (note the spaces in " Telehealth ")
$enc | Group-Object encounter_type -CaseSensitive | Sort-Object Name | Format-Table Name, Count

# 2. Exact duplicate rows (re-sent records)
($enc | Group-Object encounter_id | Where-Object Count -gt 1).Count         # 20

# 3. Negative charges (data-entry errors)
($enc | Where-Object { [double]$_.total_charges -lt 0 }).Count              # 10

# 4. Discharge before admission (impossible)
($enc | Where-Object { $_.discharge_datetime -ne '' -and
                       [datetime]$_.discharge_datetime -lt [datetime]$_.admission_datetime }).Count   # 9

# 5. Missing insurance, and patients still admitted (no discharge yet)
($enc | Where-Object { $_.insurance_type -eq '' }).Count                    # 53
($enc | Where-Object { $_.discharge_datetime -eq '' }).Count                # 319
```

Then the cross-day behaviour that drives the Silver MERGE in Lab_03:

```powershell
$day2 = Import-Csv C:\DataEngineering\sk07\data\source_drops\epic\encounters_2026-03-18.csv
($day2 | Where-Object { $_.encounter_id -like 'E20260317*' }).Count         # 134
```

Those 134 rows are 17 March encounters re-sent on 18 March because the patient has now been discharged — the "late update" problem in the guided lab's *Why Delta MERGE* row.

Finally, the supply extract:

```powershell
$sup = Import-Csv C:\DataEngineering\sk07\data\source_drops\sap\supply_orders_2026-03-17.csv
($sup | Where-Object { $_.total_cost -eq '' }).Count                        # 17 (derive from quantity × unit_cost)
($sup | Where-Object { $_.quantity -eq '0' }).Count                         # 4  (no cost -> filtered out)
```

✅ **Checkpoint:** Your numbers match the comments. Write down which defects you think should be *fixed* and which should be *rejected* — Lab_03 gives the answer.

### Step 7 — Optional: install the Capacity Metrics app

If you're the trial's capacity admin, install the **Microsoft Fabric Capacity Metrics** app from the Fabric/Power BI apps marketplace. It shows CU consumption per item, which is how you'll spot a forgotten Spark session or stream. Skip this if your organisation blocks app installs.

---

## 5. Production Engineering Practices

1. **Decide the region once.** Choose the capacity region where the data and its consumers are; moving Fabric items between regions means deleting and recreating them.
2. **Separate workspaces by audience and sensitivity**, not by tool. Engineering (raw PHI) and analytics (curated) are different security boundaries.
3. **Name things predictably.** `medvista_<layer>` for lakehouses, `NN_<verb>_<object>` for notebooks, `MedVista_<Purpose>` for pipelines. You'll reference these names in code.
4. **Treat cost as a requirement.** Write down who pauses the capacity, which schedules exist, and which streams run continuously.
5. **Seed your synthetic data and document its defects.** Reproducible data makes failures reproducible, and lets a reviewer check your row counts.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "This email is a personal email address" at sign-up | Personal Microsoft account | Use a work or school account; ask your organisation or instructor for one |
| No **Start trial** button in the Account manager | Trials disabled by your tenant admin, or you already have a Power BI trial | Try creating a Lakehouse in **My workspace**; if no trial prompt appears, ask your Fabric admin to enable *Users can try Microsoft Fabric paid features* or to share a capacity |
| "Trial limit reached" | Your tenant has used its quota of trial capacities | Ask a colleague who has a trial to assign your workspaces to it (they must give you Contributor permission on their trial capacity) |
| Workspace has no Lakehouse/Notebook items | Workspace is on Pro, not on Fabric capacity | Workspace settings → **License info** → **Edit** → Trial |
| `python` opens the Microsoft Store | Windows app-execution alias | Use `py -3.12`, or disable the alias in *Settings → Apps → Advanced app settings → App execution aliases* |
| `Activate.ps1 cannot be loaded because running scripts is disabled` | PowerShell execution policy | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`, then retry |
| Generator counts differ from the expected output | Edited script or different Python major version | Re-copy the script exactly; use Python 3.11+ (the random module's algorithms are stable across these versions) |
| `PermissionError` writing CSVs | Folder is OneDrive-synced or a file is open in Excel | Use `C:\DataEngineering\sk07` and close Excel |

---

## 7. Reflection

### Interview questions (answer out loud first)

1. **"What is a Fabric capacity and how is it billed?"** — A capacity is a pool of compute measured in capacity units (F2 = 2 CUs … F2048). All workloads in assigned workspaces share it. Pay-as-you-go capacities bill per hour while running (pausable); reservations give a discount for a 1-year commitment. Storage in OneLake is billed separately.
2. **"Your team shares one trial and everything is slow. What do you check?"** — The Capacity Metrics app for which items are consuming CUs: usually forgotten Spark sessions, a pipeline on a tight schedule, or a continuous stream. Stop them; separate heavy dev work onto its own capacity if it keeps happening.
3. **"Why not put all MedVista items in one workspace?"** — Workspace roles are the main access boundary. Raw PHI and analyst-facing views need different audiences; separate workspaces keep that boundary simple and auditable.

### Common interview traps

- Saying Fabric "charges per query" like a serverless warehouse. It doesn't: you pay for capacity time, and queries consume that capacity.
- Forgetting that a paused capacity still bills storage.

### Key takeaways

- Fabric = one SaaS platform, one lake (OneLake), one compute pool (capacity).
- The trial is free but finite and throttles when overused; paid capacity bills by the hour until paused.
- Workspaces are security boundaries — design them deliberately.
- Your seeded data has known defects with known counts; Lab_03 fixes them.

---

## Next

[Lab_01 — Lakehouses, OneLake and Bronze Schemas](Lab_01_Lakehouses_OneLake_and_Bronze_Schemas.md): create the three medallion lakehouses, upload today's source drops to OneLake, and create the typed Bronze tables.

# Lab 07 — Real-Time Intelligence: Eventstreams and KQL

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Goal:** Stream MedVista bed events through an Eventstream custom endpoint into an Eventhouse KQL table, and answer "which hospitals are near capacity right now?" with KQL — then shut the stream down so it stops consuming capacity.
> **Time:** 3–4 hours | **Difficulty:** Intermediate | **Prerequisites:** [Lab_06](Lab_06_Warehouse_Shortcuts_and_SQL_Views.md); Python `.venv` from Lab_00 (`C:\DataEngineering\sk07\.venv`)
> **Guided lab coverage:** Part 6, Steps 6.1–6.4, and the Teardown section
> **Cost:** $0 on the trial — but this is the one part of SK-07 that **runs continuously**. An active Eventstream and an Eventhouse receiving data consume capacity even when nobody is looking. Do the Teardown (Step 9) the same day.

---

## Learning objectives

By the end of this lab you can:

- Explain Eventstreams, Eventhouses and KQL databases, and when streaming beats a daily batch.
- Create a KQL table with an explicit schema using a management command.
- Connect a Python producer to an Eventstream **custom endpoint** with the Event Hubs protocol, keeping the connection string out of your code.
- Write KQL with `where`, `summarize`, `arg_max`, `countif`, `bin`, `next()` and `render`.
- Explain why "current state" queries need a look-back longer than the longest gap between events.
- Stop streaming resources so they stop consuming capacity.

---

## 1. Environment Setup

1. Workspace `medvista-data-engineering` → **+ New item** offers **Eventhouse** and **Eventstream**.
2. Locally:

   ```powershell
   cd C:\DataEngineering\sk07
   .\.venv\Scripts\Activate.ps1
   python -m pip install azure-eventhub
   python -c "import azure.eventhub; print(azure.eventhub.__version__)"
   ```

   **Expected:** a version number (5.x). This is the only package SK-07 installs.

---

## 2. Business Context

MedVista's bed managers phone each ward to find a free bed. Bed status changes every few minutes at peak times; a dashboard refreshed by last night's batch is useless for this. They need "how many beds are free in each hospital, right now", and an alert when a hospital passes 85% occupancy so they can divert ambulances early.

The bed management system can publish an event every time a bed changes state:

| event_type | Meaning | Bed is… |
|---|---|---|
| `ADMIT` | Patient placed in the bed | occupied |
| `DISCHARGE` | Patient left | being cleaned |
| `CLEAN` | Cleaning finished | available |
| `MAINTENANCE` | Taken out of service | unavailable |

---

## 3. Concept Explanation

### 3.1 The Real-Time Intelligence pieces

| Item | Role | MedVista |
|---|---|---|
| **Eventstream** | Receives events from sources, optionally transforms them, routes them to destinations | `medvista_bed_events` |
| **Custom endpoint** source | An Event Hubs / AMQP / Kafka-compatible endpoint your own application sends to | the bed system (here: a simulator) |
| **Eventhouse** | Container for KQL databases; scales compute for ingestion and queries | `medvista_eventhouse` |
| **KQL database** | Holds tables optimised for time-series and log data, queried with KQL | `medvista_realtime` |
| **KQL queryset / Real-Time Dashboard** | Saved queries / auto-refreshing tiles | occupancy board |

### 3.2 KQL in five operators

KQL reads top to bottom, each `|` passing a table to the next step:

```kql
bed_events                                   // start from a table
| where event_timestamp > ago(1h)            // filter rows
| summarize events = count() by hospital_id  // aggregate
| order by events desc                       // sort
| render barchart                            // visualise
```

`arg_max(event_timestamp, event_type) by bed_id` returns, for each bed, the `event_type` on the row with the latest timestamp — the bed's **current state**.

### 3.3 Events vs state, and the look-back trap

An event table records *changes*. "Current state" = the latest change per bed. If you only look at the last hour, a bed that hasn't changed for two hours — a patient sleeping in it — disappears from the count, and occupancy looks *lower* exactly when the hospital is calmest and full. The look-back must be longer than the longest gap between two events for one bed. The queries use 7 days; the Reflection discusses what happens past that.

### 3.4 `next()` needs a serialised, partition-aware order

`next(col)` reads the following row. It only means something after `sort by` (which serialises the rows), and in a list sorted by `bed_id` the "next" row can belong to a **different bed**. Turnover time (discharge → next admit) must therefore check `next(bed_id) == bed_id`.

### 3.5 Secrets

The custom endpoint's connection string contains a shared access key. Anyone with it can send events to your stream. Keep it in an environment variable, never in a script or Git; regenerate the key if it leaks.

---

## 4. Step-by-Step Implementation

### Step 1 — Create the Eventhouse, database and table

1. `medvista-data-engineering` → **+ New item** → **Eventhouse** → `medvista_eventhouse`.
2. Inside it, **+ Database** → `medvista_realtime`. Don't enable **Minimum consumption** (it keeps compute reserved, and consuming, all the time).
3. Open `medvista_realtime` and its query editor (the KQL queryset that comes with the database). Run:

   ```kql
   .create table bed_events (
       event_id: string,
       bed_id: string,
       ward_id: string,
       hospital_id: string,
       event_type: string,
       event_timestamp: datetime,
       patient_id: string
   )
   ```

**Expected output:** one row describing the new table. `bed_events | count` returns `0`.

✅ **Checkpoint:** `.show table bed_events cslschema` returns `event_id:string,bed_id:string,ward_id:string,hospital_id:string,event_type:string,event_timestamp:datetime,patient_id:string`. An explicit schema means timestamps are real datetimes, not strings, from the first event.

### Step 2 — Create the Eventstream with a custom endpoint

1. **+ New item** → **Eventstream** → `medvista_bed_events`.
2. **Use custom endpoint** → Source name `bed_management_system` → **Add**.
3. On the canvas, **Add destination** (or **Transform events or add destination**) → **Eventhouse**:
   - Ingestion mode: **Event processing before ingestion**
   - Workspace `medvista-data-engineering`, Eventhouse `medvista_eventhouse`, KQL database `medvista_realtime`
   - Destination table: existing → `bed_events`
   - Input data format: **JSON**
4. (Optional) A second destination, **Lakehouse** → `medvista_gold`, new table `bed_events_archive`, for long-term storage.
5. **Publish**.

**Expected output:** the canvas switches to live view; the source and destination nodes show as active.

### Step 3 — Copy the connection string

Select the `bed_management_system` source node → **Details** pane → **Event Hub** tab → **SAS Key Authentication** → copy **Connection string-primary key**. It looks like:

```
Endpoint=sb://eventstream-xxxxxxxx.servicebus.windows.net/;SharedAccessKeyName=key_xxxxxxxx;SharedAccessKey=xxxxxxxx;EntityPath=es_xxxxxxx
```

```powershell
$env:MEDVISTA_ES_CONN = "<paste it here>"
```

This lasts for the current PowerShell window only — nothing is written to disk.

✅ **Checkpoint:** `$env:MEDVISTA_ES_CONN.StartsWith("Endpoint=sb://")` → `True`.

### Step 4 — Save the simulator and dry-run it

Save as `C:\DataEngineering\sk07\bed_event_simulator.py` (same script as the guided lab, Step 6.3):

```python
"""bed_event_simulator.py - stands in for MedVista's bed management system.

Sends ADMIT / DISCHARGE / CLEAN / MAINTENANCE events for 240 beds (12 hospitals x
20 beds) to a Fabric Eventstream custom endpoint, using the Event Hubs protocol.

  1. Backfill: replays the last N hours (default 24) with past timestamps, so the
     KQL queries have history to work with immediately.
  2. Live: one event every ~1 second for --minutes minutes (default 15).

Usage (PowerShell):
  python bed_event_simulator.py --dry-run --minutes 1                  # print only, no Azure needed
  $env:MEDVISTA_ES_CONN = "<Connection string-primary key>"
  python bed_event_simulator.py --minutes 15

--dry-run uses the standard library only. Sending needs:  python -m pip install azure-eventhub
"""
import argparse
import json
import os
import random
import sys
import time
import uuid
from datetime import datetime, timedelta, timezone

# Target occupancy per hospital: H001, H005 and H009 run hot so the capacity alert fires.
TARGET_OCCUPANCY = {"H001": 0.97, "H002": 0.80, "H003": 0.75, "H004": 0.60,
                    "H005": 0.92, "H006": 0.70, "H007": 0.65, "H008": 0.78,
                    "H009": 0.87, "H010": 0.55, "H011": 0.40, "H012": 0.45}
WARDS_PER_HOSPITAL, BEDS_PER_WARD = 4, 5


def build_beds(rng):
    beds = {}
    for hospital, target in TARGET_OCCUPANCY.items():
        for w in range(1, WARDS_PER_HOSPITAL + 1):
            for b in range(1, BEDS_PER_WARD + 1):
                bed_id = f"{hospital}-W{w}-B{b:02d}"
                state = "ADMIT" if rng.random() < target else "CLEAN"
                beds[bed_id] = {"hospital_id": hospital, "ward_id": f"{hospital}-W{w}",
                                "state": state, "target": target}
    return beds


def next_state(bed, rng):
    """Return the next event type for this bed, or None if nothing happens."""
    s, target = bed["state"], bed["target"]
    if s == "ADMIT":
        # Discharge probability chosen so long-run occupancy is close to the target
        return "DISCHARGE" if rng.random() < (1 - target) / (1 + target) else None
    if s == "DISCHARGE":
        return "CLEAN"                                  # housekeeping always finishes
    if s == "MAINTENANCE":
        return "CLEAN" if rng.random() < 0.5 else None
    # CLEAN (available)
    if rng.random() < 0.02:
        return "MAINTENANCE"
    return "ADMIT" if rng.random() < target else None


def make_event(bed_id, bed, event_type, ts, rng):
    return {
        "event_id": str(uuid.UUID(int=rng.getrandbits(128))),
        "bed_id": bed_id,
        "ward_id": bed["ward_id"],
        "hospital_id": bed["hospital_id"],
        "event_type": event_type,
        "event_timestamp": ts.strftime("%Y-%m-%dT%H:%M:%SZ"),
        "patient_id": f"P{rng.randint(1, 40000):06d}" if event_type == "ADMIT" else "",
    }


def step(beds, ts, rng):
    bed_id = rng.choice(sorted(beds))
    bed = beds[bed_id]
    event_type = next_state(bed, rng)
    if event_type is None:
        return None
    bed["state"] = event_type
    return make_event(bed_id, bed, event_type, ts, rng)


class Sender:
    def __init__(self, conn_str, dry_run):
        self.dry_run, self.sent = dry_run, 0
        if not dry_run:
            try:
                from azure.eventhub import EventHubProducerClient, EventData
            except ImportError:
                sys.exit("azure-eventhub is not installed: python -m pip install azure-eventhub")
            self.EventData = EventData
            # The custom endpoint connection string already contains EntityPath=es_...
            self.client = EventHubProducerClient.from_connection_string(conn_str)

    def send(self, events):
        if not events:
            return
        if self.dry_run:
            for e in events:
                print(json.dumps(e))
        else:
            batch = self.client.create_batch()
            for e in events:
                try:
                    batch.add(self.EventData(json.dumps(e)))
                except ValueError:                      # batch full -> send and start a new one
                    self.client.send_batch(batch)
                    batch = self.client.create_batch()
                    batch.add(self.EventData(json.dumps(e)))
            self.client.send_batch(batch)
        self.sent += len(events)

    def close(self):
        if not self.dry_run:
            self.client.close()


def main():
    p = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    p.add_argument("--connection-string", default=os.environ.get("MEDVISTA_ES_CONN"))
    p.add_argument("--backfill-hours", type=float, default=24)
    p.add_argument("--minutes", type=float, default=15)
    p.add_argument("--seed", type=int, default=7)
    p.add_argument("--dry-run", action="store_true", help="print events instead of sending")
    args = p.parse_args()
    if not args.dry_run and not args.connection_string:
        sys.exit("Set MEDVISTA_ES_CONN or pass --connection-string (or use --dry-run)")

    rng = random.Random(args.seed)
    beds = build_beds(rng)
    sender = Sender(args.connection_string, args.dry_run)
    now = datetime.now(timezone.utc).replace(microsecond=0)

    # 1) Backfill: initial state of every bed, then ~one step per 15 simulated seconds
    t = now - timedelta(hours=args.backfill_hours)
    sender.send([make_event(b, beds[b], beds[b]["state"], t, rng) for b in sorted(beds)])
    batch = []
    while t < now:
        t += timedelta(seconds=15)
        e = step(beds, t, rng)
        if e:
            batch.append(e)
        if len(batch) >= 200:
            sender.send(batch)
            batch = []
    sender.send(batch)
    print(f"Backfill complete: {sender.sent} events covering {args.backfill_hours:g}h", file=sys.stderr)

    # 2) Live: one step per second, real timestamps
    try:
        for _ in range(int(args.minutes * 60)):
            e = step(beds, datetime.now(timezone.utc).replace(microsecond=0), rng)
            if e:
                sender.send([e])
            if not args.dry_run:
                time.sleep(1)
    except KeyboardInterrupt:
        print("Stopped by user", file=sys.stderr)
    finally:
        sender.close()
    occupied = sum(1 for b in beds.values() if b["state"] == "ADMIT")
    print(f"Done. {sender.sent} events sent. Occupied beds now: {occupied}/{len(beds)}", file=sys.stderr)


if __name__ == "__main__":
    main()
```

```powershell
python bed_event_simulator.py --dry-run --minutes 0 | Select-Object -First 2
```

**Expected output:** two JSON events (the initial state of `H001-W1-B01` and `H001-W1-B02`, timestamped 24 hours ago), then on stderr `Backfill complete: 2149 events covering 24h` and `Done. 2149 events sent. Occupied beds now: 167/240`. The simulator is seeded, so the counts are the same every time; only the timestamps change.

✅ **Checkpoint:** you can name every field in the event and say which KQL column it lands in.

### Step 5 — Stream for real

```powershell
python bed_event_simulator.py --minutes 15
```

**Expected output:** `Backfill complete: 2149 events covering 24h` within a few seconds, then the script keeps running, sending roughly one event every 3 seconds. Leave it running.

In the Eventstream live view, the source shows incoming events. In the KQL queryset:

```kql
bed_events | count
bed_events | take 5
bed_events | summarize min(event_timestamp), max(event_timestamp)
```

**Expected output:** a count of 2,149 or more (ingestion can lag a minute behind); timestamps spanning the last 24 hours.

✅ **Checkpoint:** the count grows when you re-run it a minute later.

### Step 6 — Build up the occupancy query

Start with one bed's history:

```kql
bed_events
| where bed_id == "H001-W1-B01"
| order by event_timestamp asc
| project event_timestamp, event_type, patient_id
```

Then current state per bed, then per hospital:

```kql
bed_events
| where event_timestamp > ago(7d)
| summarize arg_max(event_timestamp, event_type) by bed_id, hospital_id
| summarize total_beds = count(), occupied = countif(event_type == "ADMIT") by hospital_id
| order by hospital_id asc
```

**Expected output:** 12 rows, `total_beds` = 20 for every hospital.

Now repeat the last query with `ago(1h)` instead of `ago(7d)`.

**Expected output:** far fewer than 20 beds per hospital, and occupancy figures that make no sense. Beds whose state hasn't changed in the last hour have vanished. That's the trap from Section 3.3.

✅ **Checkpoint:** you can explain why a 1-hour window is wrong for state but fine for "admissions in the last hour".

### Step 7 — The four dashboard queries

These are the guided lab's Step 6.4 queries. Run each one separately:

```kql
// ── Query 1: Current bed occupancy across all hospitals ──────────────────────
// arg_max returns the most recent event_type per bed — its current status.
// The look-back must be longer than the longest gap between two events for one
// bed: a patient can occupy a bed for days without a new event.
bed_events
| where event_timestamp > ago(7d)
| summarize arg_max(event_timestamp, event_type) by bed_id, ward_id, hospital_id
| summarize
    total_beds    = count(),
    occupied      = countif(event_type == "ADMIT"),
    available     = countif(event_type == "CLEAN"),
    being_cleaned = countif(event_type == "DISCHARGE"),
    maintenance   = countif(event_type == "MAINTENANCE")
    by hospital_id
| extend occupancy_pct = round(todouble(occupied) / todouble(total_beds) * 100, 1)
| order by occupancy_pct desc
| render barchart with (title="Current Bed Occupancy by Hospital")


// ── Query 2: Hourly admissions trend (24 hours) ──────────────────────────────
bed_events
| where event_timestamp > ago(24h) and event_type == "ADMIT"
| summarize admits = count() by hospital_id, hour = bin(event_timestamp, 1h)
| render timechart with (title="Admissions per Hour")


// ── Query 3: Bed turnover time (discharge → next admit on the SAME bed) ──────
// WHY: Faster turnaround = more effective capacity without adding beds.
// next() reads the following row in sort order; the bed_id check stops a
// DISCHARGE on one bed being paired with an ADMIT on the next bed in the list.
bed_events
| where event_timestamp > ago(7d)
| where event_type in ("DISCHARGE", "ADMIT")
| sort by bed_id asc, event_timestamp asc
| extend next_bed = next(bed_id), next_event = next(event_type), next_time = next(event_timestamp)
| where event_type == "DISCHARGE" and next_event == "ADMIT" and next_bed == bed_id
| extend turnover_minutes = datetime_diff('minute', next_time, event_timestamp)
| summarize
    avg_turnover_min = round(avg(turnover_minutes), 0),
    p90_turnover_min = round(percentile(turnover_minutes, 90), 0)
    by hospital_id, day = bin(event_timestamp, 1d)
| project day, hospital_id, avg_turnover_min
| render timechart with (title="Daily Average Bed Turnover Time (Minutes)")


// ── Query 4: Alert — hospitals approaching capacity (>= 85%) ─────────────────
// Same 7-day state look-back as Query 1: a 1-hour window would only see beds
// that happened to change in the last hour and report nonsense occupancy.
bed_events
| where event_timestamp > ago(7d)
| summarize arg_max(event_timestamp, event_type) by bed_id, hospital_id
| summarize
    total    = count(),
    occupied = countif(event_type == "ADMIT")
    by hospital_id
| extend occupancy_pct = round(todouble(occupied) / todouble(total) * 100, 1)
| where occupancy_pct >= 85
| project hospital_id, occupancy_pct, occupied, total,
    beds_not_occupied = total - occupied,
    alert_level = case(
        occupancy_pct >= 95, "CRITICAL",
        occupancy_pct >= 90, "WARNING",
        "WATCH"
    )
| order by occupancy_pct desc
```

**Expected output:**

- **Query 1:** 12 bars; `total_beds` 20 each; H001 highest.
- **Query 2:** a time chart of admissions per hour over 24 hours.
- **Query 3:** average turnover minutes per hospital per day (the simulator's beds turn over within minutes to hours).
- **Query 4:** typically three rows — H001 `CRITICAL` (≥ 95%), H005 `WARNING` (≥ 90%), H009 `WATCH` (≥ 85%). Percentages drift while the simulator runs.

Check the turnover fix: remove `and next_bed == bed_id` from Query 3 and run it. A few extra pairs appear where one bed's last DISCHARGE was paired with the *next bed's* first ADMIT, often with negative or nonsensical minutes. Put the condition back.

✅ **Checkpoint:** Query 4 returns at least one hospital, and you can explain each alert level.

### Step 8 — Pin to a Real-Time Dashboard

In the queryset, run Query 1 → **Pin to dashboard** → **New Real-Time Dashboard** `MedVista Bed Board`. Pin Query 4 to the same dashboard. On the dashboard, set **Auto refresh** to 30 seconds and watch it update while the simulator runs.

(Optional: in the dashboard or queryset, **Set alert** on Query 4 to create an Activator rule that emails you when it returns rows. Delete the rule in Teardown.)

### Step 9 — Teardown: stop the stream

When the simulator finishes (or press Ctrl+C):

1. Eventstream `medvista_bed_events` → switch to **Edit** → toggle the source and destinations to **Deactivate** (or delete the Eventstream).
2. Delete any Activator rule you created.
3. Leave the Eventhouse without minimum consumption; with no incoming data and no queries it can suspend after a period of inactivity. If you won't use it again, delete it.
4. Close the PowerShell window (this discards `$env:MEDVISTA_ES_CONN`).
5. Paid capacity only: pause it in the Azure portal.

✅ **Checkpoint:** the Eventstream shows its source as inactive, and `bed_events | count` stops growing.

---

## 5. Production Engineering Practices

1. **Explicit schemas in streaming too.** Create the KQL table first; don't let the first event decide that a timestamp is a string.
2. **State queries need state-aware design.** Use a long enough look-back, or have the source emit periodic snapshot/heartbeat events, or maintain a "latest state" table with a materialized view (`arg_max` per bed).
3. **Partition-aware row navigation.** Any `next()`/`prev()` over a sorted list must check it's still the same entity.
4. **Secrets in environment variables or Key Vault**, never in code. Rotate keys when people leave.
5. **Streams cost money all the time.** Every running stream has an owner, a reason, and a shutdown plan.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `azure-eventhub is not installed` | Package installed outside the venv | Activate `.venv`, then `python -m pip install azure-eventhub` |
| `Set MEDVISTA_ES_CONN or pass --connection-string` | Environment variable not set in *this* PowerShell window | Set it again (Step 3) |
| `AuthenticationError` / `Unauthorized` | Copied the key instead of the full connection string, or the key was regenerated | Copy **Connection string-primary key** again |
| Events arrive in the Eventstream but `bed_events` stays empty | Destination not configured or not published; wrong table; format not JSON | Check the destination node for errors; re-select table `bed_events` and JSON; **Publish** |
| Columns are empty / null in `bed_events` | Field names in the JSON don't match column names | Keep the simulator's field names; they match the `.create table` columns exactly |
| Query 1 shows fewer than 20 beds per hospital | Look-back shorter than the data (e.g. `ago(1h)`), or the backfill hasn't been ingested yet | Use `ago(7d)`; wait a minute after the backfill |
| Query 4 returns nothing | Simulator stopped long ago and the hot hospitals have changed, or queries run before ingestion caught up | Re-run the simulator for a few minutes and query again |
| Capacity throttling after the lab | Stream left running | Do Step 9 |

---

## 7. Reflection

### Interview questions (with model answers)

1. **"Batch or streaming for bed availability?"** — Streaming: the value of the answer decays in minutes, and the question is about *current* state. Batch is fine for monthly readmission reports where the answer doesn't change by the minute.
2. **"A long-stay bed has had no event for 8 days and your query looks back 7. What happens and how do you fix it?"** — The bed disappears from the count, so occupancy is under-stated. Fix at the source (periodic snapshot or heartbeat events), or in KQL with a materialized view holding `arg_max` per bed over all time, so current state never depends on a time window.
3. **"Why check `next(bed_id) == bed_id`?"** — After sorting by bed and time, the row after a bed's last event belongs to the next bed. Without the check you pair events from different beds and compute meaningless durations.

### Common interview traps

- Putting `ago(10m)` on a *state* query because "alerts should be time-bounded". Time-bound the **events you count**, not the **state you derive**.
- Using `dcount()` for exact counts: it's an estimate (fine for millions of IDs, misleading for 20 beds). After `arg_max` there's one row per bed, so `count()` and `countif()` are exact.

### Key takeaways

- Eventstream (ingest and route) → Eventhouse KQL table (store and query) → dashboard/alert (act).
- Current state = `arg_max` per entity over a look-back longer than any gap between events.
- `next()` needs a sort and an entity check.
- Streaming resources run continuously: deactivate them when you're done.

---

## Next

You've built every part of the [guided lab](../Labs/SK-07_End_to_End_Guided_Lab.md). Answer its five Reflection Questions, then move on to the [SK-07 Capstone Project](../Project/SK-07_Capstone_Project.md).

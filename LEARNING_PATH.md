# Data Engineering Learning Path

A visual guide to navigating the curriculum.

---

## The Journey

```
                          YOUR DATA ENGINEERING JOURNEY
    ═══════════════════════════════════════════════════════════════

    START HERE
        │
        ▼
    ┌─────────────────────────────────────────────────────────────┐
    │                    BRIDGE PROGRAMME                         │
    │                      (3 weeks)                              │
    │  Python for DE • SQL Beyond Basics • Cloud Literacy • CLI   │
    └─────────────────────────────────────────────────────────────┘
        │
        ▼
    ══════════════════════════════════════════════════════════════
                         CORE FOUNDATION
                     (Everyone completes this)
    ══════════════════════════════════════════════════════════════
        │
        ▼
    ┌──────────────────┐
    │     SK-01        │
    │  Python DE       │  ←── Start your first skillset here
    │  Practitioner    │
    │    (4 weeks)     │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │     SK-02        │
    │  SQL & Data      │
    │   Modeling       │
    │    (4 weeks)     │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │     SK-03        │
    │  Batch ETL &     │
    │  Orchestration   │
    │    (4 weeks)     │
    └────────┬─────────┘
             │
             ▼
    ══════════════════════════════════════════════════════════════
                       CHOOSE YOUR PLATFORM
                    (Pick ONE track to start)
    ══════════════════════════════════════════════════════════════
             │
       ┌─────┴─────┐
       ▼           ▼
    AWS TRACK    MICROSOFT TRACK
       │              │
       ▼              ▼
    ┌──────────┐  ┌──────────┐
    │  SK-04   │  │  SK-06   │
    │  AWS     │  │ Power BI │
    │  Cloud   │  │ Analytics│
    │(4 weeks) │  │(3 weeks) │
    └────┬─────┘  └────┬─────┘
         │             │
         ▼             ▼
    ┌──────────┐  ┌──────────┐
    │  SK-05   │  │  SK-07   │
    │ Real-Time│  │ Microsoft│
    │ Streaming│  │  Fabric  │
    │(4 weeks) │  │(4 weeks) │
    └────┬─────┘  └────┬─────┘
         │             │
         └──────┬──────┘
                │
                ▼
    ══════════════════════════════════════════════════════════════
                        ADVANCED LAYER
                   (After completing a platform track)
    ══════════════════════════════════════════════════════════════
                │
        ┌───────┼───────┐
        ▼       ▼       ▼
    ┌───────┐┌───────┐┌───────┐
    │ SK-08 ││ SK-09 ││ SK-10 │
    │Lake-  ││Govern-││DataOps│
    │house  ││ance & ││ CI/CD │
    │Archit.││Securit││       │
    │(3 wks)││(3 wks)││(3 wks)│
    └───┬───┘└───┬───┘└───┬───┘
        └────────┼────────┘
                 │
                 ▼
    ┌──────────────────┐
    │     SK-11        │
    │  NoSQL &         │
    │  DynamoDB        │
    │    (3 weeks)     │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │     SK-12        │  ←── CAPSTONE SKILLSET
    │  ML & Data       │
    │  Science for DE  │
    │    (4 weeks)     │
    └──────────────────┘
             │
             ▼
    ══════════════════════════════════════════════════════════════
                        COMPLETION
    ══════════════════════════════════════════════════════════════
```

---

## Time Estimates

| Phase | Skillsets | Duration | Hours |
|-------|-----------|----------|-------|
| Bridge | Foundation | 3 weeks | ~45 hrs |
| Core | SK-01, SK-02, SK-03 | 12 weeks | ~150 hrs |
| Platform (AWS) | SK-04, SK-05 | 8 weeks | ~100 hrs |
| Platform (Microsoft) | SK-06, SK-07 | 7 weeks | ~85 hrs |
| Advanced | SK-08, SK-09, SK-10, SK-11, SK-12 | 16 weeks | ~200 hrs |
| **Total (AWS path)** | | **~39 weeks** | **~495 hrs** |
| **Total (Microsoft path)** | | **~38 weeks** | **~480 hrs** |

**Recommended pace:** 15-20 hours/week for working professionals

---

## Skillset Dependencies

```
SK-01 Python ──────┬──→ SK-02 SQL ──────→ SK-03 Batch ETL
                   │
                   └──→ No other skillset should be attempted
                        before completing SK-01

SK-03 Batch ETL ───┬──→ SK-04 AWS ──────→ SK-05 Streaming
                   │
                   └──→ SK-06 Power BI ──→ SK-07 Fabric

SK-04 or SK-07 ────┬──→ SK-08 Lakehouse
                   ├──→ SK-09 Governance
                   └──→ SK-10 DataOps

SK-08 + SK-09 ─────→ SK-11 NoSQL ──────→ SK-12 ML
```

---

## How to Use This Map

1. **Always start with Bridge Programme** (unless you pass the prerequisites check)
2. **Complete Core Foundation in order** (SK-01 → SK-02 → SK-03)
3. **Choose ONE platform track** based on your career goals or employer needs
4. **Advanced layer unlocks** after completing any platform track
5. **Badge earned** upon completing each skillset's capstone

---

## Progress Tracker

Copy this to track your journey:

```
[ ] Bridge Programme
    [ ] Module 1: Python for Data Engineering
    [ ] Module 2: SQL Beyond Basics
    [ ] Module 3: Cloud & Infrastructure Literacy
    [ ] Module 4: Command Line & Git
    [ ] Module 5: The Modern Data Stack

[ ] SK-01: Python Data Engineering Practitioner
    [ ] Resources (Modules A-D)
    [ ] Guided Lab
    [ ] Capstone → Badge Earned: ___/___/____

[ ] SK-02: SQL and Data Modeling Specialist
    [ ] Resources
    [ ] Guided Lab
    [ ] Capstone → Badge Earned: ___/___/____

... (continue for all skillsets)
```

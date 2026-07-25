# AI Employee: Onboarding & Retention Orchestrator

**Autopilot Asia 2026 Hackathon — Track 5: HR & People Ops**  
**Role:** Technical lead — architecture, integration, and orchestration  
**Collaborator:** Accounting major (domain logic & business rules)  
**Build window:** 48 hours

---

## What this is

An AI Employee that watches every new hire's 90-day onboarding journey, detects early risk signals, and triggers governed interventions before at-risk hires quit — without exposing sensitive employee disclosures to the wrong people. This is to address issues faced by employees before deciding to quit the company.

**Outcome metric:** Task completion rate + Day-90 retention prediction across a 60-hire cohort.

---

## The problem it solves

HR teams manually track onboarding tasks, IT provisioning, and engagement scores across dozens of new hires simultaneously. At-risk signals (blocked laptop access, stalled compliance training, declining pulse scores) are often caught too late — after a hire has already disengaged or quit.

This system automates detection and intervention, with one hard constraint: **sensitive employee disclosures (health, harassment, welfare) must route confidentially to HR only — never to a manager, never to a cohort report.**

---

## Architecture

7-operator system coordinated by a central Orchestrator:

New Hire Trigger (Employee_ID)
↓
Intake & Data Prep ← validates hire record, starts 90-day clock
↓
Manager Lookup ← fetches manager contact from directory
↓
┌──────────────────────────────────────┐
│ PARALLEL FAN-OUT │
│ Provisioning Watch Operator │ ← detects blocked day-one access
│ Task Cadence Operator │ ← identifies stalled/overdue tasks
│ Pulse & Sentiment Operator │ ← flags disengagement + sensitive disclosures
└──────────────────────────────────────┘
↓
Operator Synchronization ← waits for ALL 3 before proceeding
↓
Risk Scoring Operator ← combines signals → risk_level
↓
BRANCH:
├─ On Track → Log & Continue
├─ At Risk → Intervention & Escalation → Slack nudge to manager
├─ Sensitive → Intervention & Escalation → Confidential Outlook to HR only
└─ Needs Review → Auto Workbench Escalation → Human review
↓
Cohort / Retention Report ← cohort-wide outcome metrics
↓
Final Report
---

## Operator roles

| Operator | Job | Data source |
|---|---|---|
| Intake & Data Prep | Register hire, compute day_in_program, validate record | `workers` table |
| Provisioning Watch | Detect Blocked/missing day-one resources (Laptop/Email/System Access) | `provisioning_integration` |
| Task Cadence | Detect stalled/overdue onboarding steps, flag compliance gaps | `onboarding_tasks` |
| Pulse & Sentiment | Flag disengaged hires, screen sensitive disclosures (semantic, not keyword) | `peakon_engagement` |
| Risk Scoring | Combine signals → On Track / At Risk / Needs Review | No direct DB read — pure logic |
| Intervention & Escalation | Route: Slack nudge (manager) or confidential Outlook (HR only) | `manager_directory` + Slack + Outlook |
| Cohort/Retention Report | Cohort-wide outcome metrics | All tables |

---

## Key design decisions

**Parallel fan-out, not sequential**  
Provisioning Watch, Task Cadence, and Pulse & Sentiment run simultaneously — synchronized by an explicit fan-in gate before Risk Scoring. This keeps per-hire processing fast regardless of how many operators are running.

**Status-based logic, not date arithmetic**  
The dataset contained a real data trap: 187 of 213 "Fulfilled" provisioning rows had `Fulfilled_On` before `Requested_On` (mixed timezone bug). All operators decide purely on `Status` field values — never raw date subtraction.

**Sensitive disclosure isolation**  
Pulse & Sentiment uses semantic analysis (not keyword matching) to flag sensitive comments. If `sensitive: true`, the comment text is never copied into any output — only a minimal flag is passed downstream. The confidential route (Outlook to HR) is completely isolated from the Slack/manager path.

**Configurable thresholds — no code changes needed**  
Key decision parameters are editable via environment variables:
- `AT_RISK_PULSE_FLOOR` — pulse score that alone triggers at-risk
- `STALLED_COUNT_THRESHOLD` — how many stalled tasks = medium signal
- `disengaged_threshold` — pulse score below which = disengaged
- `medium_signals_for_at_risk` — how many medium signals → at risk

**Human-in-the-loop, not silent automation**  
Every at-risk intervention pauses at a Workbench review step — a manager or HR person must confirm the action was taken before the run finalizes.

---

## Stack

| Layer | Technology |
|---|---|
| Orchestration platform | Supervity Autos |
| System of Record | Supabase (PostgreSQL) |
| Manager nudge channel | Slack |
| Confidential HR channel | Microsoft Outlook |
| Human review | Supervity Auto Workbench |

---

## Dataset schema

Synthetic HR dataset (60 hires, 5 tables) provided by hackathon organizers.

**workers** — master hire record  
`Employee_ID, Worker_WID, Legal_Name, Preferred_Name, Business_Title, Job_Profile, Job_Family, Management_Level, Position_ID, Manager_Name, Manager_WID, Cost_Center, Location, Hire_Date, Worker_Type, Time_Type, FTE, Email_Work`

**onboarding_tasks** — per-hire task milestones  
`Event_ID, Employee_ID, Business_Process, Step_Name, Milestone, Due_Date, Status, Completed_Date, Assigned_To_Role`

**provisioning_integration** — IT resource provisioning  
`Integration_Event_ID, Employee_ID, Resource, Requested_On, Status, Fulfilled_On`

**peakon_engagement** — pulse survey responses  
`Response_ID, Employee_ID, Survey_Round, Milestone, Driver, Score, Comment, Submitted_At`

**manager_directory** — manager contacts for escalation  
`Manager_WID, Employee_ID, Name, Email_Work, Org`

---

## Known data traps (tested and handled)

| Trap | How it's handled |
|---|---|
| Mixed-timezone dates in provisioning (Fulfilled_On before Requested_On) | Status-only logic, never date arithmetic |
| Disengagement comment mistaken for sensitive disclosure | Semantic classification — "not the right fit" = disengagement, not confidential |
| Genuine sensitive disclosures (health, harassment, conduct) | Flagged confidentially, comment text never copied downstream |
| Missing manager email | Graceful fallback — no message sent, Workbench flagged instead |
| Missing/null hire date | Intake escalates to Workbench, workflow stops rather than guessing |

---

## What I learned

**Orchestrating distributed components is fundamentally harder than building individual ones.**

Each of the 7 operators worked reliably in isolation — tested individually with real data, real integrations, confirmed outputs. Coordinating them in a single orchestration layer exposed real distributed systems challenges:

- Platform-level execution state vs. application-level logic
- Field name mismatches across component boundaries
- Timeout enforcement at code level vs. platform level (different layers)
- Wrong join keys discovered only when running end-to-end
- Silent failures vs. loud errors — sequential bugs are easier to diagnose than distributed ones

These are the same problems that motivated Kubernetes, message queues, and service meshes in production systems. I now understand why those tools exist.

Working with a non-technical collaborator (accounting background) also taught me that the hardest translation in systems work isn't between programming languages — it's between business logic and technical implementation. Getting the sensitive disclosure routing right required genuinely understanding the HR compliance reasoning behind it, not just building what was asked.

---

## Files in this repo
/exports — Supervity operator JSON exports (all 7 operators + orchestrator)
/architecture — workflow diagrams and system overview screenshots
/dataset — schema documentation (no real data uploaded)
README.md — this file

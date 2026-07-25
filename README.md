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
## What it does

New hires are tracked from Day 1 through their onboarding milestones. The system:

1. **Fetches** the hire's record and calculates their current milestone window
2. **Checks three signals in parallel**: IT provisioning gaps, stalled onboarding tasks, and engagement sentiment (including *semantic* — not keyword — detection of sensitive personal disclosures)
3. **Scores risk** against configurable thresholds
4. **Routes automatically**, in priority order:
   - A sensitive disclosure → confidential route straight to HR, bypassing the manager entirely, regardless of overall risk score
   - An operational risk (stalled tasks, access gaps) → a Slack/Outlook nudge to the manager
   - Anything ambiguous or with missing data → a human review form (never fabricates a result)
5. **Aggregates** company-wide retention metrics into a report, delivered via email

---
## Architecture

7-operator system coordinated by a central Orchestrator:

Intake & Data Prep → Manager Lookup
→ [Provisioning Watch ∥ Pulse & Sentiment ∥ Task Cadence] (parallel)
→ Risk Scoring
→ [Confidential Route | Manager Nudge | Human Review | Log & Continue] (branch)
→ Final Report → Send HR Report Email


**Design principles applied throughout:**
- **Tools vs. policy, kept separate.** Every operator that fetches data returns facts, not decisions. Thresholds (`AT_RISK_PULSE_FLOOR`, `STALLED_COUNT_THRESHOLD`, `disengaged_threshold`) live as editable configuration, not buried in prompt text — a business user can retune the system without touching code.
- **Never fabricate on missing data.** Every operator escalates to a human Workbench form on invalid or incomplete input, rather than guessing.
- **Retry once, then escalate.** Explicit rule enforced at the Orchestrator level for every subworkflow call.
- **Semantic judgment only where it belongs.** Sensitive-disclosure detection uses contextual understanding, not keyword matching — deliberately, since the test cases include indirect language a keyword filter would miss. Everything else (thresholds, routing) is deterministic.

See `/operators` for the exported operator definitions (real, working configuration — not pseudocode).

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

See `/operators` for the exported operator definitions (real, working configuration — not pseudocode).
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

## Real bugs hit and fixed

This wasn't a clean build — three genuine bugs surfaced during integration testing, each diagnosed from actual data/logs rather than guessed at:

1. **Field-name mismatch** — Risk Scoring was written expecting `pulse_failed`/`task_failed` flags that the upstream operators never actually produced; the real fields were `disengaged`, `sensitive`, `task_stalled_count`, etc. Traced by comparing the Orchestrator's actual data handoff against each operator's real output.
2. **Silent parallel-execution regression** — an early fix for a stalled operator accidentally serialized what should have been three concurrent checks. Caught by comparing execution timestamps in the audit trail, not by assuming a fix worked.
3. **Manager lookup join mismatch** — an operator was built expecting a `Manager_ID` column that didn't exist in the dataset; the actual join key was `Manager_WID`. Found by reading the raw CSV schema directly rather than trusting an AI-generated diagnosis at face value.
---
## Known test coverage

The dataset includes deliberately planted edge cases, each designed to test one specific behavior:

| Employee | Designed to test | Result |
|---|---|---|
| **EMP7001** | Clean onboarding, no gaps — the happy path | ✅ Confirmed: correctly resolved On Track, no escalation triggered |
| **EMP7000** | Missing provisioning (laptop/access blocked) | ✅ Confirmed: correctly flagged At Risk and escalated |
| **EMP7003 (Tariq)** | An indirect, non-keyword health-related disclosure — must bypass normal risk scoring and route confidentially to HR regardless of risk level | ⚠️ Confidential email delivery confirmed working through the full Orchestrator; the sensitive-flag detection itself was validated at the operator level but not re-confirmed on this exact case end-to-end |
| **EMP7005 (Sarah)** | A second, differently-worded sensitive disclosure — proves the detection isn't overfit to one specific phrasing | ⬜ Not re-tested after the final build; planned but not completed |
| **EMP7007 (Mei)** | Disengaged and clearly at-risk, but *not* sensitive — the sharpest edge case: must route to the manager, and must **not** trigger the confidential HR path | ⬜ Not tested — this is the single most valuable remaining test, since it's the one that would prove the sensitive/non-sensitive distinction doesn't over-trigger |

**Also confirmed:**
- Structural parallelism — verified directly from the dependency graph (Provisioning Watch, Pulse & Sentiment, and Task Cadence each depend only on Manager Lookup, not on each other)
- Retry-then-escalate logic — present and enforced at the Orchestrator level for every subworkflow call

**Honestly, what's left to verify, in priority order:** EMP7007 (Mei) first — it's the case most likely to expose a false positive in the sensitive-disclosure bypass — followed by re-confirming EMP7003 and EMP7005 through a clean, full end-to-end run each.

See `/architecture` for the Workgraph diagram and a live escalation-form screenshot.
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
/dataset - all dataset provided to test against the model built
/dataset — schema documentation (no real data uploaded)
README.md — this file

# DAYBOOK — FINAL BUILD PRD
## Autonomous Finance Department / Agent Office
### Hackathon build specification for Claude Code

---

# 0. Executive brief

## Product

**Daybook** is a deployable autonomous finance department for a company.

It is not a dashboard with AI features and it is not four independent chatbots. It is a **real multi-agent finance operating system**: specialized agents own finance responsibilities, share a common company state, create and resolve structured work, coordinate through a central harness, and produce auditable outputs.

The frontend is the CFO's office: a calm, finance-first workspace where the CFO can see the company's financial state and inspect what every agent is doing.

The backend is the actual department.

The demo company uses highly realistic synthetic finance data. The architecture must not be hard-coded to that company: the same normalized data model and agent contracts should be capable of accepting another company's bank transactions, invoices, contracts, payroll, AR/AP, GL, etc.

### Core principle

> **Code computes. Agents reason. Agents coordinate. Auditor verifies. Humans authorize consequential actions.**

Never let an LLM invent financial truth.

---

# 1. What we are actually building

Daybook continuously maintains a company's financial state:

- reconciled
- explained
- monitored
- independently verified
- ready to answer questions
- ready to produce evidence
- permissioned for legitimate external access

The CFO should be able to answer:

- What is our financial position right now?
- What changed?
- Why did it change?
- What needs attention?
- What could affect cash or covenants?
- Which customers/vendors are becoming risky?
- What is already verified?
- What does an auditor/lender/board member need to see?
- What decisions are waiting on me?

The system should turn raw financial operations into a continuously maintained **trusted company state**.

---

# 2. Primary user

## Primary user: CFO / Head of Finance

The primary user needs:

1. A high-level view of financial health.
2. Clear visibility into work being performed by the finance department.
3. Exceptions and decisions requiring attention.
4. Evidence behind important numbers.
5. Ability to inspect any agent's current work.
6. Ability to ask role-specific finance questions.
7. Ability to grant tightly scoped external access.
8. A complete audit trail.

The CFO should **not** need to understand the AI architecture to use Daybook.

The UI should feel like finance software with a visible digital office, not an AI playground.

---

# 3. Product goals

## P0 — must work

### Finance engine
- Ingest realistic financial data.
- Normalize it into a common company model.
- Maintain ledger/bank/invoice/vendor/customer relationships.
- Reconcile transactions deterministically.
- Calculate cash and finance metrics deterministically.
- Detect exceptions.
- Maintain daily financial state.

### Multi-agent harness
- Agents have explicit roles, tools, permissions and output schemas.
- Agents can create work for one another.
- Agents can wait on another agent's output.
- Agents can return work for correction.
- Agent state is persisted.
- Every meaningful action creates an event/audit record.

### AI
- Only the Analyst uses the LLM in the initial build.
- LLM output must be structured and grounded in retrieved evidence.
- LLM never creates financial numbers.
- Unsupported conclusions become `UNVERIFIED`.

### Auditor
- Independently verifies important outputs.
- Blocks unsupported publication.
- Can send work back to agents.
- Publishes the daily state only after verification.

### CFO outputs
- Live financial state.
- Exceptions.
- Decisions.
- Agent activity.
- Evidence.
- Ask Anything.
- Trust Window.
- Auditor scorecard.

---

# 4. Non-goals

Do NOT attempt to build:

- a full ERP
- real payment execution
- real banking connectivity
- tax filing
- real payroll processing
- real accounting-book posting into an external ERP
- a general-purpose autonomous company
- a fake AI conversation for every agent

For the hackathon, these are simulated or read-only.

The architecture should make future integration possible without pretending the integration already exists.

---

# 5. Agent department

The department is larger than the minimum four-agent implementation.

## Core implemented agents

### 1. Controller
**Mission:** establish what is financially true.

Owns:
- transaction ingestion
- reconciliation
- matching
- duplicate detection
- ledger consistency
- transaction exceptions
- evidence relationships

Can:
- read normalized financial data
- create reconciliation results
- create exceptions
- request Analyst investigation

Cannot:
- use AI
- approve payments
- move money
- publish trusted state
- invent accounting treatment

Primary output:

```json
{
  "reconciliation_run_id": "...",
  "matched": 0,
  "unmatched": 0,
  "duplicates": 0,
  "exceptions": [],
  "status": "complete"
}
```

---

### 2. AP Agent
**Mission:** keep payables accurate, controlled and visible.

For the hackathon, implement AP logic as a real workflow module/agent where feasible.

Owns:
- vendor bills
- invoice/PO matching
- payment status
- duplicate bill detection
- overdue AP
- payment exceptions

Core workflow:

`PO → Invoice → Payment → GL`

Checks:
- PO exists
- quantities/prices agree
- invoice total agrees
- payment is not duplicated
- payment terms are respected
- missing approvals/evidence are flagged

Cannot:
- release money
- override Controller/Auditor
- fabricate approvals

---

### 3. AR Agent
**Mission:** keep receivables and collections visible.

Owns:
- customer invoices
- incoming payment matching
- AR aging
- overdue invoices
- collection-risk flags

Core workflow:

`Customer → Invoice → Due Date → Payment → AR Aging`

Detects:
- chronic late payers
- aging deterioration
- partial payments
- unexplained receipts
- credit notes/refunds

Can create collection tasks/drafts.

Cannot:
- alter payment records
- fabricate payment dates
- send external messages without authorization

---

### 4. Treasurer
**Mission:** understand liquidity and financial risk.

Owns:
- cash position
- cash forecast
- runway
- current ratio
- leverage
- covenant headroom
- major upcoming cash commitments

All calculations are deterministic.

Example:

```text
Cash = verified bank balances
Runway = available cash / normalized monthly burn
Current Ratio = current assets / current liabilities
Leverage = debt / defined EBITDA measure
```

Treasurer can:
- flag risk
- request investigation
- create decision items

Treasurer cannot:
- move money
- alter underlying accounting records
- use AI for calculations

---

### 5. FP&A Agent
**Mission:** explain performance versus plan and model forward outcomes.

Owns:
- budget vs actual
- forecast vs actual
- MTD performance
- variance analysis
- scenario calculations
- forecast assumptions

For initial demo, use deterministic models and optionally Analyst support for narrative explanations.

Never let AI calculate core metrics.

---

### 6. Analyst
**Mission:** investigate ambiguous or material financial changes.

This is the only AI-enabled agent in the initial architecture.

The Analyst receives structured work from other agents.

Example:

```text
Controller:
"Cloud expense increased 31%."

Analyst:
"Investigate cause."
```

The Analyst retrieves relevant evidence:

- invoices
- contracts
- POs
- tickets
- SOWs
- historical transactions
- vendor/customer records
- ledger relationships

Then TensorMux / `glm-4-7-flash` produces a structured explanation.

Required output:

```json
{
  "exception_id": "...",
  "finding": "...",
  "cause": "...",
  "impact": "...",
  "evidence_ids": ["..."],
  "confidence": 0.0,
  "status": "SUPPORTED|UNVERIFIED|WRONG",
  "recommended_action": "..."
}
```

Rules:
- never invent amounts
- never invent documents
- every factual claim must reference evidence IDs
- if evidence is insufficient, return `UNVERIFIED`
- no ledger mutation
- no payment approval
- no publication

---

### 7. Auditor
**Mission:** independently determine whether the company's published state is defensible.

The Auditor should distrust the other agents.

It verifies:
- source rows
- reconciliation outputs
- metric calculations
- evidence links
- Analyst claims
- duplicate detection
- covenant calculations
- daily state consistency

Possible result:

```text
SUPPORTED → publish
WRONG → block + reopen
UNVERIFIED → block + request evidence
```

Only Auditor can mark:

`DAILY_STATE = PUBLISHED`

The Auditor should be able to send work back to the originating agent.

---

### 8. CFO Orchestrator
**Mission:** coordinate the department and surface what matters to the CFO.

This is not a calculator and should not replace the specialists.

It:
- receives important agent events
- prioritizes material exceptions
- assigns work
- waits for dependencies
- coordinates cross-agent workflows
- creates CFO decision items
- summarizes department status
- escalates unresolved material issues

Example:

```text
AR:
"Meridian has paid 45 days late for 3 consecutive months."

↓

CFO Orchestrator:
"Collection risk is material."

↓

Treasurer:
"₹4.2L expected next week."

↓

FP&A:
"Another 30-day delay reduces projected cash buffer."

↓

CFO Orchestrator:
"Decision required: escalate account / revise forecast."
```

The orchestrator must use structured state and events, not a free-form LLM conversation.

---

# 6. Agent contract

Every agent must have:

```text
ROLE
MISSION
INPUTS
TOOLS
OWNED STATE
OUTPUT SCHEMA
PERMITTED ACTIONS
FORBIDDEN ACTIONS
TRIGGERS
ESCALATION CONDITIONS
DEPENDENCIES
```

Each agent has a persistent state:

```text
IDLE
WORKING
WAITING
VERIFYING
BLOCKED
DONE
```

These states must be real backend state and drive the frontend.

Do not animate an agent as working if the backend has no work for it.

---

# 7. Multi-agent harness

## This is the heart of the backend

Create a central orchestration layer.

Suggested modules:

```text
agents/
  controller.py
  ap.py
  ar.py
  analyst.py
  treasurer.py
  fpa.py
  auditor.py
  cfo.py

harness/
  orchestrator.py
  task_queue.py
  event_bus.py
  agent_state.py
  permissions.py
  schemas.py
```

## Work item

Every piece of agent work becomes a structured work item.

```json
{
  "id": "WORK-0184",
  "type": "INVESTIGATE_EXCEPTION",
  "created_by": "controller",
  "assigned_to": "analyst",
  "priority": "HIGH",
  "inputs": ["EX-0184"],
  "dependencies": [],
  "status": "IN_PROGRESS",
  "created_at": "...",
  "completed_at": null,
  "outputs": []
}
```

## Event bus

Every meaningful action produces an event:

```json
{
  "timestamp": "...",
  "agent": "controller",
  "event": "EXCEPTION_CREATED",
  "entity_id": "EX-0184",
  "details": {}
}
```

The frontend consumes these events.

## Coordination

Agents can:

- create work
- assign work
- complete work
- block work
- request evidence
- request another agent
- return work
- escalate to CFO
- wait for dependencies

This is what makes the system a team.

---

# 8. Shared company state

Create a normalized internal model that is independent of the demo company's exact files.

```text
Company
├── Accounts
├── Bank Accounts
├── Transactions
├── Ledger Entries
├── Chart of Accounts
├── Vendors
├── Customers
├── Purchase Orders
├── Invoices
├── Payments
├── Payroll
├── Contracts
├── SOWs
├── Tickets
├── AR
├── AP
├── Budgets
├── Forecasts
├── Covenants
├── Exceptions
├── Decisions
├── Evidence
├── Agent Tasks
└── Audit Events
```

Build relationships:

```text
PO → Invoice → Payment → Bank Transaction → GL → Daily State
Contract → SOW → Invoice → Payment → Ledger
Customer → Invoice → Payment → AR Aging
Vendor → Invoice → Payment → AP Aging
```

This evidence graph is central to Daybook.

---

# 9. Data ingestion / company adaptability

The demo dataset is synthetic and highly realistic, but the agents must not depend on its names or specific anomalies.

Build a normalization layer:

```text
SOURCE DATA
   ↓
INGESTION
   ↓
NORMALIZATION
   ↓
COMMON DAYBOOK MODEL
   ↓
AGENT DEPARTMENT
```

Future company inputs can include:

- bank statements
- ERP/GL exports
- invoices
- vendor bills
- POs
- payroll
- customer invoices
- AR/AP aging
- contracts
- SOWs
- expense reports
- card transactions
- CRM exports
- budget/forecast files

The hackathon implementation should support the synthetic dataset cleanly and expose the normalized interfaces needed for future connectors.

---

# 10. Realistic synthetic company

All demo data must be synthetic but look and behave like real finance operations.

Target:
- ~30 days
- ~6,000+ transactions
- realistic bank statements
- invoices
- POs
- payroll
- vendor/customer masters
- AR/AP aging
- GL
- contracts
- SOWs
- internal tickets
- remittance/payment references
- covenant inputs

Use realistic business terminology and formatting.

Where appropriate for an India-oriented demo:
- INR
- GST
- TDS
- UPI
- IMPS
- NEFT
- RTGS
- UTR-like references

Never copy a real company's confidential document or PII.

## Document realism

Generate realistic-looking:
- invoice PDFs
- PO PDFs
- contract/SOW PDFs
- bank statement exports
- ticket exports
- CSV/Excel registers

No real logos or copied documents.

## Required integrity checks

- bank opening + credits - debits = closing
- invoice subtotal + tax = total
- PO quantity × unit price = total
- invoice references valid PO
- payment ties to invoice
- AR aging ties to open invoices
- AP aging ties to unpaid invoices
- GL debits = credits
- daily cash = prior cash + bank movement
- covenant inputs tie to underlying state

Use a fixed seed, e.g.:

`SEED=20260906`

---

# 11. Planted demo events

These are test fixtures, not hard-coded agent logic.

1. Snowflake tier upgrade around day 12:
   - cloud spend +₹18.4K
   - linked DevOps ticket #212

2. Duplicate Vertex payment:
   - ₹2,034
   - same invoice paid twice

3. Meridian Ltd:
   - approximately 45-day late payment behavior
   - sustained AR deterioration

4. Supplier price increase around day 20:
   - appears potentially like FX
   - contract establishes actual cause
   - Analyst may be wrong on day 22
   - Auditor should catch unsupported/wrong explanation
   - never fake a failure if the actual model gets it right

5. Contractor costs decrease:
   - SOW ended
   - benign/expected

6. Payroll split across two bank transactions:
   - legitimate reconciliation wrinkle
   - should not be treated as a duplicate

---

# 12. Daily operating loop

Implement:

```bash
python run_day.py --day 1
python run_day.py --day 2
...
python run_day.py --day 30
```

Each day:

```text
1. Ingest today's data
2. Update normalized company state
3. Controller reconciles
4. AP/AR process their new work
5. Treasurer updates cash/risk
6. FP&A updates performance/forecast
7. Analyst investigates material exceptions
8. CFO Orchestrator coordinates unresolved work
9. Auditor independently verifies
10. Auditor publishes or blocks daily state
11. Generate state/day_NN.md
12. Append audit/event logs
13. Update frontend agent states
```

The sequence can branch when agents create dependencies.

The important thing is not merely the order; it is the **actual work handoff**.

---

# 13. CFO-visible outputs

The system must always make the end goals visible.

## Daily Financial State

```text
Cash
Runway
Current Ratio
Leverage
Covenant Headroom
MTD Revenue vs Plan
AR
AP
Burn
Forecast
Reconciled Transactions
Open Exceptions
Decisions Required
Auditor Status
```

Every important number must have:
- calculation/source
- timestamp
- verification status

## Exceptions

Each exception should show:

```text
WHAT HAPPENED
WHY IT MATTERS
AMOUNT
OWNER
STATUS
EVIDENCE
AGENT WORK
RECOMMENDED ACTION
AUDITOR STATUS
```

## Decisions

CFO sees:

```text
DECISION REQUIRED
Why now
Financial impact
Evidence
Options
Recommendation
Owner
Deadline
```

Do not bury important work in chat.

---

# 14. Frontend — Agent Office

## Core visual concept

The frontend is a **realistic digital finance office**.

It should not look like an AI demo.

Avoid:
- giant chat UI
- glowing AI brains
- excessive gradients
- sci-fi dashboards
- fake robot animations
- every agent constantly talking

Prefer:
- clean modern office
- desks/workstations
- documents
- monitors
- folders
- subtle status indicators
- finance dashboards
- professional typography
- restrained motion

The office is the visual surface for the real backend.

---

# 15. Main frontend layout

## Left / main area: Office

Show workstations for:

- Controller
- AP
- AR
- Analyst
- Treasurer
- FP&A
- Auditor
- CFO / Orchestrator

Each desk has:

- agent name
- role
- status
- current task
- small relevant metric
- subtle activity indicator

Example:

```text
CONTROLLER
Reconciling
214 transactions
5 exceptions
```

```text
ANALYST
Investigating
EX-0184
Vendor payment anomaly
```

```text
TREASURER
Monitoring
13-week cash forecast
```

```text
AUDITOR
Verifying
Day 17 state
```

---

# 16. Clickable agent desks

This is mandatory.

When the CFO clicks an agent's desk, open a detail panel.

### Panel contents

**Agent**
- role
- mission
- current status

**Currently working on**
- work item
- priority
- started
- dependencies

**Monitoring**
- what this agent continuously checks

**Recent work**
- completed tasks
- findings
- exceptions

**Outputs**
- reports
- decisions
- reconciliations
- recommendations

**Waiting on**
- other agent
- evidence
- human decision

**Escalations**
- current issues requiring CFO attention

**Evidence**
- linked source documents/records

**Activity timeline**
- actual harness events

Example:

```text
TREASURER

STATUS
WORKING

CURRENTLY
Updating 13-week cash forecast

MONITORING
• Cash balance
• Upcoming AP
• Expected AR
• Covenant headroom

WAITING ON
Controller → AP reconciliation

RECENT FINDING
Cash buffer may fall below target
in week 9.

IMPACT
₹8.7L lower projected liquidity

[View Forecast]
[View Evidence]
```

This should make it obvious that the agent is doing real work.

---

# 17. CFO home view

The office should not be the only view.

Provide a primary CFO overview:

```text
DAYBOOK
Financial State: VERIFIED

₹42.8L Cash
7.9 mo Runway
1.42 Current Ratio
0.38x Covenant Headroom

────────────────────────────

NEEDS YOUR ATTENTION

2 Decisions
3 Material Exceptions
1 External Access Request

────────────────────────────

TODAY

6,412 Transactions
97% Reconciled Automatically
23 Material Movements Explained
₹2,034 Duplicate Payment Caught

────────────────────────────

FINANCE DEPARTMENT

Controller      DONE
AP              DONE
AR              WORKING
Treasurer       WORKING
FP&A            DONE
Analyst         2 ACTIVE
Auditor         VERIFYING
```

The CFO should immediately know:
**Is the company financially okay, what changed, and what do I need to do?**

---

# 18. Evidence drawer

Click any material number or finding.

Show:

```text
CLAIM
Cloud spend increased ₹18,400.

SOURCE TRANSACTIONS
...

INVOICE
...

CONTRACT
...

TICKET
DevOps #212

ANALYST EXPLANATION
...

AUDITOR
SUPPORTED ✓
```

The evidence chain should be visible without leaving Daybook.

---

# 19. Ask Anything

Provide a secondary finance command interface.

Supported personas:

```bash
python ask.py --as board "Are we going to trip the covenant this quarter?"
python ask.py --as auditor "Show support for these fourteen transactions."
python ask.py --as sales "Is Meridian still good for net-60?"
python ask.py --as lender "What is our current liquidity position?"
```

Frontend version:

```text
Ask Daybook
[ Board ▼ ]

Are we going to trip the covenant this quarter?
```

Answer must include:

- answer
- reasoning/working
- source references
- verification status

This is not a generic chatbot.

---

# 20. Trust Window

External access should be explicit and narrow.

Example:

```bash
python window.py grant \
  --party "Silverline Capital" \
  --role lender \
  --scope cash,covenant,ar_aging \
  --expires 2026-12-31
```

Viewer:

```bash
python window.py view \
  --party "Silverline Capital" \
  --question "current ratio this month"
```

Log:

```bash
python window.py log \
  --party "Silverline Capital"
```

Frontend:

```text
SILVERLINE CAPITAL
Role: LENDER

ACCESS
✓ Cash
✓ Covenant
✓ AR Aging

DENIED
✕ Payroll
✕ Contracts
✕ HR

Expires
31 Dec 2026

Recent access
...
```

Rules:
- read-only
- allow-list
- explicit party
- explicit role
- expiry
- default deny
- every view/export logged

---

# 21. Auditor scorecard

Show:

```text
AUDITOR

Daily state
VERIFIED ✓

Figures checked
128

Evidence-linked
128 / 128

Analyst findings
21 supported
1 unverified
0 unsupported published

Exceptions blocked
2

Publication
APPROVED
```

The Auditor is a real system gate, not a decorative status.

---

# 22. Persistence and audit trail

Persist:

```text
state/
  company.json
  day_01.json
  day_01.md
  ...
  day_30.json
  day_30.md

logs/
  events.jsonl
  agent_actions.jsonl
  audit_log.jsonl
  access_log.jsonl

work/
  tasks.json
  exceptions.json
  decisions.json
```

Every state transition should be traceable.

---

# 23. Failure handling

The system must fail safely.

## AI failure

If TensorMux fails:

```text
Analyst = BLOCKED
```

Do not fabricate an explanation.

Other deterministic agents continue.

## Missing evidence

```text
Analyst = UNVERIFIED
Auditor = BLOCKED
```

Do not publish unsupported state.

## Agent failure

Work item becomes:

`BLOCKED`

Orchestrator can retry or reassign.

## Conflicting outputs

Auditor creates a dispute:

```text
DISPUTED
→ request evidence
→ re-run analysis
→ human escalation if unresolved
```

---

# 24. AI boundary

Hard rule:

> **Only `agents/analyst.py` may import/call the AI model in the initial build.**

No AI imports in:

- Controller
- AP
- AR
- Treasurer
- FP&A calculations
- Auditor
- Trust Window
- Close
- core financial calculation modules

AI may eventually support more narrative functions, but the hackathon version must make the boundary obvious.

Use:

- TensorMux
- base URL: `https://api.tensormux.com/v1`
- model: `glm-4-7-flash`
- OpenAI-compatible client
- API key in `.env`

Never hard-code secrets.

---

# 25. Security / permissions

Implement agent permissions at the code level.

Example:

```text
Controller
READ financial records
WRITE reconciliation results
CREATE exceptions
NO external access

Analyst
READ evidence
CREATE explanations
CREATE recommendations
NO financial mutation
NO publication

Treasurer
READ verified financial state
WRITE forecasts/alerts
NO transaction mutation

Auditor
READ everything required for verification
WRITE verification status
PUBLISH daily state

CFO Orchestrator
CREATE/ASSIGN work
ESCALATE decisions
NO financial mutation

Trust Window
READ ONLY
SCOPE LIMITED
EXPIRING
LOGGED
```

---

# 26. API / backend interface

Expose clean APIs so the frontend is a client of the actual system.

Suggested endpoints:

```text
GET  /api/state
GET  /api/agents
GET  /api/agents/:id
GET  /api/agents/:id/activity
GET  /api/work
GET  /api/exceptions
GET  /api/decisions
GET  /api/evidence/:id
GET  /api/audit
GET  /api/access
POST /api/run/day
POST /api/ask
POST /api/window/grant
POST /api/window/view
```

The frontend must not calculate finance metrics itself.

---

# 27. Suggested project structure

```text
daybook/
├── agents/
│   ├── controller.py
│   ├── ap.py
│   ├── ar.py
│   ├── analyst.py
│   ├── treasurer.py
│   ├── fpa.py
│   ├── auditor.py
│   └── cfo.py
│
├── harness/
│   ├── orchestrator.py
│   ├── task_queue.py
│   ├── event_bus.py
│   ├── agent_state.py
│   ├── permissions.py
│   └── schemas.py
│
├── finance/
│   ├── ledger.py
│   ├── reconciliation.py
│   ├── cash.py
│   ├── ar.py
│   ├── ap.py
│   ├── forecast.py
│   └── covenants.py
│
├── data/
│   ├── generator.py
│   ├── normalized/
│   ├── raw/
│   └── documents/
│
├── evidence/
│   ├── graph.py
│   └── retrieval.py
│
├── state/
├── logs/
├── work/
│
├── frontend/
│
├── run_day.py
├── ask.py
├── window.py
├── close.py
├── seed_company.py
├── ANSWER_KEY.md
├── README.md
├── .env.example
└── requirements.txt
```

Adapt structure if the existing stack benefits from a different organization, but preserve the separation of concerns.

---

# 28. Testing / evaluation

Build tests that prove the agents actually work.

## Deterministic tests

- bank reconciliation
- duplicate detection
- invoice math
- PO matching
- AR aging
- AP aging
- cash
- runway
- current ratio
- leverage
- covenant headroom
- GL balance
- daily state consistency

## Agent tests

- Controller creates correct exceptions.
- Analyst finds correct evidence.
- Analyst cannot invent unsupported claims.
- Auditor catches incorrect Analyst explanation.
- Treasurer consumes verified data.
- Orchestrator creates correct handoffs.
- Work dependencies behave correctly.
- Agent permissions are enforced.

## End-to-end tests

At minimum:

### Scenario A
Duplicate payment → Controller detects → Analyst/Auditor confirms → CFO sees recovery issue.

### Scenario B
Cloud spend spike → Controller flags → Analyst finds invoice/ticket → Treasurer measures impact → Auditor verifies → CFO sees explanation.

### Scenario C
Late customer → AR flags → Treasurer sees cash impact → FP&A sees forecast impact → CFO gets decision.

### Scenario D
Unsupported explanation → Auditor blocks publication.

### Scenario E
External lender asks for payroll → Trust Window denies access and logs attempt.

---

# 29. Demo success metrics

Generate these from the system. Do not hard-code them.

Target metrics:

- ~6,412 transactions processed
- ~97% reconciled automatically
- 23 material movements explained
- ₹2,034 duplicate caught
- day-30 close around 11 minutes
- all published figures source-linked
- unsupported findings blocked

If implementation produces different numbers, show actual numbers.

Never fake metrics.

---

# 30. Close

Implement:

```bash
python close.py
```

It should:

1. verify all 30 days
2. confirm no unresolved blocking errors
3. produce final financial state
4. summarize exceptions
5. summarize decisions
6. produce Auditor scorecard
7. produce final evidence index
8. produce final audit log
9. report elapsed processing time

Example:

```text
DAYBOOK CLOSE

30 days processed
6,412 transactions
97.1% auto-reconciled
23 material movements
1 duplicate payment caught
0 unsupported figures published

AUDITOR
30/30 daily states verified

STATUS
CLOSE COMPLETE
```

---

# 31. AO usage requirement

This hackathon requires meaningful use of AO throughout development.

Use AO for substantial implementation work.

Recommended sessions:

### Session 1 — Company + data
Build:
- synthetic company
- normalized schema
- realistic documents
- ANSWER_KEY
- integrity tests

### Session 2 — Finance engine + Controller
Build:
- ledger
- reconciliation
- duplicate detection
- exceptions

### Session 3 — Agent department
Build:
- Analyst
- Treasurer
- AP
- AR
- FP&A
- harness

### Session 4 — Auditor + CFO surface
Build:
- Auditor
- CFO orchestrator
- daily state
- Ask Anything
- Trust Window
- frontend
- end-to-end tests

Show AO session history in the final demo.

---

# 32. Observability

Use Neatlogs where practical.

Run:

```bash
npx @neatlogs/wizard
```

Capture at least one useful trace showing:

```text
Controller
→ exception
→ Analyst
→ evidence
→ Treasurer
→ Auditor
→ published state
```

The trace should demonstrate real work, not fake events.

---

# 33. Demo script

Target 4 minutes.

### 0:00–0:25 — Problem
Finance teams reconstruct truth at month-end.

Show the five moments:

- auditor asks for support
- diligence/data room
- lender covenant test
- board question
- duplicate/risk issue

### 0:25–0:50 — Daybook
Show the office.

> “This is not a finance dashboard. This is the company's finance department.”

### 0:50–1:30 — One morning
Run a day.

Controller works.
AP/AR process.
Treasurer monitors.
Analyst investigates.
Auditor verifies.

Click desks and show actual work.

### 1:30–2:05 — Material event
Show Snowflake/cloud spend.

Drill:

`state → exception → evidence → Analyst → Auditor`

### 2:05–2:35 — CFO questions
Ask Board / Auditor / Sales / Lender questions.

Show source-backed answers.

### 2:35–3:05 — Trust Window
Grant lender access.

Show:
- permitted data
- denied payroll
- expiry
- access log

### 3:05–3:35 — Auditor honesty
Show a wrong/unsupported Analyst conclusion if the real run produces one.

Auditor blocks it.

If the actual model gets it right, show the real correct result instead.

### 3:35–3:50 — Close
Run day 30 close.

Show real metrics.

### 3:50–4:00 — Build proof
Show:
- AO usage
- Neatlogs trace
- architecture

---

# 34. What makes Daybook different

Do not claim competitors cannot do autonomous finance.

Position Daybook around:

> **A continuously verified financial state plus an actual coordinated finance department and a permissioned external trust layer.**

The key idea:

> We are not replacing the ledger. We are building the room next to it where the company's finance work happens—and where legitimate people can safely ask for proof.

The strongest differentiators are:

1. multi-agent finance team
2. shared financial state
3. actual agent-to-agent work handoffs
4. independent Auditor
5. source-level evidence
6. CFO-visible decisions
7. Trust Window
8. adaptable company data model

---

# 35. Hard product rules

1. **No fake agent activity.**
2. **No hard-coded demo answers.**
3. **No invented financial numbers.**
4. **No AI-generated accounting truth.**
5. **No unsupported publication.**
6. **No external access outside explicit scope.**
7. **No payment movement.**
8. **No hidden work the CFO cannot inspect.**
9. **No frontend-only finance calculations.**
10. **Every material claim should be traceable to evidence.**

---

# 36. Definition of done

Daybook is complete enough for the hackathon when:

### Backend
- [ ] normalized company model works
- [ ] realistic synthetic company generated
- [ ] deterministic finance engine works
- [ ] Controller works
- [ ] AP works
- [ ] AR works
- [ ] Treasurer works
- [ ] FP&A works
- [ ] Analyst works with TensorMux
- [ ] Auditor works independently
- [ ] CFO Orchestrator coordinates work
- [ ] work queue persists
- [ ] event bus persists
- [ ] agent permissions enforced

### Financial state
- [ ] daily state generated
- [ ] source-linked metrics
- [ ] exceptions tracked
- [ ] decisions tracked
- [ ] Auditor publication gate works
- [ ] day-30 close works

### Frontend
- [ ] CFO overview
- [ ] agent office
- [ ] clickable desks
- [ ] live agent status
- [ ] current work visible
- [ ] monitoring visible
- [ ] dependencies visible
- [ ] activity timeline
- [ ] evidence drawer
- [ ] exceptions
- [ ] decisions
- [ ] Ask Anything
- [ ] Trust Window
- [ ] Auditor scorecard

### Reliability
- [ ] AI failure handled
- [ ] missing evidence handled
- [ ] conflicting conclusions handled
- [ ] core financial tests pass
- [ ] end-to-end scenarios pass
- [ ] Neatlogs trace captured

### Submission
- [ ] public GitHub
- [ ] README
- [ ] architecture diagram
- [ ] evaluation methodology
- [ ] measurable results
- [ ] demo video
- [ ] AO usage demonstrated
- [ ] Devpost submission complete

---

# 37. Final implementation philosophy

Do not optimize for the appearance of autonomy.

Optimize for **real autonomous work**.

A user should be able to click Controller's desk and see an actual reconciliation run.

Click Analyst and see an actual investigation.

Click Treasurer and see what financial risks it is monitoring.

Click Auditor and see exactly what it is verifying and why publication is blocked or approved.

Click CFO and see what the department believes requires a human decision.

The office is the interface.

The harness is the department.

The normalized company state is the memory.

The evidence graph is the proof.

The Auditor is the trust boundary.

And the CFO remains the human who owns consequential decisions.

**Build that system first. Make it beautiful second.**

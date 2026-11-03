# InvCast ERP

**A full-stack ERP for a precision investment-casting foundry**, covering the whole business from the customer inquiry to the
shipped, certified part and the general ledger behind it.

> The source code is private. This repository describes what the system does, how it is built, and the engineering problems
> it had to solve. A walkthrough or demo is available on request (see [Contact](#contact)).

![Foundry process logs: wax injection through knockout, recorded per cluster and work order](screenshots/foundry-logs.png)

| | |
|---|---|
| ![Product genealogy: one cluster traced across 11 stages, from heat batch to finished goods](screenshots/genealogy.png) | ![Casting yield and scrap valuation per melt](screenshots/casting-yield.png) |
| **Product genealogy:** one cluster traced across 11 stages, from heat batch to finished goods | **Casting yield:** melt-by-melt yield, with scrap valued at standard cost |
| ![Cost of goods manufactured, built from the general ledger](screenshots/cogm.png) | ![Quality dashboard: pass rate, calibration, CAPA backlog, NCR aging](screenshots/quality-dashboard.png) |
| **COGM:** built from the general ledger, so it ties to the trial balance | **Quality:** pass rate, calibration status, CAPA backlog, NCR aging |

<sub>Screenshots use seeded sample data.</sub>

---

## At a glance

| | |
|---|---|
| Backend feature modules | **107** |
| REST endpoints | **~1,100** (OpenAPI spec generated from source) |
| Reports and dashboards | **38** report pages, plus role dashboards |
| Printable documents | **30** PDF templates (orders, invoices, certificates, travelers, check sheets, labels) |
| Background jobs | **9** scheduled jobs and alerts |
| Database | **~250 tables · 650+ foreign keys · 270+ CHECK constraints** (SQL Server) |
| Schema migrations | **~400**, versioned and replayable |
| Automated tests | **685** Jest test files · **164** Playwright end-to-end specs |
| Pre-push verification gates | **43** |
| Code | ~275k lines backend JS · ~145k lines frontend |

<sub>Figures as of November 2026.</sub>

---

## What it covers

| Domain | Highlights |
|---|---|
| **Sales & CRM** | Inquiry pipeline → quotation → sales order → delivery → AR invoice → payment; multi-currency price lists; AS9100 order review; customer complaints and account-health dashboard; new-product introduction stage gates (tooling → wax → shell → casting → NDT → PPAP) |
| **Procurement** | Purchase request → RFQ → PO → goods receipt → supplier invoice with 3-way match and tolerance → payment; landed cost allocation; GR/IR clearing; supplier evaluation and certification |
| **Foundry operations** | The investment-casting process recorded stage by stage (see below), with melt heats tracked down to the poured part |
| **Manufacturing** | BOMs, routings, work centers and machines, work orders, shop-floor confirmation with run readings, MRP, subcontracting / outside processing, tooling and dies |
| **Planning & maintenance** | Shift calendars; a planner board that proposes each machine's schedule and re-plans around breakdowns; full CMMS with preventive-maintenance plans, SLAs, independent verification of repairs, and maintenance cost posted to the ledger |
| **Quality** | Quality plans with ballooned drawings and check sheets; first-piece gate; incoming, in-process and final inspection to plan; NCR, MRB and CAPA with per-defect disposition; operator certification; calibration; quality certificates |
| **Inventory** | Multi-warehouse and bin locations, lot and heat tracking, shelf-life expiry, stock counts, perpetual stock card |
| **Finance** | Double-entry general ledger fed by every operational module; fiscal period close; multi-currency with per-currency AR/AP accounts and FX revaluation; fixed assets; bank reconciliation; withholding tax; standard costing, WIP → COGM → COGS; Indonesian Faktur Pajak with PPN and PPh 22 |
| **Platform** | Multi-company tenancy, role-based permissions, configurable approval rules, field-level audit trail, PDF document generation, document email, scheduled jobs and alerts, real-time multi-user sync, an AI assistant with read-only database tools, a document library with semantic search, in-app role guides and tours |

### The foundry process it models

```mermaid
flowchart LR
    W[Wax injection] --> C[Cluster / tree assembly]
    C --> S[Shell building<br/>per-coat logging]
    S --> D[Dewax]
    M[Melt heat<br/>chemistry] --> P[Pour]
    D --> P
    P --> K[Knockout]
    K --> H[Heat treatment]
    H --> X[Machining]
    X --> Q[Final inspection]
    Q --> F[Finished goods<br/>+ EN 10204 3.1 certificate]
```

Each stage records quantities, scrap and inspection results against the work order. The melt heat follows the part all the way
to the certificate the customer receives.

### How an order flows through the business

```mermaid
flowchart LR
    INQ[Inquiry] --> QUO[Quotation] --> SO[Sales order<br/>AS9100 order review] --> MRP[MRP]
    MRP --> WO[Work order]
    MRP --> PR[Purchase request]
    PR --> PO[RFQ and<br/>purchase order] --> GR[Goods receipt<br/>incoming inspection]
    GR --> WO
    GR --> SI[Supplier invoice<br/>3-way match] --> SP[Supplier payment]
    WO --> FS[Foundry stages<br/>and machining] --> QC[Final inspection]
    QC --> DN[Delivery<br/>packing list, mill cert] --> ARI[AR invoice<br/>PPN, e-Faktur] --> CP[Customer payment]
    QC -.->|reject| NCR[NCR, MRB, CAPA]
    GR -.->|reject| NCR
```

The sales order drives MRP, which raises both the work orders and the purchase requests for the material they need. Rejected
material at any inspection opens an NCR, goes to the MRB for disposition, and can raise a CAPA.

In the app, the **document navigator** draws this chain for any real document. Below is one sales order, three links deep:

![Document navigator: one sales order linked to 73 documents through 77 links](screenshots/document-flow.png)

<sub>73 documents and 77 links from a single sales order: quotations, work orders, material issues, operation confirmations,
production receipts, the delivery, the invoice and payment, inspections, NCRs and a CAPA, and the stock movements and journal
entries each one posted.</sub>

### How cost flows into the ledger

```mermaid
flowchart LR
    GR[Goods receipt] --> RM[(Raw material<br/>inventory)]
    RM -->|material issue| WIP[(Work in<br/>progress)]
    LO[Labour and overhead<br/>at standard rates] -->|absorbed| WIP
    SC[Subcontract fees] --> WIP
    WIP -->|production receipt| FG[(Finished<br/>goods)]
    WIP -->|revert and scrap recovery| RM
    FG -->|delivery| DEF[(Deferred COGS)]
    DEF -->|AR invoice| COGS[COGS, matched<br/>with revenue]
```

Every arrow is a journal entry posted by the operational document itself. A goods receipt credits a GR/IR clearing account that
the supplier invoice later clears against AP. A delivery is valued at what the stock ledger actually relieved, not at the item
master's cost, and that value waits in deferred COGS until the invoice recognises revenue and COGS together.

<details>
<summary><b>Full feature list</b></summary>

**Sales & CRM**
- Inquiry pipeline with follow-up tracking, quotations, and sales orders with AS9100 order review and customer requirements
- Delivery notes with packing (packages, packing list, commercial invoice), mill certificate and certificate of conformance
- AR invoices with PPN, down-payment invoices and advance application, credit memos, customer payments
- Customer returns received line by line and settled on both the goods and the money
- Customer activities (calls, visits, meetings), CRM dashboard, customer complaints, account-health scoring
- New-product introduction stage gates through PPAP; multi-currency price lists and pricing rules

**Procurement**
- Purchase requests (manual or from MRP), RFQ, purchase orders, goods receipts with incoming inspection
- Supplier invoices with configurable 3-way-match tolerance and variance approval; supplier payments and advances
- Landed-cost shipments and allocation, GR/IR clearing, purchase returns, supplier debit memos
- Supplier certifications, evaluations and scorecard; supplier status, where a blocked supplier stops new purchase orders;
  item–supplier prices and lead times

**Foundry & manufacturing**
- Wax injection, cluster assembly, shell building (per coat), dewax, melt with heat chemistry, pour, knockout and cut-off, heat
  treatment, machining
- Product data management (BOMs and routings), work centers, machines with runs, cycles and batches, work orders, rework orders
- Shop-floor operation queue with QR scanning, confirmation and downtime logging; process settings recorded before work starts
  and readings at the plan's check points; shop travelers, cluster route cards and the foundry log register
- MRP that generates work orders and purchase requests; capacity load on the shift calendar
- Subcontracting and outside processing, with approval to switch an operation to a vendor or back in-house; subcontract
  orders without a work order; a vendor's faulty rework settled by debit note
- Tooling and dies with ownership, custody and capitalization; tool crib
- Standard costs, mass cost rollup, overhead and indirect-consumable rates, service-department cost allocation
- Product genealogy, and a document navigator that draws how any document connects to the rest

**Planning & maintenance**
- Shift calendars: templates, weekly patterns, holidays and exceptions, per work center
- Planner board: an auto-schedule proposal for each machine (one die per injection press), applied and pinned as firm by the
  planner, re-planned automatically when a machine breaks down; preventive maintenance shown on the same board
- Corrective maintenance: anyone can request, a supervisor accepts, and the repair is verified by someone who did not do it
- Preventive-maintenance plans with approval and revisions, task checklists, per-shift daily checks and an annual baseline
- SLA per priority with due times and an escalation job; failure codes
- Spare parts and outside service charged to the maintenance order and posted to the ledger
- Reports: MTBF and MTTR, PM compliance, maintenance cost, breakdown Pareto, each machine's history

**Quality**
- Quality plans with characteristics measured at each shop-floor stage, ballooned drawings printed into the plan and its check
  sheets, and acknowledgement by operations management; AQL sampling levels
- A first-piece QC gate; results judged against the plan by the system, with a supervisor verifying the rest (never their own)
- Incoming, in-process and final inspection to plan, with multi-defect disposition; MRB; NCR; CAPA
- NDT records, calibration records and alerts, operator certifications that gate work centers
- Quality certificates, cost of quality, scrap Pareto, supplier scorecard

**Inventory**
- Warehouses and bin locations, stock movements, stock counts, lot and heat tracking, shelf-life expiry, QR labels for items
  and locations

**Finance**
- Chart of accounts, posting profiles routed by business group, journal approvals, posting queue, void center
- Fiscal periods with period and year-end close, opening balances, budgets, cost centers
- Multi-currency: each currency posts only to the AR and AP accounts designated for it, and an order in a currency its
  supplier's AP account cannot hold is refused up front; FX revaluation runs that can be reversed
- Bank accounts, bank statement import and reconciliation, petty cash, cash-flow forecast
- Indonesian tax: PPN by the customer's tax treatment; the Faktur Pajak as its own document over one or several invoices, with
  "bill together" for one customer; withheld PPN and PPh 22 recorded on the customer's receipt; the weekly tax exchange rate;
  e-Faktur export and PPN reconciliation
- Fixed assets, withholding tax

**Reports (38)**
- Financial: trial balance, profit and loss, balance sheet, GL activity, GL integrity, subledger reconciliation, AR and AP aging
  and outstanding, customer and supplier ledgers, PPN reconciliation, GR/IR reconciliation
- Cost and production: COGM, COGS, cost variance, variance analysis, floor variance, outside-operation variance, WIP quantity,
  material issue, rework movement, casting yield, revert recovery, scrap Pareto, cost of quality
- Operations: OEE, downtime, capacity load, sales summary, outstanding orders and delivery detail, purchase summary and
  tracking, item price history, fixed-asset mutation and disposal

**Platform**
- Multiple companies, users, roles and permissions, configurable approval rules, KPI targets
- Field-level audit trail, where a journal line's history leads back to its source document; attachments, notifications,
  real-time updates across users; records past their editable stage open read-only
- 30 printable PDF documents; document email with correspondence history; inbound mailbox polling; scheduled reports
- A public QR verification page for printed documents
- An AI assistant with read-only database tools, where every question and every query it runs is logged
- An AI Library: documents uploaded, approved, split into passages and indexed for semantic search
- In-app onboarding: role playbooks, a guide drawer on every screen, and a spotlight tour

</details>

---

## Architecture

```mermaid
flowchart LR
    U[Browser<br/>React SPA] -- HTTPS / REST --> N[Nginx]
    U <-- WebSocket --> N
    N --> A[Express API<br/>107 modules]
    A --> DB[(SQL Server)]
    A --> PDF[Puppeteer / PDFKit<br/>pdf-lib<br/>document rendering]
    A --> MAIL[SMTP / IMAP<br/>document email]
    A --> JOB[node-cron<br/>scheduled jobs]
    A --> AI[LLM providers<br/>read-only tools]
    EMB[Python indexer<br/>multilingual embeddings] --> DB
```

- **Backend:** each feature module follows the same `routes → controller → service → validation` structure. Every route passes
  through authentication, permission checks, input validation and audit logging.
- **Frontend:** React 18 with React Query for server state, React Hook Form + Zod for forms, and a shared component library
  (data grid, drawers, form inputs) used by every page.
- **Real time:** write endpoints emit company-scoped Socket.IO events, so open screens refresh when another user changes the data.
- **Delivery:** GitHub Actions runs quality checks, then deploys to a Linux server (PM2 + Nginx).

---

## Hard problems worth talking about

### 1. Making the books tie
Every operational event (a goods receipt, a material issue, a production receipt, a scrap decision, a shipment) posts its own
journal entry. The hard part was not posting but **proving the ledger agrees with the subledgers**: inventory value with the
stock ledger, WIP with open work orders, AR and AP with open documents, in both document currency and base currency. Each
currency posts only to the receivable and payable accounts designated for it, which are the accounts FX revaluation restates,
so a foreign balance can't end up somewhere revaluation never looks. Posted entries are immutable at the database level, and
posting is blocked outside an open fiscal period.

### 2. Multi-company tenancy without leaks
One database serves several companies. Every company-scoped query filters by company; the active company is validated
server-side rather than trusted from the client; document numbers run on per-company sequences; and uniqueness constraints are
scoped per company. All of it was verified by a dedicated tenancy audit.

### 3. Aerospace-grade traceability (AS9100)
A shipped part can be traced back through machining, heat treatment, the pour and the melt heat with its chemistry, and
forward to the customer. That trace generates an **EN 10204 3.1 material certificate**, and printed documents carry a QR code
that verifies them against the database. Approvals enforce segregation of duties (no self-approval), operators must hold valid
certifications to confirm work at certified work centers, and every change records its before and after values.

### 4. Correctness under concurrency
Multi-table postings run in a single database transaction. Write endpoints use optimistic concurrency, so when two users edit
the same document, the second gets a clear conflict response instead of silently overwriting the first.

### 5. Quality decisions that move stock *and* money
One inspection can record several defects, each with its own disposition (accept, rework, scrap, return to supplier, hold for
MRB). Each disposition routes stock to the right warehouse and posts the right journal entry, and a part held for rework goes
back into production through a rework work order.

### 6. Scheduling against the shop's real calendar
Every machine and work center runs on a shift calendar (weekly patterns, holidays, one-off exceptions), resolved in one place.
The planner board proposes a schedule for each machine, for example one die per injection press, and the planner can apply it
and pin bars as firm. When a machine breaks down, its remaining work is re-planned automatically. OEE loading, capacity and
maintenance SLA clocks are all measured on that same calendar, and past periods are frozen, so editing a calendar never
rewrites history.

---

## Engineering discipline

A system with this many interacting rules breaks in ways ordinary tests miss, so verification became its own body of work:

- **A 43-gate pre-push suite.** It checks things a linter can't, for example that every enum value a script sends is one the
  validator accepts, or that cost and currency figures keep their scale through a calculation.
- **Schema parity.** Every migration is applied to both the sample and the production database, and a gate fails if their
  schemas ever differ.
- **SQL compiled against the live schema before it runs.** A full-lifecycle seed script builds a complete sample company,
  from master data through a closed accounting period. Every SQL statement in it is first compiled against the real schema with
  `SET NOEXEC ON`, which catches misspelled columns and reserved-word aliases before a long rebuild fails halfway through.
- **A module wiring linter** that flags a new module missing its route registration, authentication, permission seeds, menu
  entry or audit-trail hookup.
- **Generated API documentation.** A pre-commit hook regenerates the OpenAPI spec and the endpoint index from the route
  definitions, so the docs can't drift from the code.
- **Focused audits**, each with its own checklist and verification tooling: security, tenancy, RBAC and segregation of duties,
  transaction atomicity, input validation, concurrency, financial posting and costing.

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, React Router, TanStack React Query, React Hook Form, Zod, Tailwind CSS, Recharts / ApexCharts, React Flow (document graph), html5-qrcode (shop-floor scanning) |
| Backend | Node.js, Express, `mssql`, express-validator, JWT, Socket.IO, Winston, node-cron, Helmet, rate limiting, Multer (uploads), Swagger UI (API explorer) |
| Documents | Puppeteer and PDFKit (PDF), pdf-lib (merging drawings into plans), QR codes, Nodemailer + ImapFlow + mailparser (email) |
| AI | Google Gemini or any OpenAI-compatible gateway (OpenRouter, Cerebras), restricted to read-only tools; Python with sentence-transformers (multilingual E5) for semantic search |
| Database | Microsoft SQL Server |
| Testing | Jest, Supertest, Playwright |
| Delivery | GitHub Actions, PM2, Nginx |

---

## How it was built

I designed and built InvCast ERP, working closely with the foundry's own processes and accounting rules. Development was
AI-assisted: I used coding agents for much of the implementation, and I owned the domain model, the architecture, the
business rules and the verification system above, which exists so that nothing ships on an agent's say-so. The AI Library
module was contributed by a team member.

---

## Contact

**Handy Ban** · [banhandy.vercel.app](https://banhandy.vercel.app/)

Happy to walk through the system live or talk through any of the problems above.

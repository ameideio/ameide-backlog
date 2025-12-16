Below is a **first-pass Finance capability definition** in **ArchiMate terms**, aligned to **Backlog 528’s capability template** (Strategy → Business → Application → Technology → Implementation & Migration), and designed to be realizable via **Ameide primitives** and **proto-first EDA contracts**.

Assumption (so we don’t design the wrong thing): **“Finance” here means enterprise finance operations** (GL/AP/AR/close/treasury/expenses/assets/reporting), not capital-markets trading.

---

## Market baseline: what “leading finance platforms” converge on

Across major suites, the common backbone is a **general ledger** with connected sub-ledgers and controls:

* **General ledger + AP/AR + assets/expenses + reporting** is explicitly called out by vendors like Oracle Fusion Cloud Financials and Workday Financial Management. ([Oracle][1])
* Oracle’s Financials Cloud overview enumerates a “complete solution” including **General Ledger, Intercompany Accounting, Payables, Receivables, Payments, Cash Management, Tax, Expenses, Assets**—a good “minimum viable finance surface.” ([Oracle Docs][2])
* Suites aimed at mid-market also emphasize the same “core financials”: Sage Intacct lists **AP, AR, cash management, general ledger** (plus adjacent order/purchasing). ([Sage][3])
* NetSuite positions finance as a bundle of **accounting + revenue recognition + reporting + consolidation + GRC**, with add-ons like billing and budgeting. ([NetSuite][4])
* “Record-to-report” (R2R) is the canonical end-to-end finance value stream: collect/record transactions and produce financial statements and reports. ([SAP Learning][5])

Best-of-breed adjacencies show where finance needs **dedicated workflows and matching**:

* **Financial close & reconciliation** is a specialized market (e.g., BlackLine’s focus on reconciliation workflow/automation). ([bl-prod][6])
* **Treasury/cash management** is often separated (e.g., Kyriba positions treasury and cash/forecasting as its own platform category). ([Kyriba][7])
* **Spend management / P2P** (procure-to-pay) is frequently its own suite (e.g., Coupa’s guided buying + workflows + payments). ([Coupa][8])
* **Expense** is frequently separate-but-integrated (SAP Concur emphasizes unified expense/travel/vendor invoice spend). ([concur.com][9])
* **AP automation + global payments + tax/compliance** are also often separated (Tipalti highlights AP automation + tax compliance and global payments). ([Tipalti][10])
* **Revenue recognition automation** is frequently attached to billing/payment platforms for subscription businesses (Stripe revenue recognition docs). ([Stripe Docs][11])

**Design implication for Ameide:** a Finance capability should treat the **ledger as the authoritative accounting truth**, while recognizing that **P2P, expense, close, treasury, revenue recognition** can be realized as **multiple processes + integrations** over a single coherent accounting domain.

---

## Finance capability definition (ArchiMate-aligned per 528)

Backlog 528 defines a capability as Strategy-layer intent expressed via value streams, nouns, invariants, and EDA contracts realized by primitives. 

### Layer header

* **Primary ArchiMate layer(s):** Strategy (Capability, Value Stream) 
* **Secondary layers referenced:** Business, Application, Technology, Implementation & Migration 
* **Realization pattern:** capability contracts → primitives (Domain/Process/Projection/Integration/UISurface/Agent). 

---

# 1) Strategy layer

## 1.1 Capability

**Capability:** Finance (Enterprise Accounting & Close)

**Goal (what this capability provides):**

* A governed, auditable, multi-entity **system of record for accounting** (GL + sub-ledger postings)
* A standard, proto-first contract surface for:

  * posting/accounting intents
  * financial facts/events
  * read-side reporting queries
  * external finance integrations (payments, banking, tax, expense, P2P)

**Non-goals (explicitly out of scope):**

* Owning sales orders, fulfillment, procurement sourcing, payroll/HR as systems of record
  (those are other capabilities; Finance consumes their facts and produces accounting truth)
* Real-time trading, portfolio/risk analytics (capital markets)

## 1.2 Outcomes / value propositions

* **Faster and safer close:** reduce cycle time, increase confidence (close status + reconciliation completeness)
* **Auditability:** immutable evidence trail from source events → postings → reports
* **Control & compliance:** approval workflows, segregation-of-duties signals, period controls
* **Financial visibility:** trial balance, P&L, balance sheet, aging, cash position with consistent dimensions

## 1.3 Value streams (Strategy layer)

These are stable “what finance delivers” streams:

1. **Record-to-Report (R2R)**
   Capture/validate transactions and produce statements and operational reporting. ([SAP Learning][5])
2. **Procure-to-Pay (P2P)**
   Ingest vendor bills, approvals, payment execution, and accounting. ([Coupa][8])
3. **Invoice-to-Cash (I2C)**
   Issue invoices/receivables, apply cash, collections, write-offs, accounting. ([Oracle Docs][2])
4. **Expense-to-Reimburse**
   Employee expense capture/approval/reimbursement and accounting. ([concur.com][9])
5. **Treasury & Cash Visibility**
   Bank feeds, cash positioning, forecasting inputs, bank reconciliation. ([Kyriba][7])
6. **Close & Reconciliation**
   Reconciliation completion + close governance (tasks, sign-off, evidence). ([bl-prod][6])

---

# 2) Business layer

## 2.1 Business roles (not exhaustive)

* CFO / Controller
* Accountant (GL)
* AP clerk / AP manager
* AR clerk / Collections specialist
* Treasury analyst
* Internal auditor / Compliance officer

## 2.2 Business processes (Business layer) that realize value streams

(These are “Business Processes” in ArchiMate terms; later we map them to Process primitives.)

### A. Record-to-Report business processes

* **Maintain Chart of Accounts & Accounting Dimensions**
* **Record & Validate Journal Entries**
* **Period Management (open/soft-close/hard-close)**
* **Financial Statement Preparation**
* **Management Reporting & Variance Analysis**

### B. Procure-to-Pay business processes

* **Vendor Bill Ingestion**
* **Invoice Approval & Exception Handling** (workflow/policy-driven; Dynamics explicitly emphasizes invoice approval workflows + policies.) ([Microsoft Learn][12])
* **Payment Run & Remittance**
* **AP Accounting (postings, accruals, tax)**

### C. Invoice-to-Cash business processes

* **Customer Invoice Issuance**
* **Cash Application (match receipts to open items)**
* **Collections / Dunning**
* **AR Accounting (revenue, tax, adjustments)**

### D. Expense-to-Reimburse business processes

* **Expense Submission**
* **Policy/Manager Approval**
* **Reimbursement Execution**
* **Expense Accounting (coding, allocations)**

### E. Treasury & cash business processes

* **Bank Statement Ingestion**
* **Transaction Matching / Reconciliation**
* **Cash Positioning**
* **Cash Forecast Inputs & Exceptions**

### F. Close & reconciliation business processes

* **Account Reconciliation Tracking**
* **Close Task Orchestration & Sign-offs**
* **Close Evidence Packaging (audit trail)**
  (Close & reconciliation is a major “best-of-breed” focus area in the market.) ([bl-prod][6])

---

# 3) Canonical nouns and invariants

Backlog 528 requires canonical nouns/identity axes and invariants. 

## 3.1 Canonical nouns (Business Objects / Data Objects)

* **Ledger** (with **ChartOfAccounts**)
* **Account** (GL account)
* **Dimension** (cost center, project, product, region, etc.)
* **JournalEntry**, **JournalLine**, **Posting**
* **AccountingPeriod** (status: open/soft-closed/closed)
* **LegalEntity**
* **FinancialDocument** (typed): VendorBill, CustomerInvoice, CreditMemo, Payment, CashReceipt, ExpenseReport, Asset, DepreciationRun
* **Reconciliation** (bank/account reconciliation)
* **CloseTask**, **CloseRun**, **CloseEvidenceBundle**
* **TaxCode / TaxJurisdiction / TaxDetermination**
* **Allocation / Accrual**

## 3.2 Identity axes (used in intents/facts/queries)

* `tenant_id`
* `legal_entity_id`
* `ledger_id`
* `accounting_period_id`
* `document_id` (+ `document_type`)
* `counterparty_id` (customer/vendor)
* `currency` (+ `fx_rate_id` or `fx_rate_effective_at`)
* `source_system` + `source_ref` (for traceability & dedupe)
* `correlation_id` / `traceparent` (end-to-end trace)
* `idempotency_key` (especially for integrations/payment events)

## 3.3 Policies / invariants (what must always be true)

Examples (capability-level, implementable):

* **Balanced postings:** every JournalEntry must balance (debits == credits) before posting.
* **Period control:** postings must not be accepted into a closed period; reversals/adjustments must follow policy.
* **Immutability & audit trail:** once posted, a JournalEntry is immutable; corrections are via reversal/adjusting entries.
* **Deterministic dimension rules:** account+dimension validation rules must be deterministic and versioned.
* **Idempotent ingestion:** external receipts/payments/bank lines must be deduped via stable keys.
* **Segregation-of-duties signals:** approvals and evidence must be recorded such that audit can reconstruct “who did what.”

---

# 4) Application layer: contract surface (proto-first)

Backlog 528 requires the capability to define EDA contracts (application services/interfaces/events) realized by primitives. 
Backlog 528 also standardizes topic family naming for intents/facts/process facts. 

## 4.1 Proto module namespace (proposal)

* `package ameide_core_proto.finance.v1;`

  * `ameide_core_proto.finance.ledger.v1`
  * `ameide_core_proto.finance.ap.v1`
  * `ameide_core_proto.finance.ar.v1`
  * `ameide_core_proto.finance.close.v1`
  * `ameide_core_proto.finance.treasury.v1`

(Still **one domain primitive**; subpackages are just schema organization.)

## 4.2 EDA topic families (Application Events / async commands)

* `finance.domain.intents.v1` → `FinanceDomainIntent` 
* `finance.domain.facts.v1` → `FinanceDomainFact` 
* `finance.process.facts.v1` → `FinanceProcessFact` 

## 4.3 Application Services (gRPC) and Application Events

**Application Services (write-side; realized by the Finance Domain primitive):**

* `FinanceLedgerCommandService`

  * `PostJournalEntry`
  * `RecordSourcePosting` (posting request from another capability with `source_ref`)
  * `RequestPeriodClose`
* `FinancePayablesCommandService`

  * `SubmitVendorBill`
  * `ApproveVendorBill`
  * `ScheduleVendorPayment`
* `FinanceReceivablesCommandService`

  * `IssueCustomerInvoice`
  * `ApplyCashReceipt`
  * `RecordWriteOff`
* `FinanceExpenseCommandService`

  * `SubmitExpenseReport`
  * `ApproveExpenseReport`

**Application Services (read-side; realized by Finance Projection primitives):**

* `FinanceReportingQueryService`

  * `GetTrialBalance`
  * `GetGeneralLedgerDetail`
  * `GetAP_Aging`
  * `GetAR_Aging`
  * `GetCashPosition`
  * `GetCloseStatus`
* `FinanceAuditQueryService`

  * `GetAuditTimeline(document_id)`
  * `SearchEvidence(...)`

**Application Events (Domain Facts):**

* `JournalEntryPosted`
* `VendorBillSubmitted` / `VendorBillApproved` / `VendorPaymentScheduled` / `VendorPaymentSettled`
* `CustomerInvoiceIssued` / `CashReceiptApplied`
* `ExpenseReportSubmitted` / `ExpenseReportApproved` / `ExpenseReimbursed`
* `AccountingPeriodClosed`
* `ReconciliationMatched` / `ReconciliationExceptionRaised`

**Application Events (Process Facts):**

* `PeriodCloseStarted` / `PeriodCloseStepCompleted` / `PeriodCloseCompleted`
* `PaymentRunStarted` / `PaymentRunCompleted`
* `BankReconciliationRunCompleted`

## 4.4 Contract rules (Ameide primitives stack alignment)

Finance uses the standard Ameide stack: **operators (control plane) + protobuf/Buƒ (behavior plane) + CI gates (guardrails plane)**. 
Finance proto contracts must remain contracts (not runtime policy), and environment bindings/secrets stay out of proto. 

---

# 5) Realization by primitives

Backlog 528’s mapping: Domains are single-writer; Processes orchestrate cross-domain workflows; Projections build read models; Integrations handle external seams; UISurfaces are thin; Agents consume read models and produce intents/commands. 

## 5.1 Application Components (Ameide primitives)

### Domain primitive (exactly one)

**Application Component:** `finance-ledger-domain` (Domain primitive)
**Responsibility:**

* authoritative writer for:

  * chart of accounts & dimension rules (or references to master data)
  * journal entries/postings
  * finance documents that must be auditable (payables/receivables/expenses/assets postings)
  * period state (open/close)
* emits `FinanceDomainFact` via outbox

### Process primitives (multiple)

**Application Components (Process primitives):**

1. `finance-period-close-process`
   Orchestrates close runs (task checklist, dependencies, sign-offs, evidence bundling).
2. `finance-procure-to-pay-process`
   Orchestrates AP lifecycle: approvals, exception handling, payment runs (calls payment integration).
3. `finance-invoice-to-cash-process`
   Orchestrates AR: dunning/collections steps, cash application exception workflows.
4. `finance-bank-reconciliation-process`
   Orchestrates statement ingestion + matching runs + exception resolution loops.
5. `finance-expense-reimbursement-process`
   Orchestrates expense approval routes + reimbursement execution.

> Design note: these Processes must not become systems-of-record for finance state. 

### Projection primitives (read models)

**Application Components (Projection primitives):**

* `finance-trial-balance-projection`
* `finance-ledger-lines-projection`
* `finance-ap-aging-projection`
* `finance-ar-aging-projection`
* `finance-cash-position-projection`
* `finance-close-dashboard-projection`
* `finance-audit-timeline-projection`

### Integration primitives (external seams)

**Application Components (Integration primitives):**

* `finance-payments-integration` (payment provider connectors; webhooks → idempotent ingestion)
* `finance-banking-integration` (bank feed / statements ingestion)
* `finance-tax-integration` (tax determination / tax filing providers)
* `finance-expense-integration` (e.g., Concur feeds)
* `finance-p2p-integration` (e.g., Coupa feeds)
* `finance-erp-sync-integration` (when coexisting with SAP/Oracle/etc.)

### UISurface primitives (experiences)

**Application Components (UISurface primitives):**

* `finance-ops-portal` (day-to-day AP/AR/GL)
* `finance-close-cockpit` (close tasks, reconciliation status, sign-offs)
* `finance-approvals-surface` (invoices/expenses/adjustments)

UISurfaces read via projections and write via domain commands/intents. 

### Agent primitives (governed assistants)

**Application Components (Agent primitives):**

* `finance-close-assistant-agent`
  Reads close status + reconciliations; proposes next actions; can create “close step” intents (approval gated).
* `finance-ap-coding-agent`
  Suggests GL coding/dimensions for vendor bills; proposes exceptions; submits “coding suggestion” intents.
* `finance-collections-agent`
  Prioritizes delinquent receivables; drafts outreach; proposes dunning steps; can request human approval to send actions.
* `finance-anomaly-monitor-agent`
  Scans for anomalies/exception patterns (mirrors what suites market as AI anomaly detection). ([Workday][13])

---

# 6) Technology layer (constraints & topology)

This capability assumes the standard platform services described by the Ameide primitives stack: operators manage lifecycle/wiring; protos drive contracts + codegen; CI gates enforce determinism/drift. 

## 6.1 Required platform services

* **Workflow runtime:** Temporal (for Process primitives)
* **Transactional store:** Postgres (ledger/documents, outbox, projections)
* **Event transport:** Kafka/NATS/etc (topic families above)
* **API ingress:** Gateway API / HTTPRoute
* **Observability:** OpenTelemetry traces/metrics/logs

## 6.2 Topology modes

* **Cloud-first** (default): all primitives deployed centrally
* **Edge/offline (optional)**:

  * read-only projections cached locally
  * write intents queued and replayed
  * degraded mode defined per value stream (e.g., “cannot hard-close while offline”)

---

# 7) Implementation & Migration view

Backlog 528 requires phases/plateaus/work packages and explicit acceptance slices. 

## 7.1 Plateaus (pragmatic)

**Plateau 0 — Contract-first foundation**

* Deliverables:

  * finance proto module skeletons
  * topic families + envelope metadata
  * CI gates (`buf lint`, `buf breaking`, regen-diff) per Ameide stack 

**Plateau 1 — Ledger posting + trial balance**

* Deliverables:

  * `finance-ledger-domain` basic journal posting + period control
  * `finance-trial-balance-projection`
  * end-to-end slice #1 (below)

**Plateau 2 — AP + payments + reconciliation**

* Deliverables:

  * AP commands + facts (vendor bills → approvals → payment scheduling)
  * payments integration primitive (webhooks + idempotency)
  * bank ingestion + reconciliation process/projection

**Plateau 3 — Close orchestration + audit evidence**

* Deliverables:

  * close process primitive + close dashboard projection
  * evidence bundling projection (auditor-ready timeline)

---

# 8) Acceptance slices (end-to-end)

1. **Vendor bill → approval → payment → posting → trial balance update**

   * Vendor bill ingested → approved → payment executed → `VendorPaymentSettled` fact → journal entries posted → trial balance projection updates.
2. **Bank statement → matching → exception workflow → reconciliation complete → close step satisfied**

   * Bank feed ingested → matching run → exceptions routed → reconciliation completion fact emitted → close cockpit shows step complete.
3. **Customer invoice → cash receipt → cash application → AR aging update**

   * Invoice issued → cash receipt ingested → applied → AR aging projection updates and exceptions tracked.

---

# 9) Application interfaces for Agents (MCP)

Backlog 528 requires making agent access explicit, including a tool catalog table. 
Also note: protocol adapters (MCP servers) belong in Integration primitives; they translate to proto-first services. 

## 9.1 What Finance publishes

* **MCP tools:** Yes (default-deny, explicit allowlist)
* **MCP resources:** Yes (read-only resources backed by Projection query services)
* **MCP prompts:** Optional, discouraged unless versioned/promoted

## 9.2 Tool catalog (initial)

| Tool                           | Kind    | Canonical Application Service                         | Owning primitive | Scopes / risk tier       | Approval | Latency (P95) | Failure modes                            | Evidence / audit                       |
| ------------------------------ | ------- | ----------------------------------------------------- | ---------------- | ------------------------ | -------- | ------------- | ---------------------------------------- | -------------------------------------- |
| `finance.post_journal_entry`   | command | `FinanceLedgerCommandService.PostJournalEntry`        | Domain           | `finance.write` (tier 4) | yes      | seconds       | invalid-arg, permission-denied, conflict | `JournalEntryPosted` + audit timeline  |
| `finance.submit_vendor_bill`   | command | `FinancePayablesCommandService.SubmitVendorBill`      | Domain           | `ap.write` (tier 3)      | maybe    | seconds       | invalid-arg, unavailable                 | `VendorBillSubmitted`                  |
| `finance.approve_vendor_bill`  | command | `FinancePayablesCommandService.ApproveVendorBill`     | Domain           | `ap.approve` (tier 4)    | yes      | seconds       | permission-denied, conflict              | `VendorBillApproved`                   |
| `finance.request_period_close` | command | `FinanceLedgerCommandService.RequestPeriodClose`      | Domain           | `finance.close` (tier 5) | yes      | seconds       | conflict, failed-precondition            | `PeriodCloseRequested` + process facts |
| `finance.get_trial_balance`    | query   | `FinanceReportingQueryService.GetTrialBalance`        | Projection       | `finance.read` (tier 2)  | no       | <1s           | unavailable, partial                     | query log + projection watermark       |
| `finance.search_transactions`  | query   | `FinanceReportingQueryService.GetGeneralLedgerDetail` | Projection       | `finance.read` (tier 2)  | no       | <1s           | unavailable, partial                     | query log + watermark                  |

## 9.3 Resource catalog (initial)

| Resource URI pattern                           | Backing query service                          | Primitive  | Latency (P95) | Notes                            |
| ---------------------------------------------- | ---------------------------------------------- | ---------- | ------------- | -------------------------------- |
| `finance://document/{document_id}`             | `FinanceAuditQueryService.GetAuditTimeline`    | Projection | <1s           | auditor-ready evidence           |
| `finance://trial-balance/{ledger_id}/{period}` | `FinanceReportingQueryService.GetTrialBalance` | Projection | <1s           | cached by period                 |
| `finance://close/{ledger_id}/{period}`         | `FinanceReportingQueryService.GetCloseStatus`  | Projection | <1s           | shows step completion + blockers |

---

## Summary (what you can iterate next)

This Finance capability is defined as a Strategy-layer capability with value streams, business processes, nouns/invariants, and a proto-first contract surface—then realized via **exactly one Domain primitive** plus multiple Process/Projection/Integration/UISurface/Agent primitives, consistent with Backlog 528’s mapping. 

If the next step is “iterate on primitives one by one,” the natural order is:

1. **Finance Domain primitive** (ledger + documents + facts)
2. **Period Close Process**
3. **AP/P2P Process** + Payments Integration
4. **Bank Reconciliation Process** + Banking Integration
5. **Reporting projections** + UISurfaces
6. **Agents** + MCP tool allowlist (governed)

(And all of it stays within the Ameide primitives stack constraints: operators for lifecycle, protos for contracts, CI gates for drift/determinism.) 

[1]: https://www.oracle.com/erp/financials/?utm_source=chatgpt.com "Fusion Cloud Financials"
[2]: https://docs.oracle.com/en/cloud/saas/financials/24c/facsf/overview-of-oracle-financials-cloud.html?utm_source=chatgpt.com "Overview of Oracle Financials Cloud"
[3]: https://www.sage.com/en-us/sage-business-cloud/intacct/product-capabilities/core-financials/?utm_source=chatgpt.com "Sage Intacct Core Accounting Software"
[4]: https://www.netsuite.com/portal/products/erp/financial-management.shtml?utm_source=chatgpt.com "NetSuite Financial Management"
[5]: https://learning.sap.com/learning-journeys/outlining-the-financial-accounting-overview-in-sap-s-4hana/executing-the-steps-of-record-to-report-on-transactions-and-identify-which-sap-solutions-are-applicable_a1a2071e-cf94-4457-9d54-5c4ec8aac18f?utm_source=chatgpt.com "Executing the Steps of Record to Report on Transactions ..."
[6]: https://www.blackline.com/products/financial-close/account-reconciliations/?utm_source=chatgpt.com "Account Reconciliation Software"
[7]: https://www.kyriba.com/products/treasury/?utm_source=chatgpt.com "Intelligent treasury and cash management solutions"
[8]: https://www.coupa.com/products/procure-to-pay/?utm_source=chatgpt.com "Procure-to-Pay Software for Smarter Spend Control"
[9]: https://www.concur.com/blog/article/how-concur-expense-works?utm_source=chatgpt.com "How Concur Expense Works"
[10]: https://tipalti.com/ap-automation/?utm_source=chatgpt.com "AP Automation: End-to-End Accounts Payable Software"
[11]: https://docs.stripe.com/revenue-recognition/methodology?utm_source=chatgpt.com "How revenue recognition works"
[12]: https://learn.microsoft.com/en-us/dynamics365/finance/accounts-payable/accounts-payable?utm_source=chatgpt.com "Accounts payable home page - Finance | Dynamics 365"
[13]: https://www.workday.com/en-us/products/financial-management/accounting-finance.html?utm_source=chatgpt.com "Enterprise Accounting and Finance Software"

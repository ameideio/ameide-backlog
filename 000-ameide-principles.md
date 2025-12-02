## 1. How Ameide sees the business

### 1.1 Universal DDD

Ameide models the whole business landscape as **bounded domains**, not products.

* No ERP vs CRM vs HCM mindset.
* Domains like Customers, Orders, Inventory, Finance, HR, Projects, etc.
* Shared ubiquitous language across services and UIs.

### 1.2 Process‑first (L2C, O2C, P2P, H2R…)

End‑to‑end flows are **first‑class citizens**; apps are just views on those flows.

* Lead‑to‑Cash, Order‑to‑Cash, Procure‑to‑Pay, Hire‑to‑Retire, Record‑to‑Report…
* Implemented as explicit workflows (Temporal) that cross domains.
* **Automation & performance first**,
  manual workspaces are second‑class tools for exceptions, approvals, and investigations.

---

## 2. How Ameide handles change

### 2.1 Transformation at the core

Change is not a side‑project; it’s a **dedicated domain**.

* Requirements → FeatureSpecs → Extensions → Rollouts.
* All governed, versioned, auditable.
* “Implementation” is a productized capability, not a consulting black box.

### 2.2 Self‑developing universal core

The platform learns from real tenant usage.

* Recurring patterns in extensions become **catalog items** or core capabilities.
* Tenant‑specific logic stays tenant‑specific; only truly universal patterns are promoted.
* The core gets richer over time without leaking private behavior.

---

## 3. How intelligence participates

### 3.1 Agentic from any angle

Agents are first‑class actors in Ameide, but always constrained by platform rules.

* On **processes**: design, simulate, reroute, optimize L2C/O2C/etc.
* On **transformations**: propose FeatureSpecs, wire hooks, create extensions.
* On **analysis**: explain why things happen (or are stuck) using workflow + change history.
* Agents operate only via well‑defined tools / APIs, never by free‑hand hacking the core.

---

## 4. How we build the platform

### 4.1 Code‑first, proto‑first, performance‑first

Core capabilities are built like real software, not runtime DSLs.

* Native code (e.g. TS/Go) + **proto‑first contracts** (Buf/BSR).
* Strong typing end to end; minimal runtime magic.
* Declarative patterns (e.g. “invoice should be POSTED”) implemented via controllers/workflows, not giant meta‑engines.

### 4.2 Extensible at the edges, stable at the core

The core is a rock; everything else plugs in.

* Stable core services and schemas per domain.
* Extensibility via:

  * domain events,
  * workflow hooks,
  * UI extension slots,
  * lightweight custom attributes.
* Extensions are compiled, tested packages (backend + frontend), not ad‑hoc scripts.

---

## 5. How we operate in a multi‑tenant, open ecosystem

### 5.1 Tenant‑first, governed & observable

Multi‑tenant power with single‑tenant peace of mind.

* Everything is scoped by `tenantId`: data, workflows, extensions.
* Change and agent actions are **governed**:

  * policies, approvals, rollouts, rollback.
* Deep observability:

  * “What changed for this tenant?”
  * “Which extensions and processes are involved?”

### 5.2 Open by design – APIs & SDKs everywhere

No lock‑in; Ameide plays well with the rest of your stack.

* Every domain exposes **versioned APIs** (proto/gRPC/HTTP).
* Official SDKs for backend and frontend.
* Data can flow **in and out** at any time (integration, export, your own data platform).
* Partners and customers build on the same contracts as Ameide itself.

---

### Super‑short cheat sheet (for a slide)

* **Universal DDD** – domains, not products.
* **Process‑first** – L2C/O2C/P2P as first‑class flows, automation before UI.
* **Transformation at the core** – change is a governed domain.
* **Self‑developing core** – recurring patterns become shared capabilities.
* **Agentic from any angle** – agents for processes, change, and analysis.
* **Code‑first & proto‑first** – real code, typed contracts, fast paths.
* **Extensible edges, stable core** – hooks + slots, no core hacks.
* **Tenant‑first & observable** – scoped, auditable, explainable.
* **Open by design** – APIs/SDKs so you’re never trapped.

If you want, next step we can turn this into a formatted “Ameide Principles” page (like a Notion doc or README section) with a tiny intro and closing.

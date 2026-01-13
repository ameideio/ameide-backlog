# **Ameide Cloud “North‑Star” — Camunda SaaS‑style Operations Blueprint**

> **Audience** Engineering, SRE, Product Management
> **Status** Draft v0.9 (living document – update per quarterly OKR)
> **Objective** Define the target cloud architecture, operating model, and service contracts that give Ameide customers the *same day‑0‑to‑day‑2* experience Camunda SaaS delivers, while preserving Ameide‑specific strengths (model‑to‑code, ArchiMate links, Temporal workflows).

---

## 1  Guiding Principles

| Principle                  | Rationale                              | Practical Implication                                                                    |
| -------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Client‑side first**      | Fast offline modelling, light back end | Keep the full BPMN command stack in the browser; server stores *snapshots*.              |
| **Stateless edges**        | Horizontal scale, lower MTTR           | Web & Sync hubs share no user state; all durable data sits in core services.             |
| **Immutable artefacts**    | Audit, rollbacks, reproducible builds  | Every publish stores a *content‑addressed* blob; never overwrite.                        |
| **Policy ⊂ Pipeline**      | Governance without friction            | Validation gates (JSON‑Schema, BPMN‑lint, EA rules) run synchronously in the Design API. |
| **Separation of concerns** | Design ≠ Deploy ≠ Runtime              | Modelling system is independent of execution engine (Temporal, Camunda, etc.).           |

---

## 2  Target Logical Architecture

```
Browser Modeler
  (bpmn‑js + IndexedDB CommandStore)
       │ 1. publish(snapshot, checksum)
       ▼
┌────────────┐ 2. validate & version  ┌─────────────┐
│ Design API │───────────────────────►│ Model Store │  (Postgres meta + S3 blobs)
└────────────┘                        └─────────────┘
       │3. optional webhook                              
       ▼
┌────────────┐ 4. deploy XML        ┌─────────────┐
│ Deploy API │─────────────────────►│ Runtime API │  (Temporal / Camunda)
└────────────┘                      └─────────────┘

          (optional)
Browser ◄────► Sync Hub (WebSocket / OT)
```

### 2.1  Component summary

| Service                 | Responsibilities                                                                                                                                                               | Tech notes                                                          |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------- |
| **Browser Modeler**     | – Local command log (undo/redo) <br>– Snapshot creation <br>– Optional live OT feed                                                                                            | React, `bpmn-js`, IndexedDB (Dexie)                                 |
| **Sync Hub**            | – Broadcast OT deltas between peers <br>– Presence & comments                                                                                                                  | Node/Go, WebSocket; stateless; pluggable Yjs/Automerge adapter      |
| **Design API**          | – AuthZ check (Keycloak/OPA) <br>– Synchronous policy gates <br>– Version assignment (+ semver tag) <br>– Emits **DesignPublished** event                                      | FastAPI + Pydantic; BPMN‑lint; Zod schema; OpenTelemetry            |
| **Model Store**         | – Postgres (metadata, revision graph) <br>– S3 (XML snapshot, thumbnail SVG) <br>– Immutable; soft‑delete via tombstone                                                        | Flyway migrations; SHA‑256 hash key                                 |
| **Deploy API**          | – Receives designID + engine target <br>– Looks up snapshot <br>– Transforms (if needed) <br>– Calls runtime REST/gRPC <br>– Emits **DeploymentCreated**, **DeploymentFailed** | Camunda 8: REST `/deployments` <br>Temporal: code‑gen → GitOps push |
| **Runtime API**         | – Native engine endpoints <br>– Not part of Ameide but fronted by Gateway                                                                                                      | Camunda 8 SaaS, Temporal Cloud, or self‑hosted                      |
| **Observability Stack** | – Metrics, traces, logs <br>– Process KPIs (modelTime → deployTime)                                                                                                            | Prometheus, Tempo, Loki, Grafana, k6 for SLO tests                  |

---

## 3  Primary Flows

### 3.1  Save / Publish (Single user)

1. **User hits *Save*** → `modeler.saveXML({format:true})`
2. Browser computes `snapshotSHA = SHA‑256(xml)`, `commandChecksum`.
3. `POST /designs/{diagramId}/versions` with `{ xml, snapshotSHA, commandChecksum, parentRevision }`.
4. **Design API** runs:

   * JSON‑Schema validation (shape of XML)
   * BPMN‑lint Camunda rule‑set
   * EA cross‑link checks (if ArchiMate refs present)
5. On success:

   * Inserts `design_versions` row *(revision\_id PK)*.
   * Uploads XML (S3 key = `snapshotSHA.bpmn`).
   * Responds `201 Created { revisionId }`.

### 3.2  Real‑Time Collaboration (Optional)

1. On every local command 🚀 `commandStack.execute(...)` the modeler emits OT delta to **Sync Hub**.
2. Hub rebroadcasts to other browsers in the same `roomId`.
3. Each peer applies delta to local command stack (non‑blocking).
4. Network out? – Hub queues up to *N* messages; browser reconciles later.

No server snapshot until a **Publish**.

### 3.3  Deployment

1. UI (or CI) POSTs `/deployments` with `{ revisionId, target='camunda8' }`.
2. **Deploy API** checks policy (`allowedTargets`).
3. Pulls XML from S3, injects `versionTag = revisionId`.
4. Calls Camunda 8 REST `/deployments`.
5. Stores deployment metadata (`processDefinitionKey`, `version`, `revisionId`).

---

## 4  API Contracts (excerpt)

```http
POST /designs/{id}/versions
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "xml": "<base64>",          // gzipped & b64
  "snapshot_sha": "5a2c…",
  "command_checksum": "7f91…",
  "parent_revision": "rev_41"
}
```

```http
POST /deployments
{
  "version_id": "v_42",
  "target": "camunda8",
  "deploy_as": "order-process"      // override BPMN id
}
```

```http
GET  /designs/{id}/versions/{rev}/diagram
Accept: image/svg+xml
```

*All APIs return RFC 7807 problem+json on error; correlation‑id header `X‑Request‑ID` is mandatory.*

---

## 5  Service‑Level Objectives

| Service         | SLO                                                | Notes                                           |
| --------------- | -------------------------------------------------- | ----------------------------------------------- |
| **Design API**  | *p95 latency* ≤ 300 ms <br>*Availability* ≥ 99.9 % | Validation heavy but CPU‑bound.                 |
| **Sync Hub**    | *Message fan‑out* ≤ 200 ms                         | Bucketed error budget separate from Design API. |
| **Model Store** | *Durability* 11 × 9s                               | Versioned, immutable blobs.                     |
| **Deploy API**  | *Time‑to‑engine‑ACK* ≤ 2 s                         | Depends on Camunda/Temporal latency.            |

---

## 6  Security & Compliance

* **AuthN** – OIDC (Keycloak) with JWT access tokens.
* **AuthZ** – OPA sidecar; policies on `buCode`, `role`.
* **Data egress** – S3 buckets private; presigned URL for download.
* **PII** – No personal data in XML; command log stored **only** client‑side unless collaboration ON.
* **Audit** – Append‑only `audit_log` table captures `revisionId`, `userId`, `ip`, latency.

---

## 7  Operational Playbooks

| Scenario                               | Primary detector             | Immediate action                                                                |
| -------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------- |
| **Validation spike (rule mis‑config)** | 5xx rate > 2 % on Design API | Auto‑rollback last config via ArgoCD; page DevOps.                              |
| **Hub partition**                      | Heartbeat RTT > 1 s          | Browser shows “Offline” banner; queues ops locally.                             |
| **S3 latency**                         | p95 > 200 ms                 | Switch to R2 replica; invalidate edge cache.                                    |
| **Engine reject (BPMN error)**         | Deploy API 409               | Mark deployment *Failed*; alert modellers with error payload; rollback nothing. |

---

## 8  Technical Roadmap

| Quarter  | Theme                               | Key Deliverables                                                        |
| -------- | ----------------------------------- | ----------------------------------------------------------------------- |
| Q1‑Q2    | **MVP parity with Camunda Desktop** | IndexedDB command store; Design API; deploy to Temporal & Camunda 8.    |
| Q3       | **Collaboration‑ready**             | Sync Hub alpha (Yjs); cursor presence; comment threads.                 |
| Q4       | **Advanced governance**             | EA rule engine V1; ArchiMate ↔ BPMN diff visualiser; SLA dashboards.    |
| 12‑18 mo | **Marketplace**                     | Plug‑in registry for lint rules, code‑gen templates, AI command agents. |

---

## 9  KPIs & Success Metrics

| KPI                                 | Target                    | Alignment                              |
| ----------------------------------- | ------------------------- | -------------------------------------- |
| **Median modelling‑to‑deploy time** | ≤ 10 min                  | ETF *Transform* outcome.               |
| **Revision rollback success rate**  | 100 %                     | Audit & compliance.                    |
| **Hub monthly active users (MAU)**  | > 30 % of modellers by Q4 | Validate collaboration investment.     |
| **Validation failure rate**         | < 5 % of publishes        | Indicates quality of governance rules. |

---

## 10  Open Questions (to decide by next architecture board)

1. **OT vs. CRDT** – Prefer Yjs (CRDT) for offline merits, but larger payloads?
2. **Multi‑engine deploy** – Same XML to Temporal & Camunda? Need per‑target transforms.
3. **Billing hooks** – Meter by *active modeller* or *deployment count*?

---

### **One‑Sentence North Star**

> *“Ameide Cloud offers designers the zero‑install, real‑time, and governed modelling experience of Camunda SaaS, while uniquely extending it with model‑to‑code and enterprise‑architecture traceability.”*

Embed this file at `/docs/north‑star/ameide-cloud-ops.md`, review alongside the ETF strategy every quarter, and treat deviations as architecture‑board agenda items.

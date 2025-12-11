# Unified Artifact Framework — MVP Specification

## 1 · Purpose & Background

The purpose of this epic is to consolidate the event‑sourced command architecture—originally designed for BPMN & ArchiMate diagrams—into a **single, typed, and artifact‑agnostic framework** that can handle multiple content types (e.g., BPMN, ArchiMate, Markdown) while preserving deterministic replay, auditability, and governance. The MVP deliberately avoids advanced features such as command‑payload versioning or branch/merge semantics to ensure a fast, low‑risk first release.

This specification builds upon the existing Ameide architecture which uses:
* **Proto‑based API layer** with gRPC services and REST gateway (FastAPI)
* **Domain‑driven design** with clean separation between services, domain, and storage layers
* **Multi‑tenant support** with tenant_id enforcement at all levels
* **Observability‑first** approach using OpenTelemetry for traces, metrics, and structured logging
* **Bazel build system** for consistent, reproducible builds across Python, TypeScript, and Go

## 2 · Vision Statement

> *“Any structured or semi‑structured content our platform hosts should be treated as a first‑class, event‑sourced artifact—traceable from the moment a user edits it to the point it is promoted to production.”*

## 3 · Scope (MVP)

| In‑Scope                                                                                                                                                                                                                                                                                                                                                                                                                                  | Out‑of‑Scope                                                                                                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| • Typed `GenericCommand` envelope with **no `Any` fields** and no payload versioning<br>• Artifact aggregation (`Artifact`, `Revision`, `Snapshot` objects)<br>• Support for three artifact kinds: **BPMN, ArchiMate, Markdown**<br>• Single promotion path (`/promote`) with manual approval only<br>• Deterministic snapshot/replay cycle<br>• Basic secret‑scanning validator (regex‑based)<br>• OTLP traces without per‑sequence tags | • Policy as artifacts (OPA)<br>• Branching & merging<br>• Deployment plugins other than Camunda (BPMN) and static site (Markdown)<br>• Tiered storage & delta snapshots<br>• CRDT conflict resolution beyond Yjs default<br>• AI‑assisted commands or suggestions |

## 4 · Goals & Non‑Goals

### 4.1 Goals

1. **Typed safety**: All commands have a strict protobuf schema; validation failures surface at ingestion.
2. **Deterministic replay**: Given a sequence of commands, any node recreates exactly the same state.
3. **Unified promotion endpoint**: One gRPC method covers promotion for all in‑scope artifact kinds.
4. **Audit & governance ready**: Every command carries user‑id, timestamp, and UUID for idempotency.
5. **≤ 100 ms p95 command processing time** under nominal load (50 cmds/s).

### 4.2 Non‑Goals

* No runtime schema evolution (payload versioning) in MVP.
* No multi‑region replication or offline mode.
* No enterprise‑grade policy engine—manual approvals suffice.

## 5 · Stakeholders

| Role           | Responsibilities                           | Representatives     |
| -------------- | ------------------------------------------ | ------------------- |
| Product Owner  | Defines requirements & acceptance criteria | @Sven               |
| Tech Lead      | Architecture & non‑functional oversight    | @Anika              |
| Backend Team   | Event store, APIs, persistence             | Core‑Platform Squad |
| Front‑End Team | Renderer adapters, optimistic UI           | Studio Squad        |
| Dev Ops        | Deployment pipelines, monitoring           | Platform Ops        |
| QA             | Functional & perf testing                  | Quality Guild       |

## 6 · Artifact Kinds & Editors

| Kind          | Editor Tech  | Minimal Dialect Notes                                              |
| ------------- | ------------ | ------------------------------------------------------------------ |
| **BPMN**      | Diagram‑JS   | Commands limited to create/move/update/delete shapes & connections |
| **ArchiMate** | Archimate‑JS | Same CRUD command subset as BPMN                                   |
| **Markdown**  | Monaco + Yjs | CRDT `yjs.update` command only                                     |

## 7 · Functional Requirements

1. **Create Artifact**: POST/gRPC call initializes an empty artifact of a given kind.
2. **Execute Command**: Each user action is wrapped in a typed `GenericCommand` and persisted.
3. **Snapshot Service**: After *N* commands (configurable, default 100), service consolidates to a snapshot.
4. **Read History**: Clients can stream commands or request a state at sequence *S*.
5. **Promote Revision**: Trigger validation ➜ manual approval ➜ mark revision promoted.
6. **Secret Scan**: Validator scans snapshots; fails promotion if secrets found.

## 8 · Non‑Functional Requirements (NFRs)

| Category          | Requirement                                                          |
| ----------------- | -------------------------------------------------------------------- |
| **Performance**   | ≤ 100 ms p95 for command ingest; ≤ 1 s cold replay of ≤ 1 k commands |
| **Reliability**   | 99.9 % monthly API availability                                      |
| **Scalability**   | Handle 10 concurrent editors per artifact without conflict errors    |
| **Security**      | All data at rest encrypted (AES‑256); TLS 1.3 in transit             |
| **Observability** | OTLP traces + metrics; logs correlate via `command_uuid`             |

## 9 · Constraints & Assumptions

* Postgres 15 remains primary relational store.
* EventStoreDB ≥ 22.xx is used for command streams.
* No schema migrations for command payloads during MVP.
* Front‑end teams can ship updated protobuf bundle within the same quarter.

## 10 · Success Metrics

| Metric                                       | Target       |
| -------------------------------------------- | ------------ |
| Time to onboard new artifact kind (post‑MVP) | ≤ 3 dev‑days |
| Mean command ingest latency                  | < 50 ms      |
| Snapshot creation p95 (1000 cmds)            | < 500 ms     |
| Promotion lead‑time                          | < 5 min      |

## 11 · Acceptance Criteria

1. Editing BPMN, ArchiMate, Markdown simultaneously in one workspace works without errors.
2. Replay of 1000‑command history produces byte‑identical snapshot.
3. Promotion fails if secret validator detects token pattern.
4. OTLP traces visible in Grafana with `artifact_kind` dimension.
5. Documentation published in Confluence > Architecture > Artifacts.

## 12 · Risks & Mitigations

| Risk                        | Likelihood | Impact | Mitigation                       |
| --------------------------- | ---------- | ------ | -------------------------------- |
| Front‑end protobuf mismatch | Medium     | High   | Freeze proto after Sprint #1;    |
| nightly contract tests      |            |        |                                  |
| EventStore disk growth      | Medium     | Medium | Auto‑snapshot + 90‑day retention |
| Secret‑scan false positives | Low        | Medium | Allow override via signed waiver |

## 13 · Milestones & Timeline

| Sprint | Deliverable                                              |
| ------ | -------------------------------------------------------- |
| 1      | Finalize proto & SQL schemas, scaffold BPMN plugin       |
| 2      | ArchiMate plugin, Snapshot Service, OTLP instrumentation |
| 3      | Markdown plugin, UI adapters, Secret‑scan validator      |
| 4      | Promotion API, manual approval UI, end‑to‑end tests      |
| 5      | Hardening, load testing, pilot customer rollout          |

## 14 · Glossary

* **Artifact**: Any version‑controlled content item (diagram, doc, spec).
* **GenericCommand**: Strictly‑typed envelope representing a single user action.
* **Snapshot**: Immutable checkpoint of an artifact’s state at a sequence.
* **Revision**: Named snapshot eligible for promotion.
* **Promotion**: Governance process to mark a revision as production‑ready.

## 15 · References

* Backlog item 065 — *Event‑Sourced Diagram Editor*
* Backlog item 064 — *North Star Architecture* (Camunda SaaS‑style operations)
* Architecture RFC *Unified Artifact Framework* (internal Doc‑1234)
* EventStoreDB 22.x Documentation
* Existing implementations:
  * `packages/ameide_core-storage/src/ameide_core_storage/eventstore/` — Event store abstractions
  * `packages/ameide_core_proto/proto/ameide_core_proto/` — Proto definitions
  * `services/core-api-grpc-rest/` — gRPC/REST API implementation

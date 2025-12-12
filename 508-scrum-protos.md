Below are **complete Proto3 definitions** for a clean **Scrum-only** Transformation subdomain, using **official Scrum Guide artifact/event wording** (Product Backlog, Product Backlog Item, Sprint, Sprint Backlog, Increment, Product Goal, Sprint Goal, Definition of Done; plus Scrum Team accountabilities).

I’m naming the files with your requested prefix: **`transformation-scrum-*`** and putting them in a dedicated package:

`ameide_core_proto.transformation.scrum.v1`

No “requirements”, no “phase gates”, no “Definition of Ready”, no platform-specific evidence fields. This is *Scrum data + event sourcing contracts only*.

---

## Grounding & cross-references

- **Role in architecture:** Canonical Transformation Scrum subdomain contract (`ameide_core_proto.transformation.scrum.v1`) used by `services/transformation` and generated SDKs for other primitives.  
- **Upstream backlogs:** `470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `475-ameide-domains.md`, and `300-400/367-1-scrum-transformation.md` define Transformation’s role and the Scrum profile this package implements.  
- **Runtime seams:** `506-scrum-vertical-v2.md` maps these types onto bus topics (`scrum.domain.intents.v1`, `scrum.domain.facts.v1`); `496-eda-principles.md` constrains envelope/meta fields (`ScrumMessageMeta`, `ScrumAggregateRef`) and event-sourcing rules.  
- **Consumers:** `499-process-operator.md` and `506-scrum-vertical-v2.md` describe Process primitives that consume/emit these messages; `505-agent-developer-v2.md` and `505-agent-developer-v2-implementation.md` describe how AmeidePO/AmeideSA/AmeideCoder use the SDKs; `507-scrum-agent-map.md` locates this package in the Stage 0–3 stack; `502-domain-vertical-slice.md` provides the implementation pattern for Transformation as a Domain primitive.

---

## `transformation-scrum-common.proto`

```proto
syntax = "proto3";

package ameide_core_proto.transformation.scrum.v1;

option go_package = "github.com/ameideio/ameide-core/packages/ameide_sdk_go/gen/ameide_core_proto/transformation/scrum/v1;scrumv1";

import "google/protobuf/timestamp.proto";
import "google/api/field_behavior.proto";

// -----------------------------------------------------------------------------
// Message envelope / metadata
// -----------------------------------------------------------------------------

message ScrumMessageMeta {
  // Globally unique identifier for this message (idempotency key).
  string message_id = 1 [(google.api.field_behavior) = REQUIRED];

  // Semantic schema version of the payload + envelope (e.g. "1.0").
  string schema_version = 2 [(google.api.field_behavior) = REQUIRED];

  // When the producer asserts the message occurred.
  google.protobuf.Timestamp occurred_at = 3 [(google.api.field_behavior) = REQUIRED];

  // Producer identity (service/workload/user) for auditability.
  string producer = 4 [(google.api.field_behavior) = REQUIRED];

  // Optional multi-tenant boundary.
  string tenant_id = 5;

  // Trace fields (optional but strongly recommended).
  string correlation_id = 6;
  string causation_id = 7;

  // Routing keys (must contain product_id; others are optional as applicable).
  ScrumSubject subject = 8 [(google.api.field_behavior) = REQUIRED];

  // Optional actor attribution (who initiated the intent / caused the change).
  ScrumActor actor = 9;
}

message ScrumSubject {
  // Scrum always operates on a Product.
  string product_id = 1 [(google.api.field_behavior) = REQUIRED];

  // Artifact identifiers (set when applicable to the message).
  string product_backlog_id = 2;
  string product_backlog_item_id = 3;
  string sprint_id = 4;
  string increment_id = 5;
  string product_goal_id = 6;
  string definition_of_done_id = 7;
}

message ScrumActor {
  ScrumAccountability accountability = 1 [(google.api.field_behavior) = REQUIRED];

  // Actor identity could be a user ID, service account, workload identity, etc.
  string actor_id = 2 [(google.api.field_behavior) = REQUIRED];

  string display_name = 3;
}

enum ScrumAccountability {
  SCRUM_ACCOUNTABILITY_UNSPECIFIED = 0;

  // Scrum Team accountabilities
  SCRUM_ACCOUNTABILITY_PRODUCT_OWNER = 1;
  SCRUM_ACCOUNTABILITY_SCRUM_MASTER = 2;
  SCRUM_ACCOUNTABILITY_DEVELOPERS = 3;

  // Non-accountability categories (kept for audit only).
  SCRUM_ACCOUNTABILITY_STAKEHOLDER = 4;
  SCRUM_ACCOUNTABILITY_SYSTEM = 5;
}

// -----------------------------------------------------------------------------
// Aggregate identity for event sourcing / idempotent consumers
// -----------------------------------------------------------------------------

message ScrumAggregateRef {
  ScrumAggregateType type = 1 [(google.api.field_behavior) = REQUIRED];
  string id = 2 [(google.api.field_behavior) = REQUIRED];

  // Monotonic per-aggregate version, used for dedupe and ordering.
  int64 version = 3 [(google.api.field_behavior) = REQUIRED];
}

enum ScrumAggregateType {
  SCRUM_AGGREGATE_TYPE_UNSPECIFIED = 0;

  // Scrum artifacts / commitments modeled as aggregates:
  SCRUM_AGGREGATE_TYPE_PRODUCT_BACKLOG = 1;
  SCRUM_AGGREGATE_TYPE_PRODUCT_BACKLOG_ITEM = 2;
  SCRUM_AGGREGATE_TYPE_SPRINT = 3;
  SCRUM_AGGREGATE_TYPE_INCREMENT = 4;
  SCRUM_AGGREGATE_TYPE_PRODUCT_GOAL = 5;
  SCRUM_AGGREGATE_TYPE_DEFINITION_OF_DONE = 6;
}
```

---

## `transformation-scrum-artifacts.proto`

```proto
syntax = "proto3";

package ameide_core_proto.transformation.scrum.v1;

option go_package = "github.com/ameideio/ameide-core/packages/ameide_sdk_go/gen/ameide_core_proto/transformation/scrum/v1;scrumv1";

import "google/protobuf/timestamp.proto";
import "google/api/field_behavior.proto";

// -----------------------------------------------------------------------------
// Scrum artifacts + commitments (data model)
// -----------------------------------------------------------------------------

// PRODUCT BACKLOG (artifact) — commitment: PRODUCT GOAL
message ProductBacklog {
  string product_backlog_id = 1 [(google.api.field_behavior) = REQUIRED];
  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  // The current Product Goal is the commitment for the Product Backlog.
  // Stored as a reference to avoid duplicating Product Goal text in multiple places.
  string current_product_goal_id = 3;

  // An explicit ordering of Product Backlog Items.
  repeated string ordered_product_backlog_item_ids = 4;

  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}

// PRODUCT GOAL (commitment for Product Backlog)
message ProductGoal {
  string product_goal_id = 1 [(google.api.field_behavior) = REQUIRED];
  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  // Short description of the Product Goal.
  string description = 3 [(google.api.field_behavior) = REQUIRED];

  ProductGoalStatus status = 4 [(google.api.field_behavior) = REQUIRED];

  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}

enum ProductGoalStatus {
  PRODUCT_GOAL_STATUS_UNSPECIFIED = 0;

  // "Current" means this goal is the active Product Goal for the Product Backlog.
  PRODUCT_GOAL_STATUS_CURRENT = 1;

  // Goal concluded states (Scrum does not prescribe these labels; they exist to record outcome).
  PRODUCT_GOAL_STATUS_ACHIEVED = 2;
  PRODUCT_GOAL_STATUS_ABANDONED = 3;
  PRODUCT_GOAL_STATUS_SUPERSEDED = 4;
}

// PRODUCT BACKLOG ITEM (element of Product Backlog)
// Scrum Guide describes attributes such as description, order, estimate, value.
message ProductBacklogItem {
  string product_backlog_item_id = 1 [(google.api.field_behavior) = REQUIRED];
  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  // Description attribute.
  string description = 3 [(google.api.field_behavior) = REQUIRED];

  // Order attribute (relative ordering key; exact semantics are implementation-defined).
  int64 order_rank = 4;

  // Estimate attribute (format can vary by context/team).
  Estimate estimate = 5;

  // Value attribute (format can vary by context/team).
  string value = 6;

  // Selection into a Sprint (becomes part of the Sprint Backlog selection).
  // If set, this item is selected for the given Sprint.
  string selected_for_sprint_id = 7;

  // Whether the item meets the Definition of Done (and thus can be included in an Increment).
  bool is_done = 8;
  string done_definition_of_done_id = 9;
  google.protobuf.Timestamp done_at = 10;

  google.protobuf.Timestamp created_at = 20;
  google.protobuf.Timestamp updated_at = 21;
}

message Estimate {
  oneof kind {
    // e.g., story points (commonly used; Scrum itself does not prescribe units)
    int32 points = 1;

    // e.g., "S", "M", "L"
    string t_shirt_size = 2;

    // Free-form estimate representation
    string text = 3;
  }
}

// SPRINT (container event) — timeboxed
message Sprint {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];
  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  google.protobuf.Timestamp start_at = 3 [(google.api.field_behavior) = REQUIRED];
  google.protobuf.Timestamp end_at = 4 [(google.api.field_behavior) = REQUIRED];

  SprintStatus status = 5 [(google.api.field_behavior) = REQUIRED];

  // Sprint cancellation is recorded if it occurs.
  string cancellation_reason = 6;

  // The Sprint Backlog is created/maintained during the Sprint (and is owned by Developers).
  // It contains Sprint Goal, selected PBIs, and a plan.
  SprintBacklog sprint_backlog = 7;

  google.protobuf.Timestamp created_at = 20;
  google.protobuf.Timestamp updated_at = 21;
}

enum SprintStatus {
  SPRINT_STATUS_UNSPECIFIED = 0;
  SPRINT_STATUS_PLANNED = 1;
  SPRINT_STATUS_STARTED = 2;
  SPRINT_STATUS_ENDED = 3;
  SPRINT_STATUS_CANCELED = 4;
}

// SPRINT BACKLOG (artifact) — commitment: SPRINT GOAL
message SprintBacklog {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];

  // Commitment for the Sprint Backlog.
  SprintGoal sprint_goal = 2 [(google.api.field_behavior) = REQUIRED];

  // Product Backlog Items selected for the Sprint.
  repeated string selected_product_backlog_item_ids = 3;

  // A plan for delivering the Increment and realizing the Sprint Goal.
  // Represented as free-form text to avoid introducing non-Scrum task models.
  string plan = 4;

  google.protobuf.Timestamp committed_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}

// SPRINT GOAL (commitment for Sprint Backlog)
message SprintGoal {
  string description = 1 [(google.api.field_behavior) = REQUIRED];
}

// DEFINITION OF DONE (commitment for Increment)
message DefinitionOfDone {
  string definition_of_done_id = 1 [(google.api.field_behavior) = REQUIRED];
  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  // Human-readable DoD. Could be checklist text or structured elsewhere.
  string text = 3 [(google.api.field_behavior) = REQUIRED];

  // Versioning is useful because DoD can evolve.
  int32 version = 4 [(google.api.field_behavior) = REQUIRED];

  google.protobuf.Timestamp effective_from = 10;
  google.protobuf.Timestamp created_at = 11;
}

// INCREMENT (artifact) — additive, meets Definition of Done
message Increment {
  string increment_id = 1 [(google.api.field_behavior) = REQUIRED];
  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  // Increments are commonly associated with a Sprint, though Scrum allows multiple increments
  // in a Sprint and does not require a 1:1 mapping. This field is optional.
  string sprint_id = 3;

  // The Definition of Done the Increment meets.
  string definition_of_done_id = 4 [(google.api.field_behavior) = REQUIRED];

  // PBIs included in this Increment (all must meet DoD).
  repeated string included_product_backlog_item_ids = 5;

  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}
```

---

## `transformation-scrum-intents.proto`

```proto
syntax = "proto3";

package ameide_core_proto.transformation.scrum.v1;

option go_package = "github.com/ameideio/ameide-core/packages/ameide_sdk_go/gen/ameide_core_proto/transformation/scrum/v1;scrumv1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/field_mask.proto";
import "google/api/field_behavior.proto";

import "ameide_core_proto/transformation/scrum/v1/transformation-scrum-common.proto";
import "ameide_core_proto/transformation/scrum/v1/transformation-scrum-artifacts.proto";

// -----------------------------------------------------------------------------
// Domain intents (commands over the bus)
//
// Intent messages request a Scrum-domain state change. They are validated and
// persisted by the Scrum Transformation domain, which then emits facts.
// -----------------------------------------------------------------------------

message ScrumDomainIntent {
  ScrumMessageMeta meta = 1 [(google.api.field_behavior) = REQUIRED];

  oneof intent {
    SetProductGoalRequested set_product_goal_requested = 10;
    CreateProductBacklogItemRequested create_product_backlog_item_requested = 11;
    RefineProductBacklogItemRequested refine_product_backlog_item_requested = 12;
    ReorderProductBacklogRequested reorder_product_backlog_requested = 13;

    CreateSprintRequested create_sprint_requested = 20;
    StartSprintRequested start_sprint_requested = 21;
    CancelSprintRequested cancel_sprint_requested = 22;
    EndSprintRequested end_sprint_requested = 23;

    CommitSprintBacklogRequested commit_sprint_backlog_requested = 30;
    ChangeSprintBacklogRequested change_sprint_backlog_requested = 31;

    DefineDefinitionOfDoneRequested define_definition_of_done_requested = 40;

    RecordProductBacklogItemDoneRequested record_product_backlog_item_done_requested = 50;
    RecordIncrementRequested record_increment_requested = 51;
  }
}

// --- Product Backlog / Product Goal ---

message SetProductGoalRequested {
  // If omitted, the domain may create a new Product Goal ID.
  string product_goal_id = 1;

  // Always required.
  string product_id = 2 [(google.api.field_behavior) = REQUIRED];
  string description = 3 [(google.api.field_behavior) = REQUIRED];

  // Desired status; typically CURRENT for a new goal.
  ProductGoalStatus status = 4 [(google.api.field_behavior) = REQUIRED];
}

message CreateProductBacklogItemRequested {
  // If omitted, the domain may create a new Product Backlog Item ID.
  string product_backlog_item_id = 1;

  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  // Description attribute is required.
  string description = 3 [(google.api.field_behavior) = REQUIRED];

  // Optional attributes.
  int64 order_rank = 4;
  Estimate estimate = 5;
  string value = 6;
}

message RefineProductBacklogItemRequested {
  string product_backlog_item_id = 1 [(google.api.field_behavior) = REQUIRED];

  // Patch + mask so refinement can be partial.
  ProductBacklogItem patch = 2 [(google.api.field_behavior) = REQUIRED];
  google.protobuf.FieldMask update_mask = 3 [(google.api.field_behavior) = REQUIRED];
}

message ReorderProductBacklogRequested {
  string product_backlog_id = 1 [(google.api.field_behavior) = REQUIRED];
  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  // Full explicit order (auditable).
  repeated string ordered_product_backlog_item_ids = 3 [(google.api.field_behavior) = REQUIRED];
}

// --- Sprint / Sprint Backlog ---

message CreateSprintRequested {
  // If omitted, the domain may create a new Sprint ID.
  string sprint_id = 1;

  string product_id = 2 [(google.api.field_behavior) = REQUIRED];
  google.protobuf.Timestamp start_at = 3 [(google.api.field_behavior) = REQUIRED];
  google.protobuf.Timestamp end_at = 4 [(google.api.field_behavior) = REQUIRED];
}

message StartSprintRequested {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];
}

message CancelSprintRequested {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];
  string reason = 2 [(google.api.field_behavior) = REQUIRED];
}

message EndSprintRequested {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];
}

// Outcome of Sprint Planning: Sprint Goal + selected PBIs + plan => Sprint Backlog.
message CommitSprintBacklogRequested {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];

  SprintGoal sprint_goal = 2 [(google.api.field_behavior) = REQUIRED];
  repeated string selected_product_backlog_item_ids = 3 [(google.api.field_behavior) = REQUIRED];

  // Plan for delivering the Increment and realizing the Sprint Goal.
  string plan = 4;
}

message ChangeSprintBacklogRequested {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];

  // Add/remove selected PBIs (optional; can also change plan).
  repeated string add_product_backlog_item_ids = 2;
  repeated string remove_product_backlog_item_ids = 3;

  // Replace plan (optional).
  string plan = 4;
}

// --- Definition of Done / Increment ---

message DefineDefinitionOfDoneRequested {
  // If omitted, the domain may create a new Definition of Done ID.
  string definition_of_done_id = 1;

  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  string text = 3 [(google.api.field_behavior) = REQUIRED];
  int32 version = 4 [(google.api.field_behavior) = REQUIRED];

  google.protobuf.Timestamp effective_from = 10;
}

// Record that a Product Backlog Item meets the Definition of Done.
message RecordProductBacklogItemDoneRequested {
  string product_backlog_item_id = 1 [(google.api.field_behavior) = REQUIRED];

  // Commonly the Sprint in which it became Done; optional in case it’s recorded later.
  string sprint_id = 2;

  // The DoD it meets.
  string definition_of_done_id = 3 [(google.api.field_behavior) = REQUIRED];

  // Optional; if omitted, domain sets done_at to now().
  google.protobuf.Timestamp done_at = 10;
}

// Record/maintain an Increment as a set of Done PBIs that meet the Definition of Done.
message RecordIncrementRequested {
  // If omitted, the domain may create a new Increment ID.
  string increment_id = 1;

  string product_id = 2 [(google.api.field_behavior) = REQUIRED];

  // Optional: associate with a Sprint context.
  string sprint_id = 3;

  string definition_of_done_id = 4 [(google.api.field_behavior) = REQUIRED];

  // Must all be Done (domain validates).
  repeated string included_product_backlog_item_ids = 5 [(google.api.field_behavior) = REQUIRED];
}
```

---

## `transformation-scrum-facts.proto`

```proto
syntax = "proto3";

package ameide_core_proto.transformation.scrum.v1;

option go_package = "github.com/ameideio/ameide-core/packages/ameide_sdk_go/gen/ameide_core_proto/transformation/scrum/v1;scrumv1";

import "google/api/field_behavior.proto";

import "ameide_core_proto/transformation/scrum/v1/transformation-scrum-common.proto";
import "ameide_core_proto/transformation/scrum/v1/transformation-scrum-artifacts.proto";

// -----------------------------------------------------------------------------
// Domain facts (immutable events emitted after persistence)
// -----------------------------------------------------------------------------

message ScrumDomainFact {
  ScrumMessageMeta meta = 1 [(google.api.field_behavior) = REQUIRED];

  // Aggregate identity + version for idempotent consumers.
  ScrumAggregateRef aggregate = 2 [(google.api.field_behavior) = REQUIRED];

  oneof fact {
    // Product Backlog / Product Goal
    ProductGoalSet product_goal_set = 10;
    ProductBacklogItemCreated product_backlog_item_created = 11;
    ProductBacklogItemRefined product_backlog_item_refined = 12;
    ProductBacklogReordered product_backlog_reordered = 13;

    // Sprint / Sprint Backlog
    SprintCreated sprint_created = 20;
    SprintStarted sprint_started = 21;
    SprintCanceled sprint_canceled = 22;
    SprintEnded sprint_ended = 23;

    SprintBacklogCommitted sprint_backlog_committed = 30;
    SprintBacklogChanged sprint_backlog_changed = 31;

    // Definition of Done / Increment
    DefinitionOfDoneDefined definition_of_done_defined = 40;
    ProductBacklogItemDoneRecorded product_backlog_item_done_recorded = 41;

    IncrementCreated increment_created = 50;
    IncrementUpdated increment_updated = 51;
  }
}

// --- Product Backlog / Product Goal facts ---

message ProductGoalSet {
  ProductGoal product_goal = 1 [(google.api.field_behavior) = REQUIRED];
}

message ProductBacklogItemCreated {
  ProductBacklogItem item = 1 [(google.api.field_behavior) = REQUIRED];
}

message ProductBacklogItemRefined {
  ProductBacklogItem item = 1 [(google.api.field_behavior) = REQUIRED];
}

message ProductBacklogReordered {
  ProductBacklog product_backlog = 1 [(google.api.field_behavior) = REQUIRED];
}

// --- Sprint / Sprint Backlog facts ---

message SprintCreated {
  Sprint sprint = 1 [(google.api.field_behavior) = REQUIRED];
}

message SprintStarted {
  Sprint sprint = 1 [(google.api.field_behavior) = REQUIRED];
}

message SprintCanceled {
  Sprint sprint = 1 [(google.api.field_behavior) = REQUIRED];
}

message SprintEnded {
  Sprint sprint = 1 [(google.api.field_behavior) = REQUIRED];
}

message SprintBacklogCommitted {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];
  SprintBacklog sprint_backlog = 2 [(google.api.field_behavior) = REQUIRED];
}

message SprintBacklogChanged {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];
  SprintBacklog sprint_backlog = 2 [(google.api.field_behavior) = REQUIRED];
}

// --- Definition of Done / Increment facts ---

message DefinitionOfDoneDefined {
  DefinitionOfDone definition_of_done = 1 [(google.api.field_behavior) = REQUIRED];
}

message ProductBacklogItemDoneRecorded {
  ProductBacklogItem item = 1 [(google.api.field_behavior) = REQUIRED];
}

message IncrementCreated {
  Increment increment = 1 [(google.api.field_behavior) = REQUIRED];
}

message IncrementUpdated {
  Increment increment = 1 [(google.api.field_behavior) = REQUIRED];
}
```

---

## `transformation-scrum-query.proto`

```proto
syntax = "proto3";

package ameide_core_proto.transformation.scrum.v1;

option go_package = "github.com/ameideio/ameide-core/packages/ameide_sdk_go/gen/ameide_core_proto/transformation/scrum/v1;scrumv1";

import "google/api/field_behavior.proto";

import "ameide_core_proto/transformation/scrum/v1/transformation-scrum-artifacts.proto";

// -----------------------------------------------------------------------------
// Read-only query service for Scrum artifacts.
// (Write-side changes are represented by ScrumDomainIntent over the bus.)
// -----------------------------------------------------------------------------

service ScrumQueryService {
  rpc GetProductBacklog(GetProductBacklogRequest) returns (GetProductBacklogResponse);
  rpc GetProductGoal(GetProductGoalRequest) returns (GetProductGoalResponse);

  rpc ListProductBacklogItems(ListProductBacklogItemsRequest) returns (ListProductBacklogItemsResponse);
  rpc GetProductBacklogItem(GetProductBacklogItemRequest) returns (GetProductBacklogItemResponse);

  rpc ListSprints(ListSprintsRequest) returns (ListSprintsResponse);
  rpc GetSprint(GetSprintRequest) returns (GetSprintResponse);

  rpc GetDefinitionOfDone(GetDefinitionOfDoneRequest) returns (GetDefinitionOfDoneResponse);
  rpc ListDefinitionsOfDone(ListDefinitionsOfDoneRequest) returns (ListDefinitionsOfDoneResponse);

  rpc ListIncrements(ListIncrementsRequest) returns (ListIncrementsResponse);
  rpc GetIncrement(GetIncrementRequest) returns (GetIncrementResponse);
}

// --- Product Backlog / Product Goal ---

message GetProductBacklogRequest {
  string product_id = 1 [(google.api.field_behavior) = REQUIRED];
}

message GetProductBacklogResponse {
  ProductBacklog product_backlog = 1;
}

message GetProductGoalRequest {
  string product_goal_id = 1 [(google.api.field_behavior) = REQUIRED];
}

message GetProductGoalResponse {
  ProductGoal product_goal = 1;
}

// --- Product Backlog Items ---

message ListProductBacklogItemsRequest {
  string product_id = 1 [(google.api.field_behavior) = REQUIRED];

  // Optional pagination.
  int32 page_size = 2;
  string page_token = 3;
}

message ListProductBacklogItemsResponse {
  repeated ProductBacklogItem items = 1;
  string next_page_token = 2;
}

message GetProductBacklogItemRequest {
  string product_backlog_item_id = 1 [(google.api.field_behavior) = REQUIRED];
}

message GetProductBacklogItemResponse {
  ProductBacklogItem item = 1;
}

// --- Sprints ---

message ListSprintsRequest {
  string product_id = 1 [(google.api.field_behavior) = REQUIRED];

  int32 page_size = 2;
  string page_token = 3;
}

message ListSprintsResponse {
  repeated Sprint sprints = 1;
  string next_page_token = 2;
}

message GetSprintRequest {
  string sprint_id = 1 [(google.api.field_behavior) = REQUIRED];
}

message GetSprintResponse {
  Sprint sprint = 1;
}

// --- Definition of Done ---

message GetDefinitionOfDoneRequest {
  string definition_of_done_id = 1 [(google.api.field_behavior) = REQUIRED];
}

message GetDefinitionOfDoneResponse {
  DefinitionOfDone definition_of_done = 1;
}

message ListDefinitionsOfDoneRequest {
  string product_id = 1 [(google.api.field_behavior) = REQUIRED];

  int32 page_size = 2;
  string page_token = 3;
}

message ListDefinitionsOfDoneResponse {
  repeated DefinitionOfDone definitions_of_done = 1;
  string next_page_token = 2;
}

// --- Increments ---

message ListIncrementsRequest {
  string product_id = 1 [(google.api.field_behavior) = REQUIRED];

  // Optional: filter by sprint context.
  string sprint_id = 2;

  int32 page_size = 3;
  string page_token = 4;
}

message ListIncrementsResponse {
  repeated Increment increments = 1;
  string next_page_token = 2;
}

message GetIncrementRequest {
  string increment_id = 1 [(google.api.field_behavior) = REQUIRED];
}

message GetIncrementResponse {
  Increment increment = 1;
}
```

---

### Next (if you want it)

If you confirm the directory path you want (e.g., `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/`), I can also output:

* a **`buf.yaml` + `buf.gen.yaml` snippet** for this package
* the **process-side protos** (Temporal governance facts) as a separate `process-scrum-*` package that remains Scrum-compliant (i.e., timebox notifications for Scrum events, without inventing Scrum artifacts).

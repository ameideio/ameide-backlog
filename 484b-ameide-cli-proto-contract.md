# 484b – Ameide CLI: Proto Contract

> **Deprecation notice (520):** Any section that positions `ameide primitive scaffold` as the canonical proto→runtime generator is deprecated. Canonical v2 uses **`buf generate`** with pinned plugins and generated-only roots. See `backlog/520-primitives-stack-v2.md`.

**Status:** Active
**Audience:** CLI implementers, SDK developers, AI agents
**Scope:** `ameide_core_proto.primitive.v1` message definitions, local vs service mode

> **Parent document**: [484 – Ameide CLI Overview](484-ameide-cli.md)
>
> **Proto naming:** The `ameide_core_proto.primitive.v1` package and any future CLI-related proto packages must follow the conventions in [509-proto-naming-conventions.md](509-proto-naming-conventions.md).

---

## 1. Overview

The CLI commands map to proto messages in `ameide_core_proto.primitive.v1`. This follows the existing proto package convention:

- `ameide_core_proto.graph.v1` → GraphService
- `ameide_core_proto.workflows_runtime.v1` → WorkflowService
- `ameide_core_proto.primitive.v1` → PrimitiveService (new)

**Key insight**: CLI flags map directly to proto request fields. JSON output matches proto response shapes. This enables future service mode without breaking consumers.

---

## 2. Local vs Service Mode

### 2.1 Now: Local CLI

```
┌─────────────────────────────────────────────────────────────────┐
│  Agent / Human                                                  │
│       │                                                         │
│       ▼                                                         │
│  ameide primitive scaffold --json                               │
│       │                                                         │
│       ▼                                                         │
│  CLI parses flags → fills ScaffoldRequest                       │
│       │                                                         │
│       ▼                                                         │
│  CLI does work on filesystem                                    │
│       │                                                         │
│       ▼                                                         │
│  CLI emits ScaffoldResponse as JSON to stdout                     │
│       │                                                         │
│       ▼                                                         │
│  Agent parses JSON (matches proto shape)                        │
└─────────────────────────────────────────────────────────────────┘
```

- CLI runs locally in devcontainer
- JSON output matches `ameide_core_proto.primitive.v1` message shapes
- No network, no server
- Agent just runs command + parses JSON

### 2.2 Later (Optional): Remote Service

```
┌─────────────────────────────────────────────────────────────────┐
│  Agent / CI Bot                                                 │
│       │                                                         │
│       ▼                                                         │
│  PrimitiveServiceClient.Scaffold(ScaffoldRequest)               │
│       │                                                         │
│       ▼                                                         │
│  gRPC/Connect to ameide-primitive-service                       │
│       │                                                         │
│       ▼                                                         │
│  Service does work (remote builds, central codegen)             │
│       │                                                         │
│       ▼                                                         │
│  Returns ScaffoldResponse                                         │
└─────────────────────────────────────────────────────────────────┘
```

If needed later:
- Wrap same proto in a gRPC/Connect server
- Use for remote builds, CI bots, central codegen
- CLI becomes thin client around the service
- **Same proto, same semantics** - just different transport

---

## 3. Proto Definitions

### 3.1 Core Types

```proto
// packages/ameide_core_proto/src/ameide_core_proto/primitive/v1/primitive.proto

syntax = "proto3";

package ameide_core_proto.primitive.v1;

option go_package = "ameide_core_proto/primitive/v1;primitivev1";

enum PrimitiveKind {
  PRIMITIVE_KIND_UNSPECIFIED = 0;
  PRIMITIVE_KIND_DOMAIN = 1;
  PRIMITIVE_KIND_PROCESS = 2;
  PRIMITIVE_KIND_AGENT = 3;
  PRIMITIVE_KIND_UISURFACE = 4;
}
```

### 3.2 Plan Request/Result

```proto
message PlanRequest {
  PrimitiveKind kind = 1;
  string name = 2;
  string proto_path = 3;
  string repo_root = 4;  // Optional: repo root path (default: cwd)
}

message PlanResponse {
  string summary = 1;
  repeated SuggestedScaffold suggested_scaffolds = 2;
  repeated TestToCreate tests_to_create = 3;
}

message SuggestedScaffold {
  PrimitiveKind kind = 1;
  string name = 2;
  map<string, string> paths = 3;  // service, tests, migrations, gitops
}

message TestToCreate {
  string id = 1;
  string description = 2;
  string level = 3;  // unit, integration, e2e
}
```

### 3.3 Scaffold Request/Result

```proto
message ScaffoldRequest {
  PrimitiveKind kind = 1;
  string name = 2;
  string proto_path = 3;
  string lang = 4;  // go, typescript, python
  string repo_root = 5;  // Optional: repo root path (default: cwd)
  bool dry_run = 6;  // If true, return what would be created without writing files
  bool include_gitops = 7;  // If true, include GitOps manifests
  bool include_test_harness = 8;  // If true, include test infrastructure
}

message ScaffoldResponse {
  string summary = 1;
  repeated string files_created = 2;
  repeated string tests_created = 3;
  repeated string next_steps = 4;
}
```

### 3.4 Verify Request/Result

```proto
message VerifyRequest {
  PrimitiveKind kind = 1;
  string name = 2;
  string repo_root = 3;  // Optional: repo root path (default: cwd)
  string mode = 4;  // "mock" (default) or "cluster"
  repeated string checks = 5;  // Optional: specific checks to run (naming, eda, security, etc.)
  string test_filter = 6;  // Optional: specific test ID to run
}

message VerifyResult {
  string summary = 1;  // "pass" or "fail"
  repeated TestResult tests = 2;
  repeated LintResult lint = 3;
  SecurityResult security = 4;
  CommandEventDiscipline command_event_discipline = 5;
  EdaReliability eda_reliability = 6;
  repeated string next_steps = 7;
}

message TestResult {
  string id = 1;
  string status = 2;  // PASS, FAIL, SKIP, TODO
  string details = 3;
  string file = 4;
  int32 line = 5;
}

message LintResult {
  string tool = 1;  // buf-lint, golangci-lint, eslint
  string status = 2;  // PASS, WARN, FAIL
  int32 issues = 3;
}

message SecurityResult {
  CheckResult secret_scan = 1;
  CheckResult dependency_vulns = 2;
  CheckResult sast = 3;
}

message CheckResult {
  string status = 1;
  string tool = 2;
  int32 critical = 3;
  int32 high = 4;
}

message CommandEventDiscipline {
  RpcNamingResult rpc_naming = 1;
  EventCoverageResult event_coverage = 2;
}

message RpcNamingResult {
  string status = 1;
  repeated RpcNamingIssue issues = 2;
}

message RpcNamingIssue {
  string rpc = 1;
  string file = 2;
  int32 line = 3;
  string suggestion = 4;
}

message EventCoverageResult {
  string status = 1;
  int32 commands_with_events = 2;
  int32 commands_without_events = 3;
}

message EdaReliability {
  CheckStatus outbox_wiring = 1;
  CheckStatus event_emission = 2;
  CheckStatus idempotency_guard = 3;
  CheckStatus tenant_validation = 4;
  CheckStatus schema_versioning = 5;
}

message CheckStatus {
  string status = 1;  // PASS, WARN, FAIL
  repeated EdaIssue issues = 2;
}

message EdaIssue {
  string handler = 1;
  string file = 2;
  int32 line = 3;
  string reference = 4;  // e.g., "see 472 §3.3.2"
}
```

### 3.5 Describe Request/Result

```proto
message DescribeRequest {
  PrimitiveKind kind = 1;  // Optional: filter by kind
  string name = 2;  // Optional: filter by name
  string repo_root = 3;
}

message DescribeResult {
  repeated PrimitiveInfo primitives = 1;
  repeated MissingPrimitive expected_but_missing = 2;
}

message PrimitiveInfo {
  PrimitiveKind kind = 1;
  string name = 2;
  string status = 3;  // EXISTS, MISSING, PARTIAL
  string path = 4;
  string proto = 5;
  DriftInfo drift = 6;
}

message DriftInfo {
  bool sdk_stale = 1;
  repeated string missing_tests = 2;
  bool gitops_missing = 3;
}

message MissingPrimitive {
  PrimitiveKind kind = 1;
  string name = 2;
  string reason = 3;
}
```

### 3.6 Drift Request/Result

```proto
message DriftRequest {
  PrimitiveKind kind = 1;
  string name = 2;
  string repo_root = 3;
}

message DriftResponse {
  repeated ProtoSdkDrift proto_sdk_drift = 1;
  repeated TestCoverageDrift test_coverage_drift = 2;
  repeated ConventionDrift convention_drift = 3;
}

message ProtoSdkDrift {
  string proto = 1;
  string sdk = 2;
  string status = 3;  // STALE, CURRENT
  string action = 4;
}

message TestCoverageDrift {
  string primitive = 1;
  string rpc = 2;
  int32 tests = 3;
  int32 expected = 4;
}

message ConventionDrift {
  string primitive = 1;
  string issue = 2;
  string severity = 3;
}
```

### 3.7 Impact Request/Result

```proto
message ImpactRequest {
  string proto_path = 1;
  string repo_root = 2;
}

message ImpactResponse {
  string proto_path = 1;
  repeated Consumer consumers = 2;
  repeated string sdks_affected = 3;
  int32 cascade_tests_required = 4;
  string estimated_scope = 5;  // LOW, MEDIUM, HIGH
}

message Consumer {
  PrimitiveKind kind = 1;
  string name = 2;
  string path = 3;
  repeated string imports = 4;  // Which types/services are imported
  map<string, string> metadata = 5; // Workflow metadata (workflow_cmd, workflow_args, etc.)
}
```

### 3.8 VerifyAll Request/Result (Cascade)

```proto
message VerifyAllRequest {
  string proto_path = 1;  // Optional: only verify consumers of this proto
  string repo_root = 2;
}

message VerifyAllResult {
  string summary = 1;
  ProtoChange proto_change = 2;
  repeated SdkResult sdk_results = 3;
  repeated ConsumerResult consumer_results = 4;
  repeated string next_steps = 5;
}

message ProtoChange {
  string path = 1;
  bool breaking = 2;
  repeated string changes = 3;
}

message SdkResult {
  string sdk = 1;
  string status = 2;
}

message ConsumerResult {
  PrimitiveKind kind = 1;
  string name = 2;
  string status = 3;
  int32 tests = 4;
  int32 failed = 5;
  string details = 6;
}
```

---

## 4. CLI Flag to Proto Field Mapping

### 4.1 `ameide primitive verify`

| CLI Flag | Proto Field |
|----------|-------------|
| `--kind domain` | `kind = PRIMITIVE_KIND_DOMAIN` |
| `--name orders` | `name = "orders"` |
| `--mode cluster` | `mode = "cluster"` |
| `--check eda` | `checks = ["eda"]` |
| `--test orders.cancel.success` | `test_filter = "orders.cancel.success"` |
| `--repo-root /path` | `repo_root = "/path"` |

### 4.2 `ameide primitive scaffold`

| CLI Flag | Proto Field |
|----------|-------------|
| `--kind domain` | `kind = PRIMITIVE_KIND_DOMAIN` |
| `--name orders` | `name = "orders"` |
| `--proto-path ...` | `proto_path = "..."` |
| `--lang go` | `lang = "go"` |
| `--dry-run` | `dry_run = true` |
| `--include-gitops` | `include_gitops = true` |
| `--include-test-harness` | `include_test_harness = true` |

### 4.3 `ameide primitive impact`

| CLI Flag | Proto Field |
|----------|-------------|
| `--proto-path ...` | `proto_path = "..."` |
| `--repo-root /path` | `repo_root = "/path"` |

---

## 5. JSON Output Examples

### 5.1 VerifyResult

```json
{
  "summary": "fail",
  "health": {"kind": "PRIMITIVE_KIND_DOMAIN", "name": "orders", "ready": false},
  "checks": [
    {"check": "Naming", "status": "CHECK_STATUS_PASS", "message": "Proto RPCs use business verbs"},
    {"check": "BufLint", "status": "CHECK_STATUS_PASS", "message": "BufLint passed"},
    {
      "check": "BufBreaking",
      "status": "CHECK_STATUS_FAIL",
      "message": "BufBreaking failed",
      "issues": [
        {"resource": "BufBreaking", "field": "proto", "expected": "clean", "actual": "rpc RemoveOrder missing in new proto"}
      ]
    }
  ],
  "next_steps": [
    "Regenerate the SDKs and re-run buf breaking to verify contract stability.",
    "Fix RemoveOrder RPC compatibility before touching cluster resources."
  ],
  "cascade": [
    {
      "kind": "PRIMITIVE_KIND_PROCESS",
      "name": "l2o",
      "path": "primitives/process/l2o",
      "language": "npm",
      "status": "FAIL",
      "details": "NPM cascade tests failed for primitives/process/l2o"
    },
    {
      "kind": "PRIMITIVE_KIND_UNSPECIFIED",
      "name": "docs-release-verify",
      "path": "release/verify_docs.sh",
      "language": "workflow",
      "status": "WARN",
      "details": "release/verify_docs.sh failed (SUMMARY.md drift)"
    }
  ],
  "buf": {
    "lint_status": "CHECK_STATUS_PASS",
    "lint_details": "BufLint passed",
    "breaking_status": "CHECK_STATUS_FAIL",
    "breaking_details": "rpc RemoveOrder missing in new proto"
  }
}
```

`checks` contains every guardrail invoked (naming, security scans, repo tests, etc.). `buf` surfaces Buf lint/breaking summaries so agents can react without scraping `checks`. `cascade` lists each downstream consumer with its primitive kind, repo path, detected language (Go, npm, pytest, workflow), and PASS/FAIL/SKIP state so agents immediately know which test runner or workflow script must go green again. Workflow consumers stuff their script metadata into `Consumer.metadata` so agents can rerun the exact command outside of the CLI if needed.

### 5.2 ImpactResponse

```json
{
  "proto_path": "ameide_core_proto/transformation/v1/transformation_service.proto",
  "consumers": [
    {"kind": "DOMAIN", "name": "orders", "path": "primitives/domain/orders", "imports": ["OrdersService", "Order"]},
    {"kind": "PROCESS", "name": "l2o", "path": "primitives/process/l2o", "imports": ["OrdersServiceClient"]},
    {"kind": "PRIMITIVE_KIND_UNSPECIFIED", "name": "docs-release-verify", "path": "release/verify_docs.sh", "imports": ["workflow"], "metadata": {"workflow_cmd": "release/verify_docs.sh"}}
  ],
  "sdks_affected": ["ameide-sdk-go", "ameide-sdk-ts"],
  "cascade_tests_required": 25,
  "estimated_scope": "MEDIUM"
}
```

---

## 6. Existing Tooling Alignment

This integrates with:
- **Buf breaking** (`buf.yaml` has `breaking: use: [FILE]`) - detects breaking changes
- **365-buf-sdks-v2** - SDK regeneration and publishing pipeline
- **430 test infrastructure** - consumer tests run in mock/cluster modes

---

## 7. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [365-buf-sdks-v2](365-buf-sdks-v2.md) | SDK generation, Buf integration |
| [430v2 unified test infrastructure](430-unified-test-infrastructure-v2.md) | Test phases and JUnit evidence for verify |
| [472 §3.3](472-ameide-information-application.md) | EDA patterns referenced in verify output |
| [496-eda-principles-v2](496-eda-principles-v2.md) | Full EDA checklist |

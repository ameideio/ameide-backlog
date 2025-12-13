## Rationale recap

Note: this document is now largely superseded by the consolidated `backlog/520-primitives-stack-v2.md`, which includes UISurface alongside Domain/Process/Agent/Projection/Integration. Keep this file as supporting background/reference. Examples use placeholder namespaces (e.g., `acme.*`)—the canonical repo proto namespace is `ameide_core_proto.*` per `backlog/509-proto-naming-conventions.md`.

### 1) Control plane is Kubernetes-native (operators own lifecycle)

You’re treating each primitive kind (Domain/Process/Agent/UISurface/Projection/Integration) as a **Kubernetes API** (CRDs) with a controller that reconciles desired state → Deployments/Services/HTTPRoutes/HPAs/secrets + status/conditions. That’s the core Kubernetes **Operator pattern**: continuously reconcile cluster state and surface health via status. ([Kubernetes][1])

Using **HTTPRoute** specifically is aligned with the modern Gateway API routing model. ([Kubernetes Gateway API][2])

For Temporal concerns (namespaces/queues), the idea “operator manages Temporal things as K8s resources” is already a known shape in the Temporal operator ecosystem. ([Temporal Operator][3])

**Why it’s good:** operators stay language-agnostic (“images, ports, env, volumes”), stable, and cluster-focused.

---

### 2) Behavior plane is IDL-first (protos → generated SDKs + skeletons)

You’re making protos the source of truth for:

* APIs (RPCs), messages
* events/facts
  …and letting code generation produce:
* language SDKs (Go/TS/Py)
* **framework-specific skeletons** (gRPC handlers, Temporal workflow/worker wiring, LangGraph graph/state stubs)

This is fundamentally the same workflow gRPC teaches: define service in `.proto`, generate code, then implement logic. ([gRPC][4])

What makes your approach *cleaner than a bespoke CLI generator* is: **use Buf remote plugins** and `buf generate` as the canonical generator runner, versioned/pinned in `buf.gen.yaml`. ([Buf][5])

**Why it’s good:** deterministic regen, no hand-maintained scaffolds drifting from the proto/SDK contracts.

---

### 3) Guardrails become standard CI checks, not a special CLI workflow

Instead of “ameide primitive verify” being the guardrail gate, the repo gate is:

* Buf lint/breaking checks
* “regen in CI and fail if git diff” (SDK/skeleton up-to-date)
* import policy checks (“no runtime imports of core protos”)
* generated structural tests (Process shape, EDA/outbox shape)
* real behavior tests written by humans/agents

This is exactly what Buf is designed to support: code generation templates + consistent invocation via CI. ([Buf][6])

**Why it’s good:** developers stay in the universal loop: *edit proto → buf generate → run tests → implement until green*.

---

## UISurface (minimal clean target)

UISurface follows the same trio as the other primitive kinds; what changes is the behavior-plane target (web UI scaffolds instead of workers/flows).

**Proto**

* UI-facing API contracts (RPC/HTTP) and typed view models.
* No per-environment endpoints, secrets, or UI policy logic in proto (those are runtime/operator concerns).
* Ameide-specific custom options MUST be SOURCE-retained so they power codegen without leaking into runtime descriptors; generators must read such options from `CodeGeneratorRequest.source_file_descriptors`.

**Generation (generated-only outputs; safe to delete)**

* Client SDKs under `packages/ameide_sdk_ts/src/proto/gen/ts/**` (and the equivalent SDK roots for other languages).
* UI scaffolds/tests under `build/generated/uisurface/**`.
* If “starter” code is desired, emit templates under generated roots and copy them into human-owned UI projects once; do not place human-owned files inside any cleaned generator `out`.

**Operator**

* Reconcile UI workload + Service + HTTPRoute, inject runtime config/secrets, and surface `status.conditions[]`.
* Standard operator↔runtime waist:
  * `PORT_HTTP` for bind
  * `/healthz`, `/readyz`, `/metrics`
* Conditions follow Kubernetes conventions: use `metav1.Condition` with `observedGeneration` set and positive-polarity condition types; for routing, map `RouteReady` from HTTPRoute `Accepted`/`ResolvedRefs`/`Programmed` where available.

---

## How it compares to existing projects

### A) Operator side: very similar to Crossplane / Knative (custom control planes)

* **Crossplane** is explicitly a framework for building control planes via CRDs + controllers that compose underlying resources. That’s extremely close to your “operators own CRDs/resources/status” view. ([Crossplane Docs][7])
* **Knative Serving** defines CRDs (Service/Route/Configuration/Revision) and controllers that manage rollout/traffic. That’s analogous to “DomainOperator produces Deployments/Routes and exposes readiness/conditions”. ([Knative][8])

**Your difference:** you’re not just deploying “a service”; you’re deploying *typed primitive kinds* with additional invariants (EDA outbox shape, Temporal wiring expectations).

---

### B) Proto/codegen side: similar to ConnectRPC + “service scaffold generators”

* **ConnectRPC** demonstrates the exact pattern “proto + Buf plugin generates client/server stubs; you implement logic”. ([Buf][9])
* Frameworks like **go-zero**, **Kratos**, and **Kitex** all offer “proto → generate a service skeleton/project scaffolding” tooling. ([Go-Zero][10])

**Your difference:** those tools tend to be *framework-specific CLIs*. You’re proposing *framework outputs via Buf/protoc plugins* so generation is centralized and language/toolchain-neutral.

---

### C) Temporal + LangGraph projection: “schema-driven framework glue”

* Temporal apps require code registration of workflows/activities via the SDK; generating that wiring/stub code is a natural plugin output (proto informs types/routes; output is normal Temporal SDK code). ([Temporal][11])
* LangGraph’s Graph API is structurally “define state → add nodes/edges → compile”. Generating a StateGraph skeleton + typed state models from proto is straightforward, leaving node implementations to humans/agents. ([LangChain Docs][12])

**Your difference:** few mainstream stacks unify *RPC + event streams + workflow + agent graph* generation under one IDL source. That’s the unique “Ameide behavior plane” value.

---

## The “why this isn’t reinventing the wheel” bottom line

You’re reusing the existing wheels at the right layer:

* **Operators**: standard Kubernetes operator/controller model ([Kubernetes][1])
* **Codegen**: standard protoc/Buƒ plugin mechanism (remote plugins, `buf generate`) ([Buf][5])
* **Dev loop**: standard “generated stubs + implement logic + tests” pattern ([gRPC][4])

You’d be reinventing the wheel primarily if you kept a bespoke CLI that grows its own proto parsing + template generation pipeline, instead of leaning on Buf/protoc plugins.

If you want, I can also summarize the main *design commitments* you’d need to lock (proto annotations vs manifest, generated vs hand-written file boundaries, deletion/regen strategy) because that’s where projects like go-zero/Kratos/Kitex succeed or get messy.

[1]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/ "Operator pattern"
[2]: https://gateway-api.sigs.k8s.io/api-types/httproute/ "HTTPRoute"
[3]: https://temporal-operator.pages.dev/ "Temporal Operator: Overview"
[4]: https://grpc.io/docs/languages/go/quickstart/ "Quick start | Go"
[5]: https://buf.build/docs/bsr/remote-plugins/ "Remote plugins - Buf Docs"
[6]: https://buf.build/docs/generate/ "Code Generation - Buf Docs"
[7]: https://docs.crossplane.io/latest/whats-crossplane/ "What's Crossplane?"
[8]: https://knative.dev/docs/serving/ "Knative Serving"
[9]: https://buf.build/connectrpc/go "connectrpc/go"
[10]: https://go-zero.dev/en/docs/tutorials/cli/rpc "goctl rpc | go-zero Documentation"
[11]: https://docs.temporal.io/namespaces "Temporal Namespace | Temporal Platform Documentation"
[12]: https://docs.langchain.com/oss/python/langgraph/graph-api "Graph API overview - Docs by LangChain"

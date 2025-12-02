# Dashboard View (/dashboard)

## Context

The AMEIDE platform provides two complementary views of the same data:
- **Main page (/)**: Conversational interface for working with artifacts through threads
- **Dashboard page (/dashboard)**: Visual tile-based overview of the same artifacts and processes

Both views access the same underlying data through the `@ameide/core-proto-web` SDK - artifacts, BPMN processes, ArchiMate elements. The dashboard simply presents this data in a monitoring/overview format rather than a conversational format.

## Overview

A **flexible, tile-based dashboard** at `/dashboard` that uses the `@ameide/core-proto-web` SDK directly with mocked data. The dashboard displays the same BPMN processes, ArchiMate elements, and artifacts that users interact with conversationally on the main page, but organized as visual tiles for operational monitoring and status tracking.

## Core Concept

Each organizational role gets a tailored starting dashboard:
- **Enterprise Architect**: Architecture graph, transformation transformations, capability gaps
- **Operations Manager**: Process performance, bottlenecks, operational metrics
- **Product Owner**: Feature delivery, backlogs, customer feedback
- **Finance Controller**: Cost tracking, budget variance, ROI metrics
- **HR Manager**: Capacity, skills gaps, training transformations
- **IT Manager**: System health, incidents, technical debt

Users can then add, remove, or rearrange tiles to suit their needs.

## Design Principles

1. **Start smart, stay flexible** - Role presets that users can modify
2. **Ontology-backed** - TOGAF/ArchiMate concepts structure the data
3. **Plain language surface** - No jargon in the UI
4. **Single container** - All tiles in one unified grid
5. **Repository-centric** - Enterprise graph tile for architecture assets
6. **Visual simplicity** - Clean, scannable tiles with clear metrics

## Component Specifications

### Process Tiles

```
┌───────────────────────────────────────────┐
│ • Order to Cash   🟠 Slow                 │
│   Speed 72h  | Stuck >24h: 47            │
│   Volume: 320 | Trend: ↓ 8h q/q          │
│   [Open queue] [Flag issue]              │
└───────────────────────────────────────────┘
```

**Data points:**
- **Status dot**: 🟢 On track / 🟠 Slow / 🔴 Blocked / ⚪ Unknown
- **Speed**: Average time (e.g., `72h`)
- **Stuck**: Count over threshold (e.g., `>24h: 47`)
- **Volume**: Items processed this period
- **Trend**: Simple arrow vs last period (↑ / ↓ / ~)
- **Actions**: Open queue, Flag issue

### Enterprise Repository Tile

```
┌───────────────────────────────────────────┐
│ ENTERPRISE REPOSITORY               [⚙️]  │
│                                           │
│ 📊 Models: 47                            │
│ 🔧 Applications: 142                     │
│ 📐 Capabilities: 89                      │
│ 🏢 Business Services: 34                 │
│ 📦 Data Objects: 156                     │
│ 🔄 Updates today: 3                      │
│                                           │
│ Recent changes:                          │
│ • Payment Service v2.1 approved          │
│ • Customer Journey map updated           │
│                                           │
│ [Browse] [Search] [Add]                  │
└───────────────────────────────────────────┘
```

**Features:**
- Quick counts of architecture elements
- Recent changes feed
- Direct access to graph browser
- Search across all architecture assets

### Initiative Tiles (CHANGES THIS QUARTER)

```
┌───────────────────────────────────────────┐
│ • Shorten Payment Approval   In progress │
│   Goal: 36h avg by Oct 31                │
│   Expected effect: ~ -6h/order           │
│   Owner: Sarah   Due: Sep 12             │
│   Linked to: Order to Cash               │
│   Risk: 🟠 Medium   Progress: ███░ 65%   │
│   [Next step] [Docs] [Update]            │
└───────────────────────────────────────────┘
```

**Data points:**
- **Title**: Plain language description
- **Stage**: Planned / In progress / Done
- **Goal**: One line with number + date
- **Expected effect**: Rough impact estimate
- **Owner & due date**
- **Linked processes**: Chip tags
- **Risk & progress**: Color indicator + progress bar
- **Actions**: Next step, Docs, Update


## Visual Language

### Color Coding
- **Blue**: Running work/processes
- **Green**: Changes/transformations
- **Red**: Trouble/blocked
- **Orange**: Warning/slow
- **Grey**: Unknown/no data

### Number Styling
- **Solid numbers**: Measured/actual
- **Dotted numbers**: Estimates
- **Bold**: Above threshold
- **Regular**: Normal range

### Layout Rules
- Maximum 3 tiles per column
- Big numbers (24px+)
- Short labels (< 5 words)
- One-line sentences

## SDK Type Mapping

### Tile to Protobuf Type Mapping

- **Process tile** → `BPMNProcess` from BPMNServiceClient
  - Uses: `processId`, `name`, `isExecutable`
  - Extended with: `ProcessMetrics` (to be added)
  
- **Initiative tile** → `ArchiMateElement` where `elementType == ARCHIMATE_ELEMENT_TYPE_WORK_PACKAGE`
  - Uses: `metadata.id`, `name`, `properties` map
  - Extended with: `InitiativeProgress` (to be added)
  
- **Artifact counts tile** → `ArtifactView` from ArtifactQueryServiceClient
  - Uses: `ListArtifactsResponse` with counts by type
  
- **Capability tile** → `ArchiMateElement` where `elementType == ARCHIMATE_ELEMENT_TYPE_CAPABILITY`
  - Uses: `metadata.id`, `name`, `properties` map
  - Extended with: `CapabilityGap` (to be added)

## Data Model (SDK Types)

### Using Protobuf Types from @ameide/core-proto-web

```typescript
import {
  BPMNProcess,
  BPMNElement,
  BpmnElementType,
} from '@ameide/core-proto-web';

import {
  ArchiMateElement,
  ArchiMateElementType,
  ArchiMateLayer,
  ArchiMateRelationship,
} from '@ameide/core-proto-web';

import {
  ResourceMetadata,
  RequestContext,
} from '@ameide/core-proto-web';

// Dashboard extends with runtime metrics
interface ProcessMetrics {
  processId: string;
  throughput: number;
  averageTime: number;
  queueSize: number;
  stuckCount: number;
  trend: 'up' | 'down' | 'stable';
  status: 'healthy' | 'warning' | 'critical';
}

interface InitiativeProgress {
  workPackageId: string;
  phase: 'planning' | 'executing' | 'completed';
  progressPercentage: number;
  targetDate: Date;
  risks: Risk[];
  deliverables: string[];
}

interface CapabilityGap {
  capabilityId: string;
  currentLevel: 1 | 2 | 3 | 4 | 5;
  targetLevel: 1 | 2 | 3 | 4 | 5;
  workPackageIds: string[];
}
```

## Tile Specifications

### Process Tile (using BPMNProcess)
```typescript
interface ProcessTileSettings {
  processId: string              // BPMNProcess.processId
  metricType: 'throughput' | 'latency' | 'errors' | 'queue';
  threshold?: number;            // For status calculation
  timeWindow?: 'hour' | 'day' | 'week';
}

// Data sources:
// - BPMNProcess from BPMNServiceClient
// - ProcessMetrics (to be added to proto)
// Display: name, status indicator, metric value, trend
```

### Initiative Tile (using ArchiMateElement)
```typescript
interface InitiativeTileSettings {
  elementId: string;             // ArchiMateElement.metadata.id
  // Must be ARCHIMATE_ELEMENT_TYPE_WORK_PACKAGE
}

// Data sources:
// - ArchiMateElement from ArchiMateServiceClient
// - InitiativeProgress (to be added to proto)
// Display: name, phase, progress bar, target date
```

### Artifact Count Tile
```typescript
interface ArtifactCountTileSettings {
  artifactTypes?: string[];      // Filter by type
  showRecent?: boolean;
}

// Data sources:
// - ListArtifactsResponse from ArtifactQueryServiceClient
// Display: counts by type, recent changes
```

## Status Rules

### Process Status (calculated from metrics)
- **Healthy**: Metric value within normal range (< threshold)
- **Warning**: Metric value exceeds threshold by < 50%
- **Critical**: Metric value exceeds threshold by > 50%
- **Unknown**: No metric data available

### Initiative Status (from ArchiMate properties)
- **Planning**: Element has property `phase: 'planning'`
- **Executing**: Element has property `phase: 'executing'`
- **Completed**: Element has property `phase: 'completed'`

### Progress Calculation
- Based on custom properties in ArchiMateElement.properties map
- Will be formalized when InitiativeProgress proto is added

## Role Configurations

### Preconfigured Tile Sets

```typescript
const rolePresets = {
  'enterprise-architect': [
    { type: 'artifact-counts', position: { x: 0, y: 0 } },
    { type: 'archimate-capabilities', position: { x: 1, y: 0 } },
    { type: 'archimate-workpackages', position: { x: 2, y: 0 } },
    { type: 'bpmn-processes', position: { x: 0, y: 1 } },
    { type: 'archimate-transformations', position: { x: 1, y: 1 } },
    { type: 'metrics-summary', position: { x: 2, y: 1 } }
  ],
  'operations-manager': [
    { type: 'process-metrics', position: { x: 0, y: 0 } },
    { type: 'process-queue', position: { x: 1, y: 0 } },
    { type: 'process-alerts', position: { x: 2, y: 0 } },
    { type: 'bpmn-processes', position: { x: 0, y: 1 } },
    { type: 'artifact-counts', position: { x: 1, y: 1 } }
  ],
  'product-owner': [
    { type: 'archimate-transformations', position: { x: 0, y: 0 } },
    { type: 'metrics-velocity', position: { x: 1, y: 0 } },
    { type: 'metrics-backlog', position: { x: 2, y: 0 } },
    { type: 'bpmn-processes', position: { x: 0, y: 1 } },
    { type: 'artifact-counts', position: { x: 1, y: 1 } }
  ]
}
```

## Empty States

### No Processes
```
RUNNING TODAY
"Tell us roughly how long one order takes."
[Add number]
```

### No Initiatives
```
CHANGES THIS QUARTER
"Add one change you're starting this month."
[Add change]
```

### No Data
```
WATCHLIST
"Pick 1-3 things to watch. Type numbers now, connect data later."
[Add metric]
```

## Technical Implementation

### SDK Integration

```typescript
import {
  BPMNServiceClient,
  ArchiMateServiceClient,
  ArtifactQueryServiceClient,
} from '@ameide/core-proto-web';

// Initialize clients
const bpmnClient = new BPMNServiceClient('/api');
const archimateClient = new ArchiMateServiceClient('/api');
const artifactClient = new ArtifactQueryServiceClient('/api');

// Mock clients for development
class MockBPMNServiceClient extends BPMNServiceClient {
  async listBPMNProcesses(request: ListBPMNProcessesRequest) {
    return mockBPMNProcesses();
  }
}

class MockArchiMateServiceClient extends ArchiMateServiceClient {
  async listArchiMateElements(request: ListArchiMateElementsRequest) {
    return mockArchiMateElements();
  }
}
```

### 1. Tile API + Registry (Core Contract)

```typescript
// tiles/types.ts
import { z } from "zod";

export type Size = { w: number; h: number };
export type Pos = { x: number; y: number };

export interface TileProps<T> {
  id: string;
  settings: T;
  filters: { area?: string; time?: string; owner?: string };
  mode: "view" | "arrange";
}

export interface TileDefinition<TSettings = unknown> {
  type: string;                           // "process-status", "transformation-card", etc.
  title: string;                          // human label in the catalog
  version: number;                        // for migrations
  settingsSchema: z.ZodSchema<TSettings>; // validates config at runtime
  defaultSize: Size;
  minSize?: Size;
  Component: (p: TileProps<TSettings>) => JSX.Element;
  migrate?: (oldSettings: any, fromVersion: number) => TSettings; // forward compatibility
}

// tiles/registry.ts
export const registry: Record<string, TileDefinition<any>> = {};
export const register = <T>(def: TileDefinition<T>) => (registry[def.type] = def);
```

### 2. Initial Tile Catalog

**BPMN Tiles:**
- `bpmn-process-status` - Shows BPMNProcess with metrics
- `bpmn-process-list` - List of processes from BPMNServiceClient
- `process-metrics` - ProcessMetrics visualization (when proto extended)

**ArchiMate Tiles:**
- `archimate-workpackage` - WorkPackage elements (transformations)
- `archimate-capability` - Capability elements with maturity
- `archimate-elements-list` - List of ArchiMate elements by type

**Artifact Tiles:**
- `artifact-counts` - Counts from ArtifactQueryServiceClient
- `artifact-recent` - Recent artifact changes

**Metric Tiles:**
- `metric-value` - Single KPI display
- `metric-trend` - Trend chart

### 3. Role Presets

```typescript
// presets/roles.ts
export const rolePresets = {
  "enterprise-architect": [
    { 
      type: "artifact-counts", 
      pos: { x:0, y:0 }, size: { w:4, h:3 },
      settings: {}
    },
    { 
      type: "archimate-capability", 
      pos: { x:4, y:0 }, size: { w:4, h:3 },
      settings: { elementType: ArchiMateElementType.ARCHIMATE_ELEMENT_TYPE_CAPABILITY }
    },
    { 
      type: "archimate-workpackage", 
      pos: { x:8, y:0 }, size: { w:4, h:3 },
      settings: { elementType: ArchiMateElementType.ARCHIMATE_ELEMENT_TYPE_WORK_PACKAGE }
    },
    { 
      type: "bpmn-process-list", 
      pos: { x:0, y:3 }, size: { w:6, h:3 },
      settings: {}
    },
  ],
  "operations-manager": [
    { 
      type: "process-metrics", 
      pos: { x:0, y:0 }, size: { w:4, h:3 },
      settings: { metricType: 'queue' }
    },
    { 
      type: "bpmn-process-status", 
      pos: { x:4, y:0 }, size: { w:4, h:3 },
      settings: { metricType: 'throughput', timeWindow: 'day' }
    },
    { 
      type: "metric-value", 
      pos: { x:8, y:0 }, size: { w:4, h:3 },
      settings: { metricId: 'sla-performance' }
    },
  ],
} as const;
```

### 4. Board Configuration Schema

```typescript
// board/schema.ts
export const TileInstance = z.object({
  id: z.string(),
  type: z.string(),
  pos: z.object({ x: z.number(), y: z.number() }),
  size: z.object({ w: z.number(), h: z.number() }),
  settings: z.record(z.any()).default({}),
});
export type TileInstance = z.infer<typeof TileInstance>;

export const Board = z.object({
  id: z.string(),
  title: z.string(),
  filters: z.object({ 
    area: z.string().optional(), 
    time: z.string().optional(), 
    owner: z.string().optional() 
  }),
  tiles: z.array(TileInstance),
  version: z.number().default(1),
});
export type Board = z.infer<typeof Board>;
```

### 5. View vs Arrange Modes

**View Mode:**
- Tiles are interactive (click to drill down)
- Read-only dashboard experience
- Opens drawers/modals for details

**Arrange Mode:**
- Drag & drop tiles with react-grid-layout
- Resize tiles within constraints
- Add/remove from tile catalog
- Edit tile settings in right panel
- Maximum 6-9 tiles visible for clarity

## Plain Language Surface

### UI Labels Stay Simple
- Tile titles: "Process Performance", "Active Changes", "Repository"
- Field labels: "Speed", "Stuck", "Owner", "Progress"
- Actions: "Open", "Flag", "Next Step"

### Ontology in Context
- **Surface**: Shows "Payment Process - 72h average"
- **Context drawer**: Shows full ontology:
  ```
  BusinessProcess: payment-process-001
  Relations:
    - serves → BusinessService: order-fulfillment
    - accesses → DataObject: invoice
    - realizes → Capability: payment-processing (Level 2)
  Requirement: SLA-001 (target: 24h)
  Plateau: Baseline
  ```

### Export/Import
- Always uses ontology IDs and ArchiMate relation names
- Can round-trip between tools that understand ArchiMate

## Implementation Checklist

### Phase 1: SDK Setup & Mock Data
- [ ] Install `@ameide/core-proto-web` package
- [ ] Create mock SDK clients for BPMN and ArchiMate
- [ ] Implement mock data generators matching protobuf types
- [ ] Setup TanStack Query with SDK clients
- [ ] Create dashboard layout with react-grid-layout

### Phase 2: Core Tiles
- [ ] Implement `process-status` tile using BPMNProcess type
- [ ] Implement `transformation-card` tile using ArchiMateElement (WorkPackage)
- [ ] Implement `graph` tile using ArtifactQueryService
- [ ] Create tile registry and plugin system
- [ ] Add View/Arrange mode toggle

### Phase 3: Real SDK Integration
- [ ] Connect to actual gRPC-Web endpoints
- [ ] Implement authentication with bearer tokens
- [ ] Handle streaming updates via subscriptions
- [ ] Add error handling and retry logic

### Phase 4: Proto Extensions
- [ ] Define dashboard-specific proto messages
- [ ] Add ProcessMetrics message type
- [ ] Add WorkPackageProgress message type
- [ ] Regenerate TypeScript SDK with extensions

## Mockup - Enterprise Architect View

```
┌────────────────────────┐ ┌────────────────────────┐ ┌────────────────────────┐
│ ARTIFACT COUNTS        │ │ CAPABILITIES           │ │ WORK PACKAGES         │
│                        │ │                        │ │                        │
│ 📊 BPMN Models: 47    │ │ Payment Processing     │ │ Payment Automation    │
│ 🔧 ArchiMate: 142     │ │ Customer Service       │ │ ███████░░░ 65%        │
│ 📐 Artifacts: 189     │ │ Data Analytics         │ │                        │
│ 🔄 Updates today: 3   │ │                        │ │ API Gateway           │
│                        │ │ [View all]             │ │ ████░░░░░░ 30%        │
│ [Browse]              │ │                        │ │                        │
└────────────────────────┘ └────────────────────────┘ │ [All transformations]      │
                                                      └────────────────────────┘

┌──────────────────────────────────────────────────┐ ┌────────────────────────┐
│ BPMN PROCESSES                                   │ │ METRICS SUMMARY        │
│                                                   │ │                        │
│ Order to Cash         [View] [Edit]              │ │ Throughput: 1.2k/h     │
│ Quote to Order        [View] [Edit]              │ │ Avg Time: 72h          │
│ Procure to Pay        [View] [Edit]              │ │ Queue Size: 234        │
│                                                   │ │                        │
│ [View all processes]                             │ │ [Configure]            │
└──────────────────────────────────────────────────┘ └────────────────────────┘
```

## Mockup - Operations Manager View

```
┌────────────────────────┐ ┌────────────────────────┐ ┌────────────────────────┐
│ PROCESS METRICS        │ │ BPMN PROCESS STATUS    │ │ METRIC VALUE          │
│                        │ │                        │ │                        │
│ Queue Size: 234        │ │ Order Processing       │ │ SLA Performance       │
│ Avg Wait: 3.2h         │ │ 🟠 Warning             │ │                        │
│ Throughput: 450/h      │ │ Throughput: 450/h      │ │ 87%                   │
│                        │ │                        │ │                        │
│ Trend: ↗ worsening     │ │ Payment Processing     │ │ Target: 95%           │
│                        │ │ 🟢 Healthy             │ │                        │
│ [Details]              │ │ Throughput: 1.2k/h     │ │ [Configure]           │
└────────────────────────┘ └────────────────────────┘ └────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ BPMN PROCESSES LIST                                                        │
│                                                                             │
│ Name                   Status    Throughput    Queue    Actions            │
│ Order to Cash          🟠        450/h         234      [View] [Metrics]   │
│ Quote to Order         🟢        1.2k/h        12       [View] [Metrics]   │
│ Procure to Pay         🟢        320/h         45       [View] [Metrics]   │
│                                                                             │
│ [Refresh]                                                                   │
└────────────────────────────────────────────────────────────────────────────┘
```

## Key Architectural Decisions

### 1. Plugin Architecture
- **Tiles are self-contained plugins** with their own settings schema
- **Single registry** for all tiles - easy to add new ones
- **Runtime validation** with Zod ensures type safety

### 2. Configuration-Driven
- **Board state is just JSON** - easy to persist, share, version
- **Role presets are data** not code - can be loaded from API
- **Settings are validated** at runtime, not just compile time

### 3. Separation of Concerns
- **Tiles don't know about the grid** - just render in their space
- **Grid doesn't know tile internals** - just positions and sizes
- **Filters propagate down** - tiles react to context changes

### 4. Progressive Enhancement
- **Start with 10 core tiles** - cover 80% of use cases
- **Add domain-specific tiles** later (capability-gaps, sla-performance)
- **Custom tiles** can be registered by power users

### 5. User Experience First
- **Maximum 6-9 tiles** - prevent information overload
- **Two modes only** - View (use) vs Arrange (customize)
- **Role presets** - good defaults, full customization

## Technical Stack

- **React** - Component framework
- **react-grid-layout** - Drag & drop grid system
- **Zod** - Runtime type validation
- **Tailwind CSS** - Styling
- **Zustand** - State management for board config
- **React Query** - Data fetching and caching

## Proto Extensions Required

The following message types need to be added to the protobuf definitions:

```proto
// dashboard/v1/metrics.proto
message ProcessMetrics {
  string process_id = 1;
  double throughput = 2;
  double average_time = 3;
  int32 queue_size = 4;
  int32 stuck_count = 5;
  enum Trend {
    TREND_UNSPECIFIED = 0;
    TREND_UP = 1;
    TREND_DOWN = 2;
    TREND_STABLE = 3;
  }
  Trend trend = 6;
  google.protobuf.Timestamp timestamp = 7;
}

message InitiativeProgress {
  string element_id = 1;  // ArchiMate WorkPackage ID
  enum Phase {
    PHASE_UNSPECIFIED = 0;
    PHASE_PLANNING = 1;
    PHASE_EXECUTING = 2;
    PHASE_COMPLETED = 3;
  }
  Phase phase = 2;
  double progress_percentage = 3;
  google.protobuf.Timestamp target_date = 4;
  repeated string deliverable_ids = 5;
  repeated string risk_ids = 6;
}

message CapabilityGap {
  string capability_id = 1;  // ArchiMate Capability ID
  int32 current_level = 2;   // 1-5
  int32 target_level = 3;    // 1-5
  repeated string work_package_ids = 4;
}

// dashboard/v1/dashboard.proto
message DashboardConfig {
  string id = 1;
  string user_id = 2;
  string name = 3;
  repeated TileInstance tiles = 4;
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp updated_at = 6;
}

message TileInstance {
  string id = 1;
  string tile_type = 2;
  Position position = 3;
  Size size = 4;
  google.protobuf.Struct settings = 5;
}

message Position {
  int32 x = 1;
  int32 y = 2;
}

message Size {
  int32 width = 1;
  int32 height = 2;
}
```

## Success Metrics

1. **Type Safety**: 100% protobuf type coverage
2. **Mock Coverage**: All tiles work with mock SDK clients
3. **Performance**: Initial load < 2s with mocked data
4. **SDK Integration**: Seamless switch from mock to real clients
5. **Proto Alignment**: All data flows through generated SDK

## Next Steps

1. Install `@ameide/core-proto-web` package
2. Create mock SDK client implementations
3. Implement tile components using protobuf types
4. Setup react-grid-layout for dashboard
5. Add TanStack Query for data fetching
6. Define proto extensions for dashboard-specific needs
7. Generate updated SDK with extensions

## References

- SDK Package: `@ameide/core-proto-web` - Generated gRPC-Web client
- Protobuf Definitions: `/workspace/packages/ameide_core_proto/proto/`
- Design System: Tailwind CSS with shadcn/ui components
- Authentication: Keycloak SSO with bearer tokens for SDK
# Artifact-Centric Graph Architecture - Domain Model

## Core Interfaces

```typescript
interface Artifact {
  id: string;                    // UUID
  kind: string;                   // e.g., "common::Document", "bpmn::Process"
  name: string;
  description?: string;
  createdAt: Date;
  createdBy: string;              // User ID
  lastEventSeq: number;           // For causality tracking
  properties: Record<string, any>; // Kind-specific properties
}

interface ArtifactRelation {
  id: string;
  fromId: string;                  // Source artifact
  toId: string;                    // Target artifact
  kind: string;                    // e.g., "structural::hasPart", "bpmn::triggers"
  properties: Record<string, any>; // Relation-specific properties
  createdAt: Date;
  createdBy: string;
}
```

## Kind Taxonomy

### Common Core Kinds (`common::`)

```yaml
# Basic structural artifacts
- kind: common::folder
  description: Container for organizing artifacts
  properties: [name, description, order]

- kind: common::document
  description: Text-based content (markdown, rich text)
  properties: [title, content, format, version]

- kind: common::image
  description: Visual assets (PNG, SVG, etc)
  properties: [title, url, mimeType, dimensions]

- kind: common::table
  description: Structured tabular data
  properties: [name, columns[], rows[]]

- kind: common::codeblock
  description: Source code snippets
  properties: [language, content, filename]

# Annotations and metadata
- kind: common::note
  description: Annotation on any artifact
  properties: [content, author, timestamp]

- kind: common::tag
  description: Categorization label
  properties: [name, color, category]

- kind: common::comment
  description: Discussion thread
  properties: [content, author, timestamp, resolved]
```

### Structural Relations (`structural::`)

```yaml
# Composition (parent owns child lifecycle)
- kind: structural::haspart
  description: Composite aggregation (UML-style)
  symmetric: false
  properties: [order, role]

# Dependency
- kind: structural::dependson
  description: Requires for completeness
  symmetric: false
  properties: [type, strength]

# Reference (loose coupling)
- kind: structural::references
  description: Mentions or links to
  symmetric: false
  properties: [context, anchor]

# Derivation
- kind: structural::derivedfrom
  description: Generated or transformed from
  symmetric: false
  properties: [method, timestamp]
```

### Document Structure Kinds (`doc::`)

```yaml
# Document hierarchy
- kind: doc::book
  description: Top-level document container
  properties: [title, author, isbn, publishedDate]
  children: [doc::chapter, doc::frontmatter, doc::backmatter]

- kind: doc::chapter
  description: Major document division
  properties: [number, title, abstract]
  children: [doc::section, doc::figure, doc::table]

- kind: doc::section
  description: Sub-division of chapter
  properties: [number, title, level]
  children: [doc::subsection, doc::paragraph, doc::list]

- kind: doc::paragraph
  description: Block of text
  properties: [content, style]
  children: [doc::sentence, doc::inlineimage, doc::footnote]

# Document elements
- kind: doc::figure
  description: Embedded illustration
  properties: [caption, source, number]
  relations: [references -> common::image]

- kind: doc::table
  description: Embedded data table
  properties: [caption, number]
  relations: [references -> common::table]

# Specialized document types
- kind: doc::spreadsheet
  description: Excel-like document
  children: [doc::sheet]

- kind: doc::presentation
  description: Slide deck
  children: [doc::slide]
```

### View/Diagram Kinds (`view::`)

```yaml
# Canvas and diagrams
- kind: view::canvas
  description: 2D drawing surface
  properties: [width, height, viewBox, zoom]
  children: [view::shape, view::connector]

- kind: view::diagram
  description: Specialized canvas with semantics
  properties: [type, notation, version]
  subtypes: [view::bpmndiagram, view::archimatediagram, view::umldiagram]

# Visual elements
- kind: view::shape
  description: Visual node/box
  properties: [x, y, width, height, style]
  relations: [represents -> artifact]

- kind: view::connector
  description: Visual edge/line
  properties: [sourceX, sourceY, targetX, targetY, points[], style]
  relations: [represents -> relation]
```

### ArchiMate Kinds (`archimate::`)

```yaml
# Strategy Layer
- kind: archimate::capability
  properties: [name, description, maturity]
  relations: [realizes -> archimate::outcome]

- kind: archimate::resource
  properties: [name, type]

- kind: archimate::courseofaction
  properties: [name, description, priority]

# Business Layer
- kind: archimate::businessactor
  properties: [name, type, external]

- kind: archimate::businessrole
  properties: [name, description]

- kind: archimate::businessprocess
  properties: [name, description]
  relations: [realizes -> archimate::businessservice]

# Application Layer
- kind: archimate::applicationcomponent
  properties: [name, version, vendor]

- kind: archimate::applicationservice
  properties: [name, description, external]

- kind: archimate::dataobject
  properties: [name, type, format]

# Technology Layer
- kind: archimate::node
  properties: [name, type, location]

- kind: archimate::systemsoftware
  properties: [name, version, vendor]

# Relations
- kind: archimate::realizes
- kind: archimate::serves
- kind: archimate::assignment
- kind: archimate::access
- kind: archimate::triggering
- kind: archimate::flow
```

### BPMN Kinds (`bpmn::`)

```yaml
# Process elements
- kind: bpmn::process
  properties: [name, executable, type]
  children: [bpmn::task, bpmn::event, bpmn::gateway]

- kind: bpmn::task
  properties: [name, type] # User, Service, Script, etc.

- kind: bpmn::event
  properties: [name, type, trigger] # Start, End, Intermediate

- kind: bpmn::gateway
  properties: [name, type] # Exclusive, Parallel, Inclusive

# Flow objects
- kind: bpmn::sequenceflow
  properties: [name, condition]
  relations: [from -> flownode, to -> flownode]

- kind: bpmn::messageflow
  properties: [name, message]
  relations: [from -> participant, to -> participant]

# Containers
- kind: bpmn::lane
  properties: [name, actor]
  parent: bpmn::pool

- kind: bpmn::pool
  properties: [name, participant]
  children: [bpmn::lane]
```

## Data Modeling Best Practices

### Promote Hot Fields for Indexing

```typescript
// ❌ BAD: Nested properties can't be indexed
{
  kind: "archimate::ApplicationComponent",
  properties: {
    status: string;      // Can't index efficiently
    approved: boolean;   // Can't query directly
    priority: number;    // Slow to filter
  }
}

// ✅ GOOD: Top-level indexed fields
{
  kind: "archimate::ApplicationComponent",
  
  // Indexed for queries
  status: "production",    // CREATE INDEX artifact_status
  approved: true,          // CREATE INDEX artifact_approved
  priority: 1,             // CREATE INDEX artifact_priority
  
  // Non-indexed metadata in properties
  properties: {
    documentation: "...",
    notes: "...",
    customFields: {}
  }
}
```

### Search Field Pattern

```typescript
// Combine searchable text for full-text index
{
  kind: "bpmn::Task",
  name: "Validate Order",
  description: "Check inventory and pricing",
  
  // Computed field for full-text search
  search: "validate order check inventory pricing validation task",
  
  // CREATE FULLTEXT INDEX artifact_search
  // FOR (a:Artifact) ON EACH [a.search]
}
```

### Binary/Large Content Storage

```typescript
// Store references, not content
{
  kind: "doc::Figure",
  name: "System Architecture",
  
  // Reference to S3/MinIO
  contentUrl: "s3://artifacts/diagrams/sys-arch-v2.png",
  mimeType: "image/png",
  sizeBytes: 245823,
  
  // Thumbnail for quick preview
  thumbnailUrl: "s3://artifacts/thumbs/sys-arch-v2-thumb.jpg"
}
```

### Message Thread Modeling

```typescript
// Message threads as artifacts with edges
{
  thread: {
    kind: "message::Thread",
    subject: "Design Review",
    participants: ["user1", "user2"],
    lastActivity: "2024-01-15T10:30:00Z"
  },
  
  messages: [
    {
      kind: "message::Post",
      content: "Please review the attached design",
      author: "user1",
      timestamp: "2024-01-15T09:00:00Z"
    }
  ],
  
  // Edges connect messages to thread
  edges: [
    "(Thread)-[:HAS_MESSAGE {order: 1}]->(Post1)",
    "(Thread)-[:HAS_MESSAGE {order: 2}]->(Post2)"
  ]
}
```

## Change Proposal Domain Model

### ChangeSet Aggregate

```typescript
interface ChangeSet {
  id: string;                      // UUID
  tenantId: string;
  graphId: string;
  status: ChangeSetStatus;
  proposerId: string;               // User or agent ID
  title: string;
  description: string;
  rationale?: string;               // Why this change
  aiGenerated?: boolean;
  aiModel?: string;
  confidenceScore?: number;          // For AI proposals
  riskSummary?: string;
  createdAt: Date;
  updatedAt: Date;
  touches: TouchedStream[];         // Streams affected
  proposedEvents: ProposedEvent[];  // Events to apply
}

enum ChangeSetStatus {
  DRAFT = 'draft',
  PENDING_REVIEW = 'pending_review',
  APPROVED = 'approved',
  REJECTED = 'rejected',
  CONFLICTED = 'conflicted',
  COMMITTED = 'committed'
}

interface TouchedStream {
  streamId: string;
  expectedVersion: number;
}

interface ProposedEvent {
  targetStreamId: string;
  expectedVersion: number;
  eventType: string;
  eventPayload: any;
  intent?: Intent;                  // For rebase capability
}

interface Intent {
  commandType: string;
  commandPayload: any;
  streamId: string;
}
```

### ChangeSet Events

```typescript
// Domain events for ChangeSet lifecycle
interface ChangeSetCreated {
  changesetId: string;
  tenantId: string;
  graphId: string;
  title: string;
  proposerId: string;
  aiGenerated: boolean;
  createdAt: Date;
}

interface ProposedEventsAdded {
  changesetId: string;
  events: ProposedEvent[];
  addedAt: Date;
}

interface ChangeSetSubmittedForReview {
  changesetId: string;
  submittedAt: Date;
  submittedBy: string;
}

interface ChangeSetApproved {
  changesetId: string;
  approvedAt: Date;
  approvedBy: string;
  comment?: string;
}

interface ChangeSetRejected {
  changesetId: string;
  rejectedAt: Date;
  rejectedBy: string;
  reason: string;
}

interface ChangeSetCommitted {
  changesetId: string;
  committedAt: Date;
  committedBy: string;
  affectedStreams: string[];
}

interface ChangeSetConflicted {
  changesetId: string;
  conflicts: StreamConflict[];
  detectedAt: Date;
}

interface ChangeSetRebased {
  changesetId: string;
  oldEvents: ProposedEvent[];
  newEvents: ProposedEvent[];
  rebasedAt: Date;
  rebasedBy: string;
}

interface StreamConflict {
  streamId: string;
  expectedVersion: number;
  actualVersion: number;
}
```

### Preview Models

```typescript
interface ChangeSetPreview {
  changesetId: string;
  diffs: ArtifactDiff[];
  graphOverlay?: GraphOverlay;
  impacts: ImpactAssessment[];
}

interface ArtifactDiff {
  streamId: string;
  artifactId: string;
  before: ArtifactState;
  after: ArtifactState;
  changes: FieldChange[];
}

interface ArtifactState {
  name: string;
  kind: string;
  status?: string;
  properties: Record<string, any>;
}

interface FieldChange {
  field: string;
  oldValue: any;
  newValue: any;
  operation: 'add' | 'update' | 'remove';
}

interface GraphOverlay {
  addedNodes: NodePreview[];
  updatedNodes: NodePreview[];
  deletedNodes: string[];
  addedEdges: EdgePreview[];
  updatedEdges: EdgePreview[];
  deletedEdges: string[];
}

interface NodePreview {
  id: string;  // MUST use "preview_" prefix for ephemeral IDs
  kind: string;
  properties: Record<string, any>;
  isEphemeral?: boolean;  // True for uncommitted preview nodes
}

interface EdgePreview {
  fromId: string;  // MUST use "preview_" prefix if ephemeral
  toId: string;    // MUST use "preview_" prefix if ephemeral
  kind: string;
  properties: Record<string, any>;
  isEphemeral?: boolean;  // True for uncommitted preview edges
}

// ID generation for previews
function generatePreviewId(): string {
  // Prevent collision with real UUIDs by using prefix
  return `preview_${uuidv4()}`;
}

function isPreviewId(id: string): boolean {
  return id.startsWith('preview_');
}

interface ImpactAssessment {
  artifactId: string;
  impactType: 'direct' | 'indirect';
  distance: number;
  description: string;
  severity: 'low' | 'medium' | 'high';
}
```
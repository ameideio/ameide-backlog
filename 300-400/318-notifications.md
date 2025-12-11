> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 318: Notification System Implementation

**Status:** Backlog
**Priority:** High
**Estimated Effort:** 8-12 weeks
**Dependencies:** Event Store, Repository Service, www_ameide_platform

---

## Problem Statement

The platform currently lacks a notification system to inform users about relevant activities, mentions, assignments, and updates across repositories, elements, and transformations. Users need to:
- Receive real-time notifications for important events
- Browse activity feeds in context (org, graph, transformation)
- Manage notification preferences
- Access notification history
- See what's new since their last visit

---

## Solution Overview

Implement a notification system that builds on the platform's event-sourced architecture, treating notifications as projections of domain events with user-specific state.

### Core Principles

1. **Event-Driven by Default** - Notifications consume events from the event store, not triggered directly by command handlers
2. **Eventual Consistency** - Notification delivery can lag seconds to minutes behind events
3. **Notifications as Read Model** - Notifications are projections optimized for user consumption
4. **Separation of Activity and Notification** - Activities exist independently; notifications are filtered views with personal state
5. **Multi-Tenant Aware** - Clear org attribution, per-tenant preferences, cross-org aggregation
6. **Calm by Design** - Actionable triage, quiet hours, subtle real-time feedback (Material Design principles)

---

## Functional Architecture

### Component Model

```
Event Store (Single Source of Truth)
    ↓
Activity Processor
    ↓
Activity Feed Store (Context-scoped, immutable)
    ↓
Notification Rule Engine (Recipient resolution)
    ↓
Notification State Store (User-scoped, mutable)
    ↓
Delivery Channels (WebSocket, Email, Webhook)
```

### Key Distinctions

| Aspect | Notifications | Activity Feeds |
|--------|--------------|----------------|
| **Purpose** | Alert, interrupt | Discover, browse |
| **Recipient** | Specific users | Context observers |
| **State** | Read/Unread/Dismissed | No intrinsic state |
| **Filtering** | Highly personalized | Contextually scoped |
| **Retention** | Days to weeks | Months to years |
| **Delivery** | Push + Pull | Pull only |
| **Count** | Important metric (unread badge) | Not meaningful |

---

## UX Architecture

### Dual Inbox Pattern

**Global Inbox** (`/inbox`):
- Aggregates activity from all organizations
- Reduces context switching for multi-org users
- Default sort: time-based, newest first
- Org filter chips for multi-select scoping

**Context Inboxes**:
- `/org/[orgId]/inbox` - Organization-scoped
- `/org/[orgId]/repo/[graphId]/inbox` - Repository-scoped
- `/org/[orgId]/transformations/[transformationId]/inbox` - Initiative-scoped
- Auto-scoped when inside a context, one-click to widen

**Industry References**: Slack multi-workspace, GitHub inbox triage

### Notification Bell Placement

**Header Actions** (Fixed, z-50):
- Bell icon with unread badge (total across orgs)
- Click opens popover: recent 10 items, org filter chips, "View full inbox" link
- Popover shows per-org counts without overwhelming user
- Single focus point reduces anxiety (Material badge guidelines)

### Multi-Tenant UX Rules

1. **Always show origin**: Org chip (name + color dot) + scope crumb (Org › Repo › Item)
2. **Two default scopes**: Global (all orgs) and Context (auto-scoped inside org)
3. **Counts communicate, not nag**: Single bell badge, per-org counts in popover
4. **Privacy by design**: Re-check permissions at display time, never leak cross-tenant content
5. **Actionability**: Done, Save, Unsubscribe, Snooze, Mute - all in-place actions (GitHub pattern)

### Triage Verbs

| Action | Purpose | Reference |
|--------|---------|-----------|
| **Done** | Clear clutter, item viewable in history | GitHub "Mark as Done" |
| **Save** | Park for review later, never expires | GitHub "Saved" |
| **Unsubscribe/Unwatch** | Reduce future noise from thread/resource | GitHub inbox |
| **Snooze** | Pause until later (today/tomorrow/next week) | Common inbox UX |
| **Mute** | Silence repo/transformation without revoking access | Slack channel mute |

### Real-Time Feedback

- **Toast/Snackbar**: Non-blocking, brief, single arrival (Material Design)
- **Badge**: Increment unread count, max "99+", high contrast
- **ARIA live region**: Announce "N new notifications" for screen readers
- **Placement**: Above threads footer (`pb-24` space), bottom edge

### Preferences (Per-Tenant)

Located in `/user/profile/settings` → Notifications tab (extends existing settings):

**Per-Organization Controls**:
- Delivery: In-app (always), Email (immediate/daily/weekly), Mobile push (future)
- Signal level: All / Participating (mentions/assigned) / None
- Watch rules: Auto-watch on contribution (default: Off)
- Muting: Specific repos, transformations, elements
- Quiet hours/Working hours: Suppress real-time, shift to digest

**Global Settings**:
- Default signal level for new orgs
- Digest schedule (daily/weekly, sorted by org)
- Saved filters
- Keyboard shortcuts

### Data Model

```sql
-- Immutable activity log (feed foundation)
CREATE TABLE activity_feed (
  id UUID PRIMARY KEY,
  event_id UUID NOT NULL REFERENCES events(id),
  timestamp TIMESTAMPTZ NOT NULL,
  org_id UUID NOT NULL,              -- Multi-tenant: owning organization
  actor_id UUID NOT NULL,
  activity_type VARCHAR(50) NOT NULL,
  resource_type VARCHAR(50) NOT NULL,
  resource_id UUID NOT NULL,
  scope_type VARCHAR(20) NOT NULL, -- 'org', 'repo', 'transformation'
  scope_id UUID NOT NULL,
  metadata JSONB DEFAULT '{}'
);

-- User-specific notification state
CREATE TABLE notifications (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  org_id UUID NOT NULL,              -- Multi-tenant: source organization
  activity_id UUID REFERENCES activity_feed(id),
  read BOOLEAN DEFAULT FALSE,
  read_at TIMESTAMPTZ,
  saved BOOLEAN DEFAULT FALSE,       -- "Save for later" state
  snoozed_until TIMESTAMPTZ,         -- Snooze functionality
  priority VARCHAR(10) CHECK (priority IN ('low', 'medium', 'high')),
  reason VARCHAR(50),                -- 'mention', 'assigned', 'watching', etc.
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ
);

-- User preferences (per-org)
CREATE TABLE notification_preferences (
  user_id UUID NOT NULL REFERENCES users(id),
  org_id UUID NOT NULL,              -- Per-tenant preferences
  email_enabled BOOLEAN DEFAULT TRUE,
  email_frequency VARCHAR(20) DEFAULT 'immediate', -- 'immediate', 'daily', 'weekly'
  in_app_enabled BOOLEAN DEFAULT TRUE,
  signal_level VARCHAR(20) DEFAULT 'all', -- 'all', 'participating', 'none'
  auto_watch_on_contribute BOOLEAN DEFAULT FALSE,
  subscribed_events JSONB DEFAULT '[]',
  muted_resources JSONB DEFAULT '[]', -- {type, id} pairs
  quiet_hours JSONB DEFAULT '{}',     -- {start, end, timezone}
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (user_id, org_id)
);

-- Global user preferences
CREATE TABLE notification_preferences_global (
  user_id UUID PRIMARY KEY REFERENCES users(id),
  default_signal_level VARCHAR(20) DEFAULT 'participating',
  digest_schedule VARCHAR(20) DEFAULT 'daily',
  saved_filters JSONB DEFAULT '[]',
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_activity_org_scope ON activity_feed(org_id, scope_id, timestamp DESC);
CREATE INDEX idx_activity_scope ON activity_feed(scope_id, timestamp DESC);
CREATE INDEX idx_notifications_user_org_unread ON notifications(user_id, org_id, read) WHERE read = FALSE;
CREATE INDEX idx_notifications_user_unread ON notifications(user_id, read) WHERE read = FALSE;
CREATE INDEX idx_notifications_created ON notifications(created_at DESC);
CREATE INDEX idx_notifications_saved ON notifications(user_id, saved) WHERE saved = TRUE;
```

---

## Recipient Resolution Strategy

### Resolution Hierarchy

1. **Direct Recipients** (Highest Priority)
   - @mentions in content
   - Assigned users
   - Owners

2. **Interest-Based Recipients**
   - Watchers/Followers
   - Recent contributors
   - Participants in conversations

3. **Role-Based Recipients**
   - Organization roles (e.g., all architects)
   - Workspace/team membership
   - Permission-based (e.g., graph admins)

4. **Rule-Based Recipients** (Configurable)
   - Custom notification rules
   - Automation/integration triggers

### Resolution Timing

- **Eager Resolution** (Resolve at event time)
  - Store resolved user IDs in notification entity
  - Fast reads, simple queries
  - Doesn't reflect permission changes

- **Lazy Resolution** (Resolve at read time)
  - Store resolution criteria, resolve when queried
  - Always respects current permissions
  - Slower reads, more complex queries

**Recommendation:** Hybrid approach
- Eager for direct recipients (mentions, assignments)
- Lazy for role/permission-based recipients
- Re-evaluate permissions when displaying notifications

---

## Event-to-Activity-to-Notification Flow

```
1. Domain Event Published → Event Store
   ↓
2. Activity Processor subscribes
   ↓
3. Transform event → Activity entity
   ↓
4. Store in activity_feed table (unconditional)
   ↓
5. Notification Rule Engine evaluates
   ↓
6. Resolve recipients (conditional)
   ↓
7. Apply preference filters
   ↓
8. Create notification records (per recipient)
   ↓
9. Route to delivery channels
```

**Key Insight:** Activities are created **unconditionally**, notifications are created **conditionally** based on rules.

---

## Scaling Patterns

### Fan-Out Strategies

**Small Recipient Counts (<100):**
- Fan-out on write (eager)
- Create notification records immediately
- Fast reads, slow writes

**Large Recipient Counts (>100):**
- Fan-out on read (lazy)
- Store criteria, compute on-the-fly
- Slow reads, fast writes

**Hybrid (Recommended):**
- Mentions/direct → Eager fan-out
- Broadcasts → Lazy evaluation

### Performance Paths

**Hot Path (Real-time):**
```
Event → Quick Filter → Notification Entity → WebSocket Push
```
- <100ms total latency
- In-memory caching of preferences
- No queue

**Cold Path (Non-urgent):**
```
Event → Queue → Batch Processing → Multi-Channel Delivery
```
- Seconds to minutes latency
- Allows aggregation/optimization
- Better resource utilization

---

## Implementation Options

### Option 1: Custom Implementation (Recommended)

**Build on event-driven foundation:**

```
services/notification_service/
├── activity_projector/       # Subscribes to events
│   ├── event_handlers.py
│   └── activity_mapper.py
├── notification_engine/       # Rule evaluation
│   ├── recipient_resolver.py
│   └── preference_filter.py
├── channels/                  # Delivery channels
│   ├── websocket.py
│   ├── email.py
│   └── webhook.py
└── storage/
    ├── activity_store.py
    └── notification_store.py
```

**Protobuf contracts:**
```protobuf
// packages/ameide_core_proto/src/proto/ameide_core_proto/notification/v1/
service NotificationService {
  rpc SubscribeToNotifications(SubscribeRequest) returns (stream Notification);
  rpc GetNotifications(GetNotificationsRequest) returns (GetNotificationsResponse);
  rpc MarkAsRead(MarkAsReadRequest) returns (MarkAsReadResponse);
  rpc UpdatePreferences(UpdatePreferencesRequest) returns (UpdatePreferencesResponse);
}

service ActivityFeedService {
  rpc GetActivityFeed(GetActivityFeedRequest) returns (GetActivityFeedResponse);
}
```

**Pros:**
- ✅ Full alignment with event sourcing
- ✅ Uses existing Postgres infrastructure
- ✅ Complete control over data model
- ✅ Can leverage existing event store
- ✅ Incremental development

**Cons:**
- ❌ Longer development time
- ❌ Need to build all features from scratch
- ❌ Need to implement provider integrations

---

### Option 2: Novu Integration

**Deploy Novu as external service:**

Novu is an open-source notification infrastructure platform with:
- Multi-channel support (Email, SMS, Push, In-App, Chat)
- 50+ provider integrations
- Embeddable React components
- Workflow engine

**Architecture Analysis:**

Services:
- `api` - NestJS REST API (Node 20)
- `worker` - Background job processor (Bull queues)
- `ws` - WebSocket service (Socket.io)
- `dashboard` - React admin UI

Dependencies:
- MongoDB 8.0+ (required, cannot use Postgres)
- Redis (cache + queues)

**Kubernetes Deployment Status:**
- ❌ **No official Kubernetes operator**
- ❌ **No maintained Helm charts** (removed from main repo July 2024, commit 5b5a630468)
- ⚠️ Community helm charts exist but unmaintained
- ✅ Docker Compose files available
- ✅ Pre-built container images on ghcr.io

**Pros:**
- ✅ Complete solution out of the box
- ✅ 50+ provider integrations
- ✅ Embeddable UI components
- ✅ Production-proven

**Cons:**
- ❌ MongoDB dependency (separate DB)
- ❌ No official K8s operator/Helm charts
- ❌ No event sourcing alignment
- ❌ Multi-tenancy model may not match ours
- ❌ Additional infrastructure overhead
- ❌ Self-hosted is secondary to SaaS

**Integration Approach (if chosen):**
1. Deploy Novu services via custom Helm chart
2. Create notification projection service
3. Map domain events → Novu triggers
4. **Layer our UX on top**:
   - Wrap Novu bell/popover in HeaderClient actions
   - Pass org_id as tenant identifier in payloads
   - Render org chips via custom item renderer
   - Build Global + Context Inbox pages around Novu APIs
   - Deep-link to our routes (not Novu defaults)
   - Keep Activity Feed projection separate (our data model)
5. Embed @novu/react components with custom styling

---

### Option 3: Hybrid Approach

**Phase-based migration:**

1. **Phase 1:** Deploy Novu for quick MVP (4-6 weeks)
   - Get notifications working immediately
   - Use Novu's provider system
   - Embed Inbox component

2. **Phase 2:** Build custom projection layer (4-6 weeks)
   - Create event-to-Novu bridge
   - Map domain events to workflows
   - Maintain our event sourcing benefits

3. **Phase 3:** Selective replacement (8-12 weeks)
   - Replace Novu storage with custom
   - Keep provider integrations
   - Migrate to custom workflows engine

4. **Phase 4:** Full custom (optional)
   - Complete ownership
   - Remove Novu dependency

---

## Recommendation

**Build Custom Implementation** for the following reasons:

1. **Architecture Alignment**
   - Notifications are natural projections of our event store
   - Maintains consistency with the event-sourced pattern
   - Uses existing Postgres infrastructure

2. **No K8s Operator Blocker**
   - Novu removed Helm charts from main repo
   - No official Kubernetes operator
   - Would require significant custom K8s work anyway

3. **Incremental Development**
   - Can start with basic in-app notifications
   - Add email channel when needed
   - Scale complexity with requirements

4. **Full Control**
   - Complete ownership of data model
   - Can optimize for our use cases
   - No MongoDB dependency

5. **Learning Opportunity**
   - Understand notification patterns deeply
   - Build expertise in the domain
   - Reference Novu's architecture as needed

**Use Novu as Reference:**
- Study their provider abstraction patterns
- Learn from their workflows engine design
- Reference their API contracts
- Consider their channel implementations

---

## Implementation Phases

### Phase 1: Foundation (Weeks 1-3)

**Deliverables:**
- [ ] Activity feed projection service
- [ ] Event subscription to relevant events
- [ ] Activity storage (Postgres table)
- [ ] Basic activity feed API
- [ ] Protobuf contracts

**Events to Project:**
- ElementCreated, ElementUpdated, ElementDeleted
- CommentAdded, CommentUpdated
- RelationshipCreated
- RepositoryCreated, RepositoryUpdated
- InitiativeCreated, InitiativeUpdated

**Database Migrations:**
- V100__create_activity_feed.sql
- V101__create_notifications.sql
- V102__create_notification_preferences.sql

---

### Phase 2: Notification Engine (Weeks 4-6)

**Deliverables:**
- [ ] Recipient resolution engine
- [ ] Notification rule evaluation
- [ ] Preference filtering
- [ ] Notification creation pipeline
- [ ] Notification query API

**Features:**
- Resolve @mentions from content
- Identify watchers/followers
- Apply user preferences
- Store notification records
- Query notifications by user

---

### Phase 3: Real-Time Delivery (Weeks 7-9)

**Deliverables:**
- [ ] WebSocket/SSE service
- [ ] Real-time notification push
- [ ] React Inbox components (Global + Context)
- [ ] Notification bell with popover
- [ ] Unread count badge
- [ ] Mark as read/save/snooze/mute actions
- [ ] Notification preferences UI (per-org)

**UI Components:**
```typescript
// services/www_ameide_platform/features/notifications/
├── components/
│   ├── NotificationBell.tsx         # Header bell with badge
│   ├── NotificationPopover.tsx      # Quick preview (10 recent)
│   ├── NotificationInbox.tsx        # Full Global Inbox page
│   ├── NotificationContextInbox.tsx # Context-scoped inbox
│   ├── NotificationList.tsx         # List container with filters
│   ├── NotificationItem.tsx         # Individual notification row
│   ├── NotificationOrgChip.tsx      # Org attribution badge
│   ├── NotificationActions.tsx      # Done/Save/Snooze/Mute
│   ├── NotificationFilters.tsx      # Org/Type/Scope/Time filters
│   ├── NotificationEmpty.tsx        # Empty state ("all caught up")
│   └── NotificationPreferences.tsx  # Settings tab (per-org)
├── hooks/
│   ├── useNotifications.ts          # Query notifications
│   ├── useNotificationStream.ts     # WebSocket real-time
│   ├── useNotificationActions.ts    # Triage actions
│   └── useNotificationPreferences.ts
└── api/
    ├── notifications.ts             # gRPC client wrapper
    └── activity-feed.ts             # Activity feed queries
```

**Global Inbox Layout** (`/inbox`):
```
┌────────────────────────────────────────────────────────────────┐
│ HEADER (sticky)                                                 │
│ [Search] [All] [Mentions] [Assigned] [Watching] [Saved]       │
│ Filters: [Org chips...] [Type ▼] [Scope ▼] [Time ▼]          │
└────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────┬───────────────────────────┐
│ NOTIFICATION LIST                  │ DETAIL DRAWER (right)     │
│ ☐ [Acme] Alice mentioned you in   │ ┌───────────────────────┐ │
│   "Order Service"                  │ │ Alice mentioned you   │ │
│   2h ago [Mention]                 │ │ in Order Service      │ │
│   [Done] [Save] [Unsubscribe]      │ │                       │ │
│                                    │ │ "Hey @john, see this  │ │
│ ☐ [Acme] Status changed: Approved  │ │ diagram for..."       │ │
│   1d ago [Watching]                │ │                       │ │
│   [Done] [Save] [Mute repo]        │ │ [Open in context →]   │ │
│                                    │ └───────────────────────┘ │
│ ☐ [VendorCo] New comment on PR     │                           │
│   3d ago [Assigned]                │                           │
│   [Done] [Save] [Snooze]           │                           │
└────────────────────────────────────┴───────────────────────────┘
```

**Keyboard Shortcuts:**
- `j/k` - Navigate up/down
- `Enter` - Open in context
- `e` - Mark as done
- `s` - Save for later
- `u` - Unsubscribe/Unwatch
- `/` - Focus search
- `?` - Show keyboard help

---

### Phase 4: Email Channel (Weeks 10-12)

**Deliverables:**
- [ ] Email notification templates (per-org branded)
- [ ] Digest mode (daily/weekly, per-org or combined)
- [ ] Email delivery service
- [ ] Unsubscribe mechanism (per-org granularity)
- [ ] Email preferences (per-org in settings)

**Multi-Tenant Email Design:**
- **Subject line**: `[OrgName] Alice mentioned you in "Order Service"`
- **Org token**: Always first in subject for filtering
- **Digest format**: Option for single roll-up sorted by org or separate per-org
- **Saved parity**: Digest includes "Saved" section matching web inbox
- **Deep links**: Direct to context with org/repo/element in URL

**Provider Integration:**
- Use existing email infrastructure (SendGrid/SES)
- Templating engine (Handlebars or React Email)
- Email rendering service with org theming support

---

### Phase 5: Advanced Features (Future)

**Potential Additions:**
- Webhook delivery for integrations
- Mobile push notifications
- Notification grouping/batching
- Advanced filtering rules
- Notification analytics
- A/B testing for notification effectiveness
- Snooze functionality
- Custom notification channels

---

## Technical Specifications

### Service Architecture

```
services/notification_service/
├── Dockerfile
├── pyproject.toml
├── app/
│   ├── main.py
│   ├── activity/
│   │   ├── projector.py         # Event subscription
│   │   ├── mapper.py            # Event → Activity
│   │   └── store.py             # Activity storage
│   ├── notification/
│   │   ├── engine.py            # Rule evaluation
│   │   ├── resolver.py          # Recipient resolution
│   │   ├── creator.py           # Notification creation
│   │   └── store.py             # Notification storage
│   ├── channels/
│   │   ├── base.py              # Channel interface
│   │   ├── websocket.py         # Real-time push
│   │   ├── email.py             # Email delivery
│   │   └── webhook.py           # Webhook delivery
│   └── api/
│       ├── activity_feed.py     # Feed queries
│       └── notifications.py     # Notification queries
└── tests/
```

### Helm Chart

```
infra/kubernetes/charts/platform/notification-service/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── servicemonitor.yaml
```

### Configuration

```yaml
# values.yaml
replicaCount: 2

eventStore:
  subscriptionGroup: "notification-service"
  checkpoint:
    enabled: true
    storage: postgres

channels:
  websocket:
    enabled: true
    port: 8080
  email:
    enabled: true
    provider: sendgrid
    from: "notifications@ameide.com"
  webhook:
    enabled: false

resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

---

## Testing Strategy

### Unit Tests
- Event-to-activity mapping
- Recipient resolution logic
- Preference filtering
- Rule evaluation

### Integration Tests
- Event subscription end-to-end
- Database operations
- API endpoints
- Channel delivery

### E2E Tests
```typescript
// services/www_ameide_platform/features/notifications/__tests__/e2e/
test('user receives notification when mentioned', async ({ page }) => {
  // Create comment with @mention
  await createComment(page, '@john see this element');

  // Verify john receives notification
  await expect(page.locator('[data-testid="notification-badge"]'))
    .toContainText('1');

  // Click notification
  await page.click('[data-testid="notification-bell"]');

  // Verify notification content
  await expect(page.locator('[data-testid="notification-item"]'))
    .toContainText('mentioned you');
});
```

---

## Monitoring & Observability

### Metrics
- `notification_events_processed_total` - Events consumed
- `notification_activities_created_total` - Activities created
- `notification_notifications_created_total` - Notifications created
- `notification_delivery_latency_seconds` - Event to delivery time
- `notification_delivery_success_total` - Successful deliveries
- `notification_delivery_failure_total` - Failed deliveries
- `notification_unread_count` - Per-user unread count

### Alerts
- Event processing lag > 5 minutes
- Notification delivery failure rate > 5%
- WebSocket connection failures
- Database connection issues

---

## Security Considerations

1. **Authorization**
   - Check permissions at notification creation
   - Re-check permissions at notification display
   - Filter activities by user's access rights

2. **Rate Limiting**
   - Prevent notification spam per user
   - Rate limit notification creation
   - Throttle email delivery

3. **Data Privacy**
   - Don't include sensitive data in notifications
   - Respect user preferences
   - Support notification deletion (GDPR)
   - Implement retention policies

4. **XSS Prevention**
   - Sanitize notification content
   - Escape HTML in templates
   - Validate @mention syntax

---

## Success Metrics

### Phase 1 (Activity Feed)
- Activity feed available for all repositories
- <100ms query latency for recent activities
- 100% of domain events mapped

### Phase 2 (Notifications)
- Users receive notifications for @mentions
- <500ms notification creation latency
- User preference system functional

### Phase 3 (Real-Time)
- Real-time notifications via WebSocket
- <1s end-to-end latency (event → user sees)
- Unread count badge accuracy 99%+

### Phase 4 (Email)
- Email notifications for opted-in users
- Digest mode functional
- <5% email bounce rate

### UX Success Metrics (North Star)
- **Time-to-triage**: Median seconds to clear top 10 notifications
- **Noise reduction**: Notifications per active day per user (after 2 weeks)
- **Action follow-through**: Open/deep-link CTR from inbox
- **Preference adoption**: Users with ≥1 custom filter or per-tenant rule
- **Accessibility audits passed**: Badges, keyboard nav, screen reader announcements (WCAG AA)

---

## Open Questions

1. **Notification Retention**
   - How long to keep notifications?
   - Archive or hard delete?
   - Separate retention by priority?

2. **Watcher/Follower Model**
   - How do users "watch" repositories/elements?
   - Auto-watch on contribution?
   - Explicit follow action?

3. **Email Templates**
   - Who designs templates?
   - Branded or plain?
   - Support for i18n?

4. **Mobile Support**
   - Push notifications needed?
   - When to build mobile app?
   - Use existing push services?

5. **Notification Grouping**
   - Group similar notifications?
   - "5 people commented on your element"
   - Time window for grouping?

---

## References

- Novu source code: `/workspace/reference-code/novu`
- Activity feed functional architecture discussion (2025-10-29)
- Notification vs Feed relationships discussion (2025-10-29)

---

## Next Steps

1. **Review & Approval**
   - Architecture review with team
   - Alignment on custom vs Novu decision
   - Prioritization confirmation

2. **Design Approval**
   - UI mockups for Inbox component
   - Email template designs
   - Preference management UI

3. **Technical Spike** (1 week)
   - Prototype event subscription
   - Test activity projection performance
   - Validate Postgres schema

4. **Begin Phase 1 Implementation**

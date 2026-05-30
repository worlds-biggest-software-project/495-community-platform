# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourced architecture with Command Query Responsibility Segregation (CQRS). All state changes are captured as immutable domain events in an append-only event store. Commands validate business rules and emit events. Read-optimized projections (materialized views) are built from the event stream to serve queries. The write side (event store) and read side (projections) use different schemas optimized for their respective workloads.

## Why This Model Suits a Community Platform

A community platform is inherently event-driven: members join, post, react, RSVP, complete lessons, earn points, and churn. These are natural domain events. Event sourcing captures the full history of every interaction, which directly enables several of the platform's core AI features:

- **Churn prediction** needs a complete behavioral timeline, not just current state
- **Engagement scoring** requires replaying activity patterns over time windows
- **AI moderation audit trails** demand immutable records of content lifecycle (created, flagged, reviewed, removed)
- **Automation workflows** trigger on events natively -- no polling or change-data-capture bolted on
- **ActivityPub federation** maps directly to event publishing (Create, Like, Follow are events)

The CQRS split also addresses the read/write asymmetry inherent in communities: reads (feeds, search, leaderboards) vastly outnumber writes (posts, reactions) and benefit from purpose-built denormalized projections.

## Trade-offs

**Strengths:**
- Complete, immutable audit trail of every action in the system
- Natural fit for AI/ML pipelines that consume event streams
- Projections can be rebuilt from scratch if requirements change
- Events map directly to ActivityPub activities and webhook payloads
- Independent scaling of read and write paths
- Temporal queries ("what did this member's profile look like last month?") are trivial

**Weaknesses:**
- Significantly higher architectural complexity than a CRUD relational model
- Eventual consistency between write and read sides requires careful UX handling
- Event schema versioning (upcasting) adds maintenance burden over time
- Larger storage footprint -- events accumulate forever, plus projection storage
- Steeper learning curve for contributors; fewer developers are experienced with ES/CQRS
- Debugging and querying raw event stores is harder than querying relational tables

## Schema Definition

### Event Store (Write Side)

The event store is the single source of truth. All state changes pass through it.

```sql
-- ============================================================
-- EVENT STORE (append-only, the single source of truth)
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    stream_id       UUID NOT NULL,            -- aggregate root ID (user_id, post_id, etc.)
    stream_type     VARCHAR(64) NOT NULL,      -- 'User', 'Post', 'Space', 'Course', etc.
    event_type      VARCHAR(128) NOT NULL,     -- 'UserRegistered', 'PostCreated', etc.
    event_version   INT NOT NULL,              -- version within the stream (optimistic concurrency)
    payload         JSONB NOT NULL,            -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation_id, causation_id, actor_id, ip
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (stream_id, event_version)          -- optimistic concurrency control
);

-- Query events by stream (reconstitute an aggregate)
CREATE INDEX idx_es_stream ON event_store (stream_id, event_version);

-- Query events by type (build projections, replay specific event types)
CREATE INDEX idx_es_type ON event_store (event_type, occurred_at);

-- Global ordering for projections that consume all events
CREATE INDEX idx_es_global ON event_store (occurred_at, event_id);

-- Snapshot store for aggregates with long event histories
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(64) NOT NULL,
    version         INT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (stream_id)
);
```

### Domain Events Catalogue

Below are the core domain events with their payload schemas. Each event is a fact that has already happened.

```jsonc
// ── User Aggregate ──────────────────────────────────────────
{
  "event_type": "UserRegistered",
  "payload": {
    "user_id": "uuid",
    "username": "string",
    "email": "string",
    "display_name": "string",
    "auth_provider": "string|null"   // 'google', 'github', 'email'
  }
}

{
  "event_type": "UserProfileUpdated",
  "payload": {
    "user_id": "uuid",
    "changes": { "bio": "string", "avatar_url": "string", "timezone": "string" }
  }
}

{
  "event_type": "UserSuspended",
  "payload": { "user_id": "uuid", "reason": "string", "until": "timestamp|null", "by": "uuid" }
}

{
  "event_type": "UserEmailVerified",
  "payload": { "user_id": "uuid" }
}

// ── Space Aggregate ─────────────────────────────────────────
{
  "event_type": "SpaceCreated",
  "payload": {
    "space_id": "uuid", "name": "string", "slug": "string",
    "access_mode": "public|private|paid", "created_by": "uuid"
  }
}

{
  "event_type": "SpaceMemberJoined",
  "payload": { "space_id": "uuid", "user_id": "uuid" }
}

{
  "event_type": "SpaceMemberBanned",
  "payload": { "space_id": "uuid", "user_id": "uuid", "reason": "string", "by": "uuid" }
}

{
  "event_type": "ChannelCreated",
  "payload": {
    "channel_id": "uuid", "space_id": "uuid", "name": "string",
    "slug": "string", "channel_type": "discussion|chat|announcements|qa"
  }
}

// ── Post Aggregate ──────────────────────────────────────────
{
  "event_type": "PostCreated",
  "payload": {
    "post_id": "uuid", "channel_id": "uuid", "author_id": "uuid",
    "parent_id": "uuid|null", "title": "string|null", "body": "string"
  }
}

{
  "event_type": "PostEdited",
  "payload": { "post_id": "uuid", "body": "string", "edited_by": "uuid" }
}

{
  "event_type": "PostDeleted",
  "payload": { "post_id": "uuid", "deleted_by": "uuid", "reason": "string|null" }
}

{
  "event_type": "PostReactionAdded",
  "payload": { "post_id": "uuid", "user_id": "uuid", "emoji": "string" }
}

{
  "event_type": "PostReactionRemoved",
  "payload": { "post_id": "uuid", "user_id": "uuid", "emoji": "string" }
}

{
  "event_type": "PostFlaggedByAI",
  "payload": {
    "post_id": "uuid", "ai_score": 0.87, "ai_category": "harassment",
    "model_version": "string"
  }
}

{
  "event_type": "PostModerationResolved",
  "payload": {
    "post_id": "uuid", "decision": "approved|rejected",
    "resolved_by": "uuid", "notes": "string|null"
  }
}

// ── Direct Messaging ────────────────────────────────────────
{
  "event_type": "ConversationStarted",
  "payload": { "conversation_id": "uuid", "participants": ["uuid"], "started_by": "uuid" }
}

{
  "event_type": "DirectMessageSent",
  "payload": { "message_id": "uuid", "conversation_id": "uuid", "sender_id": "uuid", "body": "string" }
}

// ── Event Aggregate ─────────────────────────────────────────
{
  "event_type": "EventScheduled",
  "payload": {
    "event_id": "uuid", "space_id": "uuid|null", "title": "string",
    "starts_at": "timestamp", "ends_at": "timestamp", "location_url": "string|null",
    "max_attendees": "int|null", "created_by": "uuid"
  }
}

{
  "event_type": "EventRSVPSubmitted",
  "payload": { "event_id": "uuid", "user_id": "uuid", "status": "going|maybe|not_going" }
}

// ── Course Aggregate ────────────────────────────────────────
{
  "event_type": "CourseCreated",
  "payload": {
    "course_id": "uuid", "title": "string", "slug": "string",
    "is_free": true, "price_cents": 0, "currency": "USD", "created_by": "uuid"
  }
}

{
  "event_type": "LessonCompleted",
  "payload": { "course_id": "uuid", "lesson_id": "uuid", "user_id": "uuid" }
}

{
  "event_type": "CourseCompleted",
  "payload": { "course_id": "uuid", "user_id": "uuid" }
}

{
  "event_type": "CertificateIssued",
  "payload": { "certificate_id": "uuid", "course_id": "uuid", "user_id": "uuid", "certificate_num": "string" }
}

// ── Membership & Payments ───────────────────────────────────
{
  "event_type": "SubscriptionCreated",
  "payload": {
    "subscription_id": "uuid", "user_id": "uuid", "plan_id": "uuid",
    "stripe_sub_id": "string|null", "interval": "month|year"
  }
}

{
  "event_type": "PaymentSucceeded",
  "payload": {
    "payment_id": "uuid", "user_id": "uuid", "amount_cents": 2999,
    "currency": "USD", "provider_txn_id": "string"
  }
}

{
  "event_type": "SubscriptionCanceled",
  "payload": { "subscription_id": "uuid", "user_id": "uuid", "reason": "string|null" }
}

// ── Gamification ────────────────────────────────────────────
{
  "event_type": "PointsAwarded",
  "payload": { "user_id": "uuid", "points": 10, "reason": "post_created", "reference_id": "uuid|null" }
}

{
  "event_type": "BadgeEarned",
  "payload": { "user_id": "uuid", "badge_id": "uuid" }
}

{
  "event_type": "StreakUpdated",
  "payload": { "user_id": "uuid", "streak_type": "daily_login", "current_count": 7, "is_new_record": false }
}

// ── AI & Engagement ─────────────────────────────────────────
{
  "event_type": "ChurnRiskAssessed",
  "payload": { "user_id": "uuid", "risk_score": 0.72, "factors": ["no_posts_14d", "login_drop"] }
}

{
  "event_type": "ReEngagementTriggered",
  "payload": { "user_id": "uuid", "campaign_type": "email", "template_id": "string" }
}
```

### Command Handlers

Commands validate business rules before emitting events. Example pseudocode:

```typescript
// ── Example Command Handlers ────────────────────────────────

interface CreatePostCommand {
  channel_id: string;
  author_id: string;
  parent_id?: string;
  title?: string;
  body: string;
}

async function handleCreatePost(cmd: CreatePostCommand): Promise<void> {
  // 1. Load aggregates from event store
  const channel = await loadAggregate('Channel', cmd.channel_id);
  const author = await loadAggregate('User', cmd.author_id);

  // 2. Validate business rules
  if (channel.is_archived) throw new Error('Cannot post in archived channel');
  if (author.is_suspended) throw new Error('Suspended users cannot post');
  if (!await hasSpaceAccess(author, channel.space_id)) throw new Error('Access denied');

  // 3. Emit event(s)
  const post_id = uuid();
  await appendEvent({
    stream_id: post_id,
    stream_type: 'Post',
    event_type: 'PostCreated',
    payload: { post_id, channel_id: cmd.channel_id, author_id: cmd.author_id,
               parent_id: cmd.parent_id, title: cmd.title, body: cmd.body }
  });

  // 4. Downstream: AI moderation, gamification, and notifications
  //    are handled by event subscribers (process managers / sagas)
}

async function handleSubmitRSVP(cmd: { event_id: string; user_id: string; status: string }) {
  const event = await loadAggregate('Event', cmd.event_id);
  if (cmd.status === 'going' && event.max_attendees && event.attendee_count >= event.max_attendees) {
    throw new Error('Event is full');
  }
  await appendEvent({
    stream_id: cmd.event_id,
    stream_type: 'Event',
    event_type: 'EventRSVPSubmitted',
    payload: { event_id: cmd.event_id, user_id: cmd.user_id, status: cmd.status }
  });
}
```

### Read Projections (Query Side)

Projections are denormalized read models rebuilt from events. Each projection is an independent PostgreSQL table (or could be Redis, Elasticsearch, etc.).

```sql
-- ============================================================
-- PROJECTION: User Profile (denormalized read model)
-- ============================================================
CREATE TABLE proj_user_profiles (
    user_id         UUID PRIMARY KEY,
    username        VARCHAR(64) NOT NULL,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(128) NOT NULL,
    avatar_url      TEXT,
    bio             TEXT,
    total_points    INT NOT NULL DEFAULT 0,
    current_level   VARCHAR(64),
    badge_count     INT NOT NULL DEFAULT 0,
    post_count      INT NOT NULL DEFAULT 0,
    is_suspended    BOOLEAN NOT NULL DEFAULT FALSE,
    last_active_at  TIMESTAMPTZ,
    member_since    TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- PROJECTION: Channel Feed (optimized for paginated display)
-- ============================================================
CREATE TABLE proj_channel_feed (
    post_id         UUID PRIMARY KEY,
    channel_id      UUID NOT NULL,
    channel_name    VARCHAR(128),
    space_id        UUID NOT NULL,
    author_id       UUID NOT NULL,
    author_name     VARCHAR(128) NOT NULL,
    author_avatar   TEXT,
    parent_id       UUID,
    title           VARCHAR(512),
    body_preview    VARCHAR(500),
    body_html       TEXT,
    reply_count     INT NOT NULL DEFAULT 0,
    reaction_count  INT NOT NULL DEFAULT 0,
    is_pinned       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL,
    last_activity   TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_feed_channel ON proj_channel_feed (channel_id, is_pinned DESC, last_activity DESC);
CREATE INDEX idx_proj_feed_author ON proj_channel_feed (author_id, created_at DESC);

-- ============================================================
-- PROJECTION: Upcoming Events
-- ============================================================
CREATE TABLE proj_upcoming_events (
    event_id        UUID PRIMARY KEY,
    space_id        UUID,
    title           VARCHAR(256) NOT NULL,
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    location_url    TEXT,
    attendee_count  INT NOT NULL DEFAULT 0,
    max_attendees   INT,
    created_by_name VARCHAR(128)
);

CREATE INDEX idx_proj_events_upcoming ON proj_upcoming_events (starts_at) WHERE starts_at > NOW();

-- ============================================================
-- PROJECTION: Leaderboard
-- ============================================================
CREATE TABLE proj_leaderboard (
    user_id         UUID PRIMARY KEY,
    display_name    VARCHAR(128) NOT NULL,
    avatar_url      TEXT,
    total_points    INT NOT NULL DEFAULT 0,
    current_level   VARCHAR(64),
    rank            INT
);

CREATE INDEX idx_proj_leaderboard_rank ON proj_leaderboard (total_points DESC);

-- ============================================================
-- PROJECTION: Member Engagement (for AI churn prediction)
-- ============================================================
CREATE TABLE proj_member_engagement (
    user_id         UUID PRIMARY KEY,
    posts_7d        INT NOT NULL DEFAULT 0,
    posts_30d       INT NOT NULL DEFAULT 0,
    reactions_7d    INT NOT NULL DEFAULT 0,
    logins_7d       INT NOT NULL DEFAULT 0,
    logins_30d      INT NOT NULL DEFAULT 0,
    events_attended INT NOT NULL DEFAULT 0,
    last_post_at    TIMESTAMPTZ,
    last_login_at   TIMESTAMPTZ,
    churn_risk      DECIMAL(5,4),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- PROJECTION: Subscription Status (for access control)
-- ============================================================
CREATE TABLE proj_subscription_status (
    user_id         UUID NOT NULL,
    plan_id         UUID NOT NULL,
    plan_name       VARCHAR(128),
    status          VARCHAR(16) NOT NULL,
    current_period_end TIMESTAMPTZ,
    accessible_space_ids UUID[] NOT NULL DEFAULT '{}',
    PRIMARY KEY (user_id, plan_id)
);

CREATE INDEX idx_proj_sub_user ON proj_subscription_status (user_id);

-- ============================================================
-- PROJECTION: Moderation Dashboard
-- ============================================================
CREATE TABLE proj_moderation_items (
    content_id      UUID PRIMARY KEY,
    content_type    VARCHAR(16) NOT NULL,
    content_preview VARCHAR(500),
    author_id       UUID,
    author_name     VARCHAR(128),
    ai_score        DECIMAL(5,4),
    ai_category     VARCHAR(64),
    status          VARCHAR(16) NOT NULL DEFAULT 'pending',
    flagged_at      TIMESTAMPTZ NOT NULL,
    resolved_by     VARCHAR(128),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_proj_mod_pending ON proj_moderation_items (status, ai_score DESC) WHERE status = 'pending';

-- ============================================================
-- PROJECTION: Course Progress
-- ============================================================
CREATE TABLE proj_course_progress (
    user_id         UUID NOT NULL,
    course_id       UUID NOT NULL,
    course_title    VARCHAR(256),
    total_lessons   INT NOT NULL DEFAULT 0,
    completed_lessons INT NOT NULL DEFAULT 0,
    progress_pct    DECIMAL(5,2) NOT NULL DEFAULT 0,
    status          VARCHAR(16) NOT NULL DEFAULT 'active',
    enrolled_at     TIMESTAMPTZ NOT NULL,
    last_activity   TIMESTAMPTZ,
    certificate_id  UUID,
    PRIMARY KEY (user_id, course_id)
);
```

### Projection Rebuilder

```typescript
// Projections subscribe to event types and update read models.
// If a projection's schema changes, it can be dropped and rebuilt
// by replaying all events from the store.

const projectionHandlers: Record<string, (event: DomainEvent) => Promise<void>> = {
  'PostCreated': async (e) => {
    const author = await queryProjection('proj_user_profiles', e.payload.author_id);
    await upsert('proj_channel_feed', {
      post_id: e.payload.post_id,
      channel_id: e.payload.channel_id,
      author_id: e.payload.author_id,
      author_name: author.display_name,
      author_avatar: author.avatar_url,
      title: e.payload.title,
      body_preview: e.payload.body.substring(0, 500),
      created_at: e.occurred_at,
      last_activity: e.occurred_at,
    });
    await increment('proj_user_profiles', e.payload.author_id, 'post_count');
    await increment('proj_member_engagement', e.payload.author_id, 'posts_7d');
  },

  'PointsAwarded': async (e) => {
    await increment('proj_user_profiles', e.payload.user_id, 'total_points', e.payload.points);
    await recalculateRank('proj_leaderboard', e.payload.user_id);
  },

  'PostFlaggedByAI': async (e) => {
    await upsert('proj_moderation_items', {
      content_id: e.payload.post_id,
      content_type: 'post',
      ai_score: e.payload.ai_score,
      ai_category: e.payload.ai_category,
      status: 'pending',
      flagged_at: e.occurred_at,
    });
  },
};

// Rebuild a projection from scratch
async function rebuildProjection(projectionName: string): Promise<void> {
  await truncateTable(projectionName);
  const relevantEventTypes = getEventTypesForProjection(projectionName);
  const events = await streamEvents({ event_types: relevantEventTypes, order: 'asc' });
  for await (const event of events) {
    await projectionHandlers[event.event_type]?.(event);
  }
}
```

## Scalability Considerations

- **Event store partitioning**: Partition `event_store` by `occurred_at` (monthly ranges) for efficient archival and pruning of old partitions.
- **Snapshots**: Aggregates with long histories (e.g., a space with 100K member join/leave events) use periodic snapshots to avoid replaying thousands of events on every command.
- **Projection databases**: Projections can live in different databases optimized for their access patterns -- PostgreSQL for relational queries, Redis for leaderboards and online-presence, Elasticsearch for full-text search.
- **Event streaming**: Replace direct DB polling with Kafka, NATS, or Redpanda for high-throughput event distribution to projection builders, AI pipelines, webhook dispatchers, and ActivityPub publishers.
- **Event versioning**: Use an upcaster pipeline to transform old event schemas to new versions during replay, avoiding the need to migrate the event store itself.

## Migration Path

- Start with a simpler relational model (suggestion 1) and introduce event sourcing incrementally by adding a `domain_events` table alongside existing CRUD tables (dual-write pattern).
- Gradually shift read paths to projections while keeping CRUD tables as a fallback.
- Once projections are proven, drop the CRUD tables and make the event store authoritative.
- Projections can be swapped to the hybrid JSONB model (suggestion 3) or graph database (suggestion 4) without affecting the event store.

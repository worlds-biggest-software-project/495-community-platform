# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A fully normalized (3NF+) relational schema in PostgreSQL with proper foreign keys, indexes, constraints, and referential integrity. Every entity gets its own table, relationships are expressed via foreign keys and junction tables, and data duplication is minimized.

## Why This Model Suits a Community Platform

Community platforms have well-defined, interconnected entities: users belong to spaces, spaces contain posts, posts have replies, events have RSVPs, courses have modules, and memberships gate access. These relationships map naturally to relational tables with foreign key constraints. A normalized schema guarantees data consistency -- when a user is deleted, cascading rules ensure orphaned posts and memberships are handled predictably. PostgreSQL's mature ACID transactions, row-level security, full-text search, and extension ecosystem (pg_trgm, PostGIS, pgcrypto) make it the safest default for a self-hosted platform that needs to work at a range of scales.

## Trade-offs

**Strengths:**
- Referential integrity enforced at the database level
- Mature tooling, ORM support, migration frameworks (Prisma, Knex, TypeORM, Drizzle)
- Well-understood query optimization with EXPLAIN/ANALYZE
- Battle-tested at scale (Discourse runs this exact approach)
- Easiest model for contributors to understand and extend

**Weaknesses:**
- Deeply nested queries (e.g. "all posts in spaces the user has access to, with reactions and replies") require multiple JOINs
- Schema migrations for adding flexible/custom fields require ALTER TABLE or EAV patterns
- Gamification and activity feeds can become expensive aggregation queries at scale
- Horizontal sharding is non-trivial compared to document or event stores

## Schema Definition

```sql
-- ============================================================
-- EXTENSIONS
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "btree_gin";

-- ============================================================
-- CORE: USERS & AUTHENTICATION
-- ============================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username        VARCHAR(64) NOT NULL UNIQUE,
    email           VARCHAR(320) NOT NULL UNIQUE,
    display_name    VARCHAR(128) NOT NULL,
    avatar_url      TEXT,
    bio             TEXT,
    timezone        VARCHAR(64) DEFAULT 'UTC',
    locale          VARCHAR(16) DEFAULT 'en',
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    is_admin        BOOLEAN NOT NULL DEFAULT FALSE,
    is_suspended    BOOLEAN NOT NULL DEFAULT FALSE,
    suspended_until TIMESTAMPTZ,
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_username_trgm ON users USING gin (username gin_trgm_ops);
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_last_seen ON users (last_seen_at);

CREATE TABLE oauth_connections (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider        VARCHAR(32) NOT NULL,  -- 'google', 'github', 'oidc'
    provider_uid    VARCHAR(256) NOT NULL,
    access_token    TEXT,
    refresh_token   TEXT,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (provider, provider_uid)
);

CREATE INDEX idx_oauth_user ON oauth_connections (user_id);

-- ============================================================
-- COMMUNITY SPACES & CHANNELS
-- ============================================================
CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(128) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    description     TEXT,
    icon_url        TEXT,
    access_mode     VARCHAR(16) NOT NULL DEFAULT 'public'
                    CHECK (access_mode IN ('public', 'private', 'paid')),
    sort_order      INT NOT NULL DEFAULT 0,
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_spaces_slug ON spaces (slug);

CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,
    slug            VARCHAR(128) NOT NULL,
    channel_type    VARCHAR(16) NOT NULL DEFAULT 'discussion'
                    CHECK (channel_type IN ('discussion', 'chat', 'announcements', 'qa')),
    sort_order      INT NOT NULL DEFAULT 0,
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (space_id, slug)
);

-- ============================================================
-- POSTS & DISCUSSIONS
-- ============================================================
CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    channel_id      UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES posts(id) ON DELETE CASCADE,  -- threaded replies
    title           VARCHAR(512),  -- NULL for replies
    body            TEXT NOT NULL,
    body_html       TEXT,  -- pre-rendered HTML
    is_pinned       BOOLEAN NOT NULL DEFAULT FALSE,
    is_locked       BOOLEAN NOT NULL DEFAULT FALSE,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    reply_count     INT NOT NULL DEFAULT 0,
    reaction_count  INT NOT NULL DEFAULT 0,
    last_activity   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_channel ON posts (channel_id, created_at DESC);
CREATE INDEX idx_posts_author ON posts (author_id);
CREATE INDEX idx_posts_parent ON posts (parent_id);
CREATE INDEX idx_posts_last_activity ON posts (channel_id, last_activity DESC);

-- Full-text search on posts
CREATE INDEX idx_posts_fts ON posts USING gin (
    to_tsvector('english', COALESCE(title, '') || ' ' || body)
);

CREATE TABLE post_reactions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    emoji           VARCHAR(32) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (post_id, user_id, emoji)
);

CREATE INDEX idx_reactions_post ON post_reactions (post_id);

-- ============================================================
-- DIRECT MESSAGING
-- ============================================================
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    is_group        BOOLEAN NOT NULL DEFAULT FALSE,
    title           VARCHAR(256),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE conversation_participants (
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    last_read_at    TIMESTAMPTZ,
    is_muted        BOOLEAN NOT NULL DEFAULT FALSE,
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE direct_messages (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    sender_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    body            TEXT NOT NULL,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_dm_conversation ON direct_messages (conversation_id, created_at DESC);

-- ============================================================
-- ROLES & PERMISSIONS
-- ============================================================
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(64) NOT NULL UNIQUE,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,  -- admin, moderator, member
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope_type      VARCHAR(16) DEFAULT 'global'
                    CHECK (scope_type IN ('global', 'space', 'channel')),
    scope_id        UUID,  -- references space or channel ID when scoped
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    PRIMARY KEY (user_id, role_id, scope_type, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);

CREATE INDEX idx_user_roles_user ON user_roles (user_id);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(128) NOT NULL UNIQUE,  -- 'posts.create', 'spaces.manage'
    description     TEXT
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- ============================================================
-- SPACE MEMBERSHIP
-- ============================================================
CREATE TABLE space_members (
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status          VARCHAR(16) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('pending', 'active', 'banned')),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (space_id, user_id)
);

CREATE INDEX idx_space_members_user ON space_members (user_id);

-- ============================================================
-- EVENTS
-- ============================================================
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    space_id        UUID REFERENCES spaces(id) ON DELETE SET NULL,
    title           VARCHAR(256) NOT NULL,
    description     TEXT,
    location        TEXT,
    location_url    TEXT,  -- virtual meeting link
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    timezone        VARCHAR(64) NOT NULL DEFAULT 'UTC',
    is_recurring    BOOLEAN NOT NULL DEFAULT FALSE,
    recurrence_rule TEXT,  -- iCalendar RRULE
    max_attendees   INT,
    is_published    BOOLEAN NOT NULL DEFAULT TRUE,
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (ends_at > starts_at)
);

CREATE INDEX idx_events_starts ON events (starts_at);
CREATE INDEX idx_events_space ON events (space_id);

CREATE TABLE event_rsvps (
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status          VARCHAR(16) NOT NULL DEFAULT 'going'
                    CHECK (status IN ('going', 'maybe', 'not_going')),
    responded_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (event_id, user_id)
);

-- ============================================================
-- COURSES & LMS
-- ============================================================
CREATE TABLE courses (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    space_id        UUID REFERENCES spaces(id) ON DELETE SET NULL,
    title           VARCHAR(256) NOT NULL,
    slug            VARCHAR(256) NOT NULL UNIQUE,
    description     TEXT,
    cover_image_url TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    is_free         BOOLEAN NOT NULL DEFAULT TRUE,
    price_cents     INT DEFAULT 0,
    currency        VARCHAR(3) DEFAULT 'USD',
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE course_modules (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    course_id       UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    title           VARCHAR(256) NOT NULL,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE course_lessons (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    module_id       UUID NOT NULL REFERENCES course_modules(id) ON DELETE CASCADE,
    title           VARCHAR(256) NOT NULL,
    content_type    VARCHAR(16) NOT NULL DEFAULT 'text'
                    CHECK (content_type IN ('text', 'video', 'quiz', 'assignment')),
    body            TEXT,
    video_url       TEXT,
    duration_mins   INT,
    sort_order      INT NOT NULL DEFAULT 0,
    is_free_preview BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE course_enrollments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    course_id       UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status          VARCHAR(16) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'completed', 'dropped')),
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    UNIQUE (course_id, user_id)
);

CREATE TABLE lesson_progress (
    enrollment_id   UUID NOT NULL REFERENCES course_enrollments(id) ON DELETE CASCADE,
    lesson_id       UUID NOT NULL REFERENCES course_lessons(id) ON DELETE CASCADE,
    status          VARCHAR(16) NOT NULL DEFAULT 'not_started'
                    CHECK (status IN ('not_started', 'in_progress', 'completed')),
    completed_at    TIMESTAMPTZ,
    PRIMARY KEY (enrollment_id, lesson_id)
);

CREATE TABLE certificates (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    enrollment_id   UUID NOT NULL REFERENCES course_enrollments(id) ON DELETE CASCADE,
    certificate_num VARCHAR(64) NOT NULL UNIQUE,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    template_id     VARCHAR(64)
);

-- ============================================================
-- MEMBERSHIP & MONETISATION
-- ============================================================
CREATE TABLE membership_plans (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(128) NOT NULL,
    description     TEXT,
    price_cents     INT NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    interval        VARCHAR(16) NOT NULL DEFAULT 'month'
                    CHECK (interval IN ('month', 'year', 'one_time')),
    stripe_price_id VARCHAR(256),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan_id         UUID NOT NULL REFERENCES membership_plans(id) ON DELETE RESTRICT,
    stripe_sub_id   VARCHAR(256),
    status          VARCHAR(16) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('trialing', 'active', 'past_due', 'canceled', 'expired')),
    current_period_start TIMESTAMPTZ,
    current_period_end   TIMESTAMPTZ,
    canceled_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_user ON subscriptions (user_id);
CREATE INDEX idx_subscriptions_status ON subscriptions (status);
CREATE INDEX idx_subscriptions_stripe ON subscriptions (stripe_sub_id);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    subscription_id UUID REFERENCES subscriptions(id) ON DELETE SET NULL,
    amount_cents    INT NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    payment_method  VARCHAR(32) NOT NULL DEFAULT 'stripe',
    provider_txn_id VARCHAR(256),
    status          VARCHAR(16) NOT NULL DEFAULT 'succeeded'
                    CHECK (status IN ('pending', 'succeeded', 'failed', 'refunded')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_user ON payments (user_id);

-- ============================================================
-- PLAN-SPACE GATING (which plans unlock which spaces)
-- ============================================================
CREATE TABLE plan_space_access (
    plan_id         UUID NOT NULL REFERENCES membership_plans(id) ON DELETE CASCADE,
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    PRIMARY KEY (plan_id, space_id)
);

-- ============================================================
-- GAMIFICATION
-- ============================================================
CREATE TABLE badges (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(128) NOT NULL UNIQUE,
    description     TEXT,
    icon_url        TEXT,
    criteria_type   VARCHAR(32) NOT NULL,  -- 'manual', 'post_count', 'streak', 'course_complete'
    criteria_value  INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_badges (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    badge_id        UUID NOT NULL REFERENCES badges(id) ON DELETE CASCADE,
    awarded_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    awarded_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    PRIMARY KEY (user_id, badge_id)
);

CREATE TABLE user_points (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    points          INT NOT NULL,
    reason          VARCHAR(64) NOT NULL,  -- 'post_created', 'reply', 'reaction_received'
    reference_type  VARCHAR(32),
    reference_id    UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_user_points_user ON user_points (user_id);

CREATE TABLE levels (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(64) NOT NULL,
    min_points      INT NOT NULL UNIQUE,
    icon_url        TEXT
);

CREATE TABLE streaks (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    streak_type     VARCHAR(32) NOT NULL,  -- 'daily_login', 'daily_post'
    current_count   INT NOT NULL DEFAULT 0,
    longest_count   INT NOT NULL DEFAULT 0,
    last_recorded   DATE NOT NULL DEFAULT CURRENT_DATE,
    PRIMARY KEY (user_id, streak_type)
);

CREATE TABLE challenges (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title           VARCHAR(256) NOT NULL,
    description     TEXT,
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    reward_points   INT NOT NULL DEFAULT 0,
    reward_badge_id UUID REFERENCES badges(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE challenge_participants (
    challenge_id    UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    progress        INT NOT NULL DEFAULT 0,
    completed_at    TIMESTAMPTZ,
    PRIMARY KEY (challenge_id, user_id)
);

-- ============================================================
-- AI & MODERATION
-- ============================================================
CREATE TABLE moderation_queue (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    content_type    VARCHAR(16) NOT NULL CHECK (content_type IN ('post', 'message', 'profile')),
    content_id      UUID NOT NULL,
    reported_by     UUID REFERENCES users(id) ON DELETE SET NULL,
    reason          VARCHAR(64) NOT NULL,
    ai_score        DECIMAL(5,4),  -- 0.0000 to 1.0000 toxicity/violation score
    ai_category     VARCHAR(64),   -- 'spam', 'harassment', 'misinformation'
    status          VARCHAR(16) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'escalated')),
    resolved_by     UUID REFERENCES users(id) ON DELETE SET NULL,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_moderation_status ON moderation_queue (status, created_at);

CREATE TABLE ai_recommendations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    rec_type        VARCHAR(32) NOT NULL,  -- 'post', 'event', 'member', 'course'
    target_id       UUID NOT NULL,
    score           DECIMAL(5,4) NOT NULL,
    reason          TEXT,
    is_dismissed    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_rec_user ON ai_recommendations (user_id, rec_type, score DESC);

CREATE TABLE churn_predictions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    risk_score      DECIMAL(5,4) NOT NULL,  -- 0 = safe, 1 = high churn risk
    factors         TEXT[],  -- array of contributing factors
    predicted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_churn_user ON churn_predictions (user_id, predicted_at DESC);

-- ============================================================
-- AUTOMATION WORKFLOWS
-- ============================================================
CREATE TABLE automations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(256) NOT NULL,
    trigger_event   VARCHAR(64) NOT NULL,  -- 'member.joined', 'course.completed', 'payment.succeeded'
    conditions      TEXT,  -- JSON expression for filtering
    actions         TEXT NOT NULL,  -- JSON array of actions
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- NOTIFICATIONS
-- ============================================================
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type            VARCHAR(64) NOT NULL,  -- 'mention', 'reply', 'event_reminder'
    title           VARCHAR(256) NOT NULL,
    body            TEXT,
    target_type     VARCHAR(32),
    target_id       UUID,
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications (user_id, is_read, created_at DESC);

-- ============================================================
-- MEDIA & ATTACHMENTS
-- ============================================================
CREATE TABLE media (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    uploaded_by     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    filename        VARCHAR(512) NOT NULL,
    mime_type       VARCHAR(128) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,  -- S3/R2 object key
    alt_text        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE post_media (
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    media_id        UUID NOT NULL REFERENCES media(id) ON DELETE CASCADE,
    sort_order      INT NOT NULL DEFAULT 0,
    PRIMARY KEY (post_id, media_id)
);

-- ============================================================
-- ACTIVITYPUB FEDERATION
-- ============================================================
CREATE TABLE ap_actors (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,  -- NULL for remote actors
    actor_uri       TEXT NOT NULL UNIQUE,
    inbox_url       TEXT NOT NULL,
    outbox_url      TEXT,
    public_key      TEXT,
    is_local        BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ap_activities (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    actor_id        UUID NOT NULL REFERENCES ap_actors(id) ON DELETE CASCADE,
    activity_type   VARCHAR(32) NOT NULL,  -- 'Create', 'Like', 'Follow', 'Announce'
    object_uri      TEXT NOT NULL,
    payload         TEXT NOT NULL,  -- raw JSON-LD
    is_local        BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ap_activities_actor ON ap_activities (actor_id, created_at DESC);

-- ============================================================
-- MATERIALIZED VIEW: LEADERBOARD
-- ============================================================
CREATE MATERIALIZED VIEW leaderboard AS
SELECT
    u.id AS user_id,
    u.display_name,
    u.avatar_url,
    COALESCE(SUM(up.points), 0) AS total_points,
    (SELECT l.name FROM levels l WHERE l.min_points <= COALESCE(SUM(up.points), 0) ORDER BY l.min_points DESC LIMIT 1) AS current_level
FROM users u
LEFT JOIN user_points up ON u.id = up.user_id
GROUP BY u.id, u.display_name, u.avatar_url;

CREATE UNIQUE INDEX idx_leaderboard_user ON leaderboard (user_id);
CREATE INDEX idx_leaderboard_points ON leaderboard (total_points DESC);

-- Refresh periodically via pg_cron or application-level scheduler
-- REFRESH MATERIALIZED VIEW CONCURRENTLY leaderboard;
```

## Scalability Considerations

- **Read replicas**: PostgreSQL streaming replication handles read-heavy workloads (feed queries, search). Route writes to primary, reads to replicas.
- **Partitioning**: The `posts`, `notifications`, `user_points`, and `payments` tables should be partitioned by `created_at` once they exceed ~100M rows (PARTITION BY RANGE).
- **Materialized views**: The leaderboard, user activity summaries, and trending posts should use materialized views refreshed on a schedule (pg_cron every 5-15 minutes).
- **Connection pooling**: PgBouncer or Supavisor in front of PostgreSQL for high-concurrency self-hosted deployments.
- **Full-text search**: The built-in tsvector index handles moderate search loads. For large communities (1M+ posts), consider offloading to a dedicated search engine (Meilisearch, Typesense) and keeping PostgreSQL as the source of truth.

## Migration Path

This schema can evolve toward suggestions 2-4 without a full rewrite:
- Add JSONB columns to specific tables to adopt the hybrid approach (suggestion 3)
- Introduce an `events_log` append-only table alongside the existing schema to begin event sourcing (suggestion 2)
- Export relationship data to a graph database for recommendation queries while keeping PostgreSQL as the primary store (suggestion 4)

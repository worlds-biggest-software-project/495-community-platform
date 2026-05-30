# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A PostgreSQL schema that keeps core entities (users, spaces, posts) in normalized relational columns for query performance and referential integrity, while using JSONB columns for semi-structured, variable, or nested data -- custom profile fields, automation rule definitions, notification preferences, AI metadata, gamification criteria, and ActivityPub payloads. This approach captures the best of both worlds: structured data stays relational, flexible data stays schemaless.

## Why This Model Suits a Community Platform

Community platforms exhibit a tension between rigid structure (users, subscriptions, enrollments have well-defined fields) and fluid, operator-customizable data:

- **Custom profile fields**: Different communities need different member attributes (company, title, dietary preference, instrument played). JSONB avoids the EAV anti-pattern.
- **Automation rules**: Workflow conditions and actions are arbitrarily complex nested structures best stored as JSON.
- **AI metadata**: Model scores, factor lists, recommendation reasons, and moderation context vary across AI model versions and should not lock the schema.
- **Gamification criteria**: Badge unlock rules, challenge definitions, and point multipliers are configuration data that operators customize without migrations.
- **ActivityPub payloads**: Incoming federated objects are JSON-LD by specification and should be stored as-is.
- **Event/course metadata**: Different event types (webinar, workshop, meetup) and lesson types (video, quiz, assignment) have different metadata shapes.

PostgreSQL's JSONB type provides binary storage, GIN indexing, partial indexing on JSON paths, and containment/existence operators -- enabling efficient queries on dynamic data without sacrificing the relational foundation.

## Trade-offs

**Strengths:**
- Single database engine (PostgreSQL) -- no additional infrastructure
- Schema flexibility without migrations for operator-customizable fields
- GIN indexes on JSONB enable efficient queries on dynamic attributes
- Relational constraints protect core data integrity while JSONB handles the long tail
- Natural fit for multi-tenant customization (each community operator can add custom fields)
- PostgreSQL JSONB is mature, well-documented, and widely supported by ORMs

**Weaknesses:**
- JSONB columns bypass schema validation -- application-level validation (JSON Schema, Zod) is required
- JOINs on JSONB fields are possible but less performant than on indexed relational columns
- JSONB updates replace the entire document (no partial in-place updates before PG 17)
- Developers must decide which fields are relational vs. JSONB -- poor decisions lead to inconsistency
- Reporting and analytics tools may not understand JSONB columns natively
- Risk of "JSONB sprawl" where too much data ends up untyped and unvalidated

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

    -- JSONB: operator-defined custom profile fields
    -- e.g. {"company": "Acme", "role": "Engineer", "interests": ["AI", "DevOps"]}
    custom_fields   JSONB NOT NULL DEFAULT '{}',

    -- JSONB: notification and privacy preferences
    -- e.g. {"email_digest": "weekly", "dm_enabled": true, "show_activity": true}
    preferences     JSONB NOT NULL DEFAULT '{}',

    -- JSONB: OAuth/SSO provider details
    -- e.g. [{"provider": "google", "uid": "123", "email": "..."}]
    auth_providers  JSONB NOT NULL DEFAULT '[]',

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

-- GIN index on custom_fields for queries like:
-- WHERE custom_fields @> '{"company": "Acme"}'
CREATE INDEX idx_users_custom_fields ON users USING gin (custom_fields);

-- Partial GIN index on preferences for common queries
CREATE INDEX idx_users_preferences ON users USING gin (preferences);

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

    -- JSONB: space-level settings and branding
    -- e.g. {"color": "#3B82F6", "welcome_message": "...", "rules": [...],
    --       "required_fields": ["company"], "seo": {"title": "...", "description": "..."}}
    settings        JSONB NOT NULL DEFAULT '{}',

    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,
    slug            VARCHAR(128) NOT NULL,
    channel_type    VARCHAR(16) NOT NULL DEFAULT 'discussion'
                    CHECK (channel_type IN ('discussion', 'chat', 'announcements', 'qa')),
    sort_order      INT NOT NULL DEFAULT 0,
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,

    -- JSONB: channel-specific settings
    -- e.g. {"allow_attachments": true, "slow_mode_seconds": 30, "auto_close_days": 90}
    settings        JSONB NOT NULL DEFAULT '{}',

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
    parent_id       UUID REFERENCES posts(id) ON DELETE CASCADE,
    title           VARCHAR(512),
    body            TEXT NOT NULL,
    body_html       TEXT,
    is_pinned       BOOLEAN NOT NULL DEFAULT FALSE,
    is_locked       BOOLEAN NOT NULL DEFAULT FALSE,
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,

    -- JSONB: reactions aggregated for fast read
    -- e.g. {"👍": 12, "❤️": 5, "🎉": 3}
    reaction_counts JSONB NOT NULL DEFAULT '{}',

    -- JSONB: embedded media metadata
    -- e.g. [{"type": "image", "url": "...", "width": 800, "height": 600, "alt": "..."}]
    media           JSONB NOT NULL DEFAULT '[]',

    -- JSONB: AI moderation results (updated by AI pipeline)
    -- e.g. {"toxicity": 0.03, "spam": 0.01, "flagged": false, "model": "v2.1"}
    ai_analysis     JSONB NOT NULL DEFAULT '{}',

    -- JSONB: SEO and OpenGraph metadata
    -- e.g. {"og_title": "...", "og_image": "...", "canonical_url": "..."}
    seo_meta        JSONB NOT NULL DEFAULT '{}',

    reply_count     INT NOT NULL DEFAULT 0,
    last_activity   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_channel ON posts (channel_id, created_at DESC);
CREATE INDEX idx_posts_author ON posts (author_id);
CREATE INDEX idx_posts_parent ON posts (parent_id);
CREATE INDEX idx_posts_last_activity ON posts (channel_id, last_activity DESC);
CREATE INDEX idx_posts_fts ON posts USING gin (
    to_tsvector('english', COALESCE(title, '') || ' ' || body)
);

-- GIN index on ai_analysis for filtering flagged content
CREATE INDEX idx_posts_ai ON posts USING gin (ai_analysis);

-- Individual reactions (kept for audit/undo; aggregated counts in post.reaction_counts)
CREATE TABLE post_reactions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    emoji           VARCHAR(32) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (post_id, user_id, emoji)
);

-- ============================================================
-- DIRECT MESSAGING
-- ============================================================
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    is_group        BOOLEAN NOT NULL DEFAULT FALSE,
    title           VARCHAR(256),
    -- JSONB: conversation metadata
    -- e.g. {"participant_ids": ["uuid1", "uuid2"], "last_message_preview": "..."}
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE conversation_participants (
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    last_read_at    TIMESTAMPTZ,
    is_muted        BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB: per-participant settings
    -- e.g. {"pinned": true, "nickname": "Phil"}
    settings        JSONB NOT NULL DEFAULT '{}',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE direct_messages (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    sender_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    body            TEXT NOT NULL,
    -- JSONB: message attachments and embeds
    -- e.g. [{"type": "file", "url": "...", "name": "report.pdf", "size": 102400}]
    attachments     JSONB NOT NULL DEFAULT '[]',
    is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_dm_conversation ON direct_messages (conversation_id, created_at DESC);

-- ============================================================
-- ROLES & PERMISSIONS (relational -- access control stays strict)
-- ============================================================
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(64) NOT NULL UNIQUE,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB: permission set stored as a structured object
    -- e.g. {"posts.create": true, "posts.delete": "own", "spaces.manage": false,
    --       "moderation.review": true, "members.ban": true}
    permissions     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope_type      VARCHAR(16) DEFAULT 'global'
                    CHECK (scope_type IN ('global', 'space', 'channel')),
    scope_id        UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    PRIMARY KEY (user_id, role_id, scope_type, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);

-- ============================================================
-- SPACE MEMBERSHIP
-- ============================================================
CREATE TABLE space_members (
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status          VARCHAR(16) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('pending', 'active', 'banned')),
    -- JSONB: space-specific member data
    -- e.g. {"onboarding_completed": true, "intro_post_id": "uuid", "notes": "VIP member"}
    member_data     JSONB NOT NULL DEFAULT '{}',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (space_id, user_id)
);

-- ============================================================
-- EVENTS
-- ============================================================
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    space_id        UUID REFERENCES spaces(id) ON DELETE SET NULL,
    title           VARCHAR(256) NOT NULL,
    description     TEXT,
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    timezone        VARCHAR(64) NOT NULL DEFAULT 'UTC',
    is_published    BOOLEAN NOT NULL DEFAULT TRUE,
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,

    -- JSONB: flexible event details that vary by type
    -- e.g. {"type": "webinar", "platform": "zoom", "meeting_url": "...",
    --       "recording_url": "...", "max_attendees": 100,
    --       "recurrence": {"rule": "FREQ=WEEKLY;BYDAY=TU", "until": "2026-12-31"},
    --       "location": {"name": "...", "address": "...", "lat": 40.7, "lng": -74.0},
    --       "registration_fields": [{"name": "dietary", "type": "select", "options": [...]}]}
    details         JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (ends_at > starts_at)
);

CREATE INDEX idx_events_starts ON events (starts_at);
CREATE INDEX idx_events_space ON events (space_id);
CREATE INDEX idx_events_details ON events USING gin (details);

CREATE TABLE event_rsvps (
    event_id        UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status          VARCHAR(16) NOT NULL DEFAULT 'going'
                    CHECK (status IN ('going', 'maybe', 'not_going')),
    -- JSONB: registration form responses
    -- e.g. {"dietary": "vegetarian", "t_shirt_size": "L"}
    form_responses  JSONB NOT NULL DEFAULT '{}',
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

    -- JSONB: course structure and settings
    -- e.g. {"modules": [{"id": "uuid", "title": "...", "lessons": [
    --          {"id": "uuid", "title": "...", "type": "video", "duration_mins": 15,
    --           "video_url": "...", "is_free_preview": true}
    --       ]}],
    --       "certificate_template": "professional-v2",
    --       "completion_criteria": {"min_progress": 80, "require_quiz_pass": true},
    --       "drip_schedule": {"type": "fixed", "interval_days": 7}}
    structure       JSONB NOT NULL DEFAULT '{"modules": []}',

    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE course_enrollments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    course_id       UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status          VARCHAR(16) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'completed', 'dropped')),

    -- JSONB: lesson-level progress tracking
    -- e.g. {"lesson_uuid_1": {"status": "completed", "completed_at": "..."},
    --       "lesson_uuid_2": {"status": "in_progress", "progress_pct": 45}}
    lesson_progress JSONB NOT NULL DEFAULT '{}',

    -- JSONB: quiz/assignment results
    -- e.g. {"quiz_uuid_1": {"score": 85, "passed": true, "attempts": 2}}
    assessment_results JSONB NOT NULL DEFAULT '{}',

    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    UNIQUE (course_id, user_id)
);

CREATE TABLE certificates (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    enrollment_id   UUID NOT NULL REFERENCES course_enrollments(id) ON DELETE CASCADE,
    certificate_num VARCHAR(64) NOT NULL UNIQUE,
    -- JSONB: certificate metadata for rendering
    -- e.g. {"template": "professional-v2", "issued_to": "...", "course_title": "...",
    --       "completion_date": "...", "skills": ["React", "TypeScript"]}
    metadata        JSONB NOT NULL DEFAULT '{}',
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
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

    -- JSONB: plan features and access rules
    -- e.g. {"space_ids": ["uuid1", "uuid2"], "course_ids": ["uuid3"],
    --       "features": ["dm", "events", "ai_recommendations"],
    --       "limits": {"monthly_posts": 100, "storage_mb": 500}}
    access_rules    JSONB NOT NULL DEFAULT '{}',

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

    -- JSONB: payment provider details (supports multiple providers)
    -- e.g. {"provider": "stripe", "customer_id": "cus_xxx", "payment_method": "pm_xxx"}
    --   or {"provider": "paypal", "agreement_id": "I-xxx"}
    provider_data   JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_user ON subscriptions (user_id);
CREATE INDEX idx_subscriptions_status ON subscriptions (status);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    subscription_id UUID REFERENCES subscriptions(id) ON DELETE SET NULL,
    amount_cents    INT NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(16) NOT NULL DEFAULT 'succeeded'
                    CHECK (status IN ('pending', 'succeeded', 'failed', 'refunded')),

    -- JSONB: provider-specific transaction data
    -- e.g. {"provider": "stripe", "txn_id": "pi_xxx", "receipt_url": "...", "refund_id": "re_xxx"}
    provider_data   JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_user ON payments (user_id);

-- ============================================================
-- GAMIFICATION
-- ============================================================
CREATE TABLE badges (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(128) NOT NULL UNIQUE,
    description     TEXT,
    icon_url        TEXT,

    -- JSONB: badge unlock criteria (interpreted by the gamification engine)
    -- e.g. {"type": "threshold", "metric": "post_count", "value": 100}
    --   or {"type": "streak", "metric": "daily_login", "days": 30}
    --   or {"type": "course_complete", "course_id": "uuid"}
    --   or {"type": "manual"}
    criteria        JSONB NOT NULL DEFAULT '{"type": "manual"}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_gamification (
    user_id         UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    total_points    INT NOT NULL DEFAULT 0,
    current_level   VARCHAR(64),

    -- JSONB: badge list with timestamps
    -- e.g. [{"badge_id": "uuid", "name": "First Post", "awarded_at": "..."}]
    badges          JSONB NOT NULL DEFAULT '[]',

    -- JSONB: active streaks
    -- e.g. {"daily_login": {"current": 7, "longest": 14, "last": "2026-05-28"},
    --       "daily_post": {"current": 3, "longest": 10, "last": "2026-05-28"}}
    streaks         JSONB NOT NULL DEFAULT '{}',

    -- JSONB: active challenge participation
    -- e.g. [{"challenge_id": "uuid", "progress": 60, "completed_at": null}]
    challenges      JSONB NOT NULL DEFAULT '[]',

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Points ledger (append-only for audit trail)
CREATE TABLE points_ledger (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    points          INT NOT NULL,
    reason          VARCHAR(64) NOT NULL,
    reference_type  VARCHAR(32),
    reference_id    UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_points_user ON points_ledger (user_id, created_at DESC);

CREATE TABLE challenges (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title           VARCHAR(256) NOT NULL,
    description     TEXT,
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    reward_points   INT NOT NULL DEFAULT 0,
    reward_badge_id UUID REFERENCES badges(id) ON DELETE SET NULL,

    -- JSONB: challenge rules and milestones
    -- e.g. {"type": "post_count", "target": 30, "milestones": [10, 20, 30],
    --       "eligible_spaces": ["uuid1"]}
    rules           JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
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
    status          VARCHAR(16) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'escalated')),
    resolved_by     UUID REFERENCES users(id) ON DELETE SET NULL,
    resolved_at     TIMESTAMPTZ,

    -- JSONB: AI analysis details (varies by model version)
    -- e.g. {"model": "moderation-v3", "scores": {"toxicity": 0.87, "spam": 0.12},
    --       "category": "harassment", "explanation": "...", "confidence": 0.92}
    ai_result       JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_moderation_status ON moderation_queue (status, created_at);
CREATE INDEX idx_moderation_ai ON moderation_queue USING gin (ai_result);

-- Member engagement tracking for churn prediction
CREATE TABLE member_engagement (
    user_id         UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,

    -- JSONB: rolling engagement metrics (updated by background job)
    -- e.g. {"posts_7d": 3, "posts_30d": 12, "reactions_7d": 8, "logins_7d": 5,
    --       "logins_30d": 18, "events_attended": 2, "courses_active": 1,
    --       "last_post_at": "...", "last_login_at": "..."}
    metrics         JSONB NOT NULL DEFAULT '{}',

    -- JSONB: churn prediction results
    -- e.g. {"risk_score": 0.72, "factors": ["no_posts_14d", "login_frequency_drop"],
    --       "predicted_at": "...", "model_version": "churn-v2"}
    churn_prediction JSONB NOT NULL DEFAULT '{}',

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- AUTOMATION WORKFLOWS
-- ============================================================
CREATE TABLE automations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(256) NOT NULL,
    trigger_event   VARCHAR(64) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,

    -- JSONB: full workflow definition
    -- e.g. {"conditions": [{"field": "space_id", "op": "eq", "value": "uuid"}],
    --       "actions": [
    --         {"type": "send_email", "template": "welcome", "delay": "1h"},
    --         {"type": "assign_role", "role_id": "uuid"},
    --         {"type": "award_points", "points": 10}
    --       ]}
    workflow        JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- NOTIFICATIONS
-- ============================================================
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type            VARCHAR(64) NOT NULL,
    title           VARCHAR(256) NOT NULL,
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,

    -- JSONB: notification payload (varies by type)
    -- e.g. {"post_id": "uuid", "author": "Phil", "channel": "General",
    --       "preview": "Great discussion about...", "action_url": "/spaces/general/posts/123"}
    data            JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications (user_id, is_read, created_at DESC);

-- ============================================================
-- ACTIVITYPUB FEDERATION
-- ============================================================
CREATE TABLE ap_actors (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    actor_uri       TEXT NOT NULL UNIQUE,
    inbox_url       TEXT NOT NULL,
    outbox_url      TEXT,
    is_local        BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: full ActivityPub actor document (JSON-LD)
    actor_document  JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ap_activities (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    actor_id        UUID NOT NULL REFERENCES ap_actors(id) ON DELETE CASCADE,
    activity_type   VARCHAR(32) NOT NULL,
    object_uri      TEXT NOT NULL,
    is_local        BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: full JSON-LD payload (stored as-is from federation)
    payload         JSONB NOT NULL,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ap_activities_actor ON ap_activities (actor_id, created_at DESC);
CREATE INDEX idx_ap_activities_type ON ap_activities USING gin (payload);

-- ============================================================
-- MEDIA & ATTACHMENTS
-- ============================================================
CREATE TABLE media (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    uploaded_by     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    filename        VARCHAR(512) NOT NULL,
    mime_type       VARCHAR(128) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,

    -- JSONB: image/video metadata
    -- e.g. {"width": 1920, "height": 1080, "duration_secs": 120,
    --       "thumbnails": {"sm": "...", "md": "...", "lg": "..."},
    --       "alt_text": "...", "blurhash": "LGF5]+Yk^6#M@-5c,1J5@[or[Q6."}
    metadata        JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Querying JSONB: Practical Examples

```sql
-- Find all members at company "Acme"
SELECT id, display_name FROM users
WHERE custom_fields @> '{"company": "Acme"}';

-- Find all users with email digest set to "daily"
SELECT id, email FROM users
WHERE preferences @> '{"email_digest": "daily"}';

-- Find posts flagged by AI with toxicity > 0.8
SELECT id, title, ai_analysis->>'toxicity' as toxicity
FROM posts
WHERE (ai_analysis->>'toxicity')::decimal > 0.8;

-- Find events that are webinars
SELECT id, title, details->>'meeting_url' as url
FROM events
WHERE details @> '{"type": "webinar"}';

-- Get a user's streak data
SELECT streaks->'daily_login'->>'current' as login_streak
FROM user_gamification
WHERE user_id = $1;

-- Members at high churn risk
SELECT u.display_name, me.churn_prediction->>'risk_score' as risk
FROM member_engagement me
JOIN users u ON u.id = me.user_id
WHERE (me.churn_prediction->>'risk_score')::decimal > 0.7
ORDER BY (me.churn_prediction->>'risk_score')::decimal DESC;
```

## Scalability Considerations

- **GIN indexes** make containment queries (`@>`) fast but add write overhead. Only index JSONB columns that are frequently queried.
- **TOAST storage**: Large JSONB values are automatically compressed and stored out-of-line via PostgreSQL's TOAST mechanism, but very large documents (>1MB) should be reconsidered.
- **Partial indexes**: Create partial GIN indexes for hot paths (e.g., `WHERE status = 'pending'` on moderation_queue.ai_result) to reduce index size.
- **Application-level validation**: Use JSON Schema or Zod/TypeBox in the application layer to validate JSONB shapes before writes. Consider a `jsonb_valid` CHECK constraint if using pg_jsonschema extension.
- **Read replicas**: Same as the normalized model -- use streaming replication for read scaling.
- **Partitioning**: Partition `notifications`, `points_ledger`, `ap_activities`, and `payments` by `created_at`.

## Migration Path

- This is the easiest model to evolve from suggestion 1: add JSONB columns to existing tables with `DEFAULT '{}'` (instant, no rewrite).
- Existing relational columns remain for high-frequency query paths; new flexible data goes into JSONB.
- Can be combined with event sourcing (suggestion 2) by storing domain events with JSONB payloads.
- JSONB columns storing graph-like data (member connections, recommendations) can be extracted to a graph database (suggestion 4) if query patterns demand it.

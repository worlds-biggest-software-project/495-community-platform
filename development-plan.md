# Community Platform — Phased Development Plan

> Project: 495-community-platform · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased build. The product is an **AI-native, open-source, self-hostable community platform** — an alternative to Circle, Mighty Networks, Skool, and Bettermode — combining discussion spaces, events, courses, paid memberships, gamification, and AI moderation/engagement intelligence, with ActivityPub federation as a differentiator.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **TypeScript** (Node.js 22 LTS) | The product is API- and frontend-heavy with real-time features (chat, presence). A single language across the API, workers, and web frontend lowers contributor friction — a priority for an open-source project. TypeScript gives compile-time safety for the large domain model. |
| Runtime / monorepo | **pnpm workspaces + Turborepo** | Multiple deployable units (API, web, worker, MCP server) share domain types and the data layer. pnpm workspaces give a single source of truth for shared packages; Turborepo caches builds/tests. |
| API framework | **NestJS** | Provides first-class dependency injection, modular structure (one module per bounded context: spaces, posts, events, courses, billing, moderation), guards for RBAC, and built-in OpenAPI 3.1 generation via `@nestjs/swagger`. Modularity directly maps to the phase structure. |
| GraphQL | **Apollo Server via `@nestjs/graphql` (code-first)** | `standards.md` notes the market is split REST vs GraphQL; Bettermode and Circle lead with GraphQL. Code-first GraphQL reuses the same NestJS resolvers/DTOs, so REST (OpenAPI) and GraphQL ship from one source of truth. |
| Database | **PostgreSQL 17** | Chosen per **data-model-suggestion-3 (Hybrid Relational + JSONB)**. Core entities stay relational for integrity; operator-customisable data (custom profile fields, automation rules, AI metadata, gamification criteria, ActivityPub payloads) live in JSONB. Single engine = zero extra ops for self-hosters. PG 17 adds in-place JSONB updates. Mature `pg_trgm`/FTS for search. |
| ORM / migrations | **Drizzle ORM + drizzle-kit** | Type-safe SQL-first schema that mirrors the suggestion-3 DDL closely, generates TypeScript types, and produces deterministic SQL migrations. First-class JSONB support. Lighter and more transparent than Prisma for a SQL-heavy hybrid schema. |
| Append-only audit log | **`domain_events` table (additive)** | Adopts the dual-write idea from data-model-suggestion-2 without full CQRS. An append-only `domain_events` table captures every state change as an immutable fact. This powers AI churn timelines, moderation audit trails, webhook dispatch, and ActivityPub publishing — without the complexity of full event sourcing. |
| Task queue / async | **BullMQ on Redis** | Webhooks, LLM moderation calls, email digests, churn scoring, and ActivityPub delivery are async and retryable. BullMQ gives delayed jobs, retries with backoff, and repeatable (cron-like) jobs for scheduled scoring. |
| Cache / presence / pub-sub | **Redis 7** | Doubles as the BullMQ backend, the WebSocket pub/sub adapter (multi-instance fan-out), presence/typing state, rate-limit buckets, and hot leaderboard reads. |
| Real-time | **WebSocket (RFC 6455) via Socket.IO + Redis adapter; SSE for one-way feeds** | `standards.md` names WebSocket as the baseline for chat/presence and SSE as the lighter option for notifications/feeds. Socket.IO's Redis adapter scales horizontally. |
| Full-text search | **PostgreSQL FTS (`tsvector` + GIN) for MVP; Meilisearch later** | Suggestion-1/3 note built-in FTS handles moderate loads; defer a dedicated engine until communities exceed ~1M posts. |
| LLM integration | **Vercel AI SDK** with pluggable providers (OpenAI, Anthropic, local Ollama) | Moderation, recommendations, onboarding, and churn explanations need a provider-agnostic interface so self-hosters can use a local model or any API. The AI SDK gives a uniform `generateObject`/`generateText` surface with structured output. |
| Payments | **Stripe (Connect) SDK** | De facto standard per `research.md`/`standards.md`. Connect supports the marketplace/revenue-share model. PCI scope stays SAQ-A by never touching card data (Checkout / Payment Element). |
| Frontend | **Next.js 15 (App Router) + React 19, Tailwind CSS, shadcn/ui** | Server components for SEO-friendly public community pages (Schema.org JSON-LD), client components for interactive feeds/chat. shadcn/ui gives accessible (WCAG 2.2 AA target) primitives. |
| Auth | **Auth.js (NextAuth) + custom OAuth2/OIDC provider layer** | Implements OIDC/OAuth 2.0 + PKCE (RFC 7636) for SSO per `standards.md`. The platform is both an OIDC *client* (social login) and an OIDC/OAuth *provider* (for third-party API access and the member API). |
| Containerisation | **Docker + docker-compose** | Self-hosted deployment is a core value prop. compose bundles Postgres, Redis, API, worker, and web for a one-command local/self-host run. |
| Testing | **Vitest** (unit), **Supertest + Testcontainers** (integration), **Playwright** (E2E) | Vitest is fast and TS-native. Testcontainers spins up real Postgres/Redis for integration tests. Playwright drives the web UI and validates accessibility (axe-core) for WCAG checks. |
| Code quality | **ESLint + Prettier + `tsc --noEmit`** | Standard TS toolchain; enforced in CI and the per-phase Definition of Done. |
| Validation | **Zod** | Validates JSONB payloads (which bypass DB schema), API inputs, config, and LLM structured outputs. Shared Zod schemas live in a workspace package and double as the contract for OpenAPI/GraphQL. |
| Email | **Nodemailer + MJML templates; pluggable transport (SMTP/SES/SendGrid)** | Self-hosters configure SMTP; cloud operators plug in a provider. MJML produces accessible, responsive emails for digests and lifecycle messages. |
| Object storage | **S3-compatible (AWS S3 / Cloudflare R2 / MinIO)** | Media uploads via presigned URLs. MinIO in compose for local/self-host; any S3 API in production. |
| Licence | **Apache 2.0** | `features.md` recommends a permissive licence (MIT/Apache 2.0) to maximise adoption versus AGPL-licensed Discourse. Apache 2.0 adds an explicit patent grant. |

### Project Structure

```
community-platform/
├── package.json                  # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── tsconfig.base.json
├── docker-compose.yml            # postgres, redis, minio, api, worker, web
├── Dockerfile                    # multi-stage; targets: api, worker, web
├── .env.example
├── README.md
├── LICENSE                       # Apache 2.0
├── packages/
│   ├── db/                       # Drizzle schema, migrations, client, seed
│   │   ├── src/
│   │   │   ├── schema/           # one file per domain (users.ts, spaces.ts, ...)
│   │   │   ├── client.ts
│   │   │   ├── events.ts         # appendDomainEvent() helper
│   │   │   └── seed.ts
│   │   └── drizzle.config.ts
│   ├── contracts/                # Zod schemas + shared DTO/types (single source of truth)
│   │   └── src/                  # post.ts, event.ts, automation.ts, ai.ts, ...
│   ├── ai/                       # provider-agnostic LLM client + prompt templates
│   │   └── src/
│   │       ├── provider.ts       # AI SDK wrapper, model routing
│   │       ├── moderation.ts
│   │       ├── recommendations.ts
│   │       ├── onboarding.ts
│   │       └── prompts/
│   ├── config/                   # env loading + Zod-validated config schema
│   └── activitypub/              # AP actor/object/activity builders, HTTP signatures
├── apps/
│   ├── api/                      # NestJS REST + GraphQL
│   │   └── src/
│   │       ├── main.ts
│   │       ├── app.module.ts
│   │       ├── common/           # guards, interceptors, RBAC, pagination
│   │       ├── auth/
│   │       ├── users/
│   │       ├── spaces/
│   │       ├── posts/
│   │       ├── messaging/
│   │       ├── events/
│   │       ├── courses/
│   │       ├── billing/
│   │       ├── gamification/
│   │       ├── moderation/
│   │       ├── recommendations/
│   │       ├── automations/
│   │       ├── notifications/
│   │       ├── webhooks/
│   │       ├── federation/       # ActivityPub inbox/outbox endpoints
│   │       └── realtime/         # WebSocket gateway + SSE controllers
│   ├── worker/                   # BullMQ processors (moderation, email, churn, AP delivery)
│   │   └── src/processors/
│   ├── mcp/                      # MCP server exposing community data to AI agents
│   │   └── src/
│   └── web/                      # Next.js frontend
│       └── src/app/
└── tests/
    ├── integration/              # Testcontainers-backed API tests
    └── e2e/                      # Playwright specs + axe accessibility checks
```

The structure is grouped by **bounded context**, not by phase — each phase fills in modules/packages without restructuring.

---

## Phase 1: Foundation — Monorepo, Database, Config, Auth Skeleton

### Purpose
Establish the repository, build tooling, containerised dev environment, the hybrid relational+JSONB schema foundation, the append-only `domain_events` table, configuration loading, and an authentication skeleton. After this phase, a developer can `docker compose up`, hit a health endpoint, register/log in, and run migrations and tests. Everything else builds on this.

### Tasks

#### 1.1 — Monorepo & tooling bootstrap

**What**: Initialise the pnpm/Turborepo monorepo with shared TS config, lint/format, and CI.

**Design**:
- `pnpm-workspace.yaml` includes `packages/*` and `apps/*`.
- `turbo.json` pipelines: `build`, `lint`, `test`, `typecheck` with dependency-aware ordering.
- Root `tsconfig.base.json` with `strict: true`, `noUncheckedIndexedAccess: true`, path aliases (`@cp/db`, `@cp/contracts`, `@cp/ai`, `@cp/config`).
- ESLint flat config + Prettier; `tsc --noEmit` typecheck task.
- GitHub Actions workflow: install → typecheck → lint → test on PRs.
- `docker-compose.yml` services: `postgres:17`, `redis:7`, `minio`, plus placeholders for `api`, `worker`, `web`.
- Multi-stage `Dockerfile` with `api`, `worker`, `web` targets.
- `.env.example` enumerating every env var consumed by `@cp/config`.

**Testing**:
- `Unit: pnpm install resolves with no peer-dep errors` (CI smoke).
- `Integration: docker compose up postgres redis minio → all three report healthy within 60s` (compose healthchecks).
- `Unit: turbo run typecheck on empty packages → exits 0`.

#### 1.2 — Config package with Zod validation

**What**: A `@cp/config` package that loads and validates environment configuration at startup.

**Design**:
```ts
// packages/config/src/schema.ts
export const ConfigSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(4000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  JWT_ACCESS_TTL: z.string().default('15m'),
  JWT_REFRESH_TTL: z.string().default('30d'),
  S3_ENDPOINT: z.string().url(),
  S3_BUCKET: z.string(),
  S3_ACCESS_KEY: z.string(),
  S3_SECRET_KEY: z.string(),
  STRIPE_SECRET_KEY: z.string().optional(),
  STRIPE_WEBHOOK_SECRET: z.string().optional(),
  AI_PROVIDER: z.enum(['openai', 'anthropic', 'ollama', 'disabled']).default('disabled'),
  AI_API_KEY: z.string().optional(),
  AI_MODEL: z.string().default('gpt-4o-mini'),
  PUBLIC_BASE_URL: z.string().url(),       // used for AP actor URIs, OIDC issuer
  SMTP_URL: z.string().optional(),
});
export type Config = z.infer<typeof ConfigSchema>;
export function loadConfig(env = process.env): Config { /* parse + friendly error */ }
```
- On validation failure, throw with the offending field names listed; the API/worker refuse to start.

**Testing**:
- `Unit: valid env → Config object with defaults applied`.
- `Unit: missing DATABASE_URL → throws, message contains "DATABASE_URL"`.
- `Unit: JWT_SECRET too short → throws, message contains "JWT_SECRET"`.
- `Unit: AI_PROVIDER omitted → defaults to 'disabled'`.

#### 1.3 — Core schema (users, auth, roles) + domain_events table

**What**: Drizzle schema for the identity core and the append-only event log, with migrations and a seed.

**Design** (hybrid model, from suggestion-3 + suggestion-1 core; JSONB for flexible fields):
```ts
// packages/db/src/schema/users.ts
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  username: varchar('username', { length: 64 }).notNull().unique(),
  email: varchar('email', { length: 320 }).notNull().unique(),
  displayName: varchar('display_name', { length: 128 }).notNull(),
  passwordHash: text('password_hash'),                 // null for OAuth-only
  avatarUrl: text('avatar_url'),
  bio: text('bio'),
  timezone: varchar('timezone', { length: 64 }).default('UTC'),
  locale: varchar('locale', { length: 16 }).default('en'),
  customFields: jsonb('custom_fields').$type<Record<string, unknown>>().default({}), // operator-defined
  notificationPrefs: jsonb('notification_prefs').$type<NotificationPrefs>().default({}),
  emailVerified: boolean('email_verified').notNull().default(false),
  isAdmin: boolean('is_admin').notNull().default(false),
  isSuspended: boolean('is_suspended').notNull().default(false),
  suspendedUntil: timestamp('suspended_until', { withTimezone: true }),
  lastSeenAt: timestamp('last_seen_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// oauth_connections, refresh_tokens, roles, user_roles (scoped: global|space|channel),
// permissions, role_permissions  — relational, mirroring suggestion-1 DDL.
```
```ts
// packages/db/src/schema/events.ts  (additive append-only log from suggestion-2)
export const domainEvents = pgTable('domain_events', {
  eventId: uuid('event_id').primaryKey().defaultRandom(),
  streamId: uuid('stream_id').notNull(),               // aggregate id (user, post, ...)
  streamType: varchar('stream_type', { length: 64 }).notNull(),
  eventType: varchar('event_type', { length: 128 }).notNull(),
  payload: jsonb('payload').notNull(),
  metadata: jsonb('metadata').notNull().default({}),   // actorId, correlationId, ip
  occurredAt: timestamp('occurred_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  byStream: index('idx_de_stream').on(t.streamId, t.occurredAt),
  byType: index('idx_de_type').on(t.eventType, t.occurredAt),
}));
```
```ts
// packages/db/src/events.ts
export async function appendDomainEvent(tx: DbTx, e: NewDomainEvent): Promise<void>;
// Always called inside the same transaction as the state mutation (dual-write).
```
- Seed: one admin user, default roles (`admin`, `moderator`, `member`), the baseline permission set (`posts.create`, `spaces.manage`, `moderation.review`, ...).

**Testing**:
- `Integration (Testcontainers): run migrations → all tables and indexes exist (query pg_catalog)`.
- `Integration: insert user with duplicate email → unique violation`.
- `Integration: appendDomainEvent within a tx that rolls back → no event row persists`.
- `Unit: NotificationPrefs Zod schema validates default {} and rejects unknown channel keys`.

#### 1.4 — Auth skeleton (register, login, JWT, RBAC guard)

**What**: Email/password registration, login issuing access+refresh JWTs, and a NestJS RBAC guard.

**Design**:
- Endpoints:
  - `POST /v1/auth/register` → `{ username, email, password, displayName }` → 201 `{ user, accessToken, refreshToken }`. Emits `UserRegistered`.
  - `POST /v1/auth/login` → `{ email, password }` → 200 `{ accessToken, refreshToken }`.
  - `POST /v1/auth/refresh` → `{ refreshToken }` → 200 new pair (rotation; old refresh revoked).
  - `GET /v1/me` → current user (requires bearer).
- Passwords hashed with `argon2id`. Access JWT claims: `sub`, `roles`, `exp`. Refresh tokens stored hashed in `refresh_tokens` for rotation/revocation.
- `@Roles('admin')` decorator + `RolesGuard` reading scoped roles. `@CurrentUser()` param decorator.
- Bearer usage follows RFC 6750. OIDC/social login and the OAuth *provider* role are deferred to Phase 7.

**Testing**:
- `Integration: register then login → valid JWT, /v1/me returns the user`.
- `Integration: login with wrong password → 401, no token`.
- `Integration: expired access token → 401; refresh → new pair, old refresh rejected on reuse`.
- `Integration: non-admin hits an @Roles('admin') route → 403`.
- `Unit: argon2 verify true for correct password, false otherwise`.

---

## Phase 2: Spaces, Channels, Membership & RBAC

### Purpose
Build the organisational backbone: spaces (public/private/paid), channels within spaces, space membership, and scoped role-based access control. This is the access-control substrate every content feature depends on. After this phase the platform can model "who can see and do what, where."

### Tasks

#### 2.1 — Spaces & channels CRUD

**What**: Create/read/update/archive spaces and channels with slugs and access modes.

**Design**:
```ts
// schema/spaces.ts — relational core + JSONB settings
spaces: { id, name, slug (unique), description, iconUrl,
  accessMode: 'public'|'private'|'paid', sortOrder, isArchived,
  settings: jsonb (layout, theme, customFields schema), createdBy, createdAt, updatedAt }
channels: { id, spaceId(fk cascade), name, slug, channelType:
  'discussion'|'chat'|'announcements'|'qa', sortOrder, isArchived, unique(spaceId, slug) }
```
- REST: `POST/GET/PATCH /v1/spaces`, `.../{id}/archive`, nested `/v1/spaces/{id}/channels`.
- GraphQL: `spaces`, `space(slug)`, `createSpace`, `updateSpace` resolvers reusing the same service.
- Slugs auto-generated from name, collision-suffixed. Emits `SpaceCreated`, `ChannelCreated`.
- Requires `spaces.manage` permission.

**Testing**:
- `Integration: create space "My Space" → slug "my-space"; second with same name → "my-space-2"`.
- `Integration: create channel in archived space → 409`.
- `Integration: GraphQL space(slug) returns channels ordered by sortOrder`.
- `Integration: member without spaces.manage creates space → 403`.

#### 2.2 — Space membership & access resolution

**What**: Join/leave spaces and a central access-resolution service.

**Design**:
```ts
space_members: { spaceId, userId, status: 'pending'|'active'|'banned',
  joinedAt, PRIMARY KEY(spaceId,userId) }
```
```ts
// apps/api/src/common/access.service.ts
canAccessSpace(userId, spaceId): Promise<AccessDecision>
// public → allow; private → must be active member or have role; paid → defer to billing (Phase 6),
// for now: active member OR admin. Returns { allowed: boolean, reason: string }.
```
- `POST /v1/spaces/{id}/join`, `/leave`. Public → instant active; private → `pending` until approved by moderator. Emits `SpaceMemberJoined`, `SpaceMemberBanned`.

**Testing**:
- `Integration: join public space → status active`.
- `Integration: join private space → status pending; moderator approves → active`.
- `Integration: banned member re-joins → 403`.
- `Unit: canAccessSpace for public space, anonymous user → allowed=true`.

#### 2.3 — Scoped RBAC enforcement

**What**: Extend the RBAC guard to evaluate space/channel-scoped roles.

**Design**:
- `user_roles.scopeType ∈ {global, space, channel}` with `scopeId`. Resolution precedence: global admin > space moderator > member.
- `@RequirePermission('posts.create')` checks effective permissions for the target scope (resolved from the route's `spaceId`/`channelId` param).
- Permissions cached per-request in Redis-backed memo keyed by `userId:scopeId`.

**Testing**:
- `Integration: user with space-scoped moderator role → can moderation.review in that space only, 403 in another`.
- `Integration: global admin → passes every scoped permission check`.
- `Unit: permission resolver merges global + space-scoped grants correctly`.

---

## Phase 3: Posts, Threads, Reactions & Search (Core Value)

### Purpose
Deliver the heart of the product: threaded discussions, replies, reactions, and search. This is the primary day-to-day surface members interact with. It must ship early per the phase-design principle that the core value lands in phases 3–5.

### Tasks

#### 3.1 — Posts & threaded replies

**What**: Create posts and nested replies in channels, with edit/delete and counters.

**Design**:
```ts
posts: { id, channelId(fk cascade), authorId(fk), parentId(self fk, null=root),
  title (null for replies), body, bodyHtml, isPinned, isLocked, isDeleted,
  replyCount, reactionCount, lastActivity, metadata: jsonb (mentions[], attachments[]),
  createdAt, updatedAt }
// indexes: (channelId, createdAt desc), (channelId, lastActivity desc), parentId, FTS gin on title+body
```
- `body` stored as Markdown; `bodyHtml` server-rendered + sanitised (DOMPurify-equivalent server side) to satisfy OWASP A03 (XSS). Allowed tag/attr allowlist.
- `POST /v1/channels/{id}/posts`, `POST /v1/posts/{id}/replies`, `PATCH/DELETE /v1/posts/{id}`.
- Deletion is soft (`isDeleted=true`). Editing root after replies keeps history note in `metadata.editedAt`.
- Counters (`replyCount`, `reactionCount`) updated in the same tx. Emits `PostCreated`, `PostEdited`, `PostDeleted`.
- `@mentions` parsed → stored in `metadata.mentions` → trigger notifications (Phase 8 hook published as event now).

**Testing**:
- `Integration: create post then reply → parent.replyCount == 1, parent.lastActivity updated`.
- `Integration: post body with <script> → bodyHtml strips it (XSS allowlist)`.
- `Integration: delete post → isDeleted true, body redacted in API response, replies preserved`.
- `Integration: edit locked post by non-moderator → 403`.
- `Unit: markdown renderer + sanitiser produces only allowlisted tags`.

#### 3.2 — Reactions

**What**: Emoji reactions on posts, unique per (post, user, emoji).

**Design**:
```ts
post_reactions: { id, postId(fk cascade), userId(fk cascade), emoji, createdAt,
  unique(postId,userId,emoji) }
```
- `PUT /v1/posts/{id}/reactions/{emoji}` (idempotent add), `DELETE /v1/posts/{id}/reactions/{emoji}`.
- Updates `posts.reactionCount`. Emits `PostReactionAdded`/`Removed`. Recipient gamification points hook (Phase 5) via event.

**Testing**:
- `Integration: add same emoji twice → single row, count == 1 (idempotent)`.
- `Integration: remove reaction → count decremented, not below 0`.

#### 3.3 — Full-text search

**What**: Search posts (and later events/courses/members) via PostgreSQL FTS.

**Design**:
- GIN index `to_tsvector('english', coalesce(title,'') || ' ' || body)`.
- `GET /v1/search?q=...&type=posts&spaceId=...&cursor=...` → ranked by `ts_rank`, filtered to spaces the caller can access (joins access resolution).
- Cursor pagination (keyset on `(rank, id)`). Trigram fallback for short/typo queries via `pg_trgm`.

**Testing**:
- `Integration: seed 3 posts, search exact term → matching post ranked first`.
- `Integration: search excludes posts in private spaces the user can't access`.
- `Integration: pagination cursor returns next page with no overlap`.

---

## Phase 4: Real-Time Messaging, Chat & Presence

### Purpose
Add real-time communication: direct messages, chat-type channels, typing indicators, presence, and live post updates. This makes the community feel alive and matches the chat-first expectations set by Heartbeat and Discourse chat. Requires Phases 1–3.

### Tasks

#### 4.1 — Direct messaging

**What**: 1:1 and group conversations with messages and read state.

**Design**:
```ts
conversations: { id, isGroup, title (null for 1:1), createdAt }
conversation_participants: { conversationId, userId, lastReadAt, isMuted, joinedAt, PK(conversationId,userId) }
direct_messages: { id, conversationId(fk cascade), senderId(fk), body, isDeleted, createdAt }
```
- `POST /v1/conversations` (dedupes existing 1:1), `POST /v1/conversations/{id}/messages`, `GET .../messages?cursor=`, `POST .../read`.
- Emits `ConversationStarted`, `DirectMessageSent`.

**Testing**:
- `Integration: start 1:1 twice with same pair → same conversation id returned`.
- `Integration: send message → recipient unread count increments until /read`.

#### 4.2 — WebSocket gateway + Redis adapter

**What**: Socket.IO gateway authenticated by JWT, scaled across instances via the Redis adapter (RFC 6455 transport).

**Design**:
- Namespaces: `/chat` (DMs + chat channels), `/presence`.
- Handshake auth: JWT in `auth.token`; reject with `unauthorized` on failure.
- Rooms: `conv:{conversationId}`, `channel:{channelId}`, `user:{userId}`.
- Events emitted: `message.new`, `message.read`, `typing.start/stop`, `presence.update`.
- Server subscribes to `domain_events` fan-out (via Redis pub/sub published by services) to push `post.created`/`reaction.added` to channel rooms.

**Testing**:
- `Integration: two clients in conv room → sender emits message → both receive message.new`.
- `Integration: connect with invalid JWT → disconnected with unauthorized`.
- `Integration (2 API instances sharing Redis): client on instance A receives event published from instance B`.

#### 4.3 — Presence & notifications stream (SSE)

**What**: Online/typing presence over WebSocket; one-way notification feed over SSE.

**Design**:
- Presence stored in Redis with TTL heartbeat (`presence:{userId} = lastPing`, 30s TTL). `presence.update` broadcast to followers/space members.
- `GET /v1/notifications/stream` → `text/event-stream` (SSE per `standards.md`) pushing new notification rows for the authenticated user.

**Testing**:
- `Integration: client heartbeats → appears online; stops 30s → presence expires, offline broadcast`.
- `Integration: SSE stream receives a notification event within 1s of insert`.

---

## Phase 5: Events, Courses/LMS & Gamification

### Purpose
Round out the "all-in-one" surface that distinguishes the product from forum-only tools: event scheduling with RSVP, a native course/LMS module co-located with the community, certificates, and the gamification system (points, levels, badges, streaks, leaderboards). Skool-grade gamification is a named differentiator.

### Tasks

#### 5.1 — Events & RSVP

**What**: Scheduled events (one-off and recurring) with RSVP and calendar export.

**Design**:
```ts
events: { id, spaceId(fk set null), title, description, location, locationUrl,
  startsAt, endsAt, timezone, isRecurring, recurrenceRule (iCal RRULE),
  maxAttendees, isPublished, metadata: jsonb (eventType, agenda), createdBy,
  CHECK(endsAt>startsAt) }
event_rsvps: { eventId, userId, status: 'going'|'maybe'|'not_going', respondedAt, PK(eventId,userId) }
```
- `POST/GET/PATCH /v1/events`, `POST /v1/events/{id}/rsvp`, `GET /v1/events/{id}.ics` (RFC 5545 VEVENT export).
- RSVP `going` blocked when `maxAttendees` reached. Emits `EventScheduled`, `EventRSVPSubmitted`.
- Public event pages render Schema.org `Event` JSON-LD (SEO per `standards.md`).

**Testing**:
- `Integration: RSVP going to full event → 409 event full`.
- `Integration: GET .ics → valid VEVENT with DTSTART/DTEND in UTC`.
- `Unit: RRULE parser expands weekly rule into correct next-5 occurrences`.

#### 5.2 — Courses, lessons, progress & certificates

**What**: Course → module → lesson hierarchy, enrollment, progress tracking, completion certificates.

**Design**:
```ts
courses: { id, spaceId, title, slug(unique), description, coverImageUrl,
  isPublished, isFree, priceCents, currency, metadata: jsonb (dripSchedule), createdBy }
course_modules: { id, courseId(fk cascade), title, sortOrder }
course_lessons: { id, moduleId(fk cascade), title, contentType: 'text'|'video'|'quiz'|'assignment',
  body, videoUrl, durationMins, sortOrder, isFreePreview, metadata: jsonb (quiz questions) }
course_enrollments: { id, courseId, userId, status:'active'|'completed'|'dropped',
  enrolledAt, completedAt, unique(courseId,userId) }
lesson_progress: { enrollmentId, lessonId, status:'not_started'|'in_progress'|'completed',
  completedAt, PK(enrollmentId,lessonId) }
certificates: { id, enrollmentId(fk cascade), certificateNum(unique), issuedAt, templateId }
```
- `POST /v1/courses/{id}/enroll`, `POST /v1/lessons/{id}/progress`. When all lessons complete → enrollment `completed`, emit `CourseCompleted`, auto-issue certificate (PDF generated by worker), emit `CertificateIssued`.
- Lesson lifecycle states: `not_started → in_progress → completed`.

**Testing**:
- `Integration: complete final lesson → enrollment completed, certificate row created with unique number`.
- `Integration: access paid course lesson (non-preview) without entitlement → 403` (entitlement via Phase 6).
- `Integration: free-preview lesson accessible without enrollment`.

#### 5.3 — Gamification (points, levels, badges, streaks, leaderboard)

**What**: Award points/badges/streaks on activity events; compute levels and leaderboard.

**Design**:
```ts
user_points: { id, userId, points, reason, referenceType, referenceId, createdAt }
levels: { id, name, minPoints(unique), iconUrl }
badges: { id, name(unique), description, iconUrl, criteria: jsonb (type+threshold) }
user_badges: { userId, badgeId, awardedAt, awardedBy, PK(userId,badgeId) }
streaks: { userId, streakType, currentCount, longestCount, lastRecorded, PK(userId,streakType) }
challenges + challenge_participants  (relational, suggestion-1)
```
- A `GamificationListener` subscribes to `PostCreated`, `PostReactionAdded`, `LessonCompleted`, `daily login` → applies a rules table (points per reason, configurable in space settings JSONB). Emits `PointsAwarded`, `BadgeEarned`, `StreakUpdated`.
- Leaderboard: Redis sorted set `leaderboard:{spaceId|global}` updated on `PointsAwarded`; periodic reconciliation job rebuilds from `user_points`.
- `GET /v1/leaderboard?scope=global|space:{id}&period=all|week`.

**Testing**:
- `Integration: create post → PointsAwarded(reason=post_created) → user total increases by configured amount`.
- `Integration: cross level threshold → BadgeEarned for level badge`.
- `Integration: post on consecutive days → streak currentCount increments; skip a day → resets`.
- `Integration: leaderboard ranks users by points desc; tie broken by earliest reach`.

---

## Phase 6: Membership, Payments & Access Gating (Stripe)

### Purpose
Enable monetisation — the gap that makes Discourse insufficient and the reason operators pay for Circle/Skool. Paid membership plans, Stripe Checkout/subscriptions, webhook-driven entitlements, and paid-space/paid-course gating. Requires Phases 2 and 5.

### Tasks

#### 6.1 — Membership plans & Stripe Checkout

**What**: Define plans and start subscription checkout via Stripe.

**Design**:
```ts
membership_plans: { id, name, description, priceCents, currency,
  interval:'month'|'year'|'one_time', stripePriceId, isActive, metadata: jsonb }
subscriptions: { id, userId, planId(fk restrict), stripeSubId, status:
  'trialing'|'active'|'past_due'|'canceled'|'expired', currentPeriodStart, currentPeriodEnd, canceledAt }
payments: { id, userId, subscriptionId, amountCents, currency, paymentMethod,
  providerTxnId, status:'pending'|'succeeded'|'failed'|'refunded', createdAt }
plan_space_access: { planId, spaceId, PK(planId,spaceId) }   // which plans unlock which paid spaces
```
- `POST /v1/billing/checkout` → creates Stripe Checkout Session for a plan → returns redirect URL. Never handles card data (SAQ-A; PCI delegated to Stripe).
- `POST /v1/billing/portal` → Stripe Billing Portal session for self-service management.

**Testing**:
- `Integration (Stripe mock): checkout for active plan → returns session url`.
- `Integration: checkout for inactive plan → 400`.

#### 6.2 — Stripe webhooks & entitlement sync

**What**: Process Stripe webhooks to keep subscriptions/payments and entitlements in sync.

**Design**:
- `POST /v1/billing/webhook` verifies the `stripe-signature` header against `STRIPE_WEBHOOK_SECRET` before parsing (reject 400 on mismatch). Enqueues a BullMQ job (idempotent by Stripe event id) so processing is retryable.
- Handles `checkout.session.completed`, `customer.subscription.updated/deleted`, `invoice.payment_succeeded/failed`. Upserts `subscriptions`/`payments`. Emits `SubscriptionCreated`, `PaymentSucceeded`, `SubscriptionCanceled`.
- Entitlement = active subscription whose plan grants the space via `plan_space_access`. Extends `AccessService.canAccessSpace` for `accessMode='paid'`.

**Testing**:
- `Integration: webhook with invalid signature → 400, no job enqueued`.
- `Integration: checkout.session.completed → subscription active, paid space now accessible`.
- `Integration: duplicate webhook event id → processed once (idempotent)`.
- `Integration: subscription.deleted → access to paid space revoked`.

---

## Phase 7: AI-Native Capabilities — Moderation, Recommendations, Onboarding, Churn

### Purpose
Deliver the core differentiator: AI woven into moderation, content surfacing, onboarding, and engagement forecasting — capabilities incumbents lack or bolt on. The provider-agnostic `@cp/ai` package keeps self-hosters able to run local models. Requires the content (Phase 3) and event log (Phase 1) to be in place.

### Tasks

#### 7.1 — Provider-agnostic AI client

**What**: A `@cp/ai` wrapper over the Vercel AI SDK with model routing and structured output.

**Design**:
```ts
// packages/ai/src/provider.ts
export interface AiClient {
  classify<T>(args: { prompt: string; schema: ZodSchema<T>; system: string }): Promise<T>;
  generate(args: { prompt: string; system: string }): Promise<string>;
  embed(text: string): Promise<number[]>;
}
// Routes to openai|anthropic|ollama from config; when AI_PROVIDER='disabled',
// returns a NoOpAiClient so the platform runs fully without an LLM.
```
- All AI calls run in the worker (BullMQ), never inline on the request path.

**Testing**:
- `Unit: AI_PROVIDER=disabled → NoOpAiClient.classify returns the safe default (no violation)`.
- `Unit (mocked SDK): classify returns object validated against the Zod schema; invalid output → retry once then default`.

#### 7.2 — AI moderation pipeline

**What**: Score new content for policy violations, queue for human review, enforce escalation policy.

**Design**:
```ts
moderation_queue: { id, contentType:'post'|'message'|'profile', contentId, reportedBy,
  reason, aiScore numeric(5,4), aiCategory, status:'pending'|'approved'|'rejected'|'escalated',
  resolvedBy, resolvedAt, metadata: jsonb (modelVersion, rationale), createdAt }
```
- `PostCreated`/`DirectMessageSent`/report → enqueue `moderate` job. Worker calls `ai.classify` with a moderation prompt returning `{ violation: boolean, category, score, rationale }`.
- Escalation policy (per-space JSONB config): `score >= autoRemoveThreshold` → hide content + status `rejected`; `>= reviewThreshold` → `pending` in queue; else pass. Emits `PostFlaggedByAI`, `PostModerationResolved`.
- Moderation prompt template (in `packages/ai/src/prompts/moderation.ts`):
  > System: "You are a community moderator for {communityName}. Community norms: {norms}. Classify the content for: harassment, spam, hate, sexual, self-harm, off-topic. Respond with JSON matching the schema." User: the content.
- Human review endpoints: `GET /v1/moderation/queue`, `POST /v1/moderation/{id}/resolve` (requires `moderation.review`).

**Testing**:
- `Integration (mocked AI): post scored 0.95 harassment with autoRemove=0.9 → content hidden, status rejected, PostFlaggedByAI emitted`.
- `Integration (mocked AI): score 0.6 with review=0.5 → status pending, visible in queue`.
- `Integration: moderator resolves pending item approved → content restored, PostModerationResolved emitted`.
- `Integration: AI disabled → content passes, no queue entry`.

#### 7.3 — Recommendations & personalised feed

**What**: Rank posts/events/members/courses for each member by relevance.

**Design**:
```ts
ai_recommendations: { id, userId, recType:'post'|'event'|'member'|'course', targetId,
  score numeric(5,4), reason, isDismissed, createdAt }
```
- Nightly (BullMQ repeatable) job builds per-user recommendations: embed recent activity (posts the user engaged with) → similarity against candidate content embeddings → top-N with an LLM-generated short `reason`. Emits nothing user-facing until queried.
- `GET /v1/feed?type=post` returns recommendations joined to live content, access-filtered, recency-blended.

**Testing**:
- `Integration (mocked embeddings): user who engaged with topic A → recommendations skew to topic-A posts`.
- `Integration: dismissed recommendation excluded from feed`.
- `Integration: recommendation for inaccessible private space filtered out`.

#### 7.4 — AI onboarding journeys

**What**: Personalised onboarding sequence for new members.

**Design**:
- On `UserRegistered` + first space join, worker generates an onboarding plan: suggested spaces/channels, 3 member connections, and milestone prompts based on `customFields`/stated goals. Stored in `user.metadata.onboarding` (JSONB) and surfaced via `GET /v1/me/onboarding`.
- Prompt produces a structured `OnboardingPlan` (Zod-validated).

**Testing**:
- `Integration (mocked AI): new user with goal "learn X" → plan recommends spaces tagged X`.
- `Integration: AI disabled → static default onboarding checklist returned`.

#### 7.5 — Engagement forecasting / churn prediction

**What**: Score members for churn risk and trigger re-engagement.

**Design**:
```ts
churn_predictions: { id, userId, riskScore numeric(5,4), factors text[], predictedAt }
```
- A `proj_member_engagement`-style materialised summary (posts_7d/30d, logins, last activity) is maintained by an events listener. Daily job computes `riskScore` (heuristic baseline: inactivity decay; optional LLM rationale for `factors`). Emits `ChurnRiskAssessed`; if `risk >= threshold` enqueues re-engagement (email/notification), emits `ReEngagementTriggered`.
- Admin: `GET /v1/admin/churn?minRisk=0.7`.

**Testing**:
- `Integration: member inactive 30d → riskScore high, ChurnRiskAssessed emitted`.
- `Integration: high-risk member → ReEngagementTriggered, email job enqueued`.
- `Integration: active member → low risk, no re-engagement`.

---

## Phase 8: Notifications, Automation Workflows & Email Digests

### Purpose
Operationalise engagement: a unified notification system, operator-defined automation workflows triggered by lifecycle events, and customisable email digests. This turns the event log into proactive member communication. Requires Phases 3, 5, 6.

### Tasks

#### 8.1 — Notifications

**What**: Persistent notifications with in-app, SSE, and email delivery respecting preferences.

**Design**:
```ts
notifications: { id, userId, type, title, body, targetType, targetId, isRead, createdAt }
```
- `NotificationListener` subscribes to events (`mention`, `reply`, `event_reminder`, `badge_earned`, ...) → inserts rows → pushes via the Phase 4 SSE stream → enqueues email if the user's `notificationPrefs` allow that type/channel.
- `GET /v1/notifications`, `POST /v1/notifications/read-all`.

**Testing**:
- `Integration: @mention in post → mentioned user gets a notification row + SSE push`.
- `Integration: user disabled email for replies → no email job, in-app still created`.

#### 8.2 — Automation workflows

**What**: Operator-defined trigger→condition→action rules on lifecycle events.

**Design**:
```ts
automations: { id, name, triggerEvent (e.g. 'member.joined','course.completed','payment.succeeded'),
  conditions: jsonb (expression tree), actions: jsonb (ordered action list), isActive, createdBy }
```
- An `AutomationEngine` subscribes to `domain_events`; for matching `triggerEvent`, evaluates the JSONB condition expression (safe interpreter, no `eval`), then runs actions: `send_email`, `add_role`, `grant_badge`, `post_message`, `add_to_space`, `call_webhook`.
- Conditions/actions validated by Zod on save. Pre-built templates (join welcome, course-complete certificate email) seeded per `features.md` (Circle ships 15+ templates).

**Testing**:
- `Integration: automation on member.joined with action add_role → new member gets role`.
- `Integration: condition (plan == 'pro') false → actions skipped`.
- `Integration: save automation with invalid action type → 400 (Zod)`.
- `Unit: condition interpreter rejects unknown operators (no arbitrary code execution)`.

#### 8.3 — Email digests

**What**: Scheduled, customisable activity digests.

**Design**:
- BullMQ repeatable jobs (daily/weekly per user pref). Digest content = unread notifications + top posts + upcoming events, rendered from MJML templates, sent via the pluggable transport.
- Per-user `notificationPrefs.digest = { frequency, sections[] }`.

**Testing**:
- `Integration (mocked transport): weekly digest job → email captured containing top posts and upcoming events`.
- `Integration: user with digest disabled → no email`.

---

## Phase 9: Public API, Webhooks, OpenAPI/GraphQL Schema & OIDC Provider

### Purpose
Open the developer platform — the underserved area `research.md`/`standards.md` highlight (Skool/Mighty/Heartbeat have weak APIs). Publish OpenAPI 3.1 + GraphQL schema, outbound webhooks, OAuth2/OIDC provider for third-party app access, and SSO as an OIDC client. No paid add-on gate (versus Bettermode's $199/mo).

### Tasks

#### 9.1 — OpenAPI 3.1 + GraphQL schema publication

**What**: Auto-generate and serve API specs and a GraphQL playground.

**Design**:
- `@nestjs/swagger` emits OpenAPI 3.1 (JSON Schema 2020-12 compatible) at `/v1/openapi.json` + Swagger UI at `/docs`. Every DTO derives from `@cp/contracts` Zod schemas.
- GraphQL schema (code-first) served at `/graphql` with playground in non-prod.
- API keys: `api_keys` table (hashed, scoped); `Authorization: Bearer <key>` accepted alongside JWTs.

**Testing**:
- `Integration: GET /v1/openapi.json → valid OpenAPI 3.1 (schema-validate the document)`.
- `Integration: every REST route appears in the spec (route-count == path-count)`.
- `Integration: GraphQL introspection returns the expected core types`.

#### 9.2 — Outbound webhooks

**What**: Operator-registered webhooks receiving signed event deliveries.

**Design**:
```ts
webhooks: { id, url, secret, events text[], isActive, createdBy }
webhook_deliveries: { id, webhookId, eventId, status, statusCode, attempts, lastAttemptAt }
```
- `WebhookDispatcher` subscribes to `domain_events`; for each active webhook subscribed to the event type, enqueues a delivery job. Payload signed with HMAC-SHA256 in `X-CP-Signature`; retries with exponential backoff up to N attempts; deliveries logged.

**Testing**:
- `Integration: register webhook for post.created → creating a post delivers a signed POST (verify HMAC)`.
- `Integration: receiver returns 500 → retried with backoff, attempts incremented`.
- `Integration: inactive webhook → no delivery`.

#### 9.3 — OAuth2 / OIDC provider + SSO client

**What**: Act as an OIDC provider for third-party/member API access and as an OIDC client for social/enterprise SSO.

**Design**:
- Authorization Code + PKCE (RFC 7636), Bearer tokens (RFC 6750), discovery at `/.well-known/openid-configuration`, JWKS endpoint. Issuer = `PUBLIC_BASE_URL`.
- Client side: pluggable providers (Google, GitHub, generic OIDC) via Auth.js; links to `oauth_connections`.

**Testing**:
- `Integration: auth-code+PKCE flow → exchanges code for tokens; wrong code_verifier → invalid_grant`.
- `Integration: GET /.well-known/openid-configuration → required fields present`.
- `Integration (mocked IdP): OIDC login creates/links user via oauth_connections`.

---

## Phase 10: ActivityPub Federation & MCP Server

### Purpose
Ship the headline differentiator no incumbent offers: ActivityPub federation for data portability across the Fediverse, plus an MCP server exposing community data to AI agents. Both are additive integration layers built on the event log. Requires Phases 3 and 9.

### Tasks

#### 10.1 — ActivityPub actors, inbox/outbox, delivery

**What**: Represent local users as AP actors; publish activities; receive remote activities.

**Design** (uses `@cp/activitypub`, schema from suggestion-1 `ap_actors`/`ap_activities`):
```ts
ap_actors: { id, userId(null=remote), actorUri(unique), inboxUrl, outboxUrl, publicKey, isLocal, createdAt }
ap_activities: { id, actorId(fk cascade), activityType:'Create'|'Like'|'Follow'|'Announce',
  objectUri, payload jsonb (raw JSON-LD), isLocal, createdAt }
```
- WebFinger `/.well-known/webfinger`, actor docs at `/users/{username}` (Activity Streams 2.0 / JSON-LD per `standards.md`), `/inbox` + `/outbox`.
- HTTP Signatures sign outbound delivery; verify inbound. A `FederationListener` maps local `PostCreated`→`Create(Note)`, `PostReactionAdded`→`Like`, follows→`Follow`, and enqueues delivery to follower inboxes.
- Federation is opt-in per space/user (config flag).

**Testing**:
- `Integration: GET /.well-known/webfinger?resource=acct:user@host → links to actor`.
- `Integration: GET /users/{username} with Accept: application/activity+json → valid AS2 actor JSON-LD`.
- `Integration: inbound Follow with valid HTTP signature → Accept sent, follower recorded; invalid signature → 401`.
- `Integration: local post in federated space → Create activity enqueued to follower inboxes`.

#### 10.2 — MCP server

**What**: An MCP server exposing read tools over community data for AI agents.

**Design** (per `standards.md` MCP, TypeScript SDK, `apps/mcp`):
- Tools: `search_posts`, `get_member`, `list_events`, `community_health`, `list_moderation_queue` (read-only first). Auth via scoped API key. Respects the same access-control service.

**Testing**:
- `Integration: MCP search_posts tool → returns access-filtered results for the key's scope`.
- `Integration: tool call without valid API key → rejected`.

---

## Phase 11: Frontend Web Application

### Purpose
Deliver the member- and operator-facing web UI consuming the API — mobile-responsive, accessible (WCAG 2.2 AA target), SEO-friendly. Can begin in parallel once Phase 3 stabilises; full coverage requires the feature APIs.

### Tasks

#### 11.1 — App shell, auth, and feed

**What**: Next.js App Router shell with auth flows, navigation, and the discussion feed.

**Design**:
- Server components for public/SEO pages with Schema.org JSON-LD (`DiscussionForumPosting`, `Event`, `Course`, `Person`). Client components for feed, composer, reactions.
- shadcn/ui + Tailwind; data via typed client generated from the OpenAPI spec. WebSocket/SSE clients for live updates.

**Testing**:
- `E2E (Playwright): register → create space → post → reply appears live`.
- `E2E: axe-core scan of feed and composer → no WCAG 2.2 AA violations`.

#### 11.2 — Events, courses, gamification & admin

**What**: UI for events/RSVP, course player, leaderboard/badges, moderation queue, and admin/automation builder.

**Design**:
- Course player tracks lesson progress; calendar view for events; leaderboard page; moderation dashboard (queue + resolve); automation builder (visual trigger/condition/action form bound to `@cp/contracts`).

**Testing**:
- `E2E: enroll in course, complete lessons → certificate downloadable`.
- `E2E: RSVP to event → appears in calendar and attendee list`.
- `E2E: moderator resolves a flagged post from the dashboard`.
- `E2E: keyboard-only navigation through primary flows (WCAG operable)`.

---

## Phase 12: Hardening, Compliance, Analytics & Release

### Purpose
Production-readiness: security hardening (OWASP Top 10:2025 / ASVS L2), privacy compliance (GDPR/CCPA, ISO 27701-aligned), the analytics dashboard with open export, observability, and packaged self-host release. Requires all prior phases.

### Tasks

#### 12.1 — Security hardening

**What**: Apply OWASP ASVS L2 controls across the platform.

**Design**:
- CSP headers, CSRF tokens on cookie-auth web routes, prepared statements (Drizzle parameterised — A03), rate limiting (Redis token bucket) on auth and write endpoints, input validation at every boundary via Zod, secure headers (helmet), object-level authorization checks (A01) audited per endpoint.

**Testing**:
- `Integration: rate limit on /auth/login → 429 after threshold`.
- `Integration: cross-user object access (fetch another user's DM) → 403`.
- `Security: dependency audit (pnpm audit) and SAST in CI → no high/critical`.

#### 12.2 — Privacy compliance (GDPR/CCPA)

**What**: Consent, data export (DSAR), and right-to-erasure workflows.

**Design**:
- `GET /v1/me/export` → machine-readable archive of the member's data (portability; ISO 19941-aligned). `DELETE /v1/me` → erasure: anonymise authored content, delete PII, retain financial records per legal hold. Consent records stored with versioned policy id. Configurable data-retention policies.

**Testing**:
- `Integration: data export → archive contains user's posts, messages, profile`.
- `Integration: erasure → PII removed, posts anonymised, payment records retained`.

#### 12.3 — Analytics dashboard & open export

**What**: Operator analytics with no data lock-in.

**Design**:
- Metrics from the engagement projection + `domain_events`: member growth, active members, post volume, retention cohorts, churn-risk distribution, revenue (MRR). `GET /v1/admin/analytics` + `GET /v1/admin/analytics/export?format=csv|json` (open export — a named gap competitors don't fill).

**Testing**:
- `Integration: analytics endpoint returns member-growth series matching seeded data`.
- `Integration: CSV export parses to expected rows`.

#### 12.4 — Observability & release packaging

**What**: Logging/metrics/tracing and a one-command self-host release.

**Design**:
- Structured JSON logs (pino), OpenTelemetry traces, Prometheus `/metrics`, health/readiness probes. Versioned `docker-compose.yml` + published multi-arch images + `.env.example` + upgrade/migration docs. Tag a release.

**Testing**:
- `Integration: /healthz and /readyz report dependency status`.
- `E2E (clean machine): docker compose up → register, post, RSVP, checkout (Stripe test) all succeed`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, config, auth, domain_events)
    │ required by everything
    ▼
Phase 2: Spaces, Channels, Membership, RBAC ── requires P1
    ▼
Phase 3: Posts, Threads, Reactions, Search ── requires P2  (CORE VALUE)
    ├──────────────┬───────────────┬────────────────────┐
    ▼              ▼               ▼                    ▼
Phase 4:        Phase 5:        Phase 6:             Phase 11:
Real-time       Events/        Membership/          Web Frontend
Messaging       Courses/       Payments             (can start after P3,
(req P3)        Gamification   (req P2,P5)           grows with P4–P10)
                (req P2,P3)
    └──────────────┴───────────────┘
                    ▼
Phase 7: AI Capabilities (moderation, recs, onboarding, churn) ── requires P3 + P1 event log
                    ▼
Phase 8: Notifications, Automation, Digests ── requires P3,P5,P6 (consumes events)
                    ▼
Phase 9: Public API, Webhooks, OpenAPI/GraphQL, OIDC ── requires stable P2–P8 surface
                    ▼
Phase 10: ActivityPub Federation + MCP ── requires P3,P9
                    ▼
Phase 12: Hardening, Compliance, Analytics, Release ── requires all
```

**Parallelism opportunities:**
- After **Phase 3**, Phases **4, 5, and 6** can be developed concurrently by separate contributors (distinct modules, shared only via the access service and event log).
- **Phase 11 (frontend)** can start as soon as Phase 3's API stabilises and grow alongside Phases 4–10 (the OpenAPI-generated client decouples it).
- Within **Phase 7**, tasks 7.2–7.5 are independent of each other once 7.1 (AI client) exists.
- **Phase 9** sub-tasks (OpenAPI, webhooks, OIDC) are independent and parallelisable.

---

## Definition of Done (per phase)

A phase is complete only when every item below holds:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`turbo run test`).
3. `tsc --noEmit` typecheck passes across all affected packages/apps.
4. ESLint and Prettier pass with no errors.
5. New Drizzle migrations are generated, committed, and apply cleanly on a fresh database.
6. New env/config options are added to `@cp/config` schema and `.env.example`.
7. Every new REST endpoint appears in the auto-generated OpenAPI 3.1 spec, and new GraphQL types resolve via introspection (from Phase 9 onward).
8. Every state-mutating operation appends the corresponding `domain_events` record in the same transaction.
9. New JSONB fields have a Zod schema in `@cp/contracts` and are validated at the application boundary.
10. `docker compose up` builds and runs all services with the new code; `/healthz` is green.
11. The phase's headline capability works end-to-end (demonstrated by an integration or E2E test).
12. Security-relevant endpoints enforce RBAC/object-level authorization (verified by a negative-path test).
13. User-facing UI added in the phase (or Phase 11) passes an axe-core WCAG 2.2 AA scan with no violations.
```

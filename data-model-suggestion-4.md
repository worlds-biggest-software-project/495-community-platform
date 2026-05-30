# Data Model Suggestion 4: Graph Database Model (Neo4j)

## Approach

A graph-native data model using Neo4j, where community entities are nodes and all relationships -- membership, authorship, reactions, follows, RSVPs, enrollments, recommendations, and federation links -- are first-class edges with properties. This approach treats the community as a connected graph rather than a collection of tables, enabling traversal-based queries that are central to engagement, recommendations, and moderation.

## Why a Graph Database Suits a Community Platform

A community platform is fundamentally a network of relationships: members connect to spaces, post in channels, reply to each other, react to content, attend events, enroll in courses, follow other members, and earn social standing. The README explicitly calls for several features that are graph-native problems:

- **AI-powered recommendations**: "Surfaces threads, events, and member connections based on each individual's interests and activity history." This is a graph traversal: find content liked by members similar to the target user (collaborative filtering via shared edges).
- **Churn prediction**: Identifying at-risk members requires analyzing their position in the social graph -- isolated members with few connections churn faster than well-connected ones. Graph centrality metrics (degree, betweenness) directly predict engagement.
- **ActivityPub federation**: The Fediverse is a graph of actors following actors across instances. Graph databases model this natively.
- **Engagement networks**: "Who replied to whom, who reacted to whose posts, who attended the same events" forms a weighted social graph that drives community health metrics.
- **Role-based access with inheritance**: Permission propagation through role hierarchies and space memberships is a graph traversal problem.

Relational databases can model these relationships with junction tables and JOINs, but the queries become increasingly expensive as the depth of traversal grows. Graph databases execute multi-hop relationship queries in constant time per hop, regardless of total dataset size.

## Trade-offs

**Strengths:**
- Multi-hop relationship queries (recommendations, friend-of-friend, influence paths) run in milliseconds instead of requiring expensive recursive CTEs or multiple JOINs
- Social graph analysis (centrality, community detection, shortest path) is built into the query language (Cypher) and library (GDS)
- Schema-flexible: adding new relationship types or node properties requires no migration
- Natural representation of ActivityPub's actor-activity-object model
- Visual graph exploration aids debugging and community health analysis
- Pattern matching queries are concise and readable in Cypher

**Weaknesses:**
- Not ideal for high-volume tabular aggregations (financial reports, payment ledgers, time-series analytics)
- Transaction throughput for writes is lower than PostgreSQL for simple CRUD operations
- Fewer developers are experienced with graph databases and Cypher
- No native equivalent of SQL's GROUP BY, window functions, or complex aggregation
- Operational maturity and hosting options lag behind PostgreSQL
- Full-text search requires integration with an external engine or use of Neo4j's built-in (less mature) FTS indexes
- Licensing: Neo4j Community Edition is GPLv3; Enterprise Edition requires a commercial license

## Recommended Architecture

Use Neo4j as the **primary relationship and recommendation engine** alongside PostgreSQL for transactional data (payments, subscriptions, audit logs). This polyglot persistence approach plays to each database's strengths.

```
┌─────────────────────┐     ┌──────────────────────┐
│   PostgreSQL         │     │   Neo4j               │
│                      │     │                       │
│  - payments          │     │  - social graph        │
│  - subscriptions     │     │  - member connections  │
│  - certificates      │     │  - content graph       │
│  - audit logs        │     │  - recommendations     │
│  - media metadata    │     │  - ActivityPub graph   │
│  - automation defs   │     │  - access control      │
│                      │     │  - engagement network  │
└──────────┬───────────┘     └──────────┬────────────┘
           │                            │
           └────────── Sync ────────────┘
              (Change Data Capture / Events)
```

## Schema Definition (Cypher)

### Node Types

```cypher
// ============================================================
// CORE: USERS
// ============================================================
CREATE CONSTRAINT user_id_unique IF NOT EXISTS FOR (u:User) REQUIRE u.id IS UNIQUE;
CREATE CONSTRAINT user_username_unique IF NOT EXISTS FOR (u:User) REQUIRE u.username IS UNIQUE;
CREATE CONSTRAINT user_email_unique IF NOT EXISTS FOR (u:User) REQUIRE u.email IS UNIQUE;

// Example User node
CREATE (u:User {
  id: randomUUID(),
  username: 'philcal',
  email: 'phil@example.com',
  display_name: 'Phil Callender',
  avatar_url: 'https://...',
  bio: 'Community builder',
  timezone: 'Pacific/Auckland',
  locale: 'en',
  email_verified: true,
  is_admin: false,
  is_suspended: false,
  total_points: 450,
  current_level: 'Contributor',
  last_seen_at: datetime(),
  created_at: datetime()
});

CREATE INDEX user_last_seen IF NOT EXISTS FOR (u:User) ON (u.last_seen_at);
CREATE FULLTEXT INDEX user_search IF NOT EXISTS FOR (u:User) ON EACH [u.username, u.display_name, u.bio];

// ============================================================
// SPACES & CHANNELS
// ============================================================
CREATE CONSTRAINT space_id_unique IF NOT EXISTS FOR (s:Space) REQUIRE s.id IS UNIQUE;
CREATE CONSTRAINT space_slug_unique IF NOT EXISTS FOR (s:Space) REQUIRE s.slug IS UNIQUE;
CREATE CONSTRAINT channel_id_unique IF NOT EXISTS FOR (c:Channel) REQUIRE c.id IS UNIQUE;

CREATE (s:Space {
  id: randomUUID(),
  name: 'Engineering',
  slug: 'engineering',
  description: 'Technical discussions',
  access_mode: 'private',  // 'public', 'private', 'paid'
  sort_order: 1,
  is_archived: false,
  created_at: datetime()
});

CREATE (c:Channel {
  id: randomUUID(),
  name: 'General',
  slug: 'general',
  channel_type: 'discussion',  // 'discussion', 'chat', 'announcements', 'qa'
  sort_order: 0,
  is_archived: false,
  created_at: datetime()
});

// ============================================================
// POSTS
// ============================================================
CREATE CONSTRAINT post_id_unique IF NOT EXISTS FOR (p:Post) REQUIRE p.id IS UNIQUE;

CREATE (p:Post {
  id: randomUUID(),
  title: 'How to set up federation',
  body: 'I have been experimenting with ActivityPub...',
  body_html: '<p>I have been experimenting...</p>',
  is_pinned: false,
  is_locked: false,
  is_deleted: false,
  reply_count: 3,
  reaction_count: 7,
  last_activity: datetime(),
  created_at: datetime()
});

CREATE FULLTEXT INDEX post_search IF NOT EXISTS FOR (p:Post) ON EACH [p.title, p.body];

// ============================================================
// EVENTS
// ============================================================
CREATE CONSTRAINT event_id_unique IF NOT EXISTS FOR (e:Event) REQUIRE e.id IS UNIQUE;

CREATE (e:Event {
  id: randomUUID(),
  title: 'Community Town Hall',
  description: 'Monthly community update',
  starts_at: datetime('2026-06-15T18:00:00Z'),
  ends_at: datetime('2026-06-15T19:30:00Z'),
  timezone: 'UTC',
  location_url: 'https://zoom.us/j/...',
  max_attendees: 200,
  is_published: true,
  recurrence_rule: null,
  created_at: datetime()
});

CREATE INDEX event_starts IF NOT EXISTS FOR (e:Event) ON (e.starts_at);

// ============================================================
// COURSES & LESSONS
// ============================================================
CREATE CONSTRAINT course_id_unique IF NOT EXISTS FOR (c:Course) REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT lesson_id_unique IF NOT EXISTS FOR (l:Lesson) REQUIRE l.id IS UNIQUE;
CREATE CONSTRAINT module_id_unique IF NOT EXISTS FOR (m:Module) REQUIRE m.id IS UNIQUE;
CREATE CONSTRAINT certificate_id_unique IF NOT EXISTS FOR (cert:Certificate) REQUIRE cert.id IS UNIQUE;

CREATE (c:Course {
  id: randomUUID(),
  title: 'Community Management 101',
  slug: 'community-management-101',
  description: 'Learn to build thriving communities',
  is_published: true,
  is_free: false,
  price_cents: 4999,
  currency: 'USD',
  created_at: datetime()
});

CREATE (m:Module {
  id: randomUUID(),
  title: 'Getting Started',
  sort_order: 0
});

CREATE (l:Lesson {
  id: randomUUID(),
  title: 'Welcome & Orientation',
  content_type: 'video',
  video_url: 'https://...',
  duration_mins: 12,
  sort_order: 0,
  is_free_preview: true
});

// ============================================================
// GAMIFICATION
// ============================================================
CREATE CONSTRAINT badge_id_unique IF NOT EXISTS FOR (b:Badge) REQUIRE b.id IS UNIQUE;
CREATE CONSTRAINT challenge_id_unique IF NOT EXISTS FOR (ch:Challenge) REQUIRE ch.id IS UNIQUE;
CREATE CONSTRAINT level_id_unique IF NOT EXISTS FOR (lv:Level) REQUIRE lv.id IS UNIQUE;

CREATE (b:Badge {
  id: randomUUID(),
  name: 'First Post',
  description: 'Created your first post',
  icon_url: 'https://...',
  criteria_type: 'post_count',
  criteria_value: 1
});

CREATE (lv:Level {
  id: randomUUID(),
  name: 'Contributor',
  min_points: 100,
  icon_url: 'https://...'
});

// ============================================================
// ROLES
// ============================================================
CREATE CONSTRAINT role_id_unique IF NOT EXISTS FOR (r:Role) REQUIRE r.id IS UNIQUE;

CREATE (r:Role {
  id: randomUUID(),
  name: 'Moderator',
  is_system: true
});

// ============================================================
// ACTIVITYPUB
// ============================================================
CREATE CONSTRAINT ap_actor_uri_unique IF NOT EXISTS FOR (a:APActor) REQUIRE a.actor_uri IS UNIQUE;

CREATE (a:APActor {
  id: randomUUID(),
  actor_uri: 'https://community.example.com/users/philcal',
  inbox_url: 'https://community.example.com/users/philcal/inbox',
  outbox_url: 'https://community.example.com/users/philcal/outbox',
  is_local: true
});

// ============================================================
// MODERATION
// ============================================================
CREATE CONSTRAINT moderation_id_unique IF NOT EXISTS FOR (mq:ModerationItem) REQUIRE mq.id IS UNIQUE;

CREATE (mq:ModerationItem {
  id: randomUUID(),
  content_type: 'post',
  reason: 'ai_flagged',
  ai_score: 0.87,
  ai_category: 'harassment',
  status: 'pending',
  created_at: datetime()
});
```

### Relationship Types

```cypher
// ============================================================
// STRUCTURAL RELATIONSHIPS
// ============================================================

// Space -> Channel hierarchy
MATCH (s:Space {slug: 'engineering'}), (c:Channel {slug: 'general'})
CREATE (s)-[:HAS_CHANNEL]->(c);

// Channel -> Post containment
MATCH (c:Channel {slug: 'general'}), (p:Post {id: $post_id})
CREATE (c)-[:CONTAINS_POST]->(p);

// Post -> Reply threading
MATCH (parent:Post {id: $parent_id}), (reply:Post {id: $reply_id})
CREATE (parent)-[:HAS_REPLY]->(reply);

// Course -> Module -> Lesson hierarchy
MATCH (c:Course {slug: 'community-management-101'}), (m:Module {id: $module_id})
CREATE (c)-[:HAS_MODULE {sort_order: 0}]->(m);

MATCH (m:Module {id: $module_id}), (l:Lesson {id: $lesson_id})
CREATE (m)-[:HAS_LESSON {sort_order: 0}]->(l);

// Event -> Space association
MATCH (e:Event {id: $event_id}), (s:Space {slug: 'engineering'})
CREATE (e)-[:HOSTED_IN]->(s);

// ============================================================
// USER-CONTENT RELATIONSHIPS
// ============================================================

// Authorship
MATCH (u:User {username: 'philcal'}), (p:Post {id: $post_id})
CREATE (u)-[:AUTHORED {at: datetime()}]->(p);

// Reactions (relationship type encodes the emoji)
MATCH (u:User {id: $user_id}), (p:Post {id: $post_id})
CREATE (u)-[:REACTED_TO {emoji: '👍', at: datetime()}]->(p);

// ============================================================
// MEMBERSHIP & ACCESS RELATIONSHIPS
// ============================================================

// Space membership
MATCH (u:User {id: $user_id}), (s:Space {id: $space_id})
CREATE (u)-[:MEMBER_OF {status: 'active', joined_at: datetime()}]->(s);

// Space creation
MATCH (u:User {id: $user_id}), (s:Space {id: $space_id})
CREATE (u)-[:CREATED]->(s);

// Role assignment (global)
MATCH (u:User {id: $user_id}), (r:Role {name: 'Moderator'})
CREATE (u)-[:HAS_ROLE {scope: 'global', granted_at: datetime()}]->(r);

// Role assignment (scoped to space)
MATCH (u:User {id: $user_id}), (r:Role {name: 'Moderator'}), (s:Space {id: $space_id})
CREATE (u)-[:HAS_ROLE {scope: 'space', scope_id: s.id, granted_at: datetime()}]->(r);

// ============================================================
// EVENT RELATIONSHIPS
// ============================================================

// RSVP
MATCH (u:User {id: $user_id}), (e:Event {id: $event_id})
CREATE (u)-[:RSVP {status: 'going', responded_at: datetime()}]->(e);

// Event creation
MATCH (u:User {id: $user_id}), (e:Event {id: $event_id})
CREATE (u)-[:CREATED]->(e);

// ============================================================
// COURSE RELATIONSHIPS
// ============================================================

// Enrollment
MATCH (u:User {id: $user_id}), (c:Course {id: $course_id})
CREATE (u)-[:ENROLLED_IN {status: 'active', enrolled_at: datetime(), progress_pct: 0}]->(c);

// Lesson completion
MATCH (u:User {id: $user_id}), (l:Lesson {id: $lesson_id})
CREATE (u)-[:COMPLETED_LESSON {completed_at: datetime()}]->(l);

// Certificate issuance
MATCH (u:User {id: $user_id}), (cert:Certificate {id: $cert_id})
CREATE (u)-[:EARNED]->(cert);

MATCH (cert:Certificate {id: $cert_id}), (c:Course {id: $course_id})
CREATE (cert)-[:FOR_COURSE]->(c);

// ============================================================
// SOCIAL RELATIONSHIPS
// ============================================================

// Following
MATCH (u1:User {id: $follower_id}), (u2:User {id: $followed_id})
CREATE (u1)-[:FOLLOWS {since: datetime()}]->(u2);

// Direct messaging
MATCH (u1:User {id: $sender_id}), (u2:User {id: $recipient_id})
CREATE (u1)-[:MESSAGED {conversation_id: $conv_id, at: datetime()}]->(u2);

// ============================================================
// GAMIFICATION RELATIONSHIPS
// ============================================================

// Badge earned
MATCH (u:User {id: $user_id}), (b:Badge {id: $badge_id})
CREATE (u)-[:EARNED_BADGE {awarded_at: datetime()}]->(b);

// Challenge participation
MATCH (u:User {id: $user_id}), (ch:Challenge {id: $challenge_id})
CREATE (u)-[:PARTICIPATING_IN {progress: 0, joined_at: datetime()}]->(ch);

// Level achieved
MATCH (u:User {id: $user_id}), (lv:Level {name: 'Contributor'})
CREATE (u)-[:AT_LEVEL {since: datetime()}]->(lv);

// ============================================================
// ACTIVITYPUB FEDERATION RELATIONSHIPS
// ============================================================

// User <-> APActor mapping
MATCH (u:User {id: $user_id}), (a:APActor {actor_uri: $uri})
CREATE (u)-[:HAS_AP_IDENTITY]->(a);

// Federation follows (cross-instance)
MATCH (local:APActor {actor_uri: $local_uri}), (remote:APActor {actor_uri: $remote_uri})
CREATE (local)-[:AP_FOLLOWS {accepted: true, since: datetime()}]->(remote);

// ============================================================
// MODERATION RELATIONSHIPS
// ============================================================

// Flagged content
MATCH (mq:ModerationItem {id: $mod_id}), (p:Post {id: $post_id})
CREATE (mq)-[:FLAGS]->(p);

// Reporter
MATCH (u:User {id: $reporter_id}), (mq:ModerationItem {id: $mod_id})
CREATE (u)-[:REPORTED]->(mq);

// Resolver
MATCH (u:User {id: $mod_user_id}), (mq:ModerationItem {id: $mod_id})
CREATE (u)-[:RESOLVED {decision: 'approved', at: datetime()}]->(mq);

// ============================================================
// MEMBERSHIP PLAN RELATIONSHIPS
// ============================================================

// Plan grants access to spaces
MATCH (plan:MembershipPlan {id: $plan_id}), (s:Space {id: $space_id})
CREATE (plan)-[:GRANTS_ACCESS_TO]->(s);

// User subscribes to plan
MATCH (u:User {id: $user_id}), (plan:MembershipPlan {id: $plan_id})
CREATE (u)-[:SUBSCRIBED_TO {
  status: 'active',
  period_start: datetime(),
  period_end: datetime('2026-06-29'),
  stripe_sub_id: 'sub_xxx'
}]->(plan);
```

### Power Queries (Graph Advantage)

These queries demonstrate where the graph model excels over relational approaches.

```cypher
// ── RECOMMENDATION: Posts liked by similar members ──────────
// "Members who reacted to posts you reacted to also reacted to these posts"
MATCH (me:User {id: $user_id})-[:REACTED_TO]->(post:Post)<-[:REACTED_TO]-(similar:User)
      -[:REACTED_TO]->(recommended:Post)
WHERE NOT (me)-[:REACTED_TO]->(recommended)
  AND NOT recommended.is_deleted
RETURN recommended.id, recommended.title, COUNT(DISTINCT similar) AS score
ORDER BY score DESC
LIMIT 20;

// ── RECOMMENDATION: Suggested connections ───────────────────
// "Members in the same spaces who share the most interactions"
MATCH (me:User {id: $user_id})-[:MEMBER_OF]->(s:Space)<-[:MEMBER_OF]-(other:User)
WHERE me <> other AND NOT (me)-[:FOLLOWS]->(other)
OPTIONAL MATCH (me)-[:REACTED_TO]->(p:Post)<-[:AUTHORED]-(other)
WITH other, COUNT(p) AS interaction_score
ORDER BY interaction_score DESC
LIMIT 10
RETURN other.id, other.display_name, other.avatar_url, interaction_score;

// ── CHURN RISK: Isolated members ────────────────────────────
// Members with low graph connectivity (few relationships) are at high churn risk
MATCH (u:User)
WHERE u.last_seen_at < datetime() - duration('P14D')
OPTIONAL MATCH (u)-[r]-()
WITH u, COUNT(r) AS connections
WHERE connections < 5
RETURN u.id, u.display_name, u.last_seen_at, connections
ORDER BY connections ASC;

// ── ENGAGEMENT: Influence path ──────────────────────────────
// "How did this post spread through the community?"
MATCH path = (author:User)-[:AUTHORED]->(original:Post {id: $post_id})
              <-[:HAS_REPLY]-(reply:Post)<-[:AUTHORED]-(replier:User)
RETURN author.display_name, replier.display_name,
       reply.created_at, length(path) AS depth
ORDER BY reply.created_at;

// ── ACCESS CONTROL: Can user access this space? ─────────────
// Check direct membership OR paid plan access
MATCH (u:User {id: $user_id})
OPTIONAL MATCH (u)-[m:MEMBER_OF {status: 'active'}]->(s:Space {id: $space_id})
OPTIONAL MATCH (u)-[sub:SUBSCRIBED_TO {status: 'active'}]->(plan:MembershipPlan)
              -[:GRANTS_ACCESS_TO]->(s2:Space {id: $space_id})
RETURN m IS NOT NULL OR sub IS NOT NULL AS has_access;

// ── ACTIVITYPUB: Federation graph ───────────────────────────
// Find all remote actors that follow local community members
MATCH (remote:APActor {is_local: false})-[:AP_FOLLOWS]->(local:APActor {is_local: true})
      <-[:HAS_AP_IDENTITY]-(u:User)
RETURN remote.actor_uri, u.display_name, u.username
ORDER BY u.display_name;

// ── LEADERBOARD: Top contributors with context ─────────────
MATCH (u:User)
OPTIONAL MATCH (u)-[:AUTHORED]->(p:Post) WHERE NOT p.is_deleted
OPTIONAL MATCH (u)-[:EARNED_BADGE]->(b:Badge)
RETURN u.id, u.display_name, u.total_points,
       COUNT(DISTINCT p) AS post_count,
       COUNT(DISTINCT b) AS badge_count
ORDER BY u.total_points DESC
LIMIT 50;

// ── COMMUNITY DETECTION: Find organic sub-communities ──────
// Using Neo4j GDS (Graph Data Science) library
CALL gds.louvain.stream({
  nodeProjection: 'User',
  relationshipProjection: {
    INTERACTS: {
      type: 'REACTED_TO',
      orientation: 'UNDIRECTED'
    }
  }
})
YIELD nodeId, communityId
RETURN gds.util.asNode(nodeId).display_name AS member,
       communityId AS community
ORDER BY community, member;
```

### PostgreSQL Companion Tables (for transactional data)

```sql
-- Payment and subscription data stays in PostgreSQL
-- for ACID guarantees and financial reporting

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,  -- references Neo4j User.id
    amount_cents    INT NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    payment_method  VARCHAR(32) NOT NULL,
    provider_txn_id VARCHAR(256),
    status          VARCHAR(16) NOT NULL DEFAULT 'succeeded',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id        UUID NOT NULL,
    action          VARCHAR(128) NOT NULL,
    target_type     VARCHAR(64),
    target_id       UUID,
    details         JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_actor ON audit_log (actor_id, created_at DESC);
CREATE INDEX idx_audit_target ON audit_log (target_type, target_id);

CREATE TABLE media_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    uploaded_by     UUID NOT NULL,
    filename        VARCHAR(512) NOT NULL,
    mime_type       VARCHAR(128) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Scalability Considerations

- **Read replicas**: Neo4j supports causal clustering with read replicas for scaling graph reads.
- **Sharding**: Neo4j 5+ supports server-side routing and fabric for distributed graphs, though single-instance performance handles communities up to ~50M nodes comfortably.
- **Warm cache**: Neo4j performs best when the working set fits in memory. Size the JVM heap and page cache to cover the active user graph.
- **Batch imports**: Use `LOAD CSV` or the Neo4j Import Tool for initial data migration; use `UNWIND` for batch writes from the application.
- **CDC sync**: Use Debezium or Neo4j Streams to sync between PostgreSQL (transactional) and Neo4j (graph) in near-real-time.
- **GDS projections**: Graph Data Science library projections (for community detection, PageRank, similarity) run on in-memory graph projections and do not lock the operational database.

## Migration Path

- Start with PostgreSQL (suggestion 1 or 3) and add Neo4j when recommendation or social graph queries become bottlenecks.
- Use change data capture (Debezium) to replicate relevant PostgreSQL tables into Neo4j nodes and relationships.
- Gradually move recommendation, feed ranking, and access control queries to Neo4j while keeping PostgreSQL as the system of record for CRUD and payments.
- If Neo4j licensing is a concern, consider Apache AGE (graph extension for PostgreSQL) or Memgraph (open-source, Cypher-compatible) as alternatives.

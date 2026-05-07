# Community Platform — Feature & Functionality Survey

> Candidate #495 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Circle | All-in-one creator community | SaaS (from $49/mo) | https://circle.so |
| Mighty Networks | Creator + course community | SaaS (from $41/mo) | https://mightynetworks.com |
| Skool | Simple forum + courses | SaaS ($9–$99/mo) | https://skool.com |
| Bettermode | Flexible enterprise community | SaaS (from $49/mo + API add-on $199/mo) | https://bettermode.com |
| Discourse | Forum software | Open source (AGPL-3.0) / SaaS (from $100/mo) | https://discourse.org |
| Heartbeat | Chat-first engagement | SaaS (from $49/mo) | https://heartbeat.chat |
| Kajabi Communities | Community module within Kajabi | SaaS (from $69/mo) | https://kajabi.com |

---

## Feature Analysis by Solution

### Circle

**Core features**
- Spaces (topic-specific sub-communities with access controls)
- Native live events with up to 1,000 attendees, screen sharing, breakout rooms, and Q&A queuing
- Courses with video hosting, structured modules, and drip scheduling
- Memberships and payment gating (Stripe-based)
- Automation workflows with 15+ pre-built templates; triggers on join, course completion, payment, RSVP
- Native email marketing (add-on cost above base plan)
- Post editor with "/" command structure supporting embedded video, audio, PDFs, forms, surveys

**Differentiating features**
- Headless/API capability for embedding community into external sites (Admin API + Member API, GraphQL)
- White-label branded mobile apps on Business/Enterprise tiers
- Full AI content moderation, AI-assisted onboarding flows, and content recommendation engine
- Sandbox environment and OpenAPI specs for developer integrations

**UX patterns**
- Notion-style block editor for rich content
- Progressive feature gating: Basic plan lacks custom domain, automation available only on Business ($199/mo)
- Wizard-driven community setup for new operators

**Integration points**
- REST + GraphQL APIs; Admin API and Member API with JWT authentication
- Webhooks, Postman/Insomnia collections, OpenAPI/Swagger specs
- Zapier, Make, and native integrations with email/CRM tools

**Known gaps**
- Email marketing costs extra; not included in base tiers
- Courses cannot issue certificates without third-party integration
- Automation and full AI suite gated behind highest paid tier
- No ActivityPub / federated identity support

**Licence / IP notes**
- Proprietary SaaS; no open-source components publicly released

---

### Mighty Networks

**Core features**
- Spaces with Private, Secret, and Paid access modes
- Activity feed, sub-groups, member categories, chat messaging
- Native video hosting (no external YouTube/Vimeo dependency)
- Cohort-based and on-demand courses with video tracking
- Voice messaging with auto-transcription
- Native iOS and Android mobile apps
- Gamification with engagement badges and streaks
- Live events with strong interactive features

**Differentiating features**
- Mighty Co-Host™ AI: generates community names, descriptions, and course outlines
- Mobile-first architecture with polished native app experience
- Voice messaging as a first-class communication channel

**UX patterns**
- Historically large/clunky UI but substantially redesigned in 2025–2026
- Community and course content co-located so members never leave the community context

**Integration points**
- Limited direct API (partially conflicting reports; most integrations require Zapier)
- Native ConvertKit email integration for newsletters and drips
- Scale plan ($215/mo) required for most third-party integrations
- Webhooks available via Zapier

**Known gaps**
- No profanity filters or spam rules — problematic for public communities
- Stripe only for payments; no PayPal or alternative payment options
- API access limited and poorly documented compared to competitors
- Integrations locked to highest pricing tier

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### Skool

**Core features**
- Community forum (threaded discussions, posts, comments)
- Classroom (multi-module courses with video, now with native video upload and auto-captions)
- Calendar with event scheduling
- Gamification: points, levels, leaderboards, unlockable content, badges
- Membership management
- Native mobile app
- Affiliate/referral programme built-in

**Differentiating features**
- Arguably the most effective gamification in the category: streaks, leaderboards, level-locked content
- Skool Games: community-wide competitive engagement mechanic between Skool communities
- Freemium model from $9/mo (Hobby tier introduced 2026) makes it the most accessible entry point
- Flat-fee pricing model (no per-member fees) simplifies cost forecasting

**UX patterns**
- Deliberately minimal UI — low cognitive overhead for both operators and members
- Onboarding is rapid; new operators can be live in under 30 minutes

**Integration points**
- Limited native integrations; Zapier-dependent for most automation
- No public API or webhook documentation
- Basic Stripe payment integration

**Known gaps**
- Very limited customisation — no white-label, no custom branding beyond a logo
- No email marketing module
- No AI features as of 2026
- Weak analytics and reporting
- No moderation automation; human-only moderation

**Licence / IP notes**
- Proprietary SaaS

---

### Bettermode

**Core features**
- Fully customisable community spaces with drag-and-drop layout editor
- Knowledge base and resource hub templates
- Private messaging
- Badges and leaderboards (both automated smart badges and manual badges)
- Branded themes and custom domains
- Advanced search
- Multilingual support
- Moderation controls with detailed audit trails

**Differentiating features**
- GraphQL API as first-class interface (not an afterthought); developer portal with full schema, playground, and mutation docs
- 30+ layout templates supporting many community archetypes (product feedback, customer success, brand advocacy)
- Liquid template engine for custom script injection
- Widest integration ecosystem: Amplitude, Discord, FullStory, Google Analytics, Hotjar, Intercom, Jira, Mixpanel, Segment, SendGrid, Stripe, and more

**UX patterns**
- More complex setup than Skool or Circle; aimed at technical and enterprise operators
- Progressive disclosure via template selection before customisation

**Integration points**
- GraphQL API (POST-only) with app access tokens and member-specific auth
- Webhooks natively supported
- Zapier and Slack native integrations
- API + Webhooks is a paid add-on at $199/mo on top of base plan

**Known gaps**
- API/Webhooks add-on cost makes developer access expensive
- Steeper learning curve than simpler platforms
- Live events capability weaker than Circle or Heartbeat
- No native course or LMS module

**Licence / IP notes**
- Proprietary SaaS; developer platform documentation publicly available

---

### Discourse

**Core features**
- Threaded forum discussions with trust-level system (new users sandboxed, privileges unlocked progressively)
- Real-time chat (channel-based, separate from forum threads)
- Private messaging
- Rich-text editor (plain text, Markdown, HTML)
- Native mobile layout and iOS/Android apps
- Plugin architecture with hundreds of official and community-contributed plugins
- Email-in and email-out (reply-by-email)
- Topic voting, polls, and badges

**Differentiating features**
- Only major platform with a mature open-source (AGPL-3.0) self-hosted option
- Trust Level system is a unique community-governance mechanism not replicated elsewhere
- SEO-optimised thread structure by design
- GitHub integration with badge awards based on contributions

**UX patterns**
- Forum-centric mental model; less suited to feed-based or course-centric use cases
- `/` (admin settings) and plugin management through web UI; developer-friendly

**Integration points**
- Comprehensive REST API at https://docs.discourse.org
- SSO via DiscourseConnect (custom) or third-party OAuth 2.0 providers
- Integrations: WordPress (WP Discourse plugin), Zendesk (ticket creation from topics), GitHub, Patreon, Memberful
- Extensive webhook support

**Known gaps**
- Limited built-in monetisation; no native paid membership flow without plugins
- No native video hosting or course module
- Hosted plans start at $100/mo — self-hosted is free but requires server ops expertise
- No AI features in core; AI features require third-party plugins

**Licence / IP notes**
- AGPL-3.0 open source; self-hosted use is free. Commercial hosted service is separate
- Plugin ecosystem is a mix of MIT, GPL, and proprietary licences — check each plugin individually

---

### Heartbeat

**Core features**
- Chat-first communication (channels, DMs)
- Cohort-based and evergreen courses
- Native events with Zoom integration
- Automation workflows without needing external Zapier
- Pulse AI co-builder (channel structure suggestions, onboarding message sequences, automated DM flows)
- Native mobile app

**Differentiating features**
- Pulse AI is the most integrated AI co-builder in the category: generates channel structure, writes onboarding sequences, and drafts automated DM flows triggered by join/lesson/event actions
- Native workflow automation reduces dependency on third-party tools
- Cohort-based course model co-located with community chat

**UX patterns**
- Chat-centric layout (more Slack-like than forum-like)
- Praised for intuitive interface and responsive support team
- Active listening to user feedback with fast feature iteration

**Integration points**
- Zoom integration for live events
- Workflow automation (native, no Zapier required for common flows)
- Limited public API documentation compared to Bettermode or Circle

**Known gaps**
- Smaller ecosystem and fewer integrations than Circle or Bettermode
- Member cap pricing model: $49/mo (up to 350 members), $149/mo (up to 5,000), $849/mo (unlimited)
- No native white-label or custom domain on lower tiers
- GraphQL or REST API not publicly documented

**Licence / IP notes**
- Proprietary SaaS

---

### Kajabi Communities

**Core features**
- Community spaces (discussions, direct messages)
- Tightly integrated with Kajabi courses, email marketing, and sales funnels
- Inbound and outbound webhooks for purchase and access events
- Member access management tied to product ownership
- Mobile app (shared with wider Kajabi platform)

**Differentiating features**
- Native integration with Kajabi's course delivery, email automation, pipelines, and checkout — no third-party glue required for creators already on Kajabi
- Purchase-triggered community access (webhook events: Purchase Created, Purchase Succeeded, Cart Purchase)

**UX patterns**
- Community is a secondary feature within a broader creator platform context
- Users already familiar with Kajabi admin experience minimal friction

**Integration points**
- Kajabi API (API Key + API Secret authentication)
- Inbound and outbound webhooks
- Zapier, Make integrations
- REST-based API (no GraphQL)

**Known gaps**
- Community feature is secondary to courses/email; limited standalone community capability
- No gamification
- No AI moderation or engagement features
- Expensive if not already using the full Kajabi stack

**Licence / IP notes**
- Proprietary SaaS

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Member profiles with activity history and role assignment
- Spaces/channels/sub-groups for topic organisation
- Threaded or chat-based discussion
- Basic event scheduling and RSVP
- Mobile-responsive web and native iOS/Android app
- Membership gating (free and paid tiers)
- Stripe payment integration
- Email notification system
- Basic moderation (report, remove, ban)
- Admin analytics dashboard (member growth, engagement metrics)

### Differentiating Features
- Gamification: points, levels, leaderboards, unlockable content (Skool leads)
- Native live events with breakout rooms and Q&A (Circle, Heartbeat)
- AI moderation and AI-driven onboarding workflows (Circle, Heartbeat/Pulse)
- Headless / embedded community via GraphQL API (Bettermode, Circle)
- White-label branded mobile app (Circle Enterprise, Mighty Networks)
- Trust Level system for progressive community governance (Discourse)
- Native course LMS co-located with community (Circle, Mighty Networks, Heartbeat)
- Open-source self-hosted option (Discourse only)
- Voice messaging with auto-transcription (Mighty Networks)

### Underserved Areas / Opportunities
- ActivityPub / federated identity: no mainstream commercial platform supports it; data portability is a growing demand
- Certificate and credential issuance for course completion without third-party dependency
- Sophisticated AI-moderation that understands community-specific norms (not just profanity filtering)
- Proactive churn prediction and automated re-engagement at the member level
- Cross-community discovery and networking (the "LinkedIn layer" for community members)
- Open analytics export: most platforms lock data inside their own dashboards
- Accessibility compliance (WCAG 2.1 AA): largely unaddressed in platform marketing
- Multi-currency and non-Stripe payment options (PayPal, SEPA, LATAM methods)
- Aggregated community health scoring with recommended interventions

### AI-Augmentation Candidates
- Content moderation: rule-based filters are insufficient at scale; AI can detect nuanced policy violations and context-specific toxicity
- Member onboarding journeys: today largely template-based; AI can personalise sequencing based on member profile and early engagement signals
- Engagement forecasting: identifying at-risk members before they churn (currently absent in all platforms)
- Event recommendations: AI can match members to events based on interest graph (manual/admin-driven today)
- Content surfacing: most platforms surface content by recency; AI relevance ranking is a clear upgrade

---

## Legal & IP Summary

No patent concerns were identified in researching this feature set. The dominant open-source platform (Discourse) is licensed under AGPL-3.0, which requires derivative works to be open-sourced if distributed. The plugin ecosystem is mixed (MIT, GPL, proprietary) and requires per-plugin licence review. All other analysed platforms are proprietary SaaS products with no publicly released code. An AI-native open-source community platform should adopt a permissive licence (MIT or Apache 2.0) to maximise adoption. There are no known active patent disputes in the community platform space.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Member profiles, roles, and access controls
- Spaces / channels for topic organisation
- Threaded discussion posts and direct messaging
- Event scheduling, RSVP, and calendar
- Paid and free membership gating via Stripe
- AI moderation with configurable escalation policies
- Basic analytics dashboard (member activity, post volume, retention)

**Should-have (v1.1)**
- Gamification system (points, leaderboards, level-locked content)
- Native course/LMS module co-located with community
- Automation workflows triggered by member lifecycle events
- Personalised content recommendation feed
- Email notification management with customisable digests
- REST + GraphQL API with webhooks for third-party integrations
- Mobile-responsive web (native app in later phase)

**Nice-to-have (backlog)**
- ActivityPub federation for data portability and cross-instance interaction
- Branded white-label mobile app builder
- AI engagement forecasting with churn risk alerts
- Certificate/credential issuance for course completion
- Multi-currency checkout with non-Stripe payment options
- Cross-community member directory and networking graph
- WCAG 2.1 AA accessibility audit and remediation tooling

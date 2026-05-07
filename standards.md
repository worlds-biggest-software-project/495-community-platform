# Standards & API Reference

> Project: Community Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management systems. Provides the framework for establishing, implementing, maintaining, and continually improving an information security management system (ISMS). Essential for any community platform handling member credentials, payment data, and private communications. https://www.iso.org/standard/27001

- **ISO/IEC 27701:2025** — Privacy information management systems. Extends ISO 27001 with specific requirements for managing personally identifiable information (PII). The 2025 edition is now a stand-alone standard that no longer requires an existing ISO 27001 ISMS, giving organisations more flexibility. Directly applicable to community platforms acting as PII controllers and processors. https://www.iso.org/standard/27701

- **ISO/IEC 40500:2025 (WCAG 2.2)** — Web content accessibility. This is the ISO adoption of the W3C WCAG 2.2 standard, providing internationally recognised accessibility requirements. Community platforms must target Level AA conformance to meet legal obligations in the EU (European Accessibility Act), US (ADA), and other jurisdictions. https://www.w3.org/TR/WCAG22/

- **ISO/IEC 19941:2017** — Cloud computing interoperability and portability. Defines portability types and interoperability concepts for cloud services. Relevant to ensuring member data can be exported and migrated between community platform instances or providers. https://www.iso.org/standard/66639.html

### W3C & IETF Standards

- **ActivityPub (W3C Recommendation, 2018)** — Federated social networking protocol providing client-to-server and server-to-server APIs. Powers the Fediverse (Mastodon, Lemmy, etc.). A new W3C Social Web Working Group was chartered in January 2026 to release a backwards-compatible update in Q3 2026, including work on LOLA (Live Online Account Portability) for server-to-server account migration. https://www.w3.org/TR/activitypub/

- **Activity Streams 2.0 (W3C Recommendation)** — JSON-based syntax for expressing social activities, conforming to a subset of JSON-LD. The data model underlying ActivityPub. Defines core object types (Create, Like, Follow, Announce) essential for federated community interactions. https://www.w3.org/TR/activitystreams-core/

- **JSON-LD 1.1 (W3C Recommendation)** — A method of encoding Linked Data using JSON. Used by Activity Streams 2.0 and ActivityPub as the encoding mechanism for social data. Also used for Schema.org structured data (SEO) on community pages. https://www.w3.org/TR/json-ld11/

- **RFC 6749 — OAuth 2.0 Authorization Framework (IETF)** — The foundational authorisation protocol for delegated access. Community platforms must implement OAuth 2.0 for third-party integrations, SSO with external identity providers, and API authentication. https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 6750 — OAuth 2.0 Bearer Token Usage (IETF)** — Defines how to use bearer tokens in HTTP requests to access OAuth 2.0 protected resources. https://datatracker.ietf.org/doc/html/rfc6750

- **OpenID Connect Core 1.0** — Identity layer built on top of OAuth 2.0 that answers "who is this user?" Maintained by the OpenID Foundation. Essential for single sign-on, social login, and enterprise identity federation in community platforms. https://openid.net/specs/openid-connect-core-1_0.html

- **RFC 6455 — The WebSocket Protocol (IETF)** — Full-duplex communication protocol over a single TCP connection. Critical for real-time features in community platforms: live chat, typing indicators, presence detection, live event participation, and push notifications. https://datatracker.ietf.org/doc/html/rfc6455

- **WCAG 2.2 (W3C Recommendation, October 2023)** — Web Content Accessibility Guidelines with 13 guidelines under 4 principles (Perceivable, Operable, Understandable, Robust) at three conformance levels (A, AA, AAA). WCAG 2.2 adds 9 new success criteria focused on low vision, cognitive disabilities, and motor disabilities. Level AA is the target for legal compliance in most jurisdictions. https://www.w3.org/TR/WCAG22/

- **Schema.org / JSON-LD Structured Data** — Vocabulary for structured data markup. Relevant types for community platforms include `Event`, `Organization`, `Person`, `DiscussionForumPosting`, and `Course`. Implementing JSON-LD markup improves SEO visibility and enables rich results in search engines. https://schema.org/

### Data Model & API Specifications

- **OpenAPI Specification 3.1 / 3.2** — The standard for describing RESTful APIs. OpenAPI 3.1 achieved full compatibility with JSON Schema Draft 2020-12. Community platforms should publish OpenAPI specs for their public APIs to enable SDK generation, documentation, and third-party integration. The latest version is 3.2.0. https://spec.openapis.org/oas/v3.2.0.html

- **GraphQL (Linux Foundation)** — Query language and runtime for APIs. Used by Bettermode and Circle as their primary API interface. GraphQL enables clients to request exactly the data they need, reducing over-fetching. Particularly suited to community platforms where frontend components need flexible queries across members, posts, events, and spaces. https://graphql.org/

- **JSON Schema (Draft 2020-12)** — Vocabulary for annotating and validating JSON documents. Used for API request/response validation, configuration schemas, and data model documentation. Fully compatible with OpenAPI 3.1+. https://json-schema.org/

- **Server-Sent Events (SSE, W3C)** — HTTP-based protocol for server-to-client streaming. Lighter-weight alternative to WebSockets for one-directional real-time updates (notifications, activity feeds, status changes). https://html.spec.whatwg.org/multipage/server-sent-events.html

### Security & Authentication Standards

- **OWASP Top 10:2025** — The latest edition (confirmed January 2026) identifies the ten most critical web application security risks based on analysis of 175,000+ vulnerabilities. Key risks for community platforms include: A05 Injection (SQL injection, XSS), Broken Access Control (A01), Security Misconfiguration (A02), and Server-Side Request Forgery. Community platforms handling user-generated content must implement robust input sanitisation, Content Security Policy headers, CSRF tokens, and prepared statements. https://owasp.org/Top10/2025/en/

- **OWASP Application Security Verification Standard (ASVS)** — Detailed security requirements and testing criteria for web applications. Provides three levels of verification depth. Community platforms should target Level 2 for standard applications or Level 3 for high-value platforms handling payment data. https://owasp.org/www-project-application-security-verification-standard/

- **OAuth 2.0 + PKCE (RFC 7636)** — Proof Key for Code Exchange. Mitigates authorisation code interception attacks in public clients (mobile apps, SPAs). Required for community platform mobile apps and JavaScript clients using OAuth. https://datatracker.ietf.org/doc/html/rfc7636

- **GDPR (EU General Data Protection Regulation)** — Requires explicit consent for data processing, right to erasure, data portability, and breach notification. Enforcement has reached EUR 7.1 billion in cumulative fines. Community platforms collecting member data, payment information, and behavioural analytics must implement consent management, DSAR workflows, and data retention policies. https://gdpr.eu/

- **CCPA/CPRA (California Consumer Privacy Act / California Privacy Rights Act)** — US state-level privacy law requiring transparency about data collection and opt-out mechanisms for data sales/sharing. As of 2026, 20 US states have comprehensive privacy laws. Community platforms serving US members must implement state-specific compliance controls. https://oag.ca.gov/privacy/ccpa

- **PCI DSS 4.0** — Payment Card Industry Data Security Standard. Applicable to community platforms processing membership payments. Most platforms delegate PCI compliance to Stripe Connect or similar payment processors, but must still maintain SAQ-A compliance for their integration layer. https://www.pcisecuritystandards.org/

### MCP Server Specifications

- **Model Context Protocol (MCP)** — Open protocol standardised by the Agentic AI Foundation (Linux Foundation) enabling LLM applications to integrate with external tools and data sources. Relevant to community platforms as an integration point: MCP servers could expose community data (posts, members, events, analytics) to AI agents for automated moderation, content generation, member support, and engagement analysis. Over 500 public MCP servers exist as of early 2026. Official SDKs available for TypeScript, Python, C#, Java, and Swift. https://modelcontextprotocol.io/specification/2025-11-25

---

## Similar Products — Developer Documentation & APIs

### Circle

- **Description:** All-in-one creator community platform combining forums, courses, live events, and paid memberships. Offers a headless mode for embedding community features into external sites.
- **API Documentation:** https://api.circle.so/
- **SDKs/Libraries:** Postman/Insomnia collections available; OpenAPI/Swagger specs published. No official SDK; third-party wrappers available.
- **Developer Guide:** https://api.circle.so/apis/headless (Headless), https://api.circle.so/apis/admin-api (Admin API), https://api.circle.so/apis/headless/member-api (Member API)
- **Standards:** REST + GraphQL APIs; Admin API and Member API with JWT authentication; OpenAPI specs available
- **Authentication:** JWT-based authentication; API keys for admin access; OAuth for member-facing integrations

### Bettermode

- **Description:** Highly flexible enterprise community platform with 30+ templates, GraphQL-first API, and deep customisation via a Liquid template engine. Strongest developer platform in the category.
- **API Documentation:** https://developers.bettermode.com/docs/graphql/schema/
- **SDKs/Libraries:** No official SDKs; GraphQL schema available for code generation
- **Developer Guide:** https://developers.bettermode.com/docs/guide/ (Getting Started), https://developers.bettermode.com/docs/guide/graphql/getting-started/ (GraphQL Guide)
- **Standards:** GraphQL API (POST-only); full schema with queries, mutations, inputs, and objects documented
- **Authentication:** App Access Token and Member Access Token; rate limits use burst (10-second window) and daily buckets

### Discourse

- **Description:** Open-source (AGPL-3.0) discussion forum platform with a mature plugin architecture, trust-level system, and comprehensive REST API. The only major open-source option in the community platform category.
- **API Documentation:** https://docs.discourse.org/
- **SDKs/Libraries:** Ruby API client (https://github.com/discourse/discourse_api); community Python and JavaScript clients available
- **Developer Guide:** https://meta.discourse.org/t/discourse-rest-api-comprehensive-examples/274354
- **Standards:** REST API with JSON responses; OpenAPI spec auto-generated from source via rswag; supports `.json` extension or `Accept: application/json` header
- **Authentication:** API Key + API Username via HTTP headers; DiscourseConnect SSO (custom) or third-party OAuth 2.0 providers

### Mighty Networks

- **Description:** Creator and business platform for community, courses, and memberships with a mobile-first native app experience. Includes Mighty Co-Host AI for content generation.
- **API Documentation:** https://docs.mightynetworks.com/guides
- **SDKs/Libraries:** Community-developed Python SDK covering 137 endpoints across 18 resource categories (https://github.com/pkshahid/mighty_networks_sdk); no official SDKs
- **Developer Guide:** https://docs.mightynetworks.com/guides
- **Standards:** REST API; limited public documentation; most integrations require Zapier on Business/Pro plans
- **Authentication:** API key-based authentication; Zapier integration for third-party connections

### Heartbeat

- **Description:** Chat-first community platform with cohort-based courses, native events, and Pulse AI co-builder for automated onboarding and engagement workflows.
- **API Documentation:** https://help.heartbeat.chat/hc/en-us/articles/33257714954001-Heartbeat-API
- **SDKs/Libraries:** No official SDKs; Pabbly Connect and Pipedream integrations available
- **Developer Guide:** Help center documentation at https://www.heartbeat.chat/help
- **Standards:** REST API (base URL: `https://api.heartbeat.chat/v0`); limited public endpoint documentation
- **Authentication:** API key generated from Settings > Automation > API Keys; Business plan required

### Skool

- **Description:** Simplified community platform combining forums, courses, and gamification (points, levels, leaderboards) for coaching and course creators. Known for simplicity and flat-fee pricing.
- **API Documentation:** https://docs.skoolapi.com/
- **SDKs/Libraries:** No official SDKs; Zapier integration available on Pro plan
- **Developer Guide:** https://docs.skoolapi.com/ (early-stage documentation)
- **Standards:** REST API with webhook support; OpenAPI-style documentation; limited endpoint coverage (expanding)
- **Authentication:** Group API key + Group URL; Zapier-based automation

### Kajabi

- **Description:** All-in-one creator platform with community as a module alongside courses, email marketing, and sales funnels. Community features are secondary to the broader content delivery platform.
- **API Documentation:** https://help.kajabi.com/en/articles/12696419-using-kajabi-s-api
- **SDKs/Libraries:** No official SDKs; Zapier and Make integrations available
- **Developer Guide:** https://help.kajabi.com/articles/api-integrations/webhooks/webhooks-explained
- **Standards:** REST API; inbound and outbound webhooks (Purchase Created, Purchase Succeeded, Cart Purchase events)
- **Authentication:** API Key + API Secret; webhook signature verification

### Stripe Connect (Payment Infrastructure)

- **Description:** Payment platform for marketplaces and platforms. The de facto standard for community platform monetisation — membership payments, revenue sharing, and payouts. Used by Circle, Skool, Kajabi, and most community platforms.
- **API Documentation:** https://docs.stripe.com/api
- **SDKs/Libraries:** Official SDKs for Node.js, Python, Ruby, Go, Java, .NET, PHP (https://stripe.com/docs/libraries)
- **Developer Guide:** https://docs.stripe.com/connect (Connect), https://docs.stripe.com/connect/end-to-end-marketplace (Marketplace Guide)
- **Standards:** REST API; OpenAPI spec published; PCI DSS Level 1 certified; supports 135+ currencies and 40+ payment methods
- **Authentication:** API keys (publishable + secret); OAuth 2.0 for Connect account onboarding; webhook signature verification via `stripe-signature` header

---

## Notes

- **Federation gap:** No mainstream commercial community platform currently supports ActivityPub or any federated protocol. This is a significant opportunity for an open-source AI-native platform to differentiate through data portability and interoperability. The W3C Social Web Working Group rechartered in January 2026 is actively working on updates including LOLA (account portability), which could accelerate adoption.

- **API maturity varies widely:** Bettermode and Discourse have the most complete developer platforms (full GraphQL/REST with documentation). Circle is catching up with its headless offering. Mighty Networks, Heartbeat, Skool, and Kajabi have limited or Zapier-dependent API access, representing a clear underserved area.

- **GraphQL vs REST:** The market is split — Bettermode and Circle use GraphQL; Discourse, Kajabi, Heartbeat, and Skool use REST. A new platform should consider offering both, or prioritising GraphQL with a REST compatibility layer, to serve the widest developer audience.

- **Real-time standards:** WebSocket (RFC 6455) is the baseline for chat and live features. Server-Sent Events offer a simpler alternative for one-way streaming (notifications, feeds). Both should be supported.

- **Security landscape:** The OWASP Top 10:2025 identifies injection (including XSS) as a persistent risk, particularly relevant for community platforms handling user-generated content. Content Security Policy, input sanitisation, and CSRF protection are non-negotiable.

- **Privacy regulation expansion:** With 20+ US states now having comprehensive privacy laws alongside GDPR, community platforms must implement jurisdiction-aware consent management. ISO 27701:2025 provides a structured framework that maps to both GDPR and CCPA/CPRA requirements.

- **MCP as emerging integration layer:** The Model Context Protocol is an emerging standard that could become a key integration point for AI-native community platforms, enabling AI agents to interact with community data through a standardised protocol rather than bespoke API integrations.

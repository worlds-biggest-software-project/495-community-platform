# Community Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source community platform that unifies forums, groups, events, courses, and paid memberships with intelligent moderation and engagement tools.

Community Platform is an open-source alternative to Circle, Mighty Networks, and Skool for creators, SaaS companies, and professional associations who need an owned-community solution without vendor lock-in or steep per-feature pricing. It combines discussion spaces, live events, course delivery, and membership monetisation in a single self-hostable platform, augmented by AI capabilities that go beyond what any incumbent offers today.

---

## Why Community Platform?

- **Incumbent pricing gates core features.** Circle locks automation and AI behind its $199/month Business tier. Bettermode charges an additional $199/month just for API and webhook access. Heartbeat caps members at 350 on its entry plan. Operators on a budget are forced to choose between features and affordability.
- **No open-source option covers the full stack.** Discourse is the only mature open-source community platform, but it lacks native monetisation, course delivery, live events, and AI moderation. Every other major platform is proprietary SaaS with no self-hosted option.
- **Data portability is absent.** No mainstream commercial platform supports ActivityPub or federated identity. Community operators are locked into vendor-specific data silos with limited export capabilities.
- **AI moderation is superficial or missing.** Skool has no AI features at all. Mighty Networks lacks even basic profanity filters. Where AI exists (Circle, Heartbeat), it is bolted on rather than foundational, and cannot learn community-specific norms.
- **Engagement intelligence does not exist.** No current platform offers churn prediction at the member level or proactive re-engagement automation. Operators discover disengaged members only after they have already left.

---

## Key Features

### Community Spaces & Discussion

- Member profiles with activity history and role-based access controls
- Spaces and channels for topic organisation with public, private, and paid access modes
- Threaded discussion posts, direct messaging, and chat-based communication
- Advanced search across all community content

### Events & Courses

- Event scheduling with RSVP, calendar integration, and live event hosting
- Native course and LMS module co-located with community content
- Certificate and credential issuance for course completion

### Membership & Monetisation

- Paid and free membership gating via Stripe integration
- Multi-currency checkout with support for non-Stripe payment methods (PayPal, SEPA)
- Flat-fee pricing model with no per-member charges for self-hosted operators

### Gamification & Engagement

- Points, levels, leaderboards, and unlockable content
- Streaks, challenges, and badges for sustained participation
- Automation workflows triggered by member lifecycle events (join, course completion, payment, RSVP)

### Developer Platform & Integrations

- REST and GraphQL APIs with webhook support for third-party integrations
- ActivityPub federation for data portability and cross-instance interaction
- SSO via OpenID Connect and OAuth 2.0
- Open analytics export -- no data locked inside proprietary dashboards

---

## AI-Native Advantage

Community Platform is built with AI at its foundation, not bolted on as an upsell. AI-powered moderation detects nuanced policy violations and context-specific toxicity that rule-based filters miss, with configurable escalation to human moderators. A personalised recommendation engine surfaces threads, events, and member connections based on each individual's interests and activity history rather than simple recency. Engagement forecasting identifies members at risk of churn and triggers targeted re-engagement campaigns before they go quiet -- a capability absent from every incumbent platform. AI-assisted onboarding sequences adapt introductory content, connection suggestions, and milestone prompts to each new member's profile and stated goals.

---

## Tech Stack & Deployment

- **Self-hosted, cloud, or hybrid** deployment to give operators full control over their data and infrastructure
- **ActivityPub (W3C)** support for federation and data portability across the Fediverse
- **OpenID Connect / OAuth 2.0** for single sign-on and identity federation
- **WCAG 2.1 AA** accessibility compliance for enterprise and public-sector operators
- **Stripe Connect** integration for membership payments and revenue sharing; additional payment providers on the roadmap

---

## Market Context

The online community platform market is valued at approximately USD 1.4 billion in 2026, growing at over 20% CAGR through 2030, driven by creator economy expansion and enterprise adoption of owned communities. Entry-level plans from incumbents range from USD 19--99/month, with growth-tier plans running USD 100--400/month and enterprise plans reaching USD 5,000+/month. Primary buyers include independent creators converting social media audiences into paid memberships, SaaS companies building customer success communities, professional associations replacing legacy forum software, and brands seeking owned alternatives to Facebook Groups.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context. The features survey recommends adopting a permissive licence (MIT or Apache 2.0) to maximise adoption.

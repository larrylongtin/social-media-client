# Cross-Platform Social Messenger — Development Plan

**App concept:** A single iOS/Android client that composes and sends posts to X (Twitter), Threads, Tumblr, Mastodon, and Bluesky simultaneously; mirrors likes/reposts/reactions back to the originating platform; and applies sentiment analysis to both incoming and outgoing content — warning the user before they post something that reads as bullying or hateful.

---

## 1. Feasibility & Platform API Reality Check (read this first)

Before committing to an architecture, budget for very different access models per platform. This changes as of mid-2026:

| Platform | API access model | Cost/limits | Notes for this app |
|---|---|---|---|
| **X (Twitter)** | Official API moved to **pay-per-use** in Feb 2026, no free tier for new developers. ~$0.015/post created ($0.20 if it contains a link), ~$0.005/post read, 2M reads/month cap before Enterprise pricing (~$42k/mo). Legacy Basic ($200/mo) and Pro ($5,000/mo) subscriptions still exist for grandfathered developers. | Real, ongoing operating cost, scales with users | This is the single biggest cost and risk driver for the product. Budget per-user API cost into your business model from day one, or plan to require users to bring their own X API key/subscription. |
| **Threads** | Meta's official Threads API is publicly available (Graph-API style), free at the time of writing, OAuth-based, rate-limited per app/user. Recently expanded to support more post types, replies, insights, webhooks. | Free but rate-limited; subject to Meta app review | Reasonably solid for posting and reading a user's own content/replies. |
| **Tumblr** | Official REST API (OAuth 1.0a/2.0), free, well-documented, has been stable for years. | Free, generous rate limits | Straightforward integration. |
| **Mastodon** | Open protocol (ActivityPub) + REST API per-instance. No central authority — each server is its own OAuth provider. | Free | Requires "pick your home instance" UX and per-instance app registration flow. |
| **Bluesky** | AT Protocol, open, free, official SDKs available; supports app passwords or OAuth. | Free, generous | Most developer-friendly of the five. |

**Implication for planning:** X API costs and Threads/X's platform policy risk (both have historically restricted or banned third-party clients and aggregator-style posting tools) are the top two risks to validate before writing code. This plan treats that validation as Phase 0.

---

## 2. Product Requirements (from your spec, restated precisely)

1. **Compose-once, post-everywhere:** an original message composed in the app is published to all five connected platforms (with per-platform adaptation: character limits, image handling, content warnings, etc.), only to the platforms the user has connected/enabled.
2. **Reaction sync:** when a user likes, reposts/reblogs/boosts, replies to, or otherwise reacts to an incoming message inside the app, that reaction is sent back to the *originating* platform via that platform's native API (not simulated in-app only).
3. **Inbound sentiment tagging + filtering:** every incoming message (from feeds/timelines the user follows) is scored for sentiment (e.g., positive / neutral / negative, plus finer categories) and the user can filter their unified feed by sentiment.
4. **Outbound sentiment guardrail:** every original message the user is about to send is scored before submission; if it's classified as **bullying** or **hateful**, the user sees an interstitial warning and must explicitly confirm before it's allowed to send.

---

## 3. High-Level Architecture

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│   Mobile App (iOS/Android)   │        │        Backend Platform        │
│  React Native (single         │◄──────►│  API Gateway / BFF             │
│  codebase) or native Swift +   │  REST/  │  Auth & token vault             │
│  Kotlin                        │  WS     │  Fan-out publish service        │
│                                 │        │  Ingestion/aggregation workers   │
│  - Unified compose screen       │        │  Sentiment analysis service      │
│  - Unified feed + filters       │        │  Reaction relay service          │
│  - Account connections          │        │  Notification service            │
│  - Sentiment warning modal      │        │  Postgres + Redis + queue (SQS/  │
└─────────────────────────────┘        │  Kafka) + object storage         │
                                        └──────────────────────────────┘
                                                     │
                     ┌──────────────┬────────────────┼────────────────┬──────────────┐
                     ▼              ▼                ▼                ▼              ▼
                  X API         Threads API      Tumblr API      Mastodon API    Bluesky (AT Proto)
```

**Key architectural decisions to make explicitly (and why):**

- **Mobile framework:** React Native or Flutter for a single codebase across iOS/Android, given five external integrations and a small team — native Swift/Kotlin duplicates effort for integrations that are almost entirely backend-driven anyway. Recommend **React Native** if the team has JS/TS depth, **Flutter** if not.
- **Thin client, fat backend:** the mobile app should not talk to five different platform APIs directly. All platform calls, OAuth token storage, rate-limit handling, and sentiment scoring live server-side. This lets you patch platform API changes without an app store release, keeps API secrets off-device, and centralizes rate-limit/cost management (critical given X's pay-per-use model).
- **Event-driven ingestion:** a background worker per platform polls (or uses webhooks/streaming where available — Bluesky firehose, Mastodon streaming API, Threads webhooks) and normalizes incoming posts into a common internal schema, then runs them through the sentiment pipeline before they hit the user's feed.
- **Fan-out publish service:** a single "create post" internal API takes the normalized post + list of target platforms, adapts content per platform (length, media, alt text, links), and publishes with per-platform retry/backoff logic, returning per-platform success/failure so the UI can show partial failures ("posted to 4/5 platforms — Tumblr failed, retry?").

---

## 4. Data Model (core entities)

- **User** — app account, notification prefs, sentiment filter defaults.
- **PlatformConnection** — user_id, platform, OAuth tokens (encrypted at rest), platform user id/handle, scopes granted, status (active/expired/revoked), Mastodon instance URL where applicable.
- **ComposedMessage** — the canonical outbound message (text, media refs, link, content warning), sentiment_score, sentiment_label, per-platform delivery status, created_at.
- **PlatformPost** — the platform-specific representation actually sent (adapted text, platform post id, permalink) — one row per target platform per ComposedMessage.
- **InboundMessage** — normalized incoming post: source platform, source post id, author, text, media, timestamp, sentiment_score, sentiment_label, raw_payload (for debugging/audit).
- **ReactionEvent** — user_id, inbound_message_id, reaction_type (like/repost/reply/quote), target_platform, status (pending/sent/failed), created_at.
- **SentimentAudit** — log of every sentiment classification made (input hash, model version, score, label, whether user overrode a warning) — needed for QA, model evaluation, and defensibility if users dispute a block.

---

## 5. Sentiment Analysis Design

This is a distinct subsystem from the social integrations and deserves its own design track.

**5.1 Requirements split**
- *Inbound classification:* general sentiment (positive/neutral/negative, or a continuous score) for **filtering** — lower stakes, tolerant of some error, purely a UX convenience.
- *Outbound classification:* a **safety-critical** binary/multi-class detector for "bullying" and "hateful" content — higher stakes, needs much higher precision on the harmful classes, and a human-facing warning/confirmation flow, since false negatives let harmful content ship and false positives frustrate legitimate users (sarcasm, reclaimed language, quoting someone else to criticize them, etc.).

**5.2 Recommended approach**
- Don't try to build a hate-speech/bullying classifier from scratch initially. Options, roughly in order of speed-to-market:
  1. **Managed moderation API** (e.g., a cloud content-moderation/toxicity API) called server-side at compose time and at ingestion time — fastest to ship, offloads model maintenance, but adds per-call cost and a dependency.
  2. **Open-source toxicity/hate-speech model** (e.g., a fine-tuned transformer such as a RoBERTa-based toxicity classifier) self-hosted on the backend — more control, no per-call vendor cost at scale, but you own accuracy, retraining, and infra.
  3. **Hybrid:** general sentiment (positive/neutral/negative) via a lightweight, cheap model or lexicon-based approach for the *filtering* feature; a separate, higher-precision model/API specifically for the *bullying/hateful* outbound gate, since that one has real consequences if wrong.
- **Human-in-the-loop confirmation, not silent blocking:** requirement #4 says "warned," not "blocked" — implement as an interstitial ("This message may read as [hateful/bullying]. Are you sure you want to send it?") with explicit confirm/edit/cancel, and log the override. A hard block invites user frustration and workarounds; a warning respects user agency while adding friction to harmful posts.
- **Explainability for support/appeals:** store which phrases/spans contributed to a flag where the model supports it, so support staff can review disputes.
- **Latency budget:** outbound check must return in well under 1 second to not break the compose flow — plan for a lightweight, fast model or cached/precomputed approach rather than a slow LLM call in the hot path if possible.
- **Localization:** decide upfront whether v1 supports English-only sentiment, since most open models are English-tuned and multilingual hate speech detection is meaningfully harder and worth scoping out or explicitly limiting.
- **Bias & fairness testing:** hateful/bullying classifiers are known to have false-positive skew against certain dialects (e.g., African American Vernacular English) and against reclaimed slurs used by in-group members. Build a bias evaluation set into QA (Section 9) rather than treating this as an afterthought — this is a real user-harm and reputational risk, not just an accuracy metric.

---

## 6. UI/UX Design

**6.1 Core screens**
1. **Onboarding / Account Connections** — connect each of the 5 platforms via OAuth (Mastodon needs an extra "enter your instance" step first); show connection health/status; allow reconnect on token expiry.
2. **Unified Compose** — single text box, character-count-per-platform indicator (X's limit differs from Bluesky/Mastodon/Tumblr), per-platform toggle chips (send to X ✅, Threads ✅, Tumblr ❌...), media attach, link preview, content-warning field (used natively by Mastodon/Tumblr, adapted elsewhere), **sentiment warning modal** on send if flagged.
3. **Unified Feed** — chronologically or algorithmically merged timeline from all connected platforms, each post tagged with a small source-platform badge, sentiment indicator (subtle icon/color, not intrusive), and inline reaction bar (like, repost/boost, reply, share) that fires the reaction-relay flow.
4. **Sentiment Filter Bar** — quick filter chips (All / Positive / Neutral / Negative, or custom thresholds) above the feed; persists as a user preference.
5. **Post Detail / Thread View** — per-platform native thread rendering (Mastodon/Bluesky threads, X replies, Tumblr reblog chains) — this is a real complexity point since thread models differ structurally across platforms.
6. **Delivery Status** — per-message breakdown of where a post succeeded/failed/is pending, with retry.
7. **Settings** — notification prefs, sentiment sensitivity tuning, data/privacy controls, connected-account management, sign-out/revoke.

**6.2 Design considerations**
- Treat "source platform" as a first-class visual element (small consistent badges/colors) so users always know where a post is going or came from — this avoids the classic aggregator confusion of "wait, did that get posted to X or just Bluesky?"
- The sentiment warning modal should be calm and specific, not alarming/red-alert styled for borderline cases — reserve strong visual weight for clear hateful-content flags.
- Design for partial failure everywhere: any given send/reaction action touches 5 independent third-party systems, any of which can be down, rate-limited, or reject the content (e.g., X's cost gate, Mastodon instance-specific rules).

**6.3 Deliverables for this phase**
- Low-fidelity wireframes → clickable Figma prototype → design system (typography, color, spacing tokens) → high-fidelity screens for iOS (Human Interface Guidelines) and Android (Material 3), since platform conventions differ enough that a single design won't feel native on both without adaptation.
- Usability testing on the compose flow and the sentiment-warning interstitial specifically, since these are the highest-friction, highest-stakes interactions.

---

## 7. Backend Logic — Detailed Breakdown

**7.1 Auth & token management**
- OAuth 2.0 (or 1.0a for Tumblr) flows per platform, tokens encrypted at rest (e.g., KMS-backed envelope encryption), automatic refresh where supported, clear re-auth prompts when a platform revokes/expires a token.
- Mastodon requires dynamic app registration per-instance the first time a new instance is seen.

**7.2 Fan-out publishing service**
- Accepts a canonical message + target platform list.
- Per-platform adapter layer: truncates/splits for character limits, converts markdown/rich text as needed, maps content warnings, resizes/transcodes media to each platform's requirements, injects platform-specific metadata (alt text, sensitive-media flags).
- Retries with exponential backoff on transient failures; surfaces permanent failures (auth revoked, content rejected, rate-limited) distinctly to the client.
- Idempotency keys to prevent double-posting on retry.

**7.3 Ingestion/aggregation workers**
- Platform-specific pollers/streams normalize posts into the common `InboundMessage` schema.
- Deduplication (a user might follow the same person cross-posted across platforms).
- Feed a message queue that the sentiment service consumes before messages become visible in the unified feed, so sentiment tagging never blocks ingestion latency from becoming visible to the user for long.

**7.4 Reaction relay service**
- Takes an in-app reaction event, maps it to the correct native API call on the originating platform (like/favorite, repost/boost/retweet, reply, quote), handles platform-specific quirks (e.g., X's paid-tier constraints on write actions, Mastodon boosts vs. favorites being separate calls).
- Reconciles state: if a reaction fails, don't show it as "liked" in the UI (avoid state that lies to the user).

**7.5 Sentiment service**
- Exposed as an internal API (`POST /sentiment/classify`) used both by the outbound compose flow (synchronous, low-latency) and the inbound ingestion pipeline (can be async/batched).
- Versioned model, with a `model_version` recorded per classification for auditability and future re-scoring if the model improves.

**7.6 Rate-limit & cost governor**
- Given X's pay-per-use pricing, implement a per-user and global cost/rate budget with graceful degradation (e.g., queue and batch reads, cache aggressively, warn user if their usage would exceed a free-tier allowance you provide).

**7.7 Notification service**
- Push notifications (APNs/FCM) for delivery failures, new high-sentiment interactions, reconnect-needed alerts.

---

## 8. Third-Party Integration Details (per platform)

| Platform | Auth | Post creation | Reactions | Read/streaming | Key constraints |
|---|---|---|---|---|---|
| X | OAuth 2.0 | POST /2/tweets | POST /2/likes, /2/retweets | Pay-per-use reads, 2M/mo cap | Budget per-call cost; consider requiring power users to link their own X API key/subscription to control your COGS |
| Threads | Meta OAuth | Threads Publishing API (2-step: create container → publish) | Reply/like via API where supported | Webhooks for real-time events | Meta app review required before public launch |
| Tumblr | OAuth 2.0 | POST /v2/blog/{blog}/posts | Like/reblog endpoints | REST polling | Stable, low risk |
| Mastodon | Per-instance OAuth 2.0 | POST /api/v1/statuses | Favourite/reblog endpoints | Streaming API (SSE/WebSocket) per instance | Must handle arbitrary/unknown instances gracefully |
| Bluesky | App password or OAuth | AT Protocol `com.atproto.repo.createRecord` | Like/repost as record creation | Firehose (real-time) or polling | Best DX, plan to build this integration first as your reference implementation |

**Sequencing recommendation:** build Bluesky and Tumblr integrations first (simplest, most stable, free), Mastodon second (protocol complexity but free), Threads third (needs Meta app review lead time), X last (highest cost and policy risk, and the one most likely to have its access model change again — don't let it block your MVP timeline).

---

## 9. Testing Strategy

- **Unit tests:** per-platform adapters (content transformation, character-limit handling), sentiment classification wrapper logic, token refresh logic.
- **Integration tests:** against each platform's sandbox/staging environment where available; contract tests to catch upstream API changes early (platform APIs change without much notice — Mastodon versions vary per instance too).
- **Sentiment model evaluation:** held-out labeled test sets for both general sentiment and the bullying/hateful classifier; track precision/recall separately for the harmful classes (recall matters more here — missing real harassment is worse than an occasional over-warn); dedicated **bias/fairness test set** covering dialect variation, reclaimed language, and quoting-to-criticize scenarios, tracked as a release-blocking metric alongside overall accuracy.
- **End-to-end tests:** mobile UI automation (Detox for React Native, or platform-native XCUITest/Espresso) for the compose→send→multi-platform-delivery flow and the reaction-relay flow, using mocked platform APIs in CI and a small set of real sandbox accounts in a staging environment.
- **Load/chaos testing:** simulate one or more platforms being down/rate-limited/slow and confirm the app degrades gracefully (partial send success, clear error states) rather than failing the whole compose action.
- **Security testing:** OAuth token storage, API secret handling, penetration test before launch given the app holds credentials to five external accounts per user — this is a high-value target.
- **Beta program:** closed TestFlight/Play Internal Testing beta focused specifically on stress-testing the sentiment warning flow with real, varied user-generated text before public launch.

---

## 10. Documentation

- **Architecture Decision Records (ADRs)** for the big calls: mobile framework choice, thin-client vs. fat-client, sentiment model choice, X API cost-handling strategy.
- **API documentation** for the internal backend (OpenAPI/Swagger) for the mobile client and any future integrations.
- **Platform integration runbooks** — one per platform, covering auth setup, known quirks, rate limits, and what to do when that platform's API breaks (they will, periodically).
- **Sentiment model card** — documents training data, known limitations, bias evaluation results, and versioning, both for internal QA and for user-facing transparency ("why was my post flagged?" support content).
- **User-facing help docs** — what sentiment filtering means, why a message was flagged, how to appeal/report a misclassification, privacy explanation of what data is stored.
- **Onboarding docs for engineers** — README, local dev setup, environment variables, seed data.
- **Privacy policy & terms of service** — required given the app handles OAuth tokens and content across five platforms; consult legal review given this is user-generated content moderation territory.

---

## 11. Packaging & Deployment

**Mobile:**
- iOS: Xcode build, App Store Connect listing, TestFlight beta channel, App Store review (note: apps whose core function is auto-posting/cross-posting have faced extra App Store scrutiny — check current App Store Review Guidelines around automated content and spam before submission).
- Android: Play Console listing, internal/closed/open testing tracks, Play Store review.
- CI/CD for mobile: Fastlane (or App Center/Bitrise) for automated build, sign, and store submission from CI.

**Backend:**
- Containerized services (Docker) deployed to a managed orchestrator (e.g., ECS/EKS/Cloud Run) with separate environments (dev/staging/prod).
- Infrastructure as code (Terraform) for reproducibility.
- Secrets management (e.g., AWS Secrets Manager/HashiCorp Vault) for platform API credentials and encryption keys.
- Blue/green or canary deploys for backend services given five live external integrations that must not go down simultaneously.

---

## 12. GitHub / Source Control & Workflow

- **Repo structure:** monorepo (mobile app + backend services + shared types) or polyrepo (mobile / backend / infra split) — monorepo recommended for a small team given the tight coupling between the mobile client and the fan-out API contract.
- **Branching model:** trunk-based development with short-lived feature branches, or GitFlow if release cadence is slower — trunk-based recommended given CI/CD maturity described above.
- **CI:** GitHub Actions running lint, unit tests, integration tests (mocked platform APIs), and mobile build checks on every PR; separate scheduled job to run contract tests against live sandbox platform APIs to catch upstream breakage early.
- **CD:** GitHub Actions → Fastlane for mobile store deploys on tagged releases; GitHub Actions → Terraform/container deploy for backend on merge to main (staging) and on tagged release (prod).
- **Code review:** required PR approvals, CODEOWNERS for the sentiment service and auth/token-handling code specifically given their sensitivity.
- **Issue/project tracking:** GitHub Projects or linked Jira, with labels distinguishing platform-integration bugs (likely to spike unpredictably when a platform changes its API) from core product work.
- **Release management:** semantic versioning, changelog generation, tagged releases mapped to store submissions.

---

## 13. Suggested Phased Roadmap

| Phase | Focus | Rough duration* |
|---|---|---|
| 0 | Validate feasibility: confirm current X API pricing/access, Meta app review requirements for Threads, App Store/Play Store policy check for cross-posting apps; finalize architecture decisions and ADRs | 2–3 weeks |
| 1 | Backend foundation: auth/token vault, data model, Bluesky + Tumblr integrations (simplest, free), basic fan-out publish | 4–6 weeks |
| 2 | Mobile MVP: onboarding, compose screen, unified feed (Bluesky + Tumblr only), reaction relay | 4–6 weeks |
| 3 | Sentiment pipeline: inbound classification + feed filtering, outbound bullying/hateful detector + warning modal, bias evaluation suite | 3–5 weeks (can run partly in parallel with Phase 2) |
| 4 | Add Mastodon integration (multi-instance handling) | 2–3 weeks |
| 5 | Add Threads integration (incl. Meta app review lead time — start this early, review can take weeks) | 3–4 weeks + review wait |
| 6 | Add X integration + cost-governance controls | 2–3 weeks |
| 7 | Hardening: security review/pen test, load testing, accessibility pass, closed beta | 3–4 weeks |
| 8 | Store submission, launch | 1–2 weeks + review wait |

*Durations are rough planning estimates for a small team (2–4 engineers + 1 designer) and should be recalibrated once your actual team size and the Phase 0 findings are known.

---

## 14. Key Risks to Track

1. **X API cost model** — pay-per-use pricing means your COGS scale directly with usage; needs a monetization or usage-cap strategy before wide launch.
2. **Platform policy risk** — X, and to a lesser extent Threads, have histories of restricting third-party clients and auto-cross-posting tools; monitor developer terms of service continuously, not just at launch.
3. **Sentiment model accuracy and bias** — both false positives (frustrated users, potential fairness/discrimination concerns) and false negatives (harmful content getting through) carry real cost; treat this as an ongoing evaluation program, not a one-time build.
4. **Mastodon's fragmented instance landscape** — no single point of integration; instance-specific quirks and moderation rules will generate ongoing support load.
5. **App store policy compliance** — auto-posting/cross-posting apps get extra scrutiny; review current guidelines before investing heavily in the compose/fan-out feature.
6. **Token/credential security** — the app is a high-value target since it holds live credentials to five accounts per user; security review is not optional.

---

*This document is a planning scaffold — treat the cost figures and API details in Section 1 and Section 8 as a snapshot as of mid-2026 and re-verify against each platform's current developer documentation before finalizing budgets, since these terms have changed multiple times in the past two years and are likely to change again.*

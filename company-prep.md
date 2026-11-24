# Company-Specific System Design Prep

How interview style, bar, and focus areas differ by company type. Pair this with the [study tracker](/Users/skat/.cursor/projects/Users-skat-Projects/canvases/system-design-interview-tracker.canvas.tsx) and [worked examples](./worked-examples.md).

---

## Quick reference matrix

| Company type | Interview length | Depth expected | Consistency bar | Distinctive focus |
|--------------|-------------------|----------------|-----------------|-------------------|
| FAANG (L5) | 45–60 min | High on scale + trade-offs | Eventual OK for feeds | Estimation, bottleneck ID |
| FAANG (L6+) | 60 min | Org + evolution + failure | Strong for money/data | Multi-region, operability |
| High-growth startup | 45 min | Pragmatic MVP → scale path | Context-dependent | Speed, simplicity, cost |
| Fintech / payments | 45–60 min | Correctness > scale | **Strong / exactly-once** | Ledger, idempotency, audit |
| Infra / platform | 60 min | Deep on one subsystem | Quorum, CAP nuance | APIs, multi-tenancy, SLOs |
| Ad / ML platform | 45–60 min | Data pipelines + serving | Eventual for features | Feature stores, freshness |
| Gaming / realtime | 45 min | Low latency, state sync | Session consistency | Matchmaking, sharding |

---

## FAANG (Meta, Google, Amazon, Apple, Netflix)

### What they optimize for
- Structured communication under ambiguity
- Quantitative reasoning (QPS, storage, cost)
- Identifying the **bottleneck** before optimizing
- Trade-off articulation with alternatives named

### Company nuances

**Meta**
- Heavy on social graphs: feeds, messaging, notifications
- Expect **fan-out hybrid** discussion for any feed question
- Often probes: "How would you rank?" and "How handle celebrities?"
- Practice: Twitter Feed, Instagram, WhatsApp, Notification System

**Google**
- Search, crawl, index, MapReduce-style batch + serving split
- Strong on **consistency models** and global infrastructure
- May ask open-ended: "Design Google Docs" or "Design YouTube"
- Practice: Web Crawler, YouTube, Dropbox, Metrics Pipeline

**Amazon**
- Customer obsession framing — start from user stories
- Leadership Principles bleed into design: "What's the simplest thing that works?"
- Retail/inventory overlap: Ticketmaster, payment flows
- Practice: URL Shortener, Ticketmaster, Payment System, Notification System

**Apple**
- Privacy, on-device vs cloud trade-offs for some roles
- Media and sync: photos, music, device ecosystem
- Practice: Instagram, Dropbox, YouTube

**Netflix**
- CDN, encoding pipeline, personalization at scale
- Chaos/failure injection mindset — "What breaks first?"
- Practice: YouTube, News Feed Ranking, Metrics Pipeline

### FAANG prep checklist
- [ ] Do estimation in first 10 minutes every mock
- [ ] Name 2 alternatives for every major decision
- [ ] End with "v1 vs v2" without prompting
- [ ] Handle 3 interrupt probes calmly
- [ ] Draw separate read path and write path

### Typical bar
- **L5 / E5:** Solid design that works at stated scale; one good deep dive
- **L6 / E6:** Evolution story, operability, cross-region, explicit failure modes
- **L7+:** Org constraints, build-vs-buy, multi-year roadmap

---

## High-growth startup (Series B–D)

### What they optimize for
- **Pragmatism:** monolith-first, managed services, ship in 6 months
- Can you design something a 5-person team can actually build?
- Cost awareness (RDS vs self-hosted, serverless trade-offs)

### How to adjust your answer
```
Bad:  "We'll shard Postgres across 64 nodes with custom consistent hashing."
Good: "v1: single Postgres + Redis cache handles 10K QPS. At 100K QPS we
       shard by user_id — here's the migration path."
```

### Common questions
- Design our product (they'll describe their domain)
- Rate limiter, notification system, real-time dashboard
- "How would you build this in 3 months?"

### Prep checklist
- [ ] Lead with MVP scope; label "phase 2" explicitly
- [ ] Mention managed services (SQS, CloudFront, Supabase, etc.)
- [ ] Discuss hiring/operational burden of custom infra
- [ ] Show awareness of technical debt trade-offs

---

## Fintech (Stripe, Square, Robinhood, Coinbase)

### Non-negotiables
- **Double-entry ledger** for money movement
- **Idempotency keys** on all write APIs
- **Exactly-once semantics** (or explain why at-least-once + reconciliation)
- Audit trail, PCI scope minimization, encryption

### How interviews differ
- Scale matters less than **correctness under failure**
- Expect: "Network fails after charge but before response — what happens?"
- Strong consistency for balances; eventual for analytics

### Must-know patterns
- Saga pattern for distributed transactions
- Outbox pattern for reliable event publishing
- Reconciliation jobs (expected vs actual balances)
- Optimistic locking on account rows

### Practice focus
- Payment System (worked example #14) — know cold
- Ticketmaster for inventory + overselling (similar to stock trading)
- Rate limiter for fraud prevention angle

### Sample probe answers
**"How prevent double charge?"**
- Idempotency-Key header → unique DB constraint
- State machine: pending → captured → failed (never skip states)
- Reconciliation cron flags mismatches

---

## Infrastructure / platform (Databricks, Cloudflare, MongoDB, Datadog)

### What they optimize for
- Deep expertise on **one layer** (storage engine, proxy, observability)
- Multi-tenancy, noisy neighbor isolation, API design
- SLO/SLI definitions, cardinality limits

### Common questions
- Design a distributed cache / rate limiter / key-value store
- Design a metrics or logging pipeline (high cardinality!)
- Design an API gateway

### Prep checklist
- [ ] Know CAP/PACELC cold — apply to specific component
- [ ] Discuss tenant isolation and fair scheduling
- [ ] Quantify cardinality explosion (metrics labels, log indexes)
- [ ] API versioning and backward compatibility

### Datadog-style twist
- "1M unique metric names per customer" → cardinality bomb
- Mitigations: aggregation at ingest, label limits, sketch algorithms

---

## Ad tech / ML platform (Meta ads, Pinterest, Snap)

### Distinctive topics
- Feature store (online vs offline features)
- Real-time vs batch ranking
- Freshness vs latency trade-off
- A/B testing infrastructure

### Practice focus
- News Feed Ranking (#13)
- Metrics Pipeline (#15) for experiment analysis
- Notification System for campaign delivery

---

## Gaming / realtime (Riot, Epic, Discord)

### Distinctive topics
- Matchmaking, game state sync, tick rate
- UDP vs TCP, authoritative server model
- Regional shards, cross-region latency

### Adjustments
- Latency targets in **milliseconds**, not seconds
- Stateful servers — sticky sessions, graceful drain
- Cheat prevention at protocol level

---

## Interview day playbook by company type

### Before
1. Read company's engineering blog (last 6 months) — note technologies and scale claims
2. Identify product's **core loop** (feed? transaction? search? sync?)
3. Pick 2 worked examples closest to their domain

### During — opening script tweak

**FAANG:** "Let me clarify functional and non-functional requirements before I draw anything."

**Startup:** "I'll propose an MVP scope first, then discuss how we'd evolve at 10× scale."

**Fintech:** "Before scale — let me walk through the consistency and correctness requirements for money movement."

### During — when stuck
1. Return to requirements
2. Name the bottleneck aloud
3. Ask interviewer: "Would you prefer I optimize for latency or consistency here?"

---

## 2-week cram plan by target

### Target: Meta / social
| Day | Focus |
|-----|-------|
| 1–2 | Twitter Feed + News Feed Ranking |
| 3 | Instagram + Notification System |
| 4 | WhatsApp + Rate Limiter |
| 5–6 | Mocks with celebrity/hot-key probes |
| 7 | Review weak SCALE-R dimensions |

### Target: Google / search & media
| Day | Focus |
|-----|-------|
| 1–2 | Web Crawler + URL Shortener |
| 3–4 | YouTube + Dropbox |
| 5 | Metrics Pipeline |
| 6–7 | Mocks with indexing/sharding probes |

### Target: Fintech
| Day | Focus |
|-----|-------|
| 1–2 | Payment System (full depth) |
| 3 | Ticketmaster (inventory locking) |
| 4 | Rate Limiter (fraud) |
| 5–7 | Mocks with failure/reconciliation probes |

### Target: Startup
| Day | Focus |
|-----|-------|
| 1 | URL Shortener + Rate Limiter (MVP framing) |
| 2 | Notification System |
| 3–4 | Domain-specific (research their product) |
| 5–7 | Mocks with "3-month MVP" constraint |

---

## Red flags by company type

| Type | Instant fail |
|------|--------------|
| FAANG | No estimation; can't deep dive when prompted |
| Startup | Over-engineered microservices day 1 |
| Fintech | Eventual consistency for account balances |
| Infra | Can't explain failure recovery |
| All | Silent for 5+ minutes; ignores interviewer input |

---

## Resources per company

- **Meta:** "Scaling Memcache at Facebook", "TAO", "Messing with HTTP for fun and profit"
- **Google:** Bigtable, Spanner, MapReduce papers (know concepts, not proofs)
- **Amazon:** Dynamo paper, AWS architecture blogs
- **Netflix:** CDN Open Connect, Chaos Engineering principles
- **Stripe:** "Idempotency" docs, ledger engineering posts

Log mock sessions and track company-specific weak areas in the [study tracker canvas](/Users/skat/.cursor/projects/Users-skat-Projects/canvases/system-design-interview-tracker.canvas.tsx).

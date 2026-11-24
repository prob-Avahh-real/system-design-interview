# 15 System Design Worked Examples (SCALE-R)

Full walkthroughs for the most common interview questions. Use with the [mock interviewer prompt](./mock-interviewer.md) and [study tracker](/Users/skat/.cursor/projects/Users-skat-Projects/canvases/system-design-interview-tracker.canvas.tsx).

Each example follows **SCALE-R**: Scope → Capacity → API & Data → Layout → Explore → Reliability.

---

## 1. URL Shortener (bit.ly)

### S — Scope

**Functional**
- Shorten long URL → unique short code
- Redirect short URL → original (HTTP 301/302)
- Optional: custom alias, expiration, click analytics
- Optional: authenticated users manage links

**Non-functional**
- 100:1 read/write ratio (redirects dominate)
- Redirect latency p99 < 100ms
- 99.99% availability for redirects
- Short codes: 6–8 chars, URL-safe

**Out of scope v1:** User accounts, link editing, A/B redirect rules

### C — Capacity

- 100M new URLs/month → ~40 URLs/sec write peak (~400 with burst factor)
- 40 × 100 = **4K redirect reads/sec** average; **40K/sec** peak
- Storage: 100M × 500B ≈ **50 GB/year** (negligible)
- Cache: hot 20% of links → ~20M × 500B ≈ **10 GB** Redis

### A — API & Data

```
POST /api/v1/urls   { long_url, custom_alias?, ttl_days? }
  → { short_code, short_url, expires_at }

GET  /{short_code}  → 301 Location: long_url

GET  /api/v1/urls/{code}/stats  → { clicks, last_clicked }
```

**Schema (SQL — strong uniqueness on short_code)**
```
urls(id PK, short_code UNIQUE, long_url, user_id?, created_at, expires_at)
clicks(id, short_code, ts, referrer, country)  -- async analytics
```

**Short code generation**
- v1: Base62 encode of auto-increment ID (fixed length via zero-pad)
- v2: Hash(long_url + salt) → retry on collision; counter-based for guaranteed uniqueness

### L — Layout

```
Client → CDN/LB → Redirect Service → Redis (cache-aside) → PostgreSQL
                 ↘ Analytics Queue → Click Aggregator → clicks table
Create path: API → ID generator (DB sequence or Snowflake) → DB write → cache warm
```

### E — Deep dives

**1. Redirect hot path**
- Cache-aside: GET code from Redis; miss → DB → populate Redis (TTL 24h)
- 301 vs 302: 301 for permanent (SEO, browser cache); 302 if expiration/custom rules

**2. Collision handling**
- DB UNIQUE on short_code
- Custom alias: INSERT with conflict → 409 to client

**3. Scale reads**
- Redis cluster; CDN edge cache for top 1% codes
- Read replicas if cache miss rate grows

### R — Reliability

- Idempotent creates via client request ID
- Rate limit creates per IP (prevent abuse)
- Monitor: redirect p99, cache hit rate, create error rate
- v2: Geo DNS, multi-region read replicas, bloom filter for "does code exist" before DB

---

## 2. Twitter Feed

### S — Scope

**Functional**
- Post tweet (text, media refs)
- Follow users
- Home timeline: tweets from followees, reverse chronological
- User profile timeline

**Non-functional**
- 300M DAU, 500M tweets/day
- Timeline read p99 < 500ms
- Eventual consistency OK for feed (seconds delay acceptable)

**Out of scope v1:** Trending, search, DMs, ranking beyond recency

### C — Capacity

- 500M tweets / 86400 ≈ **6K writes/sec** avg → **60K/sec** peak
- Timeline read: 300M DAU × 10 reads/day ≈ **35K/sec** avg → **350K/sec** peak
- Tweet storage: 500M × 300B/day ≈ **150 GB/day** raw
- Fan-out: avg 200 followers → 6K × 200 = **1.2M fan-out writes/sec** (problem!)

### A — API & Data

```
POST /v1/tweets        { text, media_ids[] }
POST /v1/follow        { followee_id }
GET  /v1/timeline/home?cursor=&limit=20
GET  /v1/timeline/user/{id}?cursor=
```

**Schema**
```
users(id, handle, ...)
tweets(id, user_id, text, created_at)          -- sharded by user_id
follows(follower_id, followee_id, created_at)  -- or graph DB
timeline(user_id, tweet_id, rank_ts)           -- precomputed fan-out
```

### L — Layout

```
Write: API → Tweet Service → Tweet DB
                          → Fan-out Worker → timeline tables (per follower)

Read: API → Timeline Service → Redis (user timeline cache) → timeline DB / merge
```

### E — Deep dives

**1. Fan-out on write vs read**
- **Fan-out on write:** Low read latency; bad for celebrities (10M followers = 10M writes)
- **Fan-out on read:** Cheap writes; slow reads (merge 500 followees at read time)
- **Hybrid:** Fan-out on write if followers < 10K; else merge-on-read for celebrity tweets only

**2. Celebrity problem**
- Threshold: 10K followers
- Celebrity tweet: write to `celebrity_tweets` stream; readers merge at fetch time
- Precompute for "normal" users only

**3. Timeline storage**
- `timeline:user_id` → sorted set in Redis (tweet_id, timestamp) capped at 800 entries
- Backing store: Cassandra partition key = user_id, cluster key = timestamp

### R — Reliability

- Tweet create idempotent via client UUID
- Fan-out worker: at-least-once → dedupe by (user_id, tweet_id)
- Monitor: fan-out lag, timeline p99, celebrity merge latency
- v2: Ranking layer, "catch-up" sync for offline users, media pipeline

---

## 3. Instagram (Photo Sharing)

### S — Scope

**Functional**
- Upload photo/video, add caption, tags
- Follow users, view home feed and explore
- Like, comment (basic)

**Non-functional**
- 500M DAU, heavy media (photos ~2MB avg)
- Feed p99 < 1s (includes CDN image load)
- Durability: never lose uploaded media

**Out of scope v1:** Stories, Reels transcoding pipeline, DMs

### C — Capacity

- 100M uploads/day → **1.2K uploads/sec** avg → **12K/sec** peak
- 500M DAU × 20 feed loads × 5 images ≈ **50B image views/day** → CDN handles bulk
- Metadata: 100M posts × 1KB/day ≈ **100 GB/day**
- Object storage: 100M × 2MB ≈ **200 TB/day** (tiered/lifecycle to cold storage)

### A — API & Data

```
POST /v1/media/upload     → presigned S3 URL flow
POST /v1/posts            { media_id, caption, tags[] }
GET  /v1/feed/home?cursor=
GET  /v1/users/{id}/posts
POST /v1/posts/{id}/like
```

**Schema**
```
posts(id, user_id, media_id, caption, created_at)
media(id, s3_key, width, height, status)   -- processing pipeline
follows, likes, comments (standard)
feed_cache(user_id, post_ids[])          -- same fan-out as Twitter
```

### L — Layout

```
Upload: Client → presigned URL → S3 → SQS → Thumbnail/ML worker → update media status
Feed:   API → Feed Service (Twitter hybrid fan-out) → post metadata → CDN URLs in response
```

### E — Deep dives

**1. Upload pipeline**
- Client gets presigned PUT URL; uploads direct to S3 (offload bandwidth from app servers)
- SQS message triggers: thumbnail (128px, 640px), virus scan, optional content moderation
- Post only visible when media.status = ready

**2. Feed with media**
- Feed returns post IDs + metadata; client loads images from CDN
- Edge cache images; URL includes content hash for immutable caching

**3. Explore/discovery**
- Offline: compute engagement scores batch job
- Online: serve precomputed candidate sets per user cohort; rank lightly at request time

### R — Reliability

- Multipart upload for large video; resume support
- S3 cross-region replication for durability
- Monitor: upload success rate, processing lag, CDN hit ratio
- v2: Stories (TTL objects), video transcoding (HLS), full-text search on captions

---

## 4. WhatsApp / Messenger

### S — Scope

**Functional**
- 1:1 and group messaging (text, media)
- Delivery/read receipts, online presence
- Message history sync across devices

**Non-functional**
- 2B users, 100B messages/day
- Delivery latency p99 < 500ms
- Ordering per chat session; at-least-once delivery acceptable with dedup

**Out of scope v1:** E2E encryption implementation details, voice/video

### C — Capacity

- 100B / 86400 ≈ **1.2M messages/sec** avg → **12M/sec** peak (regionally sharded)
- Avg message 200B → **24 TB/day** (compressed, replicated)
- Presence: 500M concurrent connections globally (WebSocket/long-poll)

### A — API & Data

```
WebSocket /long-poll gateway
  SEND { chat_id, client_msg_id, payload, ts }
  ACK  { server_msg_id, client_msg_id }
  SYNC { chat_id, since_server_msg_id }

GET /v1/chats/{id}/messages?before=&limit=50
```

**Schema (per chat partition)**
```
messages(chat_id, server_msg_id, sender_id, payload, ts)  -- Cassandra, PK=chat_id
user_inbox(user_id, chat_id, last_msg_ts, unread_count)
presence(user_id, status, last_seen)  -- Redis TTL
```

### L — Layout

```
Client ↔ Connection Gateway (sticky) ↔ Message Router
                                      ↔ Message Store (Cassandra)
                                      ↔ Push Notification Service (offline)
                                      ↔ Presence Service (Redis)
Group: message → fan-out to online members via pub/sub channel per user
```

### E — Deep dives

**1. Message routing**
- Hash user_id → gateway shard; maintain connection registry in Redis
- Online: push via WebSocket; offline: store + APNS/FCM push

**2. Ordering & dedup**
- server_msg_id = Snowflake (time-ordered per shard)
- Client sends client_msg_id; server dedupes on (chat_id, client_msg_id)
- Per-chat ordering guaranteed; cross-chat not required

**3. Group chat**
- Small groups (<256): fan-out write to each member's inbox index
- Large groups: read from shared chat log only; no per-user fan-out

### R — Reliability

- Store-then-forward: persist before ACK to sender
- Retry with exponential backoff; DLQ for poison messages
- Monitor: delivery lag, connection count, push failure rate
- v2: E2E keys (Signal protocol), multi-device sync protocol, media CDN

---

## 5. YouTube (Video Platform)

### S — Scope

**Functional**
- Upload video, transcode to multiple resolutions
- Stream playback with adaptive bitrate
- Search videos, channels, subscriptions feed

**Non-functional**
- 2B users, 500 hours uploaded/minute
- Playback start p99 < 2s
- Upload durability: never lose source file

**Out of scope v1:** Live streaming, comments moderation ML

### C — Capacity

- 500 hr/min × 60 × 1GB/hr ≈ **500 GB/min upload** → need distributed upload + queue
- Views: 5B views/day → **58K/sec** → CDN serves 95%+ bytes
- Transcoding: CPU-heavy; async workers scale independently
- Metadata DB: billions of videos × 2KB

### A — API & Data

```
POST /v1/videos/upload/init   → { upload_id, chunk_urls[] }
POST /v1/videos/upload/complete { upload_id, title, ... }
GET  /v1/videos/{id}/manifest  → HLS/DASH manifest URLs
GET  /v1/search?q=&page=
```

**Schema**
```
videos(id, channel_id, title, status, created_at)
video_renditions(video_id, resolution, s3_key, bitrate)
upload_sessions(id, user_id, chunks_completed)
```

### L — Layout

```
Upload: Client → chunked upload → Raw S3 → SQS → Transcode fleet → Renditions S3/CDN
         Metadata API → PostgreSQL (video record status=processing→ready)

Play: Client → CDN edge (manifest + segments) ← origin on miss
      Search: Elasticsearch index (title, tags, channel)
```

### E — Deep dives

**1. Transcoding pipeline**
- Priority queue: popular creators / paid tier first
- DAG of jobs: 360p, 720p, 1080p, thumbnail, audio-only
- Spot/preemptible instances for batch transcode

**2. Adaptive streaming**
- HLS: manifest lists variant playlists; client switches by bandwidth
- Segment size ~2–6 sec; CDN cache segments aggressively (immutable URLs)

**3. Upload reliability**
- Resumable chunked upload; checksum per chunk
- Complete call triggers transcode; idempotent on upload_id

### R — Reliability

- Raw + transcoded in separate S3 buckets, cross-region replication
- Monitor: transcode queue depth, playback error rate, CDN origin load
- v2: Live HLS (lower latency segment size), recommendation feed, copyright Content ID

---

## 6. Uber (Ride Hailing)

### S — Scope

**Functional**
- Rider requests ride (pickup, dropoff)
- Match nearby available driver
- Track trip status, ETA, fare estimate
- Driver accept/decline, navigation updates

**Non-functional**
- 100M monthly active, peak in city rush hours
- Match latency p99 < 5s
- Location updates every 3–5 sec from active drivers

**Out of scope v1:** Surge pricing algorithm details, payments settlement

### C — Capacity

- 20M rides/day → **230 rides/sec** avg → **2K/sec** peak per mega-city
- 1M active drivers × 1 update/4sec → **250K location writes/sec** globally
- Geospatial queries: "drivers within 2km" at high QPS during peak

### A — API & Data

```
POST /v1/rides/request     { pickup_latlng, dropoff_latlng, tier }
GET  /v1/rides/{id}/status
POST /v1/drivers/location  { lat, lng, heading }   -- high frequency
POST /v1/rides/{id}/accept  (driver)
```

**Schema**
```
rides(id, rider_id, driver_id?, status, pickup, dropoff, fare, created_at)
drivers(id, status, current_location, rating)
-- Geospatial index: Redis GEO / PostGIS / custom grid
```

### L — Layout

```
Location: Driver app → Location Ingestion → Redis GEO (per city cell) + Kafka archive
Match:    Request → Matching Service → query GEO radius → rank by ETA → notify drivers
Trip:     State machine service → PostgreSQL + push updates to rider/driver
```

### E — Deep dives

**1. Geospatial index**
- City divided into geohash grids (e.g., precision 6 ≈ 1.2km)
- Driver location → update cell membership in Redis GEO
- Match: query neighboring cells for available drivers

**2. Matching algorithm**
- v1: nearest available driver, broadcast to top 3
- Timeout 15s → expand radius, retry
- Avoid double-assign: optimistic lock on driver status (available → en_route)

**3. ETA calculation**
- Precomputed road graph (OSRM/Google Directions API)
- Cache common origin-dest pairs; real-time traffic overlay batch-updated

### R — Reliability

- Ride state machine: requested → matched → in_progress → completed (no illegal transitions)
- Idempotent ride request via client token
- Monitor: match time p99, cancel rate, location pipeline lag
- v2: Surge by cell demand/supply, pool/shared rides, multi-stop

---

## 7. Rate Limiter

### S — Scope

**Functional**
- Limit requests per user/API key/IP (e.g., 100 req/min)
- Return 429 with Retry-After when exceeded
- Distributed: consistent limits across N servers

**Non-functional**
- 1M checks/sec
- p99 latency < 5ms
- Approximate limit OK (not billing-critical)

**Out of scope v1:** Dynamic limits per plan tier UI, analytics dashboard

### C — Capacity

- 1M checks/sec, each check ~50 bytes state → **50 MB/sec** memory churn if naive
- Redis cluster: 10 nodes handles millions of ops/sec

### A — API & Data

Internal middleware:
```
allow(key, limit, window_sec) → { allowed: bool, remaining, reset_at }
```

No persistent DB; Redis keys:
```
rate:{key}:{window_bucket} → counter (INCR + EXPIRE)
```

### L — Layout

```
API Gateway → Rate Limiter Sidecar → Redis Cluster
              (token bucket or sliding window logic)
```

### E — Deep dives

**1. Token bucket vs sliding window**
- **Fixed window:** simple INCR per minute bucket; burst at window boundary
- **Sliding window log:** accurate but memory-heavy
- **Sliding window counter:** hybrid — previous + current bucket weighted (recommended)

**2. Distributed consistency**
- Redis INCR atomic; all gateway nodes share Redis
- Local cache of "definitely over limit" for 100ms to reduce Redis load (optional, adds inaccuracy)

**3. Hot keys**
- Global limit on celebrity API key → shard counter: `rate:{key}:{shard_id}`
- Aggregate shards on read (rare) or divide limit across shards

### R — Reliability

- Fail-open vs fail-closed: product decision (fail-open preserves availability)
- Monitor: 429 rate, Redis latency, false positive rate
- v2: Per-endpoint weights, burst allowance, admin override API

---

## 8. Web Crawler

### S — Scope

**Functional**
- Seed URLs → fetch pages → extract links → repeat
- Dedupe URLs, respect robots.txt and politeness delay
- Store page content for search index

**Non-functional**
- Crawl 10B pages/month
- Politeness: max 1 req/sec per domain
- Freshness: re-crawl popular pages more often

**Out of scope v1:** JavaScript rendering, PageRank computation

### C — Capacity

- 10B / 30 days ≈ **4K pages/sec** sustained
- Avg page 50KB → **200 MB/sec** ingress
- URL frontier: billions of URLs in queue

### A — API & Data

Internal system; optional ops API:
```
POST /crawl/seeds  { urls[] }
GET  /crawl/stats
```

**Schema**
```
url_frontier(url_hash PK, url, priority, domain, next_fetch_at, depth)
fetched_pages(url_hash, content_s3_key, fetched_at, checksum)
domain_politeness(domain, last_fetch_at, delay_ms)
robots_cache(domain, rules, cached_at)
```

### L — Layout

```
Scheduler → Priority Queue (per domain buckets) → Fetcher pool → DNS + HTTP
                                              → Content Store (S3)
                                              → Link Extractor → URL Frontier (dedup bloom + DB)
                                              → Indexer → Search index
```

### E — Deep dives

**1. Politeness**
- Separate queue per domain; scheduler picks domain where `now > last_fetch + delay`
- robots.txt cached 24h; disallow paths filtered before fetch

**2. Dedup**
- Bloom filter for "probably seen" (in-memory + periodic rebuild)
- Confirmed dedup in `url_hash` DB (MD5 of normalized URL)

**3. Priority frontier**
- Priority score: PageRank proxy, freshness, depth from seed
- Infinite crawl: never mark "done"; downgrade stale pages

### R — Reliability

- Exponential backoff on 5xx/timeout; max retries then tombstone
- Monitor: fetch rate, error rate by domain, frontier depth
- v2: Headless browser farm for JS sites, distributed politeness via domain locks

---

## 9. Notification System

### S — Scope

**Functional**
- Send notifications: push, email, SMS
- User channel preferences
- Template rendering, batching/digest

**Non-functional**
- 1B notifications/day
- Delivery within 30s for push; minutes OK for email
- At-least-once delivery with dedup

**Out of scope v1:** In-app notification center UI, A/B on copy

### C — Capacity

- 1B / 86400 ≈ **12K/sec** avg → **120K/sec** peak (event spikes)
- Push: vendor APIs (FCM/APNS) rate limits per app
- Email: batch through SES/SendGrid

### A — API & Data

```
POST /v1/notify   { user_id, template_id, payload, channels[]?, idempotency_key }
GET  /v1/users/{id}/preferences
PUT  /v1/users/{id}/preferences  { email: bool, push: bool, ... }
```

**Schema**
```
notification_events(id, user_id, template, payload, status, idempotency_key UNIQUE)
user_preferences(user_id, channel, enabled)
delivery_log(event_id, channel, status, provider_id)
```

### L — Layout

```
API → Validate prefs → Kafka topic (by channel) → Workers:
       Push Worker → FCM/APNS
       Email Worker → SES
       SMS Worker → Twilio
     → Delivery log DB
```

### E — Deep dives

**1. Idempotency**
- `idempotency_key` UNIQUE constraint; duplicate POST returns 200 with same event_id
- Workers check delivery_log before send

**2. Priority & spikes**
- Kafka partitions by channel; priority topics for OTP vs marketing
- Rate limit marketing separately; OTP bypasses digest

**3. Digest batching**
- Accumulate low-priority in Redis sorted set by (user_id, send_at)
- Cron flushes hourly digest email

### R — Reliability

- Retry 3× with backoff; DLQ for permanent failures
- Monitor: delivery latency by channel, provider error rate, queue lag
- v2: User timezone-aware scheduling, unsubscribe compliance (CAN-SPAM)

---

## 10. Distributed Cache (Redis-like)

### S — Scope

**Functional**
- get(key), set(key, value, ttl), delete(key)
- TTL expiration, LRU eviction under memory cap
- Horizontal scale across nodes

**Non-functional**
- 1M ops/sec, p99 < 5ms
- Eventual consistency across replicas OK for cache
- 99.9% availability

**Out of scope v1:** Persistence guarantees (AOF/RDB), pub/sub, complex data structures

### C — Capacity

- 1M ops/sec; avg value 1KB, 100M keys → **100 GB** working set
- 10 nodes × 16GB RAM each with replication

### A — API & Data

```
GET /cache/{key}
PUT /cache/{key}  { value, ttl_sec }
DELETE /cache/{key}
```

In-memory hash table per node; key metadata: expiry timestamp, access time (for LRU)

### L — Layout

```
Client → Proxy (consistent hashing) → Cache Node (primary)
                                    → Replica (async replication)
         Miss → origin DB (cache-aside at application layer)
```

### E — Deep dives

**1. Consistent hashing**
- Ring with virtual nodes (150 per physical node) for even distribution
- Add node: only ~1/N keys remapped

**2. Eviction**
- Maxmemory policy: allkeys-lru (approx LRU via random sample of 5 keys)
- Lazy expire on read + active expire cron

**3. Cache stampede**
- Lock on miss: first request regenerates; others wait or serve stale
- Jitter TTL to prevent simultaneous expiry

### R — Reliability

- Single node failure: replica promoted; rebalance keys
- Monitor: hit rate, evictions/sec, replication lag
- v2: Cross-region cache, hot key detection + local L1 on app server

---

## 11. Dropbox (File Sync)

### S — Scope

**Functional**
- Upload/download files, folder hierarchy
- Sync across devices (detect changes, delta sync)
- Share links, basic conflict handling

**Non-functional**
- 500M users, avg 10GB stored
- Sync notification within seconds of change
- Durability: 11 nines on blob storage

**Out of scope v1:** Real-time co-editing, fine-grained ACLs

### C — Capacity

- 500M × 10GB = **5 EB** total (S3 tiering)
- Metadata ops: list dir, sync poll — **100K/sec** peak
- Block size 4MB → delta sync reduces bandwidth

### A — API & Data

```
POST /v1/files/upload_block   { hash, data }   -- content-addressed
POST /v1/files/commit         { path, block_hashes[], revision }
GET  /v1/files/{path}
GET  /v1/sync/cursor          → changes since cursor
```

**Schema**
```
namespaces(user_id, root_inode)
inodes(id, parent_id, name, file_hash?, revision, is_dir)
blocks(hash PK, s3_key, size)   -- dedup by content hash
```

### L — Layout

```
Client → Sync API → Metadata DB (PostgreSQL)
                 → Block Storage (S3, deduplicated by hash)
Long poll / WebSocket for sync cursor updates
```

### E — Deep dives

**1. Delta sync**
- File split into 4MB blocks; roll hash (rsync-style) detects changed blocks only
- Upload only new block hashes; server dedupes if hash exists

**2. Conflict resolution**
- Per-file revision counter; commit fails if base_revision mismatch
- Conflict: keep both as "conflicted copy" — user resolves manually (v1)

**3. Dedup**
- Content-addressed storage: same hash → same S3 object (saves storage across users)

### R — Reliability

- Metadata transaction for commit (atomic inode update)
- Monitor: sync latency, block upload failures, storage growth
- v2: Operational transform for docs, team folders, encryption at rest per user key

---

## 12. Ticketmaster (Flash Sale)

### S — Scope

**Functional**
- List events, seat inventory
- Purchase flow: hold seat → payment → confirm
- Prevent overselling (1000 seats, max 1000 sold)

**Non-functional**
- Flash sale: 500K users, 10K seats, opens at fixed time
- Peak **500K requests in 60 sec**
- Strong consistency on inventory; no double-book

**Out of scope v1:** Secondary market, dynamic pricing

### C — Capacity

- 500K/sec spike at onsale → queue/virtual waiting room essential
- Inventory DB writes: only successful purchases (~10K) — manageable
- Read-heavy before sale: event details on CDN

### A — API & Data

```
GET  /v1/events/{id}
POST /v1/events/{id}/queue/join     → { queue_token, position }
POST /v1/events/{id}/hold           { seat_ids[], queue_token }
POST /v1/orders/confirm             { hold_id, payment_token }
```

**Schema**
```
events(id, name, onsale_at)
seats(event_id, seat_id, status: available|held|sold, version)
holds(id, user_id, seat_ids[], expires_at)
orders(id, user_id, seat_ids[], payment_status)
```

### L — Layout

```
CDN → Event read API
Sale: Queue Service (Redis FIFO token) → Admitted users only → Inventory Service
Inventory: PostgreSQL row locks OR Redis decrement with Lua atomic script
Payment → Payment gateway → confirm order → mark seats sold
```

### E — Deep dives

**1. Virtual waiting room**
- Before onsale: join queue → random shuffle or FIFO → admit N users/sec
- Token required on hold API; prevents thundering herd on inventory DB

**2. Inventory locking**
- `UPDATE seats SET status='held', version=version+1 WHERE seat_id=? AND status='available' AND version=?`
- Optimistic concurrency; 0 rows updated → seat taken
- Hold TTL 5 min; cron releases expired holds

**3. Alternative: Redis atomic**
- `DECR available_count` in Lua if > 0; assign seat from pre-shuffled pool
- Faster but less flexible seat selection

### R — Reliability

- Idempotent confirm with idempotency_key
- Monitor: queue depth, hold expiry rate, oversell checks (audit sum sold vs capacity)
- v2: Seat map realtime WebSocket, fraud detection, geo-fair queue

---

## 13. News Feed Ranking

### S — Scope

**Functional**
- Rank posts for home feed by relevance (not just recency)
- Features: engagement history, affinity, post type, freshness
- Serve ranked feed p99 < 500ms

**Non-functional**
- 300M DAU, 500 candidates per user per request
- Model refresh: daily batch + near-real-time feature updates
- Eventual consistency on scores OK (minutes)

**Out of scope v1:** Training infrastructure, A/B framework

### C — Capacity

- 300M × 10 feeds/day = **35K ranking requests/sec** peak
- 500 candidates × 100 features × 4B = **200 KB** feature vector per request (too big — must precompute)

### A — API & Data

```
GET /v1/feed/ranked?cursor=&limit=20

Internal:
  rank(user_id, candidate_post_ids[]) → scored ordered list
```

**Stores**
```
post_features(post_id, embedding, engagement_rate, age, ...)  -- offline computed
user_features(user_id, interests[], affinities{})             -- offline + streaming
candidate_generator(user_id) → ~500 post IDs                  -- inverted index + social graph
```

### L — Layout

```
Request → Candidate Generation (graph + popular + recent)
       → Feature Fetch (Redis user features + post feature store)
       → Ranking Model (linear model / small NN in serving layer)
       → Filter (dedupe, diversity, seen posts bloom filter)
       → Return top 20
Offline: Spark jobs compute features → feature store (Feast/custom)
```

### E — Deep dives

**1. Two-stage retrieval**
- Stage 1: cheap retrieval (500 from millions) — inverted index, social graph BFS
- Stage 2: expensive ranker on 500 only

**2. Feature freshness**
- Static features: batch daily (user country, account age)
- Dynamic: streaming update (last 1h clicks) in Redis
- Trade-off: staleness vs serving latency

**3. Diversity & exploration**
- Penalize same author 3× in a row
- ε-greedy: 5% slots for random exploration posts

### R — Reliability

- Fallback to reverse-chron if ranker timeout (500ms budget)
- Monitor: ranking latency, null feature rate, engagement lift metrics
- v2: Deep learning ranker, real-time personalization, feedback loop from clicks

---

## 14. Payment System

### S — Scope

**Functional**
- Charge customer, refund, transfer to merchant
- Idempotent API, audit ledger
- Reconciliation with payment processor

**Non-functional**
- 10K TPS peak
- **Strong consistency** on balances; exactly-once money movement
- PCI: never store raw card numbers (tokenize via Stripe)

**Out of scope v1:** Multi-currency FX, subscription billing

### C — Capacity

- 10K TPS; ledger append-only → **864M entries/day** max theoretical (actual far lower)
- Storage: ledger rows small (~200B); partitioned by account_id

### A — API & Data

```
POST /v1/charges   Idempotency-Key: uuid
  { amount_cents, currency, source_token, merchant_id }

POST /v1/refunds   { charge_id, amount_cents?, idempotency_key }
GET  /v1/balances/{account_id}
```

**Schema (double-entry ledger)**
```
accounts(id, owner_id, balance_cents, version)
ledger_entries(id, txn_id, account_id, debit, credit, created_at)
transactions(id, idempotency_key UNIQUE, status, processor_ref)
-- Invariant: sum(debits) = sum(credits) per txn_id
```

### L — Layout

```
API → Payment Service → BEGIN TX
                      → Lock account row (SELECT FOR UPDATE)
                      → Insert ledger entries (debit customer, credit merchant)
                      → Call Stripe API
                      → COMMIT or ROLLBACK
     → Reconciliation Worker (nightly: ledger vs Stripe reports)
```

### E — Deep dives

**1. Idempotency**
- Idempotency-Key stored UNIQUE; replay returns same transaction result
- State machine: created → processing → succeeded | failed

**2. Failure handling**
- Stripe timeout after debit recorded: status=processing; reconciliation resolves
- Never double-charge: check idempotency_key before any processor call

**3. Ledger design**
- Append-only entries; balance = SUM(credits) - SUM(debits) or cached with version
- Audit: every penny traceable to txn_id

### R — Reliability

- PCI scope: tokenization only; SAQ A compliance
- Monitor: failed charges, reconciliation mismatches, ledger invariant checks
- v2: Multi-region with CRDB/Spanner, payout scheduling, dispute workflow

---

## 15. Metrics / Logging Pipeline

### S — Scope

**Functional**
- Apps emit metrics (counters, gauges, histograms) and logs
- Query dashboards, alerts on thresholds
- Retention: metrics 13 months, logs 30 days

**Non-functional**
- 10M metrics/sec ingestion
- Query p99 < 3s for dashboard (last 1h, 100 series)
- High cardinality risk (unbounded label combinations)

**Out of scope v1:** Trace correlation, ML anomaly detection

### C — Capacity

- 10M points/sec × 16 bytes ≈ **160 MB/sec** raw ingest
- 1B log lines/day × 500B ≈ **500 TB/day** (compression ~10× → 50 TB)

### A — API & Data

```
POST /v1/metrics/write   (Prometheus remote write / StatsD)
POST /v1/logs/ingest     { stream, timestamp, level, message, labels }

Query:
  GET /v1/query?promql=rate(http_requests[5m])
  GET /v1/logs/search?filter=level:ERROR&since=1h
```

**Storage**
```
Metrics: time-series DB (Cortex/M3/ClickHouse) — downsampled rollups
Logs: indexed by (timestamp, service) in Elasticsearch/S3 + index
Metadata: label → series ID mapping with cardinality limits
```

### L — Layout

```
App → Agent (local batch/compress) → Kafka → Ingestion Workers
                                            → Metrics TSDB (partitions by metric name hash)
                                            → Log indexer → ES hot, S3 cold
Query API → TSDB / ES → merge → dashboard
Alert Scheduler → eval rules every 1m → PagerDuty
```

### E — Deep dives

**1. Cardinality explosion**
- Limit labels per metric (max 20); block high-cardinality labels (user_id) at ingest
- Aggregations at write time for known dashboards

**2. Downsampling**
- Raw 10s resolution 7 days → 1min rollup 90 days → 1h rollup 13 months
- Queries auto-select resolution by time range

**3. Log pipeline cost**
- Hot/warm/cold tiers: ES 7 days → S3 searchable via Athena 30 days → glacier
- Structured JSON logs; index only ERROR by default

### R — Reliability

- At-least-once ingest; dedupe by (series_id, timestamp) at compaction
- Monitor: ingest lag, dropped series count, query latency
- v2: Exemplars linking metrics→traces, per-tenant quotas

---

## How to use these examples

1. **Read one** end-to-end without notes (30 min)
2. **Whiteboard from memory** (25 min timed)
3. **Run mock** with [mock-interviewer.md](./mock-interviewer.md)
4. **Log scores** in the [study tracker canvas](/Users/skat/.cursor/projects/Users-skat-Projects/canvases/system-design-interview-tracker.canvas.tsx)
5. **Spaced rep** the topic cards 24h, 3d, 7d later

Cross-reference [company-prep.md](./company-prep.md) to emphasize the right deep dives for your target company.

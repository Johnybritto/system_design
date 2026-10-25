# 5. Twitter / X-Like Social Feed — System Design (Interview Version)

> **Goal:** Reuse our earlier Customer-Facing API foundation, but add a **social graph, asynchronous timeline fanout, caching, and a hybrid celebrity-feed strategy**. This is a simplified interview design for text posts; media upload/storage, recommendations, ads, direct messages, trending and full-text search are outside scope. Scale and SLO values are illustrative assumptions, not statements about the actual X infrastructure.

## Step 1 — Requirements

### Functional Requirements (4)

1. A user can publish and retrieve a tweet/post.
2. A user can follow or unfollow other users.
3. A user can retrieve a home feed containing posts by followed accounts, ordered by recency (simplified ranking).
4. A user can delete their own tweet; deleted and blocked/private content must not be served to unauthorized users.

### Non-Functional Requirements (4)

1. **Availability:** 99.95% API availability across three AZs.
2. **Performance:** Feed retrieval p95 below 300 ms; post creation p95 below 200 ms (interview targets).
3. **Scalability:** Assume millions of users, a read-heavy workload, 10,000 post writes/s and 100,000 feed reads/s at peak for sizing discussions.
4. **Consistency and reliability:** Newly published posts persist durably; **eventual consistency is acceptable for home-feed propagation**, but tweet deletion/visibility and authorization are checked before display.

**Critical interview distinction:** A successful `POST /tweets` stores the tweet. It does **not** promise that every follower's feed is already updated.

---

## Step 2 — API Contracts (Short)

| User operation | API |
|---|---|
| Publish tweet | `POST /v1/tweets` |
| Read one tweet | `GET /v1/tweets/{tweet_id}` |
| Home timeline | `GET /v1/feed?cursor=...&limit=20` |
| Follow account | `POST /v1/users/{user_id}/follow` |
| Unfollow | `DELETE /v1/users/{user_id}/follow` |
| Delete own tweet | `DELETE /v1/tweets/{tweet_id}` |

**Write request**
~~~http
POST /v1/tweets
Authorization: Bearer <access_token>
Idempotency-Key: post-req-001
~~~
~~~json
{"text":"Hello world!"}
~~~

**Response — 201 Created**
~~~json
{"tweet_id":"T1001","author_id":"U123","status":"PUBLISHED"}
~~~

**Read feed response — 200**
~~~json
{"tweets":[{"tweet_id":"T1001","author_id":"U123","text":"Hello world!"}],"next_cursor":"..."}
~~~

Author identity comes from the validated JWT, not the caller's chosen author_id. Use **cursor pagination**, not OFFSET over an enormous timeline.

---

## Step 3 — Core Entities and Database Indexes

| Entity | Important fields | Why |
|---|---|---|
| **Users** | user_id PK, handle UNIQUE, profile data, visibility | Account / author identity |
| **Follows** | follower_id, following_id, created_at; composite PK (follower_id, following_id) | Who a user follows |
| **Tweets** | tweet_id PK, author_id FK, text, created_at, deleted_at, visibility | Durable canonical post data |
| **Outbox** | event_id PK, tweet_id, event_type, payload, published_at | Reliable tweet-created / deleted events |
| **HomeTimeline** | Redis sorted set per user: tweet_id scored by publish time | Quickly serve recent feed candidates |

**Indexes and read patterns**
- `Tweets(tweet_id)` primary key: retrieve a specific tweet.
- `Tweets(author_id, created_at DESC)`: load an author's recent tweets, including celebrity posts for fanout-on-read.
- `Follows(follower_id, following_id)`: list accounts that Alice follows.
- `Follows(following_id, follower_id)`: find Bob's followers for fanout-on-write.
- Index unpublished Outbox rows to keep the publisher efficient.
- Redis key `feed:user:U123`: bounded sorted set of recent tweet IDs; the feed is **reconstructable** from canonical posts and follows.

**Note:** At large scale, shard the followers/following edges and tweet rows according to access patterns; indexing alone is not sufficient forever.

---

## Step 4 — Database Selection: Why?

**Initial relational source of truth: PostgreSQL Multi-AZ**
- Users, Follows, Tweets and Outbox fit a consistent relational model for a simplified design.
- Supports ACID transaction: **insert Tweet + Outbox event together**.
- Supports indexes and constraints on author/profile/follow records.
- Strong reads from the writer for critical post state; replicas can handle stale-tolerant discovery and profiles.

**Redis / ElastiCache: hot home-timeline cache**
- Stores **tweet IDs**, not the only copy of the actual tweets.
- Sorted sets support efficient recent-item retrieval.
- Redis state can be rebuilt if lost; it must not be the sole durable record of tweets.
- A feed read assembles tweet objects by fetching referenced IDs from tweet storage / a hot-tweet cache, then applies visibility checks.

**Kafka: durable event stream**
- Receives `TweetCreated` and `TweetDeleted` events and allows asynchronous fanout workers to replay after transient failures.
- Appropriate when tweet events feed several consumers (timelines, notifications, analytics).
- For a smaller application, SQS could also work; Kafka is chosen here for **multiple consumers and replay**.

**At much larger scale:** Split tweet storage and feed timelines into horizontally partitioned stores (e.g., DynamoDB/Cassandra-style keyed stores) rather than relying on one PostgreSQL primary or treating Redis as an unlimited durable database. That is an **evolution**, not necessary to memorize for the first diagram.

**Why not use SQL JOINs on every GET /feed?** Expensive at high read volume: joining follows to a large tweet dataset and ranking it per request strains database CPU, indexes and I/O. Materialize recent feed candidates asynchronously for ordinary users.

---

## Step 5 — High-Level Design

### Write path

~~~text
Client → API Gateway → EKS Tweet Service
       → PostgreSQL (Tweet + Outbox, atomic COMMIT) → HTTP 201
       → Outbox Publisher → Kafka → Fanout Workers
       → Redis home timelines of ordinary followers
~~~

### Read path

~~~text
Client → API Gateway → EKS Feed Service
       → Redis home timeline tweet IDs
       + fetch recent celebrity tweets on demand (if following celebrity)
       → merge / sort / filter by current visibility → load tweet details
       → return paginated feed
~~~

**Unchanged infrastructure:** Route 53 DNS, API Gateway authentication/throttling, VPC Link + internal NLB, EKS ingress, TLS, multi-AZ deployment, monitoring and horizontal Pod scaling.

---

## Step 6 — Deep Dive: The Most Important Decisions

### A. Exact tweet-creation flow

1. Alice calls `POST /v1/tweets`; API Gateway validates JWT and applies coarse throttling.
2. Tweet Service validates body/visibility, derives author_id from JWT and enforces an idempotency key.
3. In **one PostgreSQL transaction**, insert `Tweets` and `Outbox(TweetCreated)`; commit.
4. Return `201 Created` immediately. Followers' timelines are **not** part of the transaction.
5. Outbox Publisher reads committed unpublished events, publishes to Kafka and marks progress. It might publish duplicates after a crash; consumers must handle them.
6. Kafka consumers look up followers in batches and update ordinary followers' Redis timeline caches using idempotent tweet-ID inserts.
7. Metrics track **fanout lag** and the time until a newly posted tweet becomes visible in followers' feeds.

### B. Why do we need the Feed Service?

Without a dedicated feed service, every feed request could require:
- Look up everyone Alice follows.
- Fetch their newest tweets.
- Merge, filter, sort and paginate a potentially enormous set.

Instead, **Feed Service** performs fast timeline assembly:
1. Read recent tweet IDs from Alice's Redis home timeline.
2. Pull in recent posts from celebrities Alice follows that were deliberately not pre-fanned-out.
3. Merge / deduplicate / sort candidates, apply up-to-date visibility/deletion/block filtering.
4. Fetch tweet details; return first 20 plus a cursor.

**Responsibility split:** Tweet Service owns durable posts; Follow Service owns relationships; Feed Service owns fast timeline generation and assembly.

### C. Fanout-on-write vs Fanout-on-read vs Hybrid

| Strategy | How it works | Suitable for | Main cost |
|---|---|---|---|
| **Fanout-on-write (push)** | Workers place Bob's new tweet ID into followers' timeline caches | Ordinary accounts | Many background writes |
| **Fanout-on-read (pull)** | Feed Service fetches Bob's recent tweets when follower opens feed | Celebrity / very-high-follower accounts | More read-time computation |
| **Hybrid (our choice)** | Push normal users' tweets; pull celebrity tweets at feed-read time | Realistic mixed social graph | Merge/dedup complexity |

**Celebrity example:** Bob has 10 million followers. Pushing one tweet to 10 million individual timeline keys instantly creates enormous write amplification and lag. Instead, store his tweet **once** in the tweet database; the Feed Service fetches his recent tweets when a follower requests their feed.

**Normal user example:** Bob has 500 followers. Fanout workers can cheaply add his new tweet ID into 500 followers' Redis timelines for faster reads.

### D. Read scaling and hot content
- Redis holds recent timeline ID lists and possibly popular tweet detail caches, with bounded size/TTL.
- Read replicas can help stale-tolerant queries; never assume a read replica is immediately current after post commit.
- Pagination prevents giant response bodies. Cache hot tweets to avoid repeatedly hitting the database.
- At global scale, move large/high-throughput tweet and timeline records to appropriately sharded stores.

### E. Failures, consistency, and privacy
- **Kafka down:** Tweet and Outbox are committed; publisher retries. Tweet exists even if follower feeds temporarily lag.
- **Fanout Worker crash:** Reprocess Kafka events. Updating a Redis sorted set by tweet ID is idempotent; replay/backfill after cache loss.
- **Redis down:** Feed reads may be degraded; apply **bounded** fallback / lazy rebuild from follows and posts. Do not stampede PostgreSQL.
- **Follow or unfollow:** Update `Follows`; optionally invalidate/rebuild feed candidates. **Enforce unfollow/block/private visibility at read time** instead of assuming the cache has been perfectly cleaned.
- **Tweet delete:** Mark tweet deleted, publish `TweetDeleted`, asynchronously evict from caches, and **filter deleted tweets synchronously on reads**.
- **Celebrity goes viral:** Push-vs-pull threshold, rate-limits, batch processing and caching prevent fanout overload.
- **Ordering:** Per-author recent order can be preserved; a global total order across every user's feed is usually unnecessary. Use stable cursor order (timestamp + tweet ID) rather than unstable offset.

---

## Step 7 — Why Each Component?

| Component | Why we use it |
|---|---|
| Route 53 | Resolve public API hostname |
| API Gateway | JWT, throttling and central API access |
| VPC Link / NLB / EKS Ingress | Private routing to Kubernetes applications |
| EKS Tweet Service | Validate/write/delete canonical tweets |
| EKS Follow Service | Maintain follower/following relationships |
| EKS Feed Service | Merge precomputed feed IDs with celebrity tweets; filter/rank/page |
| PostgreSQL Multi-AZ | Durable Users, Tweets, Follows, Outbox and relational constraints |
| Outbox Publisher | Reliable DB → Kafka event transfer |
| Kafka | Replayable `TweetCreated` / `TweetDeleted` event stream |
| Fanout Worker Pods | Asynchronously populate ordinary followers' feeds |
| Redis / ElastiCache | Fast bounded home-timeline and hot-tweet caching |
| PostgreSQL read replicas (optional) | Scale stale-tolerant tweet/profile reads |
| HPA + Karpenter | Scale API, fanout workers and EKS node capacity |
| CloudWatch / Grafana / Tracing | API latency, cache hit rate, fanout lag and Kafka consumer health |
| OAuth/OIDC + TLS/KMS | Authenticated identities and encrypted data |

---

## Step 8 — Failure, Remediation, Verification

| Failure | Impact | Remediation | Verify |
|---|---|---|---|
| PostgreSQL unavailable | Tweet creation fails | Multi-AZ failover, safe reconnect and bounded idempotent retry | Successful durable posts |
| Duplicate POST | Duplicate tweet | Idempotency key + DB uniqueness | One tweet per request key |
| Kafka unavailable | Feed propagation delayed | Durable Outbox, retry publisher | Outbox lag and publish failures decline |
| Fanout worker crashes | Missing/delayed feed entries | Kafka replay, idempotent sorted-set writes | Consumer lag and propagation delay decline |
| Redis cache loss | Slow/missing cached feeds | Bounded fallback, rebuild caches, rate-limit requests | Feed latency recovers |
| Celebrity post | Massive fanout write amplification | Hybrid fanout: pull celebrity posts at read time | Worker pressure remains bounded |
| Deleted tweet still cached | Privacy/correctness issue | Read-time tombstone/visibility filter + async invalidation | Deleted tweet never returned |
| Followers DB overloaded | Slow follow/fanout | Batch reads, appropriate indexes, partition large graphs | Query/lock latencies normalize |
| One AZ lost | Lost API/worker capacity | Multi-AZ Pods, spare node headroom, DB HA | SLO and fanout lag recover |

**Monitor:** POST/GET p95/p99, read/write RPS, 5xx, Redis hit ratio/evictions, Kafka consumer lag, Outbox lag, feed freshness, PostgreSQL CPU/connections, hot celebrity spikes, and visibility/deletion correctness.

---

## Interview Summary

> I would build Tweet, Follow and Feed Services on EKS behind API Gateway. PostgreSQL stores users, relationships and canonical tweets; each tweet write atomically includes an Outbox event. The publisher sends events to Kafka, and fanout workers populate Redis timeline caches for normal accounts. For celebrities with millions of followers, I switch to fanout-on-read to avoid huge write amplification. The Feed Service merges cached normal tweets with pulled celebrity tweets, filters visibility and returns a cursor-paginated feed.

**Biggest difference from a CRUD API:** **hybrid fanout plus fast home-timeline reads**, not just storing and retrieving tweets.

---

## Easy-to-Remember Final Diagram

~~~mermaid
flowchart TD
  C["Client"] --> G["API Gateway"]
  G --> T["EKS Tweet Service"]
  T --> DB["PostgreSQL — Tweet + Outbox"]
  DB --> K["Outbox Publisher → Kafka"]
  K --> W["Fanout Workers — normal users"]
  W --> R["Redis — Home Timeline IDs"]

  G --> F["EKS Feed Service"]
  R --> F
  DB -->|"Celebrity posts — pull on read"| F
  F -->|"Merge, visibility check, paginate"| C
~~~

**Memory lines:**
- **Write:** Client → Gateway → Tweet Service → DB + Outbox → Kafka → Fanout Worker → Redis.
- **Read:** Client → Gateway → Feed Service → Redis + celebrity tweets → merge/filter → feed.
- **Key decision:** **Push for normal users, pull for celebrities — Hybrid Fanout.**

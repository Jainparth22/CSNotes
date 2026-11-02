# Placement Study Notes — Part 5: System Design (HLD)

*Companion to Parts 1 to 4. Algorithms (rate limiter, consistent hashing, ID generation, circuit breaker) have Python and C++ code; architectures use ASCII diagrams. Numbers are back-of-the-envelope: state assumptions, do not memorize answers.*

## 0. How to use these notes

Each fundamentals topic has: **what it is**, **how it works**, **when to use it**, **trade-offs**, **interview answer**, **loopholes** (the follow-up traps). Each practice system follows one template: requirements → estimates → API → data model → architecture and flow → deep dive → scaling and failure → security → observability → trade-offs.

**Contents:** H0 Interview framework · H1 Requirements · H2 Client-server and API design · H3 Latency vs throughput · H4 Capacity estimation · H5 Scaling and stateless services · H6 Load balancing · H7 Caching and Redis · H8 CDN · H9 Databases at scale (selection, replication, sharding, partitioning, indexing, pooling) · H10 Consistency, CAP, PACELC · H11 Consistent hashing · H12 Unique IDs · H13 Object storage and uploads · H14 Messaging: queues, Kafka, pub-sub · H15 REST, gRPC, GraphQL, API gateway · H16 Monolith vs microservices, service discovery, health checks · H17 WebSockets, SSE, polling · H18 Background jobs · H19 Rate limiting · H20 Idempotency · H21 Retries, timeouts, backoff, jitter · H22 Circuit breakers and fault tolerance · H23 Redundancy, HA, DR · H24 Saga and transactional outbox · H25 Distributed locks and leader election · H26 Authentication, authorization, OAuth 2.0, JWT · H27 Security and secrets · H28 Concurrency control · H29 Observability and SLOs · H30 Bottleneck analysis · S1 to S10 HLD systems (URL shortener, Rate limiter, Notification service, Chat, File storage, News feed, E-commerce, Job queue/scheduler, Appointment booking, Distributed cache) · Revision sheet.

---

## H0. The HLD interview framework

**45-minute plan:** requirements 5 · estimation 5 · API and data model 8 · high-level design 10 · deep dives 12 · failures, trade-offs, wrap-up 5.

1. **Clarify requirements.** Functional (what it does) and non-functional (scale, latency, availability, consistency, durability, security). State what is **out of scope**.
2. **Estimate.** Daily users, QPS (average and peak), read:write ratio, storage per year, bandwidth, cache size. Let the numbers justify design choices (a 1,000 QPS system needs no sharding; a 1,000,000 QPS one does).
3. **Define APIs** (endpoints, parameters, responses, auth, pagination, idempotency).
4. **Data model:** entities, relationships, access patterns, SQL vs NoSQL, indexes, partition key.
5. **Draw the high-level architecture:** clients, CDN, load balancer, API gateway, stateless services, cache, databases, queues, workers, object storage. Walk one **write flow** and one **read flow** end to end.
6. **Deep dive** into the one or two hard parts (ID generation, ranking, fan-out, consistency, hot keys).
7. **Scale and harden:** replication, sharding, caching, async processing, retries, rate limiting, failure scenarios, security, monitoring.
8. **Trade-offs:** name alternatives and why you chose this one; mention what you would build later.

**Habits that score:** think aloud; ask before assuming; start simple (single server) then evolve; label every box with its purpose; always say the **failure mode** of each component; quantify; admit unknowns; end with bottlenecks and improvements.

---

## H1. Functional and non-functional requirements

**Functional:** features visible to users ("shorten a URL", "send a message", "upload a file"), the actors (user, admin, system), core use cases, and **out of scope**.

**Non-functional requirements (NFRs): the ones that shape architecture**

| NFR | Question to ask | Typical design consequence |
| --- | --- | --- |
| **Scale** | users, QPS, data volume, growth | sharding, caching, queues |
| **Latency** | target p95/p99 for key calls | cache, CDN, async, regional deployment |
| **Availability** | 99.9% or 99.99%? | redundancy, failover, multi-AZ/region |
| **Consistency** | can users see stale data? | strong vs eventual, quorum, single leader |
| **Durability** | can we ever lose data? | replication, WAL, backups, multi-AZ |
| **Security and privacy** | PII, auth, encryption, compliance | TLS, encryption at rest, IAM, audit |
| **Cost and operability** | budget, team size | managed services vs build |

**Availability table:** 99% ≈ 3.65 days downtime/year; 99.9% ("three nines") ≈ 8.76 hours; 99.99% ≈ 52.6 minutes; 99.999% ≈ 5.26 minutes. Components in **series** multiply (0.999 x 0.999 = 0.998); **parallel redundancy** gives `1 - (1 - a)^n` (two 99% nodes in parallel = 99.99%).

**Questions to always ask:** read-heavy or write-heavy? how many users and requests? global or one region? real-time needs? data retention? mobile or web? strict ordering or uniqueness requirements? acceptable data staleness?

---

## H2. Client-server architecture and API design

```
Client (browser/mobile) --HTTPS--> DNS -> CDN/LB -> API gateway -> stateless app servers -> cache / DB / queue
```

The client sends requests; servers hold the business logic and data; the protocol is usually HTTP(S) with JSON (REST) or binary (gRPC). Details of HTTP methods, status codes, REST design, cookies/JWT are in Part 2 (C6 to C11).

**API design checklist for HLD answers**

- Resource-oriented endpoints with the right verbs: `POST /v1/urls`, `GET /v1/urls/{code}`, `DELETE /v1/urls/{code}`.
- **Request and response examples** (JSON) with status codes (`201` + `Location`, `404`, `409`, `429` + `Retry-After`).
- **Pagination:** cursor-based for large or changing lists (`?limit=20&cursor=...`); offset only for small static data.
- **Idempotency** for non-idempotent writes (`Idempotency-Key` header), **versioning** (`/v1`), **authentication** (Bearer token / API key), **rate limit headers**, consistent error format, input validation, timeouts.
- **Sync vs async:** if work takes long, return `202 Accepted` with a job id and let the client poll or receive a webhook.

```
POST /v1/urls               { "long_url": "https://example.com/a/very/long/path", "custom_alias": "promo1", "ttl_days": 365 }
201 Created                 { "short_url": "https://sho.rt/promo1", "expires_at": "2027-10-28T00:00:00Z" }
409 Conflict                { "error": "alias_taken" }
```

---

## H3. Latency vs throughput

- **Latency:** time for one request (ms). **Throughput:** requests (or bytes) processed per second (QPS, MB/s). They are different: a pipeline can have high throughput and high latency (batching), and adding servers raises throughput but not single-request latency.
- **Percentiles, not averages:** p50 (median), p95, p99. **Tail latency matters** because a page that makes 50 backend calls will almost always hit at least one p99 (probability `1 - 0.99^50 ≈ 40%`). Fan-out amplifies tail latency; mitigate with hedged requests, timeouts, caching, and fewer sequential hops.
- **Little's law:** `L = λ x W` (average concurrent requests = arrival rate x average time in system). 1,000 requests/s at 200 ms means about **200** requests in flight; that bounds threads/connections you need.
- **Queueing:** latency explodes as utilization approaches 100% (waiting time grows like `1 / (1 - utilization)`); target 60 to 70% utilization at peak, leave headroom.
- **Bandwidth vs latency:** a fast pipe does not remove the speed-of-light round trip (cross-continent \~100+ ms), so move data and compute **closer to users** (CDN, regional deployments).

**Latency numbers to remember (order of magnitude)**

| Operation | Time |
| --- | --- |
| L1 cache / main memory reference | 1 ns / 100 ns |
| Redis/Memcached get (same datacenter) | 0.2 to 1 ms |
| SSD random read | 0.1 ms |
| Read 1 MB sequentially from memory / SSD / HDD | 0.25 ms / 1 ms / 20 ms |
| Round trip within a datacenter | 0.5 ms |
| Disk seek (HDD) | 10 ms |
| Round trip California to Netherlands | 150 ms |
| Typical DB indexed query | 1 to 10 ms |

**Loopholes:** "we need low latency" is meaningless without a percentile and a number; caching improves both, async queues improve user-visible latency but not end-to-end completion time; averages hide the tail.

---

## H4. Capacity estimation (back-of-the-envelope)

**Memorize**

- 1 day = 86,400 s ≈ **10^5 s**; 1 month ≈ 2.5 x 10^6 s; 1 year ≈ 3 x 10^7 s.
- 1 million requests/day ≈ **12 QPS**; 100 million/day ≈ 1,200 QPS; 1 billion/day ≈ 12,000 QPS.
- Peak QPS ≈ 2x to 5x average (state your factor).
- Sizes: char 1 B (ASCII); int 4 B; long/timestamp 8 B; UUID 16 B; a tweet-sized record \~300 B to 1 KB; image \~200 KB to 2 MB; short video \~ 10 to 50 MB; 1 KB = 10^3, 1 MB = 10^6, 1 GB = 10^9, 1 TB = 10^12 (use powers of ten for estimates).
- Rule of thumb throughput per instance: a typical web server handles roughly 1k to 10k simple requests/s; a single relational DB node 1k to 10k QPS (depending on query cost); Redis \~100k ops/s; Kafka broker hundreds of MB/s.

**Formulas**

```
QPS_avg     = daily_requests / 86,400
QPS_peak    = QPS_avg x peak_factor
Writes/day  = DAU x actions_per_user
Storage/yr  = writes/day x bytes_per_record x 365 x replication_factor
Bandwidth   = QPS x average payload size
Cache size  = (daily unique hot data) x 20%   (80/20 rule: 20% of keys serve 80% of reads)
Servers     = peak_QPS / QPS_per_server (+ headroom 30 to 50% + redundancy N+1)
```

**Worked example: a microblogging service** (assumptions stated aloud)

```
300M monthly users, 100M DAU. Each posts 2 times/day and reads 200 posts/day (read:write = 100:1).
Writes/day = 200M            -> 200M / 86,400 = 2,300 QPS average, ~7,000 QPS at 3x peak
Reads/day  = 20B             -> 230,000 QPS average, ~700,000 QPS peak   => heavy caching + CDN + fan-out design
Storage: 200M posts x 300 B text = 60 GB/day = 22 TB/year; with 3x replication 66 TB/yr (text only).
         10% have a 500 KB image -> 20M x 0.5 MB = 10 TB/day  => object storage + CDN, not the database.
Bandwidth out (reads): 700k QPS x 1 KB text = 700 MB/s peak (text) -> CDN offload essential.
Cache: 20% of 1 day's posts read = ~12 GB of text for hot items: fits in a small Redis cluster.
```

```python
def estimate(dau, writes_per_user, read_write_ratio, bytes_per_write, peak=3, replicas=3):
    writes = dau * writes_per_user; qps_w = writes / 86400
    return dict(write_qps=round(qps_w), peak_write_qps=round(qps_w * peak),
                read_qps=round(qps_w * read_write_ratio), storage_tb_year=writes * bytes_per_write * 365 * replicas / 1e12)
print(estimate(100e6, 2, 100, 300))
```

**Loopholes:** do not chase precision (round aggressively); always separate **average** and **peak**; remember replication and indexes multiply storage; remember read amplification (one feed read touches many items); sanity-check against known system sizes.

---

## H5. Vertical vs horizontal scaling and stateless services

|  | Vertical (scale up) | Horizontal (scale out) |
| --- | --- | --- |
| How | bigger machine (CPU, RAM, disk) | more machines |
| Limit | hardware ceiling, expensive at the top | practically unlimited |
| Availability | single point of failure | redundancy built in |
| Complexity | none | load balancing, distributed state, consistency |
| Use | databases early on, quick fix | web/app tier, caches, sharded stores |

**Stateless services:** the server keeps **no per-user state between requests**; any request can go to any instance. State lives in shared stores (database, Redis, object storage) or in the client (token). Benefits: trivial horizontal scaling and autoscaling, easy rolling deploys, instant failover, easy replacement of crashed nodes.

```
                  +--> app server 1 --+
client -> LB ---->|--> app server 2 --|--> Redis (sessions/cache) , DB
                  +--> app server N --+            any server can serve any request
```

**How to make a service stateless:** store sessions in Redis or use signed tokens (JWT); upload files to object storage (not local disk); keep caches shared or accept per-instance caches that are only optimizations; use external config; make background work go through a queue; avoid sticky sessions. (The twelve-factor app principles.)

**Scaling order of operations:** optimize queries and add indexes → add caching → scale the stateless tier horizontally → read replicas → async/queue heavy work → CDN for static/media → partition/shard the database → split into services → multi-region.

**Loopholes:** stateless does not mean "no state anywhere"; local in-memory caches create inconsistency between instances; sticky sessions break even load distribution and failover; autoscaling reacts slowly (warm-up, cold caches), so pre-scale for known peaks.

---

## H6. Load balancing

**Purpose:** distribute traffic across servers for scalability, availability and zero-downtime deploys; remove unhealthy nodes.

|  | L4 (transport) | L7 (application) |
| --- | --- | --- |
| Sees | IP, port, TCP/UDP | HTTP: URL, headers, cookies |
| Speed | very fast, simple | slower, smarter |
| Features | connection-level balancing | path/host routing, TLS termination, caching, auth, rewrites, sticky by cookie |
| Examples | AWS NLB, HAProxy (tcp mode), LVS | AWS ALB, Nginx, Envoy, HAProxy (http) |

**Algorithms**

- **Round robin / weighted round robin:** simple; weights for different capacities.
- **Least connections / least response time:** adapt to uneven request cost.
- **IP hash / consistent hashing:** same client or key goes to the same server (cache locality, session affinity); consistent hashing minimizes remapping when nodes change (H11).
- **Random with two choices (power of two choices):** pick two at random, send to the less loaded; near-optimal and cheap.

**Health checks:** active (the LB probes `/health` every few seconds; unhealthy after N failures, healthy after M successes) and passive (watch real traffic errors). **Connection draining** finishes in-flight requests before removing a node.

**Other facts:** the **LB must not be a single point of failure** (active-passive pair with a virtual IP, or active-active behind DNS/anycast); **TLS termination** at the LB offloads crypto; **global load balancing** with GeoDNS/anycast routes users to the nearest region; **session affinity** (sticky sessions) via cookie/IP hash is an anti-pattern for stateless design.

```nginx
upstream api {
    least_conn;
    server 10.0.0.1:8080 max_fails=3 fail_timeout=10s;
    server 10.0.0.2:8080 weight=2;
}
server { listen 443 ssl; location / { proxy_pass http://api; } }
```

**Loopholes:** health check that only tests the process, not the dependencies (a node with a dead database looks healthy; use deep checks sparingly); thundering herd when a restarted node receives full traffic instantly (slow start); LB misconfigured timeouts shorter than backend timeouts; long-lived connections (WebSockets) skew least-connections; one hot key still lands on one server.

---

## H7. Caching, Redis and cache invalidation

**Why:** reduce latency, protect the database, absorb read spikes. **Where caches live:** browser, CDN, reverse proxy (Nginx/Varnish), application memory (local), **distributed cache (Redis/Memcached)**, database buffer pool.

**Caching patterns**

| Pattern | Read path | Write path | Notes |
| --- | --- | --- | --- |
| **Cache-aside (lazy loading)** | app checks cache; on miss reads DB and fills cache | app writes DB, then **deletes** the cache key | most common; stale window small; cold start misses |
| **Read-through** | cache library loads from DB on miss | n/a | simpler app code, cache owns loading |
| **Write-through** | n/a | write cache and DB together (synchronous) | consistent, higher write latency |
| **Write-behind (write-back)** | n/a | write cache, flush to DB asynchronously | fast writes, **risk of data loss** and complexity |
| **Refresh-ahead** | cache refreshes hot keys before expiry | n/a | avoids expiry latency spikes |

**Eviction policies:** LRU (default choice), LFU, FIFO, random, TTL-based. **TTL** bounds staleness and memory.

**Invalidation (the hard problem):** (1) **TTL** only; (2) **delete on write** (cache-aside); (3) **versioned keys** (`user:42:v7`) so old entries just stop being read; (4) **event-driven invalidation** (a change event from the DB/CDC/queue purges keys); (5) tag-based purge. Prefer **delete over update** on writes to avoid races that store stale data.

**Classic cache problems and fixes**

| Problem | Cause | Fixes |
| --- | --- | --- |
| **Cache stampede / dogpile** | a hot key expires and thousands of requests hit the DB | per-key lock or **single-flight** (one loader), serve stale while revalidating, jittered TTLs, refresh-ahead |
| **Cache penetration** | requests for keys that do not exist bypass the cache every time | cache **null results** with short TTL, **Bloom filter** in front, input validation |
| **Cache avalanche** | many keys expire together or the cache cluster dies | **TTL jitter** (add random seconds), replication, circuit breakers, gradual warm-up |
| **Hot key** | one key gets enormous traffic | local in-process cache, key replication/sharding (`key#1..#N`), CDN |
| **Inconsistency** | read-miss race: reader loads old DB value while a writer updates, then reader sets old value | delete after write, short TTL, versioning, CAS on cache write, accept eventual consistency |

**Redis essentials**

- In-memory **data structure server**, single-threaded command execution (no lock contention; fast, sub-millisecond), optional persistence: **RDB** snapshots and **AOF** append-only log.
- Structures: **string** (counters `INCR`, cache blobs), **hash** (objects), **list** (queues), **set** (unique members), **sorted set** (leaderboards, rate-limit windows, delayed jobs by score), **bitmap**, **HyperLogLog** (approximate unique counts), **stream** (log/queue with consumer groups), **geo**, **pub/sub**.
- Scaling: **replication** (primary + replicas), **Sentinel** (automatic failover), **Cluster** (16,384 hash slots sharded across nodes).
- Atomic operations: single commands, `MULTI/EXEC`, **Lua scripts**, `SET key val NX EX 30` (lock with expiry).
- Uses: cache, session store, rate limiting, leaderboards, distributed lock (with caveats, H25), queues, pub/sub, counters.

```python
import json, random, threading, time
locks, locks_guard = {}, threading.Lock()
def get_user(uid, cache, db, ttl=300):                  # cache-aside with a per-key lock (single-flight) and TTL jitter
    key = f'user:{uid}'
    if (v := cache.get(key)) is not None: return json.loads(v) if v != 'NULL' else None
    with locks_guard: lock = locks.setdefault(key, threading.Lock())
    with lock:                                          # only one thread reloads; the others wait, then hit the cache
        if (v := cache.get(key)) is not None: return json.loads(v) if v != 'NULL' else None
        row = db.fetch_user(uid)
        if row is None: cache.set(key, 'NULL', ex=30)    # negative caching stops penetration
        else: cache.set(key, json.dumps(row), ex=ttl + random.randint(0, 60))   # jitter prevents avalanche
        return row
def update_user(uid, data, cache, db):
    db.update_user(uid, data)                           # write DB first ...
    cache.delete(f'user:{uid}')                         # ... then delete the cache entry (next read repopulates)
```

```cpp
// C++: cache-aside with a mutex per key shard to prevent stampedes
std::optional<std::string> getUser(const std::string& id, Cache& cache, Db& db) {
    std::string key = "user:" + id;
    if (auto v = cache.get(key)) return v;
    static std::array<std::mutex, 64> shards;                         // lock striping: bounded memory, low contention
    std::lock_guard<std::mutex> g(shards[std::hash<std::string>{}(key) % shards.size()]);
    if (auto v = cache.get(key)) return v;                           // double-check after acquiring the lock
    auto row = db.fetchUser(id);
    if (row) cache.set(key, *row, 300 + rand() % 60);
    return row;
}
```

**Interview answer:** "I'd use cache-aside with Redis: read the cache, fall back to the DB, populate with a TTL plus jitter; on writes update the DB and delete the key. To protect the database I'd add single-flight loading for hot keys, negative caching or a Bloom filter for missing keys, and monitor the hit ratio."

**Loopholes:** caching does not fix a bad query (fix the query and index first); caches must be **optional** (the system works, slower, if the cache is down); stale reads are acceptable only if the requirements say so; do not cache per-user private data in a shared CDN/proxy without `Vary`/`private`; memory limits and eviction cause surprise misses; serialization cost for big values; Redis is not a durable primary database by default.

---

## H8. CDN (Content Delivery Network)

**What:** a geographically distributed network of **edge servers** that cache content close to users, reducing latency, origin load and bandwidth cost, and absorbing traffic spikes/DDoS.

```
User (Mumbai) --> DNS/anycast --> nearest edge POP --hit--> returns cached object (few ms)
                                       |
                                       +--miss--> (regional/origin shield cache) --> origin server / object storage; edge stores a copy
```

- **Pull CDN:** edges fetch from the origin on first request and cache by TTL (most common). **Push CDN:** you upload content to the CDN (good for large, rarely changing files).
- **What to put on a CDN:** static assets (JS, CSS, images), video segments (HLS/DASH), downloads, API responses that are cacheable; plus TLS termination, compression and DDoS protection at the edge.
- **Control:** `Cache-Control: public, max-age=...`, `ETag`/`Last-Modified` revalidation, `Vary`, the **cache key** (URL + chosen headers/query params), **signed URLs/cookies** for private media.
- **Invalidation:** explicit **purge** (slow, costly, rate-limited) or, better, **versioned/fingerprinted filenames** (`app.3f9c1.js`) with a very long TTL so a new release is just a new URL. Use short TTLs for HTML.
- **Dynamic content acceleration:** edge routes use optimized backbone paths and connection reuse even for uncacheable content; edge compute (Workers/Lambda@Edge) runs logic near users.

**Loopholes:** the **cache key** includes cookies or random query strings by mistake and the hit ratio collapses; stale content after deploys if you rely on purges; caching responses with user-specific data (privacy leak); origin overload on a cold cache or purge (**origin shield**, request collapsing); CDN adds a hop for uncacheable traffic; costs for egress.

---

## H9. Databases at scale: selection, replication, sharding, partitioning, indexing, pooling

**SQL vs NoSQL: how to choose (details in Part 2, B11)**

| Need | Choice |
| --- | --- |
| transactions, joins, constraints, reporting | PostgreSQL / MySQL |
| key lookups at huge scale, low latency, simple access | DynamoDB, Redis, Cassandra (by key) |
| flexible documents, evolving schema | MongoDB |
| high write throughput, time series, wide rows | Cassandra, ScyllaDB, TimescaleDB |
| relationships (social graph, fraud) | Neo4j or a graph layer |
| full-text search and analytics queries | Elasticsearch/OpenSearch |
| blobs/files | object storage (S3) + metadata in a DB |
| analytics/warehouse | BigQuery, Snowflake, Redshift, ClickHouse |

**Replication**

```
              writes                 async (or sync) replication
clients ----> LEADER ----------------------------> FOLLOWER 1 (reads)
                          \---------------------> FOLLOWER 2 (reads)
```

- **Leader-follower (primary-replica):** one node accepts writes; replicas serve **reads** and act as failover targets. **Synchronous** replication = no data loss but higher write latency and lower availability; **asynchronous** = fast but **replication lag** and possible loss of the last writes on failover; **semi-sync** is a compromise.
- **Replication lag effects and fixes:** a user writes and immediately reads from a lagging replica and sees old data. Fixes: **read-your-writes** (route a user's reads to the leader for a few seconds after a write, or use a version/timestamp), **monotonic reads** (stick a user to one replica).
- **Failover:** detect leader failure, promote a replica, repoint clients; **split-brain** (two leaders) is prevented by quorum/fencing tokens.
- **Multi-leader** (multi-region writes; conflict resolution needed: last-write-wins, CRDTs) and **leaderless/quorum** (Dynamo, Cassandra: `N` replicas, write to `W`, read from `R`; **`W + R > N`** gives strong-ish consistency, with `N=3, W=2, R=2` as the usual setting).

**Partitioning vs sharding:** *partitioning* splits a table into pieces (often within one database: by range/date/hash) for manageability and pruning; *sharding* spreads partitions across **multiple servers** to scale writes and storage horizontally.

**Sharding strategies**

| Strategy | How | Pros | Cons |
| --- | --- | --- | --- |
| **Range** | by key range (A-F, date) | efficient range scans | **hot shards** (latest dates), uneven sizes |
| **Hash** | `hash(key) % N` | even distribution | range queries scatter; **resharding moves most keys** (fix: consistent hashing / many virtual shards) |
| **Directory/lookup** | a mapping service says where a key lives | flexible | the lookup is a bottleneck/SPOF |
| **Geo** | by region | data locality, compliance | uneven load |

**Choosing a shard key:** high cardinality, even distribution, matches the dominant access pattern (all of a user's data on one shard), avoids hot keys. Example: shard chat messages by `conversation_id`, orders by `user_id`; do not shard by `created_date` (all writes hit one shard).

**Sharding pain points:** cross-shard joins and transactions (avoid by data modelling or use sagas, H24), global secondary indexes, **resharding** (use many logical shards mapped to physical nodes, e.g. 1,024 virtual shards), **hot/celebrity keys** (split the key: `key#random(0..9)` and merge on read), uniqueness constraints across shards, operational complexity. **Delay sharding** until vertical scaling, caching, and replicas are exhausted.

**Indexing (recap; Part 2 B6):** B+ tree indexes for lookups/ranges; composite index leftmost-prefix; covering indexes; each index slows writes and uses memory. In NoSQL the primary/partition key **is** the access path; design tables per query.

**Connection pooling**

- Opening a DB connection (TCP + TLS + auth + session setup) costs milliseconds and the database can only handle a limited number (PostgreSQL \~100s, each connection is a process/thread with memory). A **pool** reuses a fixed set of open connections.
- **Pool sizing:** more is not better; a common start is `pool = (core_count x 2) + effective_spindles` per DB, or compute from Little's law (`concurrency = QPS x avg query time`). With 50 app instances x 20 connections = 1,000 connections: too many; use a **server-side pooler** (PgBouncer, ProxySQL, RDS Proxy) to multiplex.
- Pitfalls: **connection leaks** (not returned to the pool: always use `with`/try-finally), pool exhaustion under slow queries (cascading timeouts), long transactions pinning connections, no timeout on acquiring a connection, serverless functions opening a new connection per invocation.

**Loopholes:** replicas do not help write scaling; synchronous replication to all replicas harms availability; sharding by `hash % N` makes adding a node a mass migration; a single "celebrity" key can overload one shard regardless of hashing; backups are not replication (replication copies deletes and corruption too).

---

## H10. Consistency models, ACID vs eventual consistency, CAP and PACELC

**Consistency spectrum (strongest to weakest)**

| Model | Guarantee | Example |
| --- | --- | --- |
| **Strong / linearizable** | every read sees the latest committed write; the system behaves as if there is one copy | single-leader DB reading from the leader, etcd, ZooKeeper, Spanner |
| **Sequential** | all nodes see operations in the same order (not necessarily real time) |  |
| **Causal** | causally related operations are seen in order (a reply after the message it answers) | MongoDB causal sessions |
| **Read-your-writes** | you always see your own writes | route a user to the leader after writing |
| **Monotonic reads** | you never see data go back in time | stick a user to one replica |
| **Eventual** | replicas converge **if writes stop**; reads may be stale meanwhile | DNS, Cassandra at low consistency, DynamoDB default reads |

**ACID vs BASE:** ACID (Part 2, B2) gives atomic, isolated, durable transactions, typically on one node or with distributed commit. **BASE** = Basically Available, Soft state, Eventually consistent: favour availability and scalability and accept temporary inconsistency. Choose by the **business cost of stale data**: bank balance and inventory oversell need strong guarantees; likes, view counts and feeds tolerate eventual consistency.

**CAP theorem:** in a distributed data store during a **network partition (P)** you must choose between **Consistency (C: linearizable reads/writes)** and **Availability (A: every non-failing node answers)**. Partitions will happen, so the real choice is **CP or AP when a partition occurs**; when there is no partition you can have both.

```
            partition happens
        /                       \
  CP: reject/ block requests     AP: keep answering, accept divergence (reconcile later)
  (ZooKeeper, etcd, HBase,       (Cassandra, DynamoDB, Riak, CouchDB)
   MongoDB majority writes)
```

**PACELC (more complete):** if there is a **P**artition choose **A** or **C**; **E**lse (normal operation) choose **L**atency or **C**onsistency. Replication always costs latency if you want consistency. Examples: DynamoDB and Cassandra are **PA/EL** (available and fast, eventual); Spanner, HBase, VoltDB are **PC/EC** (consistent always); MongoDB is configurable (often PA/EC or PC/EC with majority concern).

**Quorums (leaderless stores):** `N` replicas, write acknowledged by `W`, read from `R`; **`R + W > N`** guarantees the read set overlaps the latest write set (e.g. N=3, W=2, R=2). `W=N` means writes block if any replica is down; `R=1, W=1` is fastest but only eventually consistent. Supporting mechanisms: **read repair**, **hinted handoff**, **anti-entropy (Merkle trees)**, **vector clocks**/**last-write-wins**/**CRDTs** for conflicts, **sloppy quorums**.

**Interview answer:** "CAP says that under a partition we pick consistency or availability. For payments and inventory I'd choose CP (reject rather than oversell); for feeds and counters I'd choose AP with eventual consistency, and use PACELC to say that even without partitions the stronger option costs latency."

**Loopholes:** CAP's "C" is linearizability, **not** the ACID "C"; CAP is not "pick any 2 of 3" (P is not optional); a system is not simply "CP" or "AP": consistency is often **tunable per operation**; single-node databases are not subject to CAP; eventual consistency has no bound unless the system states one; a CP system is still available when there is no partition.

---

## H11. Consistent hashing

**Problem:** with `node = hash(key) % N`, changing `N` remaps almost **all** keys (cache misses everywhere, mass data movement). **Consistent hashing** maps both keys and nodes onto a **hash ring**; a key belongs to the **first node clockwise** from its position. Adding or removing a node moves only about `K / N` keys (the neighbour's share).

```
            node A (hash 40)
       .-----o-----.
     /               \        key k1 (hash 55) -> next node clockwise = B (70)
   o D (hash 300)      o B (70)
     \               /        add node E at 60: only keys in (40, 60] move from B to E
       '-----o-----'
            C (hash 200)
```

**Virtual nodes (vnodes):** each physical node is placed on the ring at many positions (100 to 256 labels like `nodeA#17`). This **evens out load**, spreads a failed node's load across many neighbours, and allows **weights** (a bigger machine gets more vnodes).

**Replication on the ring:** store each key on the next `R` **distinct physical** nodes clockwise (Dynamo, Cassandra).

```python
import bisect, hashlib
class ConsistentHashRing:
    def __init__(self, nodes=(), vnodes=100):
        self.vnodes, self.ring, self.hashes = vnodes, {}, []
        for n in nodes: self.add_node(n)
    @staticmethod
    def _h(s): return int(hashlib.md5(s.encode()).hexdigest(), 16)        # stable across processes (Python's hash() is salted)
    def add_node(self, node):
        for i in range(self.vnodes):
            h = self._h(f'{node}#{i}'); self.ring[h] = node; bisect.insort(self.hashes, h)
    def remove_node(self, node):
        for i in range(self.vnodes):
            h = self._h(f'{node}#{i}'); del self.ring[h]; self.hashes.remove(h)
    def get_node(self, key):
        if not self.hashes: raise RuntimeError('empty ring')
        i = bisect.bisect(self.hashes, self._h(key)) % len(self.hashes)    # wrap around the ring
        return self.ring[self.hashes[i]]
    def get_replicas(self, key, n=3):
        i, out = bisect.bisect(self.hashes, self._h(key)), []
        for j in range(len(self.hashes)):
            node = self.ring[self.hashes[(i + j) % len(self.hashes)]]
            if node not in out: out.append(node)
            if len(out) == n: break
        return out
```

```cpp
class ConsistentHash {
    std::map<uint64_t, std::string> ring_; int vnodes_;
    static uint64_t hash(const std::string& s) {                         // FNV-1a 64-bit: deterministic, cheap
        uint64_t h = 1469598103934665603ULL; for (unsigned char c : s) { h ^= c; h *= 1099511628211ULL; } return h;
    }
public:
    explicit ConsistentHash(int vnodes = 100) : vnodes_(vnodes) {}
    void addNode(const std::string& n) { for (int i = 0; i < vnodes_; i++) ring_[hash(n + "#" + std::to_string(i))] = n; }
    void removeNode(const std::string& n) { for (int i = 0; i < vnodes_; i++) ring_.erase(hash(n + "#" + std::to_string(i))); }
    const std::string& getNode(const std::string& key) const {
        if (ring_.empty()) throw std::runtime_error("empty ring");
        auto it = ring_.lower_bound(hash(key));                           // first position >= key hash
        return (it == ring_.end() ? ring_.begin() : it)->second;           // wrap to the start
    }
};
```

**Where it is used:** distributed caches (client-side routing to Memcached/Redis shards), Dynamo-style databases (Cassandra, Riak), CDN/load balancers (cache locality), sharded stores, partitioning in Kafka-like systems (key hash to partition is plain modulo; the ring is for node assignment).

**Alternatives:** **rendezvous (highest random weight) hashing** (each key picks the node with the highest `hash(node, key)`; simple, no ring), **jump consistent hash** (O(1) memory, nodes numbered 0..N-1), **fixed many logical shards** mapped to nodes by a table.

**Complexity:** lookup O(log(N x vnodes)); add/remove O(vnodes x log).

**Loopholes:** without vnodes the load is badly uneven; a hot key stays hot (hashing does not split a single key); removing a node must also move **data**, not just routing (and the replicas); using a non-stable hash (`hash()` salted per process) gives different rings on different machines; membership changes must be propagated consistently (gossip/ZooKeeper).

---

## H12. Unique ID generation

**Requirements to ask:** globally unique? sortable by time (for pagination and index locality)? size (64-bit vs 128-bit)? generated by many nodes without coordination? guessable or secret? rate (IDs per second)?

| Approach | Pros | Cons |
| --- | --- | --- |
| DB auto-increment | simple, ordered, compact | single point/bottleneck; hard across shards; reveals counts |
| **UUID v4** (random 128-bit) | no coordination, no collisions in practice | 16 bytes, **random inserts hurt B-tree locality**, not sortable |
| **UUID v7 / ULID** (timestamp prefix + random) | roughly time-ordered, decentralized, index-friendly | 128-bit; clock dependence |
| **Ticket server** (DB or Redis `INCR` with ranges handed out in batches) | simple, ordered | central dependency (batching mitigates); use 2 servers with odd/even steps |
| **Snowflake (Twitter)** | 64-bit, **time-ordered**, decentralized, very fast | needs unique machine ids, clock handling |
| Hash of content / random base62 (short URLs) | stateless | collisions must be handled |

**Snowflake layout (64 bits):** `1 sign bit (0) | 41 bits timestamp (ms since a custom epoch, ~69 years) | 10 bits machine id (1,024 nodes: e.g. 5 datacenter + 5 worker) | 12 bits sequence (4,096 IDs per ms per node)`. Roughly 4 million IDs per second per node.

```python
import threading, time
class Snowflake:
    EPOCH = 1_700_000_000_000                                    # custom epoch in ms (e.g. project start): extends the 41-bit range
    def __init__(self, machine_id):
        assert 0 <= machine_id < 1024
        self.machine_id, self.seq, self.last = machine_id, 0, -1
        self.lock = threading.Lock()
    def next_id(self):
        with self.lock:
            now = int(time.time() * 1000)
            if now < self.last: raise RuntimeError('clock moved backwards: refuse to generate (or wait)')
            if now == self.last:
                self.seq = (self.seq + 1) & 0xFFF                # 12-bit sequence
                if self.seq == 0:                                # exhausted this millisecond: spin until the next one
                    while now <= self.last: now = int(time.time() * 1000)
            else: self.seq = 0
            self.last = now
            return ((now - self.EPOCH) << 22) | (self.machine_id << 12) | self.seq
```

```cpp
class Snowflake {
    static constexpr uint64_t kEpoch = 1700000000000ULL; uint64_t machine_, last_ = 0, seq_ = 0; std::mutex m_;
    static uint64_t nowMs() { return std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::system_clock::now().time_since_epoch()).count(); }
public:
    explicit Snowflake(uint64_t machine) : machine_(machine & 0x3FF) {}
    uint64_t nextId() {
        std::lock_guard<std::mutex> g(m_); uint64_t now = nowMs();
        if (now < last_) throw std::runtime_error("clock moved backwards");
        if (now == last_) { seq_ = (seq_ + 1) & 0xFFF; if (seq_ == 0) while ((now = nowMs()) <= last_) {} }
        else seq_ = 0;
        last_ = now; return ((now - kEpoch) << 22) | (machine_ << 12) | seq_;
    }
};
```

**Machine id assignment:** from config, ZooKeeper/etcd lease, or the pod ordinal; two nodes with the same id will generate duplicates.

**Short IDs (URL shortener):** convert a unique counter/Snowflake to **base62** (`[0-9a-zA-Z]`): 7 characters give 62^7 ≈ 3.5 trillion codes. Random codes need collision checks (unique constraint + retry).

**Loopholes:** **clock skew/backwards jumps** (NTP) can duplicate or block IDs (refuse, wait, or use a logical clock); UUIDs as clustered primary keys fragment indexes; timestamps embedded in IDs leak creation time and volume; sequential public IDs are guessable (use separate public ids if enumeration matters); "sortable by time" is only roughly ordered across nodes (millisecond granularity, skew).

---

## H13. Object storage, file uploads and CDN integration

|  | Block | File | **Object (S3, GCS, Azure Blob)** |
| --- | --- | --- | --- |
| Unit | fixed-size blocks | files in directories | **objects** (bytes + metadata + key) in flat buckets |
| Access | OS-level disk | POSIX/NFS | HTTP API (`PUT/GET/DELETE`) |
| Strengths | low latency, databases | shared filesystem | huge scale, cheap, durable (11 nines), versioning, lifecycle rules |
| Weaknesses | not shared | limited scale | no in-place partial edit, higher latency than disk, eventual semantics for some operations (S3 now offers strong read-after-write) |

**Rules:** never store large blobs in the relational database (bloats backups, hurts cache and replication); store **blobs in object storage** and keep **metadata** (owner, size, content type, key, checksum, status) in the database.

**Upload design: presigned URLs (the client uploads directly to storage)**

```
1. Client -> API:  POST /v1/files {name, size, content_type}
2. API: authz + quota + create metadata row (status=PENDING) -> returns a presigned PUT URL (expires in minutes)
3. Client -> Object storage:  PUT bytes to the URL        (the API servers never carry the file bytes)
4. Storage -> event (ObjectCreated) -> queue -> workers: virus scan, thumbnails, transcoding, metadata extraction
5. Worker updates metadata (status=READY). Client polls or gets a notification.
6. Download: API returns a signed URL via the CDN (private bucket, short expiry).
```

```python
import boto3
s3 = boto3.client('s3')
url = s3.generate_presigned_url('put_object', Params={'Bucket': 'media', 'Key': 'u42/9f3a/report.pdf', 'ContentType': 'application/pdf'}, ExpiresIn=600)
```

**Large files: multipart / chunked upload.** Split the file into parts (5 MB to 100 MB), upload parts in **parallel**, retry only failed parts, then call **complete** with the list of part ETags; **resumable** after network loss; the final object is assembled by storage. Use **content hashing** (SHA-256) for integrity and **deduplication** (a chunk already stored is not uploaded again: Dropbox-style).

**Other design points:** unguessable keys (`user/uuid/name`), size/type limits and content sniffing, antivirus scanning before exposing, encryption in transit and at rest (SSE-KMS, optional client-side), versioning for undo, lifecycle policies (move to cheaper tiers, expire temp files), **orphan cleanup** (PENDING rows that never completed), replication across regions, CORS configuration for browser uploads, the CDN in front for downloads with signed URLs/cookies, and **range requests** for video streaming.

**Loopholes:** letting clients upload through your API servers saturates bandwidth and threads; a presigned URL that is too long-lived or overly permissive (wrong key prefix) is a security hole; trusting the client-reported content type; never verifying the upload completed (use the storage event, not the client's say-so); public buckets by mistake; listing huge buckets is slow (keep an index in the DB).

---

## H14. Messaging: queues, Kafka and pub-sub

**Why asynchronous messaging:** **decoupling** producer and consumer, **load leveling** (absorb spikes, consumers work at their own pace), **reliability** (messages survive a consumer outage; retries), **fan-out**, and keeping slow work off the request path.

|  | Message queue (point-to-point) | Pub-sub (publish/subscribe) |
| --- | --- | --- |
| Delivery | each message handled by **one** consumer (competing consumers) | each message delivered to **every** subscriber |
| Use | task distribution, work queues | events, notifications, fan-out |
| Examples | SQS, RabbitMQ queue, Celery | SNS, Kafka topics (per consumer group), Redis pub/sub, Google Pub/Sub |

**Delivery semantics**

- **At-most-once:** may lose messages, never duplicates (ack before processing).
- **At-least-once:** never loses, may **duplicate** (ack after processing; the practical default).
- **Exactly-once:** achieved in practice by at-least-once delivery + **idempotent consumers** (dedupe by message id) or transactional processing (Kafka transactions); true exactly-once across systems is not free.

**Core concepts:** producer, broker, consumer, **acknowledgement**, **visibility timeout** (SQS: a received message is hidden; if not deleted in time it reappears), **retry** with backoff, **dead-letter queue (DLQ)** for poison messages after N failures, **ordering** (per queue/partition only), **backpressure**, TTL, prefetch/concurrency, message size limits.

### Kafka

```
Topic 'orders' (3 partitions, replication factor 3)
  P0: [m0][m3][m6]...   <- append-only log, each record has an OFFSET
  P1: [m1][m4][m7]...      key hash -> partition: same key always in the same partition => per-key ORDER
  P2: [m2][m5][m8]...
Consumer group 'billing':  C1 reads P0,P1   C2 reads P2    (each partition is consumed by ONE consumer in a group)
Consumer group 'analytics': independent offsets => every group receives all messages (pub-sub)
```

- **Log-based broker:** messages are appended to a **partitioned, replicated commit log** and **retained** (time/size) rather than deleted on consumption; consumers track an **offset** and can **replay** (reprocess after a bug, bootstrap a new service).
- **Partitions = parallelism and ordering unit.** Max consumers in a group = partitions. Key choice decides ordering and hot partitions. Increasing partitions later changes key-to-partition mapping.
- **Replication:** each partition has a leader and followers (**ISR**, in-sync replicas). Producer `acks=all` + `min.insync.replicas=2` for durability; `acks=1` (faster, can lose on leader failure); `acks=0` (fire and forget). **Idempotent producer** avoids duplicates on retry; **transactions** give atomic multi-partition writes and exactly-once processing within Kafka.
- **Consumer groups and rebalancing:** members join/leave and partitions are reassigned (pauses processing; use static membership/cooperative rebalancing). **Consumer lag** (latest offset minus committed offset) is the key health metric.
- **Commit offsets after processing** (at-least-once); committing before processing risks loss.
- **Log compaction** keeps the latest value per key (changelog, cache rebuilds). Very high throughput (hundreds of MB/s per broker) through sequential disk I/O and zero-copy.
- **Kafka vs RabbitMQ vs SQS:** Kafka = high-throughput **event log**, replay, stream processing, ordering per key; RabbitMQ = flexible routing, per-message acks, low-latency task queues; SQS = fully managed simple queue (standard: at-least-once, unordered; FIFO queue: ordered, deduplicated, lower throughput).

```python
# Consumer pattern: process, then commit; make the handler idempotent
for msg in consumer:                              # e.g. confluent-kafka / kafka-python style loop
    try:
        if already_processed(msg.key, msg.offset):   # dedupe store (DB unique key or Redis SETNX)
            consumer.commit(); continue
        handle(msg.value)
        mark_processed(msg.key, msg.offset)
        consumer.commit()                           # at-least-once: commit AFTER successful processing
    except RetryableError:
        send_to_retry_topic(msg)                    # delayed retry topic with backoff
    except Exception:
        send_to_dlq(msg)                            # poison message: park it, do not block the partition
```

**Pub-sub design notes:** topics with schemas (Avro/Protobuf + schema registry) and versioning; consumers must be idempotent and tolerate reordering across partitions; use events for **notifications of facts** (`OrderPlaced`) rather than commands when you want loose coupling; watch for ordering assumptions and the **outbox pattern** (H24) to publish reliably.

**Loopholes:** a queue does not remove the need for idempotency (duplicates will happen); one slow/poison message can block a partition (use retry topics and DLQs); ordering is per partition, not global; unbounded queue growth hides an overloaded consumer (alert on depth and age); Redis pub/sub is fire-and-forget (no persistence or replay: use Streams for durability); too many partitions slow recovery and rebalancing.

---

## H15. REST, gRPC, GraphQL and the API gateway

|  | REST | gRPC | GraphQL |
| --- | --- | --- | --- |
| Style | resources + HTTP verbs | RPC with generated stubs | one endpoint, client-shaped queries |
| Format | JSON over HTTP/1.1 or 2 | **Protocol Buffers over HTTP/2** (binary) | JSON over HTTP |
| Strengths | simple, cacheable, ubiquitous, tooling | fast, small, strongly typed contracts, streaming (uni/bi-directional), deadlines | no over/under-fetching, one round trip for aggregated data, typed schema |
| Weaknesses | over/under-fetching, multiple round trips | not browser-native (needs gRPC-Web), harder to debug | caching is hard, N+1 resolver problem, query cost/abuse control, complexity |
| Best for | public APIs, CRUD | internal service-to-service, low latency, streaming | mobile/web clients with varied data needs, BFF layers |

REST details (methods, status codes, idempotency) are in Part 2 (C6, C8, C10).

**API gateway:** a single entry point in front of services that handles **cross-cutting concerns**: routing/path-based dispatch, **authentication/authorization**, **rate limiting**, TLS termination, request/response transformation, **aggregation** (compose several calls), caching, logging/metrics/tracing, CORS, circuit breaking, versioning, canary routing. Examples: Kong, Nginx/Envoy, AWS API Gateway, Apigee.

```
Clients -> CDN/WAF -> API Gateway (auth, rate limit, route) -> user-svc | order-svc | payment-svc -> their own databases
                                  \-> BFF (backend-for-frontend per client type: mobile, web)
```

**Loopholes:** the gateway can become a **single point of failure and a bottleneck** (run it redundantly, keep it thin); do not put business logic in it; every extra hop adds latency; a gateway is not a **service mesh** (the mesh handles east-west service-to-service traffic: mTLS, retries, telemetry via sidecars); GraphQL needs depth/complexity limits; keep public vs internal APIs separate.

---

## H16. Monolith vs microservices, service discovery and health checks

|  | Monolith | Microservices |
| --- | --- | --- |
| Deploy | one unit | many independent services |
| Dev speed | fast early; slows with size | parallel teams, independent releases |
| Scaling | scale everything together | scale hot services individually |
| Data | one shared database, ACID transactions | **database per service**, eventual consistency, sagas |
| Failure | one bug can take it all down | isolation, but **cascading failures** and partial failures |
| Complexity | code complexity | **distributed-systems complexity**: networking, observability, versioning, deployment, testing, data consistency |
| Choose when | small team, new product, unclear boundaries | large org, clear domains, different scaling/tech needs |

**Guidance:** start with a **modular monolith** (clear module boundaries, one deployable) and extract services when there is a real reason (team independence, scaling hot spots, isolation). Split by **business capability / bounded context (DDD)**, not by technical layer. Services own their data; communicate through APIs (sync) or events (async). Prerequisites: CI/CD, containerization, centralized logging, metrics and tracing, API versioning.

**Service discovery:** instances come and go (autoscaling, deploys), so clients need current addresses.

- **Client-side discovery:** the client queries a **registry** (Consul, etcd, Eureka) and load-balances itself.
- **Server-side discovery:** the client calls a stable LB/DNS name; the platform routes it (Kubernetes **Services** with DNS, AWS ELB).
- **Registration:** instances **register** on start and **deregister** on stop, with **heartbeats/TTL leases** so dead nodes disappear; or the platform registers automatically (Kubernetes endpoints).

**Health checks (Kubernetes probes)**

| Probe | Question | On failure |
| --- | --- | --- |
| **Liveness** | is the process stuck/deadlocked? | restart the container |
| **Readiness** | can it serve traffic **now** (dependencies up, warmed)? | remove from load balancing (no restart) |
| **Startup** | has slow initialization finished? | delay other probes |

```yaml
livenessProbe:  { httpGet: { path: /healthz, port: 8080 }, periodSeconds: 10, failureThreshold: 3 }
readinessProbe: { httpGet: { path: /ready,   port: 8080 }, periodSeconds: 5 }
```

**Deployment strategies:** **rolling** (replace gradually), **blue-green** (switch traffic between two full environments; instant rollback), **canary** (send 1% to 5% to the new version, watch metrics, then expand), feature flags to decouple deploy from release. **Graceful shutdown:** stop accepting new work, drain in-flight requests, deregister.

**Loopholes:** a liveness probe that checks the database causes restart storms when the DB is down (use readiness for dependencies); distributed monolith (services that must deploy together, share a database); chatty synchronous call chains multiply latency and failure probability; no service ownership of data; rewriting a working monolith "for microservices" is a classic mistake.

---

## H17. WebSockets, Server-Sent Events and polling

| Technique | Direction | How | Use when | Cost |
| --- | --- | --- | --- | --- |
| **Short polling** | client → server | repeat `GET` every N seconds | simple, infrequent updates | wasted requests, delay up to N s |
| **Long polling** | client → server | request held open until data or timeout, then repeat | near-real-time without special infrastructure | many held connections, reconnect overhead |
| **SSE** | server → client (one way) | one long-lived HTTP response, `text/event-stream`, auto-reconnect with `Last-Event-ID` | live feeds, notifications, progress, stock tickers | one-directional, text only, per-origin connection limits on HTTP/1.1 |
| **WebSocket** | **bidirectional** | HTTP upgrade (`101`) then a persistent TCP channel with small frames | chat, games, collaboration, trading UIs | stateful connections to scale; heartbeats; reconnect logic |

```python
# SSE (Flask): server pushes events
from flask import Flask, Response
import time, json
app = Flask(__name__)
@app.get('/events')
def events():
    def stream():
        n = 0
        while True:
            n += 1; yield f'id: {n}\ndata: {json.dumps({"tick": n})}\n\n'; time.sleep(1)    # blank line ends one event
    return Response(stream(), mimetype='text/event-stream')
```

```python
# WebSocket (asyncio + websockets library): broadcast to connected clients
import asyncio, websockets
clients = set()
async def handler(ws):
    clients.add(ws)
    try:
        async for msg in ws: websockets.broadcast(clients, msg)
    finally: clients.discard(ws)
async def main():
    async with websockets.serve(handler, '0.0.0.0', 8765): await asyncio.Future()
```

**Scaling real-time connections**

- Each connection holds state on a specific server; one server can hold \~100k mostly idle connections with tuned file-descriptor limits and an event-driven stack (epoll, Netty, Node, asyncio, Go).
- Users connect to different servers, so a message for user B must reach B's server: use a **pub/sub backplane** (Redis pub/sub or streams, Kafka, NATS) and a **presence/routing map** (`user_id -> server_id` in Redis).
- **Heartbeats** (ping/pong) detect dead connections; clients **reconnect with exponential backoff + jitter** and **resume from the last seen message id**; the load balancer must support WebSockets with long idle timeouts; authenticate during the upgrade (token in the first message or header); apply rate limits per connection; graceful deploys must drain connections (clients reconnect).

**Loopholes:** WebSocket messages are **not guaranteed delivery** across reconnects (add acks, sequence numbers, and a persisted store for history); a deploy disconnects everyone at once (reconnection stampede: add jitter); proxies/corporate firewalls may break WebSockets (fallback to SSE/long polling); SSE over HTTP/1.1 is limited to \~6 connections per browser per domain; mobile clients need push notifications when the app is in the background.

---

## H18. Background jobs and task queues

**Why:** keep slow or unreliable work (emails, image processing, report generation, payments reconciliation, third-party calls) out of the request path; smooth spikes; enable retries.

```
API --(enqueue job {id, type, payload})--> Queue (Redis/SQS/RabbitMQ/Kafka) --> Worker pool --> side effects, result store
  |<-- 202 Accepted {job_id} ; GET /jobs/{id} -> {status: queued|running|succeeded|failed, result}     (or webhook/push on completion)
```

**Job design rules**

1. **Idempotent:** the queue is at-least-once, so running a job twice must be safe (idempotency key, upsert, "already done?" check).
2. **Small payloads:** pass ids, not blobs; workers fetch fresh data.
3. **Retries with exponential backoff + jitter**, a max attempt count, and a **dead-letter queue** after exhaustion; classify errors as retryable vs permanent.
4. **Timeouts** per job and a **visibility timeout/lease** longer than the expected run time (extend with heartbeats for long jobs); **checkpointing** for long jobs.
5. **Priorities** and separate queues per class (critical vs bulk) so low-priority floods cannot starve urgent jobs.
6. **Scheduling:** delayed jobs (`run_at`) and recurring/cron jobs (a scheduler enqueues; avoid running cron on every instance: use leader election or a dedicated scheduler, see H25 and Part 4 L12).
7. **Concurrency and rate limits** per downstream dependency; **graceful shutdown** (finish or requeue current job).
8. **Observability:** queue depth, **age of oldest message**, processing time, failure rate, retries, DLQ size, worker utilization.

```python
# Celery example
from celery import Celery
app = Celery('jobs', broker='redis://localhost:6379/0')
@app.task(bind=True, max_retries=5, acks_late=True)           # acks_late: acknowledge after finishing (at-least-once)
def send_receipt(self, order_id):
    try:
        if receipt_already_sent(order_id): return                # idempotency check
        deliver_email(order_id); mark_receipt_sent(order_id)
    except TemporaryError as e:
        raise self.retry(exc=e, countdown=min(2 ** self.request.retries * 5, 300))   # exponential backoff, capped
send_receipt.delay(order_id=42)
```

Tools: Celery, RQ, Sidekiq, BullMQ, AWS SQS + Lambda, Temporal/Cadence (durable workflows with retries, timers and state), Airflow (batch DAGs).

**Loopholes:** a job enqueued **before** the database transaction commits may run against uncommitted data (enqueue after commit, or use the **outbox**, H24); jobs without idempotency cause double emails/charges; a poison job retried forever blocks capacity; unbounded queues hide failure; clock-based cron on many instances runs the task many times; failing silently with no alert on DLQ growth.

---

## H19. Rate limiting and rate-limit strategies

**Why:** protect services from overload and abuse (scrapers, brute force, DDoS), ensure **fair use** among tenants, control cost, enforce plans (free vs paid tiers). **Where:** CDN/WAF, API gateway (most common), per-service middleware. **What to key on:** user id, API key, IP, endpoint, tenant, or combinations (`user + endpoint`).

| Algorithm | How | Pros | Cons |
| --- | --- | --- | --- |
| **Fixed window counter** | counter per `key + window` (e.g. per minute); reject above the limit | simplest, O(1) memory | **boundary burst:** up to 2x the limit across a window edge (100 at 12:00:59 + 100 at 12:01:00) |
| **Sliding window log** | store a timestamp per request, drop old ones, count | exact | memory O(requests), expensive |
| **Sliding window counter** | `prev_window_count x (1 - elapsed_fraction) + current_count` | smooth, O(1) memory, approximate | assumes uniform distribution in the previous window |
| **Token bucket** | bucket of capacity `B` refills at `r` tokens/s; each request takes a token | allows **bursts** up to `B`, simple, widely used (AWS, Stripe) | two parameters to tune |
| **Leaky bucket** | queue drained at a fixed rate; overflow dropped | **smooth, constant output rate** (traffic shaping) | bursts are delayed or dropped; recent requests wait behind old ones |

```python
import threading, time
class TokenBucket:
    def __init__(self, rate, capacity, clock=time.monotonic):
        self.rate, self.cap, self.tokens, self.clock = rate, capacity, capacity, clock
        self.last, self.lock = clock(), threading.Lock()
    def allow(self, n=1):
        with self.lock:
            now = self.clock()
            self.tokens = min(self.cap, self.tokens + (now - self.last) * self.rate)    # lazy refill, no background thread
            self.last = now
            if self.tokens >= n: self.tokens -= n; return True
            return False

class SlidingWindowCounter:                        # approximate sliding window with two fixed windows
    def __init__(self, limit, window=60, clock=time.time): self.limit, self.w, self.clock, self.counts = limit, window, clock, {}
    def allow(self, key):
        now = self.clock(); cur = int(now // self.w); frac = (now % self.w) / self.w
        prev_c, cur_c = self.counts.get((key, cur - 1), 0), self.counts.get((key, cur), 0)
        if prev_c * (1 - frac) + cur_c >= self.limit: return False
        self.counts[(key, cur)] = cur_c + 1; return True       # (a real implementation also expires old keys)
```

```cpp
class TokenBucket {
    double rate_, cap_, tokens_; std::chrono::steady_clock::time_point last_; std::mutex m_;
public:
    TokenBucket(double rate, double cap) : rate_(rate), cap_(cap), tokens_(cap), last_(std::chrono::steady_clock::now()) {}
    bool allow(double n = 1) {
        std::lock_guard<std::mutex> g(m_); auto now = std::chrono::steady_clock::now();
        tokens_ = std::min(cap_, tokens_ + std::chrono::duration<double>(now - last_).count() * rate_); last_ = now;
        if (tokens_ < n) return false; tokens_ -= n; return true;
    }
};
```

**Distributed rate limiting with Redis:** the check-and-update must be **atomic**, so use a Lua script (or `INCR` + `EXPIRE` for fixed windows) rather than GET-then-SET (race).

```lua
-- KEYS[1]=bucket key; ARGV: rate (tokens/s), capacity, now (seconds, float), cost
local d = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(d[1]) or tonumber(ARGV[2])
local ts = tonumber(d[2]) or tonumber(ARGV[3])
tokens = math.min(tonumber(ARGV[2]), tokens + (tonumber(ARGV[3]) - ts) * tonumber(ARGV[1]))
local ok = 0
if tokens >= tonumber(ARGV[4]) then tokens = tokens - tonumber(ARGV[4]); ok = 1 end
redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', ARGV[3])
redis.call('EXPIRE', KEYS[1], math.ceil(tonumber(ARGV[2]) / tonumber(ARGV[1])) * 2)    -- idle buckets disappear
return ok
```

**Response contract:** HTTP **429 Too Many Requests** with `Retry-After` and headers like `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. Clients should back off with jitter (H21).

**Design decisions:** limits per tier and per endpoint (login stricter than reads); **fail-open vs fail-closed** when the limiter store is down (fail open for availability, fail closed for security-sensitive endpoints); local approximate counters synced periodically to cut Redis calls; **sharded Redis** by key (consistent hashing); clock source (use Redis time or monotonic); **cost-based limits** (expensive queries cost more tokens); concurrency limits (max in-flight) in addition to rates; **load shedding** under overload (drop low priority first).

**Abuse prevention beyond limits:** CAPTCHAs, IP reputation and WAF rules, device/behaviour fingerprinting, account lockout with exponential delay, API keys with quotas, bot detection, request signing.

**Loopholes:** keying only by IP breaks behind NAT (many users share one) and is bypassed with botnets; `X-Forwarded-For` can be spoofed unless trusted proxies set it; fixed window allows 2x bursts; per-instance in-memory limits multiply by the number of instances; GET-then-SET races let extra requests through; limiter outage can take the whole API down.

---

## H20. Idempotency and idempotency keys

**Definition:** an operation is idempotent if performing it once or many times has the same effect. Needed because networks fail: **timeouts leave the client unsure** whether the server processed the request, so clients **retry**; double-clicks and queue redelivery also repeat requests. A non-idempotent operation (create order, charge card, send email) would then execute twice.

**Naturally idempotent:** `GET`, `PUT` (replace), `DELETE`, set-to-absolute-value (`SET balance = 500`), upserts with a unique key. **Not idempotent:** `POST /payments`, `balance = balance + 100`, "append" operations.

**Idempotency key pattern (Stripe style)**

```
Client generates a unique key per logical operation:   POST /v1/payments   Idempotency-Key: 7c9e1b...   (UUID)
Server:
  1. Atomically INSERT (user_id, key, request_hash, status='IN_PROGRESS')  -- unique constraint on (user_id, key)
  2. Insert succeeded  -> perform the operation, store the response (status code + body), mark DONE, return it
  3. Key already exists:
        DONE + same request_hash      -> return the STORED response (no re-execution)
        IN_PROGRESS                   -> 409 Conflict / ask client to retry shortly
        different request_hash        -> 422 (same key reused with a different payload: client bug)
  4. Expire keys after a window (e.g. 24 hours)
```

```sql
CREATE TABLE idempotency_keys (
  user_id BIGINT, idem_key TEXT, request_hash TEXT NOT NULL, status TEXT NOT NULL,
  response_code INT, response_body JSONB, created_at TIMESTAMP DEFAULT now(),
  PRIMARY KEY (user_id, idem_key)                      -- the unique constraint IS the concurrency control
);
INSERT INTO idempotency_keys (user_id, idem_key, request_hash, status) VALUES (:u, :k, :h, 'IN_PROGRESS')
ON CONFLICT DO NOTHING;                                 -- rows_affected = 1 means this request owns the key
```

```python
def handle_payment(user_id, key, payload, db):
    h = sha256(json.dumps(payload, sort_keys=True).encode()).hexdigest()
    if db.insert_key_if_absent(user_id, key, h):                 # we own it: do the real work, in the SAME transaction as the effect if possible
        resp = charge_card(payload)
        db.complete_key(user_id, key, 201, resp); return 201, resp
    rec = db.get_key(user_id, key)
    if rec.request_hash != h: return 422, {'error': 'key reused with different payload'}
    if rec.status == 'IN_PROGRESS': return 409, {'error': 'request in progress'}
    return rec.response_code, rec.response_body                  # replay the original answer
```

**Idempotent consumers (messaging):** store processed message ids (`processed_messages(message_id PRIMARY KEY)`), or make the effect itself idempotent (upsert, conditional update `WHERE status = 'PENDING'`); insert the id **in the same transaction** as the business change. Downstream providers: pass your key to the payment gateway so the charge is deduplicated end to end.

**Loopholes:** the key must be generated **once per logical operation** (not per HTTP attempt); store the **response**, not just the key; the key check and the side effect must be atomic or a crash between them duplicates/loses work; deduplication windows expire (document it); idempotency is not exactly-once delivery, it is making duplicates harmless; keys scoped globally allow one user to replay another's request (scope by user).

---

## H21. Retries, timeouts, exponential backoff and jitter

**Timeouts (always set them):** a missing timeout lets one slow dependency exhaust threads/connections and take the whole system down. Set **connect**, **read** and an overall **deadline**; make them shorter than the caller's own timeout (**deadline propagation**: pass the remaining time budget downstream); pick values from observed p99 latency plus margin.

**Retries: when and how**

- Retry only **transient** failures (timeouts, connection reset, 429, 502/503/504), never client errors (400, 401, 404) or non-idempotent operations unless protected by an **idempotency key**.
- **Exponential backoff:** wait `base x 2^attempt` (100 ms, 200, 400, 800...), capped (e.g. 30 s), with a **max attempts**.
- **Jitter:** randomize the wait so that thousands of clients do not retry in lockstep (**thundering herd / retry storm**). **Full jitter:** `sleep = random(0, min(cap, base x 2^attempt))` (best distribution of load). **Equal jitter:** half fixed + half random. **Decorrelated jitter:** `sleep = min(cap, random(base, previous_sleep x 3))`.

```python
import random, time
def retry(fn, attempts=5, base=0.1, cap=10.0, retryable=(TimeoutError, ConnectionError)):
    for n in range(attempts):
        try: return fn()
        except retryable:
            if n == attempts - 1: raise                                    # out of attempts: surface the error
            time.sleep(random.uniform(0, min(cap, base * 2 ** n)))          # full jitter
```

```cpp
template <class F>
auto retry(F fn, int attempts = 5, double baseSec = 0.1, double capSec = 10.0) {
    std::mt19937 rng{std::random_device{}()};
    for (int n = 0;; ++n) {
        try { return fn(); }
        catch (const std::runtime_error&) {                                  // treat as transient (narrow this in real code)
            if (n == attempts - 1) throw;
            std::uniform_real_distribution<double> d(0.0, std::min(capSec, baseSec * std::pow(2.0, n)));
            std::this_thread::sleep_for(std::chrono::duration<double>(d(rng)));
        }
    }
}
```

**Avoiding retry amplification:** if each of 3 layers retries 3 times, one failure becomes 27 attempts hammering a struggling dependency. Retry at **one layer** (usually the edge/closest to the failure), use a **retry budget** (retries limited to \~10% of traffic), respect `Retry-After`, and combine with **circuit breakers** (H22). **Hedged requests:** send a second request after the p95 latency and use the first reply (reduces tail latency for idempotent reads; adds load).

**Loopholes:** retrying a non-idempotent call double-charges; retries without jitter synchronize clients; infinite retries without a cap; retrying on overload makes the overload worse; timeouts shorter than the real p99 create self-inflicted failures; long timeouts hold resources.

---

## H22. Circuit breakers and fault tolerance

**Circuit breaker:** wraps calls to a dependency and **stops calling it when it is failing**, failing fast instead of piling up slow requests and giving the dependency time to recover.

```
          failures >= threshold (or error rate over a window)
 CLOSED ---------------------------------------------> OPEN   (calls fail immediately / fallback)
   ^                                                      |
   | trial calls succeed                                  | after a cool-down timeout
   +------------------ HALF-OPEN <-----------------------+
                       (allow a few probe calls; any failure re-opens)
```

```python
import threading, time
class CircuitOpenError(Exception): ...
class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=30.0, half_open_max=1, clock=time.monotonic):
        self.threshold, self.timeout, self.half_max, self.clock = failure_threshold, reset_timeout, half_open_max, clock
        self.state, self.failures, self.opened_at, self.probes, self.lock = 'CLOSED', 0, 0.0, 0, threading.Lock()
    def call(self, fn, *a, **kw):
        with self.lock:
            if self.state == 'OPEN':
                if self.clock() - self.opened_at < self.timeout: raise CircuitOpenError()          # fail fast
                self.state, self.probes = 'HALF_OPEN', 0
            if self.state == 'HALF_OPEN':
                if self.probes >= self.half_max: raise CircuitOpenError()                          # limit trial traffic
                self.probes += 1
        try:
            result = fn(*a, **kw)
        except Exception:
            with self.lock:
                self.failures += 1
                if self.state == 'HALF_OPEN' or self.failures >= self.threshold: self.state, self.opened_at = 'OPEN', self.clock()
            raise
        with self.lock: self.state, self.failures = 'CLOSED', 0                                   # success closes the circuit
        return result
```

```cpp
class CircuitBreaker {
    enum class S { Closed, Open, HalfOpen } s_ = S::Closed; int failures_ = 0, threshold_; std::chrono::seconds timeout_;
    std::chrono::steady_clock::time_point openedAt_; std::mutex m_;
public:
    CircuitBreaker(int th, std::chrono::seconds to) : threshold_(th), timeout_(to) {}
    template <class F> auto call(F fn) {
        { std::lock_guard<std::mutex> g(m_);
          if (s_ == S::Open) { if (std::chrono::steady_clock::now() - openedAt_ < timeout_) throw std::runtime_error("circuit open"); s_ = S::HalfOpen; } }
        try { auto r = fn(); std::lock_guard<std::mutex> g(m_); s_ = S::Closed; failures_ = 0; return r; }
        catch (...) { std::lock_guard<std::mutex> g(m_); if (s_ == S::HalfOpen || ++failures_ >= threshold_) { s_ = S::Open; openedAt_ = std::chrono::steady_clock::now(); } throw; }
    }
};
```

**Other fault-tolerance patterns**

- **Timeouts and bounded retries** (H21); **bulkheads** (isolate resources per dependency: separate thread pools/connection pools so one slow service cannot starve everything, like watertight ship compartments).
- **Fallbacks / graceful degradation:** serve cached or default data, hide non-critical features (recommendations) rather than failing the page.
- **Load shedding and backpressure:** reject excess work early (429/503) to protect latency for admitted requests; bounded queues.
- **Redundancy and failover** (H23), **health checks**, **idempotent operations**, **rate limits**, and **chaos engineering** (deliberately inject failures to verify resilience).
- Avoid **cascading failures:** a slow dependency raises latency, holds threads, exhausts pools, and fails callers in turn; timeouts + breakers + bulkheads stop the spread.

**Loopholes:** a breaker with a threshold that is too sensitive flaps; the fallback path itself must be reliable and tested; a breaker per process only sees local failures (acceptable); counting only exceptions and not slow responses misses "brown-outs" (count timeouts and latency); half-open probes should be few.

---

## H23. Redundancy, high availability and disaster recovery

**High availability (HA):** eliminate **single points of failure (SPOFs)** at every layer: multiple app instances behind an LB, replicated databases with automatic failover, multi-AZ placement, redundant LBs, DNS, caches, queues, power/network. **N+1** (spare capacity for one failure), **N+2** for critical systems.

| Mode | Meaning | Trade-off |
| --- | --- | --- |
| **Active-active** | all instances serve traffic | best utilization and fast failover; needs conflict handling and capacity for the loss of a node/region |
| **Active-passive (standby)** | a standby takes over on failure | simpler; wasted capacity; failover takes time (**cold/warm/hot** standby) |

**Scope of failure:** instance → **availability zone** (data center) → **region**. Multi-AZ is standard; multi-region protects against regional outages and reduces latency but costs more and complicates data consistency (replication lag, conflict resolution).

**Disaster recovery (DR)**

- **RPO (Recovery Point Objective):** the maximum acceptable **data loss** measured in time (RPO 5 minutes = may lose up to the last 5 minutes of writes). Determined by backup/replication frequency.
- **RTO (Recovery Time Objective):** the maximum acceptable **downtime** until service is restored.

| DR strategy | RTO / RPO | Cost |
| --- | --- | --- |
| **Backup and restore** | hours / hours | lowest |
| **Pilot light** (minimal core running in another region, scale up on disaster) | tens of minutes / minutes | low |
| **Warm standby** (scaled-down full copy running) | minutes / seconds to minutes | medium |
| **Multi-site active-active** | near zero / near zero | highest |

**Backups:** the **3-2-1 rule** (3 copies, 2 media/types, 1 off-site), **immutable/air-gapped** copies against ransomware, **point-in-time recovery** (base backup + WAL/binlog), encryption, and **regularly test restores** (an untested backup is not a backup). Replication is **not** a backup (it replicates accidental deletes and corruption instantly).

**Operational HA:** automated failover with fencing (avoid **split-brain**), health checks, rolling/canary deploys and fast rollback, runbooks, **game days** and chaos drills, capacity planning for failure (can the remaining nodes carry the load?), dependency mapping (the system's availability is at most the product of its critical dependencies).

**Loopholes:** an HA design that shares one hidden SPOF (single DNS provider, single secrets store, single CI pipeline, one config service); failover that was never tested; replica promoted with replication lag losing recent writes (RPO is not zero with async replication); DR region without enough quota/capacity; backups in the same account/region as production; forgetting data **and** configuration/secrets/infrastructure-as-code in the recovery plan.

---

## H24. Distributed transactions: Saga and the transactional outbox

**Problem:** with **database per service**, one business action (place an order) spans several services/databases (orders, payment, inventory, shipping). A single ACID transaction cannot cover them. **Two-phase commit (2PC)** exists (prepare, then commit) but is **blocking** (participants hold locks while waiting for the coordinator), the coordinator is a SPOF, latency is high, and many NoSQL/message systems do not support it, so it is avoided in microservices.

### Saga pattern

A saga is a **sequence of local transactions**, each in one service. If a step fails, previously completed steps are undone by **compensating transactions** (business-level undo: `refund payment`, `release stock`), not by a rollback.

```
Happy path:  CreateOrder -> ReservePayment -> ReserveInventory -> ScheduleShipping -> ConfirmOrder
Failure at ReserveInventory:   <- RefundPayment (compensate)  <- CancelOrder (compensate)
```

|  | Choreography | Orchestration |
| --- | --- | --- |
| Control | each service reacts to events and emits the next event | a central **orchestrator** tells services what to do and tracks state |
| Coupling | decentralized, loosely coupled | orchestrator knows all steps |
| Visibility | hard to see the whole flow; risk of cyclic dependencies | clear flow, easy monitoring and error handling |
| Best for | few steps, simple flows | complex, many steps (Temporal, Camunda, Step Functions, custom) |

```python
class SagaStep:
    def __init__(self, name, action, compensation): self.name, self.action, self.compensation = name, action, compensation

def run_saga(steps, ctx):
    done = []
    for s in steps:
        try:
            s.action(ctx); done.append(s)                  # local transaction in a service (must be idempotent)
        except Exception as e:
            for d in reversed(done):                       # compensate in reverse order
                try: d.compensation(ctx)                   # compensations must be idempotent and retried until success
                except Exception: log_for_manual_intervention(d, ctx)
            return False, e
    return True, None

order_saga = [SagaStep('payment', reserve_payment, refund_payment),
              SagaStep('inventory', reserve_stock, release_stock),
              SagaStep('shipping', schedule_shipping, cancel_shipping)]
```

**Design rules:** every action and compensation must be **idempotent** and retryable; persist the saga state (so a crash resumes it); use **semantic locks/pending states** (`order.status = PENDING`) because other transactions can see intermediate states (**no isolation**); compensation can fail (retry, alert, manual); some actions are not reversible (email sent): place them last or use "pivot" steps; use timeouts for steps that never respond; **TCC** (Try-Confirm-Cancel) reserves resources first then confirms or cancels.

### Transactional outbox pattern

**Problem (dual write):** a service must update its database **and** publish an event. Doing both separately can fail halfway: the DB commit succeeds but the publish fails (lost event) or the event is published but the commit rolls back (phantom event).

**Solution:** in **one local database transaction**, write the business row **and** an `outbox` row describing the event. A separate **relay** reads the outbox and publishes to the broker, then marks the row sent. Because the outbox row commits atomically with the data, the event is **guaranteed to be published eventually (at-least-once)**.

```
  BEGIN;
    INSERT INTO orders (...);                 -- business change
    INSERT INTO outbox (id, aggregate_id, type, payload, created_at, sent_at) VALUES (...);   -- same transaction
  COMMIT;
  Relay (poller or CDC/Debezium tailing the WAL) -> publish to Kafka -> mark outbox.sent_at
  Consumers: idempotent (inbox table with processed event ids)
```

```sql
CREATE TABLE outbox (id UUID PRIMARY KEY, aggregate_id TEXT, type TEXT, payload JSONB, created_at TIMESTAMP DEFAULT now(), sent_at TIMESTAMP);
CREATE INDEX ON outbox (created_at) WHERE sent_at IS NULL;        -- cheap poll of unsent rows
SELECT * FROM outbox WHERE sent_at IS NULL ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED;   -- multiple relays without double-picking
```

**Loopholes:** the relay delivers **at least once**, so consumers must dedupe (inbox pattern or idempotent handlers); ordering is guaranteed only if the relay preserves it per aggregate (partition key = aggregate id); the outbox table needs cleanup/archival; polling adds latency (CDC is lower latency, more infrastructure); sagas lack isolation, so design for intermediate states; do not use a saga where one database transaction would do (keep related data in one service).

---

## H25. Distributed locks and leader election

**When you need them:** only one worker should run a cron/job, one node should own a shard, mutual exclusion for a shared external resource, preventing duplicate processing. **Prefer alternatives when possible:** idempotent operations, unique constraints, optimistic version checks, queues with a single consumer per partition. Use a lock when you truly need mutual exclusion across processes.

**Redis lock (single instance)**

```python
import uuid
def acquire(r, name, ttl_ms=10000):
    token = str(uuid.uuid4())
    return token if r.set(f'lock:{name}', token, nx=True, px=ttl_ms) else None       # NX = only if absent, PX = auto-expire
RELEASE = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end"
def release(r, name, token): r.eval(RELEASE, 1, f'lock:{name}', token)                # only the owner may release (compare token)
```

The TTL prevents a crashed owner from holding the lock forever; the **unique token** prevents releasing someone else's lock after your TTL expired.

**The fundamental problem and fencing tokens**

```
Client A acquires the lock (TTL 10 s) -> long GC pause / network delay (12 s) -> lock expires
Client B acquires the lock and writes -> Client A wakes up, still believes it holds the lock, and writes: CORRUPTION

Fix: a monotonically increasing FENCING TOKEN issued with each lock grant (1, 2, 3 ...).
The protected resource (database/storage) rejects writes carrying a token lower than the highest it has seen.
```

- **Redlock** (acquire on a majority of N independent Redis nodes) improves availability but still relies on bounded clock drift/pauses; for **correctness-critical** locks use a **consensus-based** system (ZooKeeper, etcd, Consul) that provides leases/ephemeral nodes and fencing/versions. For **efficiency-only** locks (avoid duplicate work occasionally) a simple Redis lock is fine.
- **Database locks** are often the simplest: `SELECT ... FOR UPDATE`, advisory locks (`pg_advisory_lock`), or a lock table row with a unique constraint and lease expiry.
- **ZooKeeper/etcd:** create an **ephemeral** node (deleted when the session dies) or use a **lease** with keep-alive; waiters watch the node ahead of them (no herd).

**Leader election:** one node among many acts as leader (scheduler, primary shard owner, controller). Mechanisms: a **lease** in etcd/ZooKeeper/Kubernetes (the leader renews periodically; if it stops, another acquires), or a consensus protocol (**Raft/Paxos**: nodes vote, a leader needs a majority, each leadership has a **term/epoch number** used for fencing, followers redirect clients to the leader). Odd cluster sizes (3, 5) tolerate `f` failures with `2f + 1` nodes.

**Loopholes:** a lock without TTL deadlocks on crash; a TTL without renewal expires mid-work (renew via a watchdog, but a paused process still cannot renew); clocks and pauses break time-based leases, so use **fencing tokens** at the resource; two leaders at once (**split-brain**) is the failure to design against; releasing without checking the token; a lock service outage must not take down the application (decide fail-open vs fail-closed); holding locks across slow I/O raises contention.

---

## H26. Authentication, authorization, OAuth 2.0 and JWT

**Authentication (AuthN):** who are you? **Authorization (AuthZ):** what may you do? (Sessions vs JWT, cookies and CSRF/XSS are in Part 2, C11.)

**Authentication building blocks**

- **Passwords:** never store plaintext or fast hashes; use a slow, salted, adaptive hash (**bcrypt, scrypt, Argon2id**); enforce rate limits and lockouts; breached-password checks.
- **MFA:** something you know + have + are (TOTP apps, WebAuthn/passkeys, hardware keys; SMS is the weakest).
- **API keys** for server-to-server (hash them at rest, scope them, rotate them); **mTLS** for service identity; short-lived **tokens**; SSO via **SAML/OIDC**.

**Authorization models**

| Model | Idea | Example |
| --- | --- | --- |
| **ACL** | list of who can do what on each object | file permissions |
| **RBAC** | permissions attached to **roles**; users get roles | admin, editor, viewer |
| **ABAC / policy-based** | decisions from attributes of user, resource and context | "doctors can read records of patients in their own hospital during shift" |
| **ReBAC** | relationships (owner, member of) | Google Drive sharing, Zanzibar/OpenFGA |

**Key rule:** enforce authorization **on the server for every request** and on the **object** being accessed (prevent **IDOR**: `GET /orders/123` must check that order 123 belongs to the caller). Deny by default; least privilege.

### OAuth 2.0 (delegated authorization) and OpenID Connect

**Roles:** **resource owner** (user), **client** (the app), **authorization server** (issues tokens), **resource server** (API). OAuth lets a user grant an app **limited access** to their resources **without sharing their password**. **OpenID Connect (OIDC)** adds an **ID token** (JWT with identity claims) on top, so it handles *login* (authentication).

```
Authorization Code flow with PKCE (recommended for web, mobile, SPA):
1. Client creates a random code_verifier, sends code_challenge = SHA256(verifier) and redirects the user to the Authorization Server (client_id, scope, redirect_uri, state)
2. User authenticates and consents at the Authorization Server
3. Authorization Server redirects back to redirect_uri with a short-lived authorization CODE (+ state)
4. Client (back channel) POSTs code + code_verifier + client_id to the token endpoint
5. Authorization Server returns ACCESS TOKEN (short-lived), REFRESH TOKEN (long-lived), and ID TOKEN (OIDC)
6. Client calls the API with  Authorization: Bearer <access_token>;  the API validates the token and scope
```

| Grant | Use |
| --- | --- |
| **Authorization Code + PKCE** | user-facing apps (default) |
| **Client Credentials** | machine-to-machine (no user) |
| **Refresh Token** | get new access tokens without re-login |
| Device Code | TVs and CLIs |
| ~~Implicit~~, ~~Password~~ | **deprecated/avoid** |

**Scopes** limit what a token can do (`orders:read`); `state` prevents CSRF on the redirect; `redirect_uri` must be exactly registered; PKCE protects public clients from code interception.

**JWT validation checklist (resource server)**

1. Parse `header.payload.signature`; verify the **signature** using the issuer's public key from its **JWKS** endpoint (RS256/ES256) or a shared secret (HS256).
2. **Whitelist the algorithm** (reject `alg: none` and algorithm confusion).
3. Check `exp` (not expired), `nbf`, `iss` (trusted issuer), `aud` (this API is the audience).
4. Check scopes/roles/claims for the requested action.
5. Keep access tokens **short-lived** (5 to 15 minutes); rotate refresh tokens; support revocation/denylist or token introspection for opaque tokens.

**Opaque vs JWT access tokens:** opaque tokens are random strings validated by calling the auth server (easy revocation, extra network hop); JWTs are validated locally (fast, scalable) but cannot be revoked before expiry unless you add a denylist.

**Loopholes:** OAuth is **authorization, not authentication** (use OIDC for login); access tokens must not be placed in URLs; storing tokens in `localStorage` exposes them to XSS (prefer HttpOnly cookies or in-memory + refresh cookie); skipping `aud` validation accepts tokens meant for other APIs; long-lived JWTs with no revocation path; authorization checks only in the UI; trusting claims from a client-supplied token without signature verification; microservices re-validating tokens inconsistently (centralize in the gateway but still verify audience and scopes in sensitive services).

---

## H27. Security fundamentals and secrets management

**Threat checklist (OWASP-style)**

| Threat | Defence |
| --- | --- |
| **Injection** (SQL, command, NoSQL) | parameterized queries, input validation, least-privilege DB users, avoid shell calls with user input |
| **XSS** | output encoding/escaping, Content-Security-Policy, HttpOnly cookies, sanitize HTML |
| **CSRF** | SameSite cookies, CSRF tokens, check Origin; token-in-header APIs are immune |
| **Broken access control / IDOR** | server-side authorization on every object, deny by default |
| **SSRF** (server fetches attacker-chosen URLs, e.g. cloud metadata endpoint) | allowlist destinations, block internal IP ranges, require IMDSv2 |
| **Broken authentication** | strong password hashing, MFA, rate limits, secure session handling |
| **Sensitive data exposure** | TLS everywhere, encryption at rest, data minimization, masking in logs |
| **Security misconfiguration** | secure defaults, no debug in production, least privilege IAM, patching |
| **Vulnerable dependencies / supply chain** | dependency scanning, lockfiles, signed artifacts, minimal images |
| **DoS / brute force** | rate limiting, WAF, autoscaling, timeouts, request size limits |
| **Insecure deserialization, path traversal, open redirects** | validate input, avoid unsafe deserializers, canonicalize paths, allowlist redirects |

**Defence in depth:** TLS in transit (including internal traffic/mTLS), **encryption at rest** (disk/DB/object storage, keys in **KMS**; **envelope encryption**: data encrypted with a data key, the data key encrypted with a master key), network segmentation (private subnets, security groups, no public DBs), **least-privilege IAM** (separate roles per service, short-lived credentials), audit logging, backups encrypted, secure SDLC (code review, SAST/DAST, dependency checks, secrets scanning).

**Hashing vs encryption vs encoding:** hashing is one-way (passwords: slow salted hash; integrity: SHA-256); encryption is reversible with a key (AES-GCM for data, RSA/ECDH for key exchange); encoding (base64) is **not** security. **Sign** (HMAC/RSA) to prove integrity and origin.

**Secrets management**

- **Never** commit secrets to git, bake them into images, put them in client-side code, or print them in logs. Secrets: DB passwords, API keys, private keys, tokens, signing keys.
- Store in a **secret manager** (HashiCorp Vault, AWS Secrets Manager / Parameter Store, GCP Secret Manager, Azure Key Vault); inject at runtime (environment variables from the orchestrator, mounted files, or fetched with workload identity).
- Prefer **short-lived, dynamically issued credentials** (Vault dynamic DB users, IAM roles/IRSA) over static long-lived keys; **rotate** regularly and automatically; scope per service; **audit** access.
- Separate secrets per environment; different from config (non-secret); detect leaks with scanners (gitleaks, GitHub secret scanning); if leaked: **revoke and rotate immediately**, then clean history.
- Local development: `.env` files ignored by git with `.env.example` committed (see Part 3, G6).

**Loopholes:** secrets in environment variables can leak via process listings, crash dumps and child processes (acceptable with care, better via secret mounts/manager); "encrypted at rest" does not protect against a compromised application that has the key; security through obscurity (hidden URLs, unguessable but unauthenticated IDs); logging request bodies with passwords/tokens; trusting internal network traffic; CORS misconfiguration (`*` with credentials); rolling your own crypto.

---

## H28. Concurrency control, race conditions, CAS and TOCTOU

The core theory is in Part 2 (A5, A9 for threads/CAS/TOCTOU; B7 to B9 for database isolation and serializability) and Part 4 (O7). The system-design view:

| Race | Example | Fixes |
| --- | --- | --- |
| **Lost update** | two users edit the same profile; last write wins silently | **optimistic locking** (`version` column, `UPDATE ... WHERE id=? AND version=?`), ETag + `If-Match` |
| **Check-then-act (TOCTOU)** | stock check then decrement oversells | atomic conditional update `UPDATE items SET stock = stock - 1 WHERE id=? AND stock > 0`, row lock `FOR UPDATE`, CHECK constraint |
| **Duplicate creation** | two signups with the same email | **unique constraint** (handle the violation), idempotency key |
| **Double spend / concurrent withdrawals** | two withdrawals read the same balance | atomic `UPDATE ... SET bal = bal - ? WHERE bal >= ?`, serializable isolation, per-account serialization via queue/partition |
| **Cache races** | stale value written to cache after an update | delete after write, versioned keys, CAS in the cache, short TTL |
| **Distributed critical section** | cron job runs on every node | lock with fencing (H25), leader election, single consumer per partition |
| **Seat booking** | two users buy the same seat | atomic conditional update on the seat row, seat locks with TTL (Part 4, L7) |

**Approaches:** (1) **avoid shared mutable state** (immutability, per-key single-threaded ownership: partition so that one worker handles all events of one key, e.g. Kafka partition per account); (2) **pessimistic locking** (`SELECT ... FOR UPDATE`: safe under high contention, risks deadlocks and reduced throughput); (3) **optimistic locking/CAS** (version check and retry: best under low contention; wasteful if conflicts are frequent); (4) **atomic database or Redis operations**; (5) **serializable transactions** with retry on serialization failure; (6) **idempotency** so retries are safe.

```sql
-- Optimistic locking
UPDATE accounts SET balance = :new, version = version + 1 WHERE id = :id AND version = :seen;    -- 0 rows affected => conflict: reload and retry
-- Pessimistic locking
BEGIN; SELECT balance FROM accounts WHERE id = :id FOR UPDATE; UPDATE accounts SET balance = balance - :amt WHERE id = :id; COMMIT;
```

**Interview answer:** "I'd make the invariant enforceable by the database: an atomic conditional update or a constraint for correctness, optimistic versioning for user edits, and per-key serialization for hot entities; locks across services only with fencing."

---

## H29. Observability: logs, metrics, traces, alerts and SLOs

**Observability** is the ability to understand a system's internal state from its outputs. The three pillars:

| Pillar | What | Tools | Good practice |
| --- | --- | --- | --- |
| **Logs** | discrete timestamped events | ELK/OpenSearch, Loki, CloudWatch, Splunk | **structured JSON**, levels, a **correlation/request id** in every line, no PII or secrets, sampling for noisy debug logs |
| **Metrics** | numeric time series (counters, gauges, histograms) | Prometheus + Grafana, Datadog, CloudWatch | low-cardinality labels, **histograms for latency** (p50/p95/p99), RED/USE dashboards |
| **Traces** | one request's path across services as **spans** | OpenTelemetry, Jaeger, Zipkin, Tempo | propagate `trace_id` through headers and queues, sample (head/tail), find which hop is slow |

**What to measure**

- **Four golden signals (Google SRE):** **latency**, **traffic**, **errors**, **saturation**.
- **RED (request-driven services):** **R**ate, **E**rrors, **D**uration. **USE (resources):** **U**tilization, **S**aturation, **E**rrors (CPU, memory, disk, network, connection pools, queue depth).
- Business metrics too: signups, orders per minute, payment success rate, queue age, cache hit ratio, replication lag, consumer lag.

**Alerts:** alert on **symptoms users feel** (error rate, latency SLO burn), not every cause (CPU 80%); each alert must be **actionable**, have an owner, a severity, and a **runbook** link; use thresholds over time windows, multi-window burn-rate alerts, and route pages vs tickets; avoid alert fatigue (delete noisy alerts); add **health/readiness endpoints** and **synthetic checks** (probes from outside).

**SLI, SLO, SLA, error budget**

- **SLI (indicator):** a measurable ratio, e.g. `good requests / total requests` or the share of requests under 300 ms.
- **SLO (objective):** the target, e.g. 99.9% of requests succeed over 30 days. **SLA** is the contractual promise with penalties (set weaker than the SLO).
- **Error budget** = `1 - SLO`. For 99.9% over 30 days: `0.001 x 30 x 24 x 60 = 43.2 minutes` of allowed unavailability. If the budget is burning fast, freeze risky releases and fix reliability; if there is budget left, ship faster.
- **Burn-rate alert:** page when the budget is being consumed (for example 14x normal) over short and long windows.

```python
# Minimal instrumentation (Prometheus client)
from prometheus_client import Counter, Histogram
REQS = Counter('http_requests_total', 'requests', ['route', 'status'])
LAT = Histogram('http_request_seconds', 'latency', ['route'], buckets=(0.05, 0.1, 0.25, 0.5, 1, 2.5))
def handle(route, fn):
    with LAT.labels(route).time():
        try: resp = fn(); REQS.labels(route, '200').inc(); return resp
        except Exception: REQS.labels(route, '500').inc(); raise
```

**Loopholes:** averages hide tail latency (use percentiles/histograms); **high-cardinality labels** (user id, request id as a metric label) explode metric storage (put them in logs/traces); logging everything is expensive and a privacy risk; traces without context propagation across queues break; alerting on causes instead of symptoms creates noise; no one owns the alert; dashboards without SLOs do not tell you whether you are okay.

---

## H30. Monitoring, latency, throughput and bottleneck analysis

**Method (do this in interviews and in real life):** (1) define the symptom with numbers (p99 latency, error rate, throughput); (2) **measure before changing anything** (metrics, traces, profiles); (3) find the **most saturated resource** (USE method) or the **slowest span** in a trace; (4) fix the **biggest** bottleneck; (5) re-measure; repeat. Amdahl's law: speeding up a part that is 10% of the time gains at most 10%.

**Common bottlenecks and typical fixes**

| Where | Symptom | Fixes |
| --- | --- | --- |
| **Database** | high CPU/IO, slow queries, lock waits | indexes, query rewrite, cache, read replicas, denormalize, partition/shard, batch writes, connection pooling |
| **N+1 queries** | request makes hundreds of DB calls | joins/batching, `IN` queries, dataloader, caching |
| **Hot key/hot partition** | one shard or cache node saturated | key splitting, local cache, better shard key |
| **Connection/thread pools** | requests queue, timeouts despite idle CPU | right-size pools, async I/O, timeouts, bulkheads |
| **Synchronous chains** | latency = sum of every hop | parallelize independent calls, async/queue non-critical work, cache |
| **Network / payload** | large responses, chatty APIs | compression, pagination, field selection, binary formats, CDN, batching |
| **CPU-bound code** | high CPU, GC pauses | profiling and flame graphs, algorithmic fixes, caching, horizontal scale |
| **Memory/GC** | latency spikes, OOM | reduce allocations, tune GC, fix leaks, bound caches |
| **Disk I/O** | high iowait | SSDs, sequential writes, batching, caching, compaction tuning |
| **Downstream dependency** | slow third-party | timeouts, circuit breaker, caching, async, fallback |
| **Single-threaded/serial section** | adding servers does not help | remove locks, shard the work, queue and parallelize |

**Load testing:** **load test** (expected peak), **stress test** (beyond peak to find the breaking point and failure mode), **spike test**, **soak test** (hours: leaks, slow degradation). Tools: k6, JMeter, Locust, Gatling, wrk. Use realistic data and traffic mix, warm caches, measure percentiles, watch resource saturation, test failure modes (kill a node). **Capacity planning:** measure per-instance capacity at target latency, add headroom (30 to 50%), plan for N+1 and peak events.

**Profiling toolbox:** CPU profilers and **flame graphs** (`perf`, py-spy, async-profiler), `EXPLAIN ANALYZE`, slow query logs, APM traces, `top`/`vmstat`/`iostat`/`ss` (Part 2, D4), heap dumps, lock/contention profilers.

**Interview answer to "the API is slow, what do you do?":** "First measure: look at p95/p99 per endpoint and a trace to see where time goes. Typically I find the database (slow query, missing index, N+1) or a slow downstream call. I'd fix the biggest hop first, then add caching or async processing where it is safe, verify with a load test, and set an SLO alert so regressions are caught."

**Loopholes:** optimizing without measuring; testing with unrealistic data; ignoring tail latency; scaling horizontally a bottleneck that is a shared database or lock; premature optimization versus ignoring a real hotspot; measuring in an environment unlike production.

---

# PART S: HLD PRACTICE SYSTEMS

Each system follows the same template: **Requirements → Estimates → API → Data model → Architecture and flow → Deep dive → Scaling and failures → Security → Observability → Trade-offs**. Numbers are assumptions you should state out loud. Cross-references (H-numbers) point to the fundamentals above.

## S1. URL shortener

**Functional:** create a short URL for a long URL (optional custom alias, optional expiry); redirect short → long; delete; basic click analytics. **Non-functional:** very high availability and low redirect latency (< 50 ms p99), read-heavy (100:1), short codes unguessable if private, no collisions, scales to billions of URLs. **Out of scope:** user accounts detail, billing.

**Estimates:** 100M new URLs/month → 40 writes/s avg (about 120/s peak); reads 100x → 4,000/s avg, about 12,000/s peak. 5 years: 6 billion URLs x about 500 B = **3 TB** (small). Code space: 7 base62 chars = 62^7 ≈ 3.5 trillion, enough. Cache the hot 20%: 4,000 x 86,400 x 0.2 x 500 B ≈ **35 GB** of hot data, fits in Redis.

**API**

```
POST   /v1/urls   { long_url, custom_alias?, expires_at? }  -> 201 { short_url, code, expires_at }   |  409 alias_taken  |  400 invalid_url
GET    /{code}                                              -> 301 or 302  Location: <long_url>      |  404 / 410 Gone (expired)
DELETE /v1/urls/{code}                                      -> 204
GET    /v1/urls/{code}/stats                                -> { clicks, by_day, by_country }
```

**301 vs 302:** `301` (permanent) is cached by browsers, so later clicks skip your servers (cheaper, but you lose analytics and cannot change/expire the target). `302/307` (temporary) always hits you: use it when you need analytics or expiry. Most services use 302.

**Data model** (a key-value access pattern: lookup by code)

```
urls(code PK, long_url, user_id, created_at, expires_at, is_active)      -- DynamoDB / Cassandra / sharded MySQL by hash(code)
clicks(code, ts, ip_hash, ua, country)                                   -- append-only event stream -> Kafka -> analytics store (ClickHouse/BigQuery)
```

**Architecture and flow**

```
              +--> CDN/edge (optional for hot redirects)
Client -> LB -> App servers (stateless) --read--> Redis cache (code -> long_url) --miss--> URL DB (sharded, replicated)
                  |   write: get unique ID from ID generator -> base62 -> insert (unique constraint on code)
                  +--> Kafka 'clicks' topic --> analytics consumers
Background: expiry cleaner, malware/phishing scanner
```

Read flow: look up `code` in Redis; on a miss read the DB, populate the cache with TTL; if expired or missing return 404/410; emit a click event asynchronously; respond 302.

**Deep dive: generating the short code**

| Approach | How | Pros | Cons |
| --- | --- | --- | --- |
| **Counter + base62** | globally unique number encoded in base62 | no collisions, short | sequential codes are guessable (add a shuffle/Feistel permutation or use random for private links); needs a distributed counter |
| **Range allocation** | a coordinator (ZooKeeper/etcd/DB) hands each app server a block (e.g. 1,000,000 IDs); the server uses it locally | no per-request coordination, survives coordinator downtime for a while | lost range if a server dies (fine, IDs are plentiful) |
| **Snowflake ID → base62** | H12 | decentralized | longer code (about 11 chars for 64 bits) |
| **Random 7 chars + unique constraint + retry** | pick random, insert, retry on conflict | unguessable | collision probability grows with fill ratio (retry loop) |
| **Hash of the URL (MD5/SHA, take 7 chars)** | deterministic dedupe | same URL gives same code | collisions need handling; cannot support different aliases per user |
| **Pre-generated key service (KGS)** | offline job fills a table of unused keys; servers take batches | very fast, no collisions | extra component, must mark used atomically |

```python
ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
def base62(n: int) -> str:
    if n == 0: return ALPHABET[0]
    out = []
    while n: n, r = divmod(n, 62); out.append(ALPHABET[r])
    return ''.join(reversed(out))
class RangeAllocator:                       # each app server holds a block of IDs from the coordinator
    def __init__(self, fetch_block, size=1_000_000): self.fetch, self.size, self.cur, self.end = fetch_block, size, 0, 0
    def next_id(self):
        if self.cur >= self.end: self.cur = self.fetch(self.size); self.end = self.cur + self.size    # atomic INCRBY on the coordinator
        self.cur += 1; return self.cur - 1
```

```cpp
std::string base62(uint64_t n) {
    static const char* A = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    if (n == 0) return "0"; std::string s; while (n) { s.push_back(A[n % 62]); n /= 62; } std::reverse(s.begin(), s.end()); return s;
}
```

**Scaling and failures:** shard the URL table by `hash(code)` (consistent hashing); read replicas and Redis cluster for reads; the redirect path touches only cache + one DB lookup; multi-region read replicas with GeoDNS; if the ID coordinator is down, servers continue on their current block; if Redis is down the DB serves (with rate limiting); **hot link** (viral URL) is served from cache/CDN; expired links removed by a background job (TTL index/lazy delete); thundering herd on a cold hot key uses single-flight (H7).

**Security and abuse:** validate scheme (`http/https` only, block `javascript:`), blocklist and malware/phishing scanning (Safe Browsing), rate-limit creation per user/IP (H19), CAPTCHA for anonymous creation, no open-redirect abuse via reserved paths, unguessable codes for private links, HTTPS only, hide user identity in analytics (hash IPs).

**Observability:** redirect latency percentiles, cache hit ratio, 404 rate, creation rate, DB shard load, click-stream lag, error budget SLO (99.99% redirect availability).

**Trade-offs:** 301 vs 302; sequential vs random codes (guessability vs simplicity); strong consistency for create (unique alias) vs eventual for analytics; SQL (simple, transactional alias uniqueness) vs NoSQL (scale, key-value fit); deduplicating identical long URLs (saves space, complicates per-user analytics).

---

## S2. Rate limiter (as a distributed service)

**Functional:** limit requests per **user / API key / IP / endpoint** using configurable rules (e.g. 100 req/min free, 1,000 paid; login 5/min); return **429** with `Retry-After`; support burst allowances. **Non-functional:** adds **< 5 ms**; works across many API servers; highly available (**fail-open** unless security critical); accurate enough; easy to change rules at runtime; low memory per key.

**Estimates:** 10M active users, 100,000 limit checks/s at peak across the fleet; each check is one Redis round trip (about 0.5 ms) with about 100k ops/s per Redis node → a **small sharded Redis cluster** (e.g. 4 to 8 shards). Memory: 10M keys x about 100 B = **1 GB** (tiny).

**API (internal) and contract**

```
check(client_id, resource, cost=1) -> { allowed: bool, limit, remaining, reset_at, retry_after }
HTTP to clients: 429 Too Many Requests + Retry-After + X-RateLimit-Limit / -Remaining / -Reset
Rules store:  rule { match: {tier, endpoint, key_type}, algorithm: token_bucket, rate, burst, window }
```

**Architecture**

```
Client -> API gateway / middleware --(1) load rule (cached locally, refreshed from config store)
                |  (2) atomic check+update in Redis (Lua)  ----> Redis Cluster (shards by hash(key)), replicas for failover
                |  allowed -> forward to the service      rejected -> 429 immediately (cheap)
                +-- async metrics -> Prometheus; rule changes via admin API -> config store (etcd/DB) -> pushed/polled to gateways
```

**Deep dive**

- **Algorithm:** token bucket per key (bursts, O(1) state: `tokens`, `last_refill`) executed in **one Lua script** so check-and-decrement is atomic (no race between instances); sliding window counter when you need strict windows (H19 has code and the Lua script).
- **Key design:** `rl:{tier}:{client_id}:{endpoint}`; TTL set to about twice the refill period so idle buckets vanish.
- **Rules:** cached in each gateway with periodic refresh, so a rule lookup costs no network call; hierarchical limits (per user AND per IP AND global) check the cheapest/strictest first.
- **Race conditions:** GET-then-SET is wrong; use atomic scripts or `INCR`.
- **Multi-region:** option A: one global limiter (cross-region latency, strongest accuracy); option B: **per-region limits with a share of the global quota** reconciled asynchronously (fast, slightly approximate); option C: local in-memory token buckets that sync counters in the background for ultra low latency.
- **Hot keys:** a single abusive IP/user can overload one Redis shard (shard load is per key): protect Redis with a local pre-filter (in-process bucket) and cheap blocklist at the edge/WAF.

**Failure handling:** Redis down → **fail-open** (allow, alert) for general APIs, **fail-closed** for login/OTP/payment; local fallback limiter with conservative limits; replica promotion with Redis Sentinel/Cluster; timeouts on the limiter call (a slow limiter must not become the outage).

**Security:** do not trust `X-Forwarded-For` unless set by your own proxy; combine IP with user/API key; rate limit **authentication endpoints** hardest; protect the admin/rules API; avoid leaking exact limits to attackers where sensitive.

**Observability:** allowed vs rejected rates by rule, top limited keys, limiter latency p99, Redis CPU/memory, fail-open events, 429 rate per client.

**Trade-offs:** accuracy vs latency (central atomic vs local approximate); fixed window (simple, 2x burst) vs token bucket (burst control); fail-open vs fail-closed; library in each service (no extra hop) vs a standalone service (central policy); per-IP vs per-user fairness.

---

## S3. Notification service

**Functional:** send **push, SMS, email, in-app** notifications triggered by events (order shipped, OTP, promotions); **templates** and localisation; **user preferences** (channels, opt-out, quiet hours); scheduling; delivery tracking (sent/delivered/opened); bulk campaigns. **Non-functional:** OTP/transactional delivered within seconds, high throughput during campaigns, **at-least-once** delivery without duplicates visible to users, resilient to provider outages, auditable.

**Estimates:** 50M users x 5 notifications/day = 250M/day → about 2,900/s average; a campaign burst of 10M in 10 minutes → about **17,000/s** (design for 100k/s with queues). Storage: 250M x 300 B ≈ 75 GB/day of history (keep 30 to 90 days hot, archive the rest).

**API**

```
POST /v1/notifications { user_id | segment_id, event_type, data{...}, channels?, priority: 'HIGH|NORMAL|LOW', idempotency_key, send_at? }  -> 202 { notification_id }
GET  /v1/notifications/{id}   -> status per channel (queued/sent/delivered/failed/opened)
PUT  /v1/users/{id}/preferences { channels: {...}, quiet_hours: {...}, unsubscribed: [...] }
Webhooks from providers (delivery receipts, bounces) -> POST /internal/receipts
```

**Architecture**

```
Producers (order, auth, marketing services) -> Notification API -> validate, dedupe (idempotency key), load preferences & template
   -> Kafka topics by priority/channel:  otp.sms (isolated, high priority) | transactional.email | marketing.push ...
   -> Channel workers (consumer groups): render template -> check rate limits/quiet hours -> call provider
        push: FCM / APNs     sms: Twilio / local aggregators     email: SES / SendGrid     in-app: WebSocket/DB inbox
   -> status events -> Status store (Cassandra/DynamoDB) + retries via delay topics -> DLQ
Preference service (SQL + cache), Template service, Device-token registry, Scheduler (delayed sends), Campaign service (segment fan-out)
```

**Deep dive**

- **Priority isolation:** separate queues/consumer pools for OTP/transactional vs marketing, so a campaign cannot delay a login code.
- **Fan-out for campaigns:** the campaign service reads the segment in **pages**, publishes batches of user ids to the queue; workers render per user; apply **throttling** to protect providers (providers rate limit) and avoid waking the whole user base at once; schedule by user timezone.
- **Reliability:** idempotency key to dedupe producer retries; **at-least-once** with retries and exponential backoff + jitter (H21); poison messages go to a **DLQ**; **provider failover** (secondary SMS/email vendor) behind an adapter interface (Part 4, P7); channel fallback (push failed → SMS for important events).
- **Preferences and compliance:** per-type/channel opt-outs, **unsubscribe links**, quiet hours, do-not-disturb, regulatory rules (e.g. DLT registration in India, GDPR/CAN-SPAM); transactional vs marketing classification.
- **Dedupe and noise control:** per-user frequency capping, collapse keys for push (replace older), batching/digests.
- **Device tokens:** register on login, remove invalid tokens when APNs/FCM report them as unregistered; multiple devices per user.
- **Delivery tracking:** providers send webhooks (delivered, bounced, complaint); store events and update status; track opens/clicks with signed tracking URLs.
- **Template rendering:** versioned templates, variables validated, HTML escaping, localisation fallback.

**Scaling and failures:** workers scale on queue lag; Kafka partitions by `user_id` (ordering per user) or by notification id; cache preferences in Redis with invalidation events; provider outage → circuit breaker + failover + queue buffering; duplicate prevention at provider (their idempotency/message id); regional deployment for latency.

**Security and privacy:** PII (phone, email) encrypted, access-controlled; never include sensitive data in push payloads or SMS beyond need; sign and verify webhooks; per-tenant API keys and quotas; abuse protection (cannot spam arbitrary numbers: rate limits, verification); audit logs.

**Observability:** queue lag per topic, send success rate per provider/channel, end-to-end latency for OTP (alert if above 5 s), bounce/complaint rate (reputation), DLQ size, cost per channel.

**Trade-offs:** at-least-once with dedupe vs at-most-once (never risk spamming); synchronous send (simple, couples callers to providers) vs queue (resilient, async status); one multi-channel pipeline vs per-channel services; build vs buy providers; push-based fan-out per user vs pull for in-app inbox.

---

## S4. Chat application (WhatsApp/Messenger style)

**Functional:** 1:1 and **group** chat; message delivery status (sent, delivered, read); **online presence** and last seen; message history sync across devices; media sharing; push notifications when offline; (extension: end-to-end encryption). **Non-functional:** low latency (< 200 ms delivery online), **no message loss**, per-conversation **ordering**, high availability, millions of concurrent connections.

**Estimates:** 50M DAU x 40 messages/day = **2B messages/day → 23,000/s** average, about 70,000/s peak. Storage: 2B x 100 B = **200 GB/day ≈ 73 TB/year** (text only; media goes to object storage). Connections: 10M concurrent WebSockets at about 100k per server → **100+ gateway servers**.

**API and events**

```
WebSocket (after authenticated upgrade):
  client -> server : send { client_msg_id, conversation_id, body, type }      server -> client: ack { client_msg_id, message_id, seq }
  server -> client : message { message_id, conversation_id, sender, body, seq, ts }     client -> server: delivered / read receipts
REST: GET /v1/conversations/{id}/messages?before_seq=...&limit=50   POST /v1/groups   POST /v1/media/upload-url   PUT /v1/devices/{id}/push-token
```

**Architecture**

```
Client <--WebSocket--> Gateway servers (stateful connections)   <--> Presence & routing map in Redis (user_id -> gateway_id, TTL by heartbeat)
                           |  
                      Chat service (stateless): authz, assign seq, persist, route
                           |--> Message store (Cassandra/HBase: partition = conversation_id, clustering = seq)   [durable BEFORE ack]
                           |--> Kafka (events: delivery, fan-out, search indexing, analytics)
                           |--> Pub/sub backplane (Redis/NATS): deliver to the recipient's gateway  -- online
                           +--> Push notification service (APNs/FCM) -- offline recipients
Media: client uploads directly to object storage via presigned URL, sends a message with the media URL; CDN for download
```

**Message flow (1:1):** A sends `{client_msg_id}` → gateway → chat service assigns `message_id` (Snowflake, H12) and a per-conversation `seq` → **persist** → ack to A ("sent") → look up B's gateway in Redis → publish to that gateway → B receives and sends a delivery receipt ("delivered") → B reads, sends "read". If B is offline: store only, trigger push; on reconnect B **syncs from its last seen seq**.

**Deep dive**

- **Ordering:** a monotonic **per-conversation sequence number** assigned at one place (the partition owner/leader for that conversation); clients order by `seq`, not by timestamp (clock skew).
- **Reliability/dedupe:** at-least-once: the client retries with the same `client_msg_id` until acked; the server dedupes on `(conversation_id, client_msg_id)`; persist before ack.
- **Groups:** small groups (up to about 100s): **fan-out on write** (deliver/insert into each member's inbox or push the event to each member); very large groups/channels: **fan-out on read** (members pull from the shared conversation log) to avoid millions of writes per message.
- **Presence:** heartbeats every 15 to 30 s refresh a Redis key with TTL; "online" = key exists; publish presence changes only to a user's subscribed friends/active chat partners (not everyone) and **debounce** flapping; last-seen is updated lazily.
- **Multi-device:** each device has its own connection and sync cursor (`last_seq` per conversation); messages are pushed to all devices; read state syncs.
- **Storage choice:** write-heavy, time-ordered, partitioned by conversation: wide-column store (Cassandra/HBase/Scylla). Hot partitions for huge groups are bucketed (`conversation_id + month`).
- **Connection management:** heartbeats, reconnect with exponential backoff + jitter, resume from last seq, graceful gateway drain on deploys; gateways are the only stateful tier.
- **Encryption:** end-to-end (Signal protocol: server relays ciphertext, keys on devices) changes server-side features (no server-side search/spam scanning of content); transport TLS regardless.

**Scaling and failures:** gateways scale horizontally behind an L4 LB; routing via the Redis map (shard by user id) + pub/sub backplane; gateway crash → clients reconnect, undelivered messages are re-sent from the store using seq; Cassandra replication factor 3 with quorum writes; multi-region with users homed to a region; backlog protections: bounded per-connection send queues and slow-consumer disconnect.

**Security and abuse:** authenticate on WebSocket upgrade (token), authorize conversation membership on every send, rate limit per user, spam/abuse detection (report/block), media scanning, retention and deletion policies, protect against enumeration of users.

**Observability:** connected clients per gateway, message delivery latency p99 (send → deliver), undelivered backlog, reconnect rate, push success rate, store write latency, error rates.

**Trade-offs:** WebSocket (real-time, stateful) vs long polling (simple, heavier); per-conversation single sequencer (ordering, potential hot spot) vs timestamps (no ordering guarantees); fan-out on write vs read for groups; storing messages server-side (history, search) vs device-only (privacy); strong vs eventual consistency for read receipts and presence (eventual is fine).

---

## S5. File-storage service (Dropbox / Google Drive style)

**Functional:** upload/download files; **sync across devices**; folders; **versioning** and restore; **sharing** with permissions (view/edit, link share); large files with resume; trash and recovery; (extension: collaborative editing, search). **Non-functional:** **durability above all** (never lose data), high availability for metadata, efficient bandwidth (sync only what changed), strong consistency for metadata, scalable storage.

**Estimates:** 10M users x 2 GB average = **20 PB** of logical data (3x replication would be 60 PB; **erasure coding** gives about 1.4x). Files: 100 files/user = 1B files. Metadata about 1 KB/file → about 1 TB (fits a sharded SQL). Upload traffic: 10M users x 20 MB/day = 200 TB/day ≈ 2.3 GB/s average. Sync notifications: about 1M active clients connected.

**API**

```
POST /v1/files/init   { path, size, content_hash, chunk_hashes[] } -> { upload_id, missing_chunks[] + presigned URLs }     (resumable, dedupe aware)
PUT  <presigned URL>  (chunk bytes -> object storage directly)
POST /v1/files/commit { upload_id }         -> creates a new file version atomically
GET  /v1/files/{id}/download                -> signed CDN URL (range requests)
GET  /v1/changes?cursor=...                 -> list of metadata changes since the cursor (sync)
POST /v1/files/{id}/share { user|link, permission, expires }      GET /v1/files/{id}/versions   POST /v1/files/{id}/restore
```

**Architecture**

```
Client app (watches folder, chunks files, hashes) --HTTPS--> API/Metadata service --> Metadata DB (sharded SQL by user/namespace)
    |   chunks (4 MB, SHA-256) -- presigned PUT --> Block store = object storage (immutable, content-addressed, erasure coded, multi-AZ)
    |<--- change notifications (long poll / WebSocket / push) <--- Notification service <--- change events (Kafka)
Background: virus scan, thumbnails/previews, GC of unreferenced chunks, version pruning, search indexing, replication to second region
Downloads: CDN with signed URLs (or direct from object storage for private/large)
```

**Deep dive**

- **Chunking and delta sync:** split files into 4 MB chunks (or content-defined chunking so inserting bytes does not shift every chunk); identify each chunk by its **content hash**; a modified file uploads **only changed chunks**; a file version = ordered list of chunk hashes.
- **Deduplication:** if a chunk hash already exists (any user, or per user for privacy), skip uploading it (cross-user dedupe saves space but has side channels; many services restrict it).
- **Metadata vs blocks:** **blocks** are immutable blobs in object storage (cheap, scalable); **metadata** (namespace tree, file → versions → chunk list, permissions, owner, quota) lives in a **strongly consistent transactional database** sharded by user/namespace; a `commit` is one metadata transaction (all chunks must exist first).
- **Sync protocol:** each client keeps a **cursor**; it asks "changes since cursor" and gets a delta of metadata changes; a **notification channel** tells idle clients to poll. Offline edits are uploaded later; **conflict** handling: if two devices edit the same base version, keep both ("conflicted copy") or merge; never silently overwrite.
- **Resumable uploads:** the server tracks which chunks of an `upload_id` arrived; the client re-requests missing chunk URLs after a failure; incomplete uploads expire and their chunks are garbage collected.
- **Versioning and trash:** every commit creates a new version pointing to chunk hashes; restoring = pointing the file to an older version; **GC** deletes chunks no version references (reference counts or mark-and-sweep with a safety delay to avoid races with in-flight uploads); retention policies.
- **Sharing and permissions:** ACL table `(resource_id, principal, role)`; folder inheritance; link sharing with unguessable tokens, expiry and password; check permissions on every API call and signed URL generation.
- **Durability:** replicate or erasure-code across AZs; checksums verified on read and by periodic **scrubbing**; cross-region replication for DR; backups of metadata with point-in-time recovery.
- **Large files and bandwidth:** parallel chunk upload, adaptive chunk size, compression where useful, client-side rate limiting, LAN sync between devices (optional).

**Scaling and failures:** stateless API tier; metadata DB sharded by namespace with replicas; hot namespaces (a viral shared folder) are cached and read-replicated; object storage scales horizontally; notification channel is the stateful part (WebSocket/long poll with a pub/sub backplane); upload failure at any chunk is retried independently; metadata DB failover with synchronous replication for no-loss commits; quotas enforced transactionally.

**Security:** TLS, encryption at rest (per-user/per-file keys via KMS; optional client-side end-to-end), authz on every request and signed-URL generation, short-lived presigned URLs scoped to one key, malware scanning, audit logs for sharing, data residency/compliance, rate limits and abuse controls on public links.

**Observability:** upload/download success and latency, sync lag (time from commit to other device seeing it), chunk dedupe ratio, storage growth, GC backlog, error budget on metadata API, integrity-check failures (alert at any non-zero rate).

**Trade-offs:** fixed vs content-defined chunking (simplicity vs better delta/dedupe); global dedupe (storage savings vs privacy leaks and complexity); server-side vs client-side encryption (features vs privacy); replication vs erasure coding (latency/simplicity vs storage cost); strong consistency for metadata (needed) vs eventual for blocks (immutable so safe); polling vs push for sync.

---

## S6. News feed / social timeline (Twitter, Instagram style)

**Functional:** users **post** (text/media); **follow** others; the **home timeline** shows posts from followed accounts (reverse-chronological or ranked); likes/comments; pagination; (extension: trending, notifications). **Non-functional:** timeline load < 200 ms p99, very read-heavy, eventual consistency is fine (a post may appear seconds late), high availability, handles **celebrity accounts** with tens of millions of followers.

**Estimates (from H4):** 100M DAU; 2 posts/user/day → 200M posts/day ≈ **2,300 writes/s** (7k peak); each user opens the feed \~10 times x 20 posts → about 20B post reads/day ≈ **230k/s** (700k peak). Average follows 200. Storing a timeline of 800 post ids per active user: 100M x 800 x 8 B = **640 GB** in Redis (shardable). Posts: 200M x 300 B = 60 GB/day; media in object storage + CDN.

**API**

```
POST /v1/posts         { text, media_ids[] }                  -> 201 { post_id }
GET  /v1/timeline?limit=20&cursor=<post_id|ts>               -> { posts[], next_cursor }     (cursor pagination, not offset)
POST /v1/follow/{user_id}   DELETE /v1/follow/{user_id}
POST /v1/posts/{id}/like
```

**Data model**

```
posts(post_id PK [Snowflake: time-ordered], author_id, text, media_urls, created_at)      -- sharded by post_id or author_id
follows(follower_id, followee_id, created_at)   -- two access paths: who do I follow (by follower) / who follows X (by followee): store both (graph DB or two tables)
timeline:{user_id}  -> Redis sorted set / list of (post_id) newest first, capped at ~800
likes(post_id, user_id) + counters(post_id -> count)   -- counters via async aggregation / Redis INCR, not row updates
```

**The core decision: how to build the timeline**

| Approach | How | Pros | Cons |
| --- | --- | --- | --- |
| **Fan-out on read (pull)** | on request, fetch recent posts of everyone you follow and merge | cheap writes, always fresh | slow reads (200 queries per feed), heavy at 230k reads/s |
| **Fan-out on write (push)** | on post, append the id to each follower's precomputed timeline | **very fast reads** (one cache read) | write amplification: 1 post x followers; a celebrity with 10M followers = 10M writes; wasted work for inactive users |
| **Hybrid (what real systems do)** | push for normal users; **do not push celebrity posts**; at read time merge the precomputed timeline with recent posts pulled from the celebrities you follow | bounded write cost and fast reads | more complex read path |

```
Post flow:   client -> Post service -> posts DB  + media -> object store/CDN
                          \-> Kafka 'post_created' -> Fan-out workers -> for each ACTIVE follower (skip celebrities and inactive users):
                                                         ZADD timeline:{follower} post_id ; ZREMRANGEBYRANK keep newest 800
Read flow:   client -> Feed service -> ZREVRANGE timeline:{me} (post ids) + pull recent posts of followed celebrities
                          -> merge by time/rank -> hydrate: batch get post bodies/authors/like counts from cache (miss -> DB)
                          -> ranking/filtering (mute, block, privacy) -> response page + next cursor
```

```python
CELEB_THRESHOLD = 100_000                                   # followers above this are NOT fanned out
def on_post_created(post, followers_of, redis, is_active):
    if followers_of.count(post.author) >= CELEB_THRESHOLD: return          # pull at read time instead
    for batch in chunked(followers_of.iter(post.author), 1000):           # page through followers
        pipe = redis.pipeline()
        for f in batch:
            if is_active(f):                                              # inactive users get their feed built when they return
                pipe.zadd(f'timeline:{f}', {post.id: post.ts}); pipe.zremrangebyrank(f'timeline:{f}', 0, -801)
        pipe.execute()
def read_timeline(me, redis, celeb_follows, recent_posts_of, limit=20, before=None):
    ids = redis.zrevrangebyscore(f'timeline:{me}', before or '+inf', '-inf', start=0, num=limit)
    ids += [p.id for c in celeb_follows(me) for p in recent_posts_of(c, before, limit)]   # pull side
    return sorted(set(ids), reverse=True)[:limit]                          # Snowflake ids sort by time
```

**Deep dive**

- **Cursor pagination:** use the last post id/time seen (Snowflake ids are time-sortable), never `OFFSET` (new posts shift pages; slow).
- **Ranking:** start with reverse chronological; ranked feeds score candidates (recency, affinity with the author, engagement, content type) with a lightweight ML model after candidate generation; keep ranking out of the fan-out path.
- **Hot keys:** a viral post's like counter and a celebrity's posts get huge traffic: cache aggressively (local cache + Redis), shard counters (`post:{id}:likes:{0..9}` and sum), serve from CDN where possible.
- **Consistency:** eventual; the author sees their own post immediately (read-your-writes: merge the author's own latest posts at read time); deletes/blocks must filter at read time (privacy), not only at fan-out.
- **Follow/unfollow:** following adds recent posts of the followee to the timeline (backfill); unfollowing filters at read time and lazily purges.
- **Social graph storage:** adjacency lists by user (sharded); cache follower/followee id lists; counts approximate.
- **Media:** upload via presigned URLs, transcoding async, CDN delivery (H13, H8).

**Scaling and failures:** fan-out workers scale on Kafka lag (partition by author id); Redis timelines are a **cache**: if a node is lost, rebuild from the social graph + posts (pull path) lazily; timeouts and partial results (return the feed without ranking or without celebrity merge rather than failing); shard Redis by user id with replicas; multi-region with async replication; backfill and replay jobs when the pipeline lags.

**Security and privacy:** private/protected accounts, blocked/muted users filtered at read, authorization on post visibility, abuse/spam detection, rate limits on posting/following, content moderation pipeline.

**Observability:** feed latency p99, fan-out lag (post → visible in followers' timelines), cache hit ratio, Kafka lag, fan-out queue depth for large accounts, error rate, timeline rebuild rate.

**Trade-offs:** push vs pull vs hybrid (read speed vs write cost vs complexity); precomputed timeline storage cost vs compute at read; freshness vs ranking quality; strong vs eventual consistency; fan-out to inactive users (waste) vs cold-start latency when they return.

---

## S7. E-commerce backend (catalog, cart, checkout, payments, inventory)

**Functional:** browse/search catalog, **cart**, **checkout**, **payment**, **order** tracking, **inventory** management, shipping, notifications, returns/refunds. **Non-functional:** catalog reads extremely fast and highly available; **never oversell** (or define the policy); **never charge twice or lose an order**; handle **flash-sale** spikes (100x); PCI compliance (never store raw card data).

**Estimates:** 10M DAU x 100 page views = 1B views/day ≈ **12k/s** (36k peak); 1M orders/day ≈ 12/s average, but a flash sale can push **10k order attempts/s** for a few minutes for a handful of hot items. Catalog 50M products x 5 KB = 250 GB (search index similar); orders 1M/day x 2 KB = 2 GB/day.

**Services (each owns its data)**

```
Clients -> CDN -> API gateway -> | Catalog svc (product DB + Elasticsearch search, Redis cache)
                                 | Cart svc (Redis/DB by user)        | Pricing/Promotions svc
                                 | Inventory svc (stock + reservations)| Order svc (orders DB, sharded by user_id)
                                 | Payment svc (gateway adapter, idempotency, webhooks) | Shipping/Logistics svc
                                 | Notification svc | User/Auth svc | Review/Recommendation svc
   Events (Kafka + outbox): OrderCreated, PaymentCaptured, StockReserved, OrderShipped ... ; search index fed by CDC from the catalog DB
```

**Browse and search:** product data in a relational/document DB as the source of truth; **Elasticsearch/OpenSearch** for full-text, filters and facets, updated through CDC/events (eventual consistency acceptable); aggressive caching of product pages (CDN + Redis); images on CDN; recommendations precomputed.

**Checkout flow as a saga (H24)**

```
1. Order svc: create order status=PENDING (idempotency key from the client prevents duplicates)
2. Inventory svc: RESERVE stock for each item with a hold that EXPIRES (e.g. 10 min)           [compensation: release]
3. Payment svc: authorize/charge via gateway with the order's idempotency key                    [compensation: refund/void]
4. On success: order status=CONFIRMED, inventory reservation -> committed (stock decremented), publish OrderConfirmed (outbox)
5. Shipping svc picks up -> notifications -> tracking
Failure at any step: run compensations in reverse (release stock, void payment), set status=FAILED/CANCELLED, notify user.
```

**Inventory consistency (the classic interview question)**

```sql
-- Atomic decrement: no oversell, no explicit lock needed
UPDATE inventory SET available = available - :qty, reserved = reserved + :qty
WHERE sku = :sku AND available >= :qty;          -- rows_affected = 0 => out of stock
```

- For **flash sales**, the single hot row becomes the bottleneck (row lock contention): keep the stock counter in **Redis** (`DECRBY` with a Lua check `if stock >= qty`), accept orders in memory at 100k/s, and reconcile asynchronously to the database; or **split stock into N buckets** (`stock:sku:{0..9}`) to spread contention; use a **virtual waiting room/queue** to admit buyers at a rate the backend can handle; pre-warm caches; rate-limit per user.
- **Reservation with TTL** (similar to seat locks, Part 4 L7) so abandoned carts return stock; a periodic job releases expired holds.
- **Overselling policy:** strict (never) vs allow small oversell with backorder/cancellation; always reconcile inventory with warehouse counts.

**Payments (critical correctness)**

- Use a **payment gateway** (Razorpay/Stripe/PayU); **never store card numbers/CVV** (PCI-DSS): use hosted fields/tokenization, store only tokens and last4.
- **Idempotency key** on charge calls (H20); handle **timeouts** by querying payment status, not blindly retrying.
- **Webhooks:** the gateway notifies asynchronously (captured/failed/refunded): verify the **signature**, process **idempotently**, tolerate out-of-order and duplicate events; reconcile daily with gateway settlement reports.
- State machine: `INITIATED → AUTHORIZED → CAPTURED → (REFUNDED)` or `FAILED`; store every attempt; amounts as integer minor units.

**Data model highlights**

```
products(id, title, attrs JSON, price, seller_id)         inventory(sku, warehouse_id, available, reserved)
carts(user_id, items[], updated_at)  [Redis with TTL or DB]
orders(id, user_id, status, total, address, created_at)   order_items(order_id, sku, qty, unit_price)   -- sharded by user_id
payments(id, order_id, gateway_ref, status, amount, idempotency_key UNIQUE)     outbox(...)
```

**Scaling and failures:** stateless services + autoscaling; catalog served mostly from cache/CDN; orders DB sharded by `user_id` with replicas (read-your-orders from the leader); async fan-out of events via Kafka; **circuit breakers and timeouts** around payment/shipping providers; degrade gracefully (recommendations/reviews off first); queue-based order intake during spikes; **price and stock re-validation at checkout** (the cart may be stale); cart persisted so users do not lose it; backups and DR for orders and payments.

**Security and compliance:** PCI scope minimization, TLS, authz on orders (IDOR), fraud/risk checks (velocity limits, address/device signals), bot protection on sales (CAPTCHA, rate limits), PII protection and data retention rules, signed webhooks, admin audit logs.

**Observability:** conversion funnel (view → cart → checkout → paid), payment success rate by gateway/bank, order latency, inventory mismatch/oversell count, stuck sagas (orders PENDING longer than N minutes), search latency, cache hit ratio, queue lag.

**Trade-offs:** strict consistency for stock/payments vs eventual for search/recommendations; saga (microservices) vs a single transactional monolith DB (simpler for small scale); Redis-first inventory (speed, risk on failure) vs DB-first (safe, slower); reserve at add-to-cart (bad: hoarding) vs at checkout (fair); build vs buy payments/search.

---

## S8. Job queue and distributed task scheduler

(The single-process design with a heap and thread pool is in Part 4, L12; this is the distributed, fault-tolerant version.)

**Functional:** submit **one-off, delayed and recurring (cron)** jobs; run them **at least once** on a worker fleet; **retries** with backoff; **priorities**; cancel; query status; per-tenant quotas. **Non-functional:** survives scheduler and worker crashes with **no lost jobs**, scales to millions of jobs/day, accurate timing (seconds), **no duplicate concurrent execution** of the same job run, observable.

**Estimates:** 50M jobs/day ≈ 580/s average, 5k/s peak; 1 KB per job record → 50 GB/day (retain 7 to 30 days; archive). Workers: if the average job takes 2 s, 5k/s peak needs about 10,000 concurrent slots (Little's law).

**API**

```
POST /v1/jobs { type, payload, run_at? | delay?, cron?, priority, max_retries, timeout_s, idempotency_key } -> 201 { job_id }
GET  /v1/jobs/{id} -> { status: PENDING|RUNNING|SUCCEEDED|FAILED|CANCELLED, attempts, last_error, next_run }
DELETE /v1/jobs/{id}
```

**Architecture**

```
             +-------------------------+       poll due jobs (partitioned)      +--> Queue (Kafka/SQS/Redis) per priority
Clients -> API -> Job store (SQL/Cassandra)  <--- Scheduler instances (N, each owns a partition range) ---+
             |  jobs(id, type, payload, run_at, status, priority, attempts, lease_until, cron, tenant)        |
             +-------------------------+                                                                    v
                          ^ status updates, heartbeats, results                                       Worker pool (autoscaled)
                          +---------------------------------------------------------------------------- execute -> ack / fail
  Retry: failed job -> run_at = now + backoff -> back to PENDING;  exhausted -> DEAD (DLQ);   Cron: on completion compute next run_at
```

**Deep dive**

- **Finding due jobs without duplicates:** each scheduler claims jobs atomically with a **lease**:

```sql
UPDATE jobs SET status = 'RUNNING', lease_until = now() + interval '5 minutes', worker_id = :me, attempts = attempts + 1
WHERE id IN (SELECT id FROM jobs WHERE status = 'PENDING' AND run_at <= now()
             ORDER BY priority DESC, run_at LIMIT 100 FOR UPDATE SKIP LOCKED)       -- multiple schedulers never grab the same row
RETURNING *;
```

Index on `(status, run_at)` (partial index on PENDING). Alternatives: a **Redis sorted set** as a delay queue (`ZADD delayed run_at job_id`; a Lua script atomically pops items with `score <= now` into the ready list), **hierarchical timing wheels** for huge timer counts, or Kafka delay topics.

- **Leases, heartbeats and crash recovery:** a worker extends `lease_until` while it works; if the worker dies the lease expires and a **reaper** returns the job to PENDING (this is why delivery is **at-least-once** and jobs must be **idempotent**, H20). Fencing: include the attempt number so a zombie worker's late result is ignored.
- **Partitioning the scheduler:** shard jobs by `hash(job_id)` across scheduler instances or use **leader election per partition** (H25) so each shard has one active scheduler; on failure another instance takes over after the lease.
- **Cron/recurring:** store the cron expression; after each run (or at creation) compute the **next fire time** with a parser and insert the next occurrence; one definition must not create duplicate runs when several schedulers exist (unique key `(cron_job_id, scheduled_time)`). **Misfire policy** when the system was down: run once now, skip, or catch up all.
- **Fairness and isolation:** per-tenant concurrency and rate limits, separate queues per priority class, weighted fair scheduling so one tenant's flood cannot starve others; max job runtime with kill/timeout.
- **Retries:** exponential backoff + jitter, retryable vs permanent errors, max attempts then **dead-letter** with alert (H21); poison job protection.
- **Dependencies/workflows:** DAG of jobs (Airflow, Temporal) with state persisted between steps; use Temporal-style durable execution for long multi-step flows rather than hand-rolling.
- **Time:** use UTC, a single time source (DB `now()`) for comparisons to avoid clock skew between schedulers, handle DST for cron in local time zones.

```python
# Redis delay queue (sketch): ZADD delayed <run_at> <job_id>; worker loop pops due jobs atomically
POP_DUE = """local ids = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', ARGV[1], 'LIMIT', 0, ARGV[2])
for _, id in ipairs(ids) do redis.call('ZREM', KEYS[1], id); redis.call('LPUSH', KEYS[2], id) end
return ids"""
# Caveat: a worker that crashes after the pop loses the job: use a reliable-queue pattern (move to a 'processing' set with a visibility deadline, as SQS does).
```

**Scaling and failures:** scheduler and API tiers are stateless; the job store is the source of truth (replicated); workers autoscale on queue depth and age; job store hot spots at `now()` (many jobs due at the same second: add jitter, partition); backpressure when workers are saturated (stop claiming more); multi-region: run schedulers per region with partition ownership.

**Security:** per-tenant isolation of payloads and secrets, authz on submit/cancel, job types whitelisted (never execute arbitrary code from payload), sandboxed workers, audit logs.

**Observability:** queue depth and **oldest job age** per priority, scheduling delay (actual start minus run\_at), success/failure/retry rates by type, DLQ size, lease expirations (zombie workers), worker utilization, per-tenant usage.

**Trade-offs:** DB polling with `SKIP LOCKED` (simple, strong consistency, DB load) vs message-queue-native delays (scalable, less flexible querying); at-least-once + idempotency vs exactly-once (impossible without idempotent effects); push to workers vs workers pull (pull gives natural backpressure); timing precision vs load; build vs adopt (Temporal, Celery, Sidekiq, Quartz, Airflow, SQS).

---

## S9. Appointment / booking system (doctors, salons, hotels)

**Functional:** providers define **availability** (weekly schedule + exceptions such as leave); customers **search** (specialty, location, date), view slots, **book**, **cancel/reschedule**, receive **reminders**; payment/deposit (optional); waitlist. **Non-functional:** **no double booking, ever** (the key invariant), search reads fast and may be slightly stale, correct **time zones**, high availability, fair under peak demand.

**Estimates:** 5M users, 100k providers, 1M bookings/day ≈ 12/s average, about 100/s at peak (low write volume; correctness matters far more than raw throughput). Slots: 100k providers x 30 slots/day x 90 days = 270M slot rows (about 50 GB): fits a sharded SQL database. Search reads about 100x writes.

**API**

```
GET  /v1/providers?specialty=cardiology&lat=..&lng=..&date=2026-11-03      -> list with next available slots
GET  /v1/providers/{id}/slots?from=2026-11-03&to=2026-11-10                -> [{slot_id, start_utc, end_utc, price}]
POST /v1/bookings { slot_id, user_id, idempotency_key }                     -> 201 { booking_id, status: CONFIRMED } | 409 slot_taken
POST /v1/bookings/{id}/cancel     POST /v1/bookings/{id}/reschedule { new_slot_id }
```

**Data model: materialize slots** (one row per bookable slot) so booking is a row-level atomic operation

```sql
CREATE TABLE slot (
  id BIGINT PRIMARY KEY, provider_id BIGINT NOT NULL, start_ts TIMESTAMPTZ NOT NULL, end_ts TIMESTAMPTZ NOT NULL,
  status VARCHAR(8) NOT NULL DEFAULT 'FREE',            -- FREE | HELD | BOOKED
  held_by BIGINT, held_until TIMESTAMPTZ,
  UNIQUE (provider_id, start_ts)                         -- structural guard: a provider cannot have two slots at the same time
);
CREATE TABLE booking (
  id BIGINT PRIMARY KEY, slot_id BIGINT NOT NULL UNIQUE REFERENCES slot(id),   -- UNIQUE slot_id: one booking per slot, enforced by the database
  user_id BIGINT NOT NULL, status VARCHAR(10) NOT NULL, idem_key TEXT, created_at TIMESTAMPTZ DEFAULT now(), UNIQUE (user_id, idem_key)
);
-- Availability rules (weekly template + exceptions) generate slots ahead of time via a background job (e.g. next 90 days).
```

**Booking without double booking (atomic conditional update; same idea as seat locking, Part 4 L7)**

```sql
BEGIN;
UPDATE slot SET status = 'BOOKED', held_by = NULL
WHERE id = :slot AND (status = 'FREE' OR (status = 'HELD' AND held_by = :user AND held_until > now()));
-- rows_affected = 0  => someone else got it: ROLLBACK and return 409
INSERT INTO booking (id, slot_id, user_id, status, idem_key) VALUES (:id, :slot, :user, 'CONFIRMED', :key);   -- the UNIQUE(slot_id) is the last line of defence
COMMIT;
```

For flows with payment: first **hold** the slot (`FREE → HELD` with `held_until = now() + 5 min`), take payment, then confirm; a background job (or lazy check) releases expired holds. Alternatives: Redis `SET slot:{id} user NX EX 300` for the hold with the database as the source of truth; or pessimistic `SELECT ... FOR UPDATE` on the slot row.

**Hotel-style inventory (room-nights, not unique slots):** keep `inventory(room_type, date, available)`; booking several nights is one transaction decrementing each date with `WHERE available > 0` and rolling back if any date fails; overbooking can be an explicit policy (airlines, hotels) with compensation.

**Architecture**

```
Client -> CDN -> LB -> Booking API (stateless) -> Booking DB (SQL, sharded by provider_id; synchronous replica for failover)
                  |-> Search service (Elasticsearch / PostGIS: provider profile, geo, specialty; availability summary updated via CDC, eventually consistent)
                  |-> Cache (Redis): provider pages, slot lists for the next days (short TTL; invalidated on booking events)
                  |-> Payment service (gateway, idempotent) -> webhooks
                  +-> Events (outbox -> Kafka) -> Notification svc (confirmations), Reminder scheduler (24h/1h before), Calendar sync (Google/Outlook), Analytics
```

**Deep dive**

- **Source of truth vs search view:** search shows **approximate availability** (eventually consistent); the **booking transaction re-validates** and is authoritative: an "available" slot may turn out taken (handle with a friendly 409 and show alternatives).
- **Sharding:** by `provider_id` keeps all of a provider's slots/bookings on one shard so the booking transaction is local; a user's "my bookings" is a secondary index/table by `user_id` (updated via event) to avoid scatter-gather.
- **Time zones:** store **UTC** timestamps plus the provider's IANA time zone; render in the user's zone; generate recurring availability in the provider's **local** time (handles DST: a 9:00 slot stays 9:00 local); never store local times without zones.
- **Cancellation and reschedule:** cancellation policy windows (free until 24 h before), release the slot to FREE (transactionally with the booking status), refund via the payment service (saga/compensation), notify the **waitlist** (first in line gets a hold); reschedule = atomically book the new slot and release the old (one transaction when on the same shard, otherwise a saga).
- **Reminders:** a scheduler/delay queue (S8) fires 24 h/1 h before; idempotent send; skip cancelled bookings.
- **Fairness at peak (concert/vaccine style):** virtual waiting room, per-user rate limits, CAPTCHA, limit holds per user; avoid hoarding.
- **Idempotency:** `Idempotency-Key` plus `UNIQUE (user_id, idem_key)` makes a retried POST return the same booking (H20).

**Scaling and failures:** the write path is low volume, so a single primary per shard with synchronous replication is enough; reads scale with replicas and caching; if the search index lags, bookings still succeed; hold expiry job failure is harmless (expired holds are treated as free in the update's `WHERE`); payment succeeded but booking failed → automatic refund; DR with PITR backups.

**Security:** authz (users see only their bookings; providers only their calendar), PII/health data protection (HIPAA-like constraints: encryption, audit logs, minimal data in notifications), rate limits and bot protection on booking, signed payment webhooks.

**Observability:** booking success/409 rate, hold-expiry rate, search-to-booking conversion, double-booking invariant checks (a nightly query that must return zero rows), reminder delivery, payment/booking mismatch, p99 latencies.

**Trade-offs:** materialized slots (simple atomic booking, more rows, generation job) vs computing availability from rules at query time (flexible, harder to make atomic); hold-then-pay (better UX, needs expiry) vs pay-then-book (risk of paying for a taken slot); strong consistency for booking vs eventual for search; SQL (constraints, transactions) vs NoSQL (needs conditional writes).

---

## S10. Distributed cache (design a Redis/Memcached-like system)

**Functional:** `get/put/delete` of key-value pairs with optional **TTL**; eviction when full; atomic operations (incr, CAS). **Non-functional:** **sub-millisecond** latency, very high throughput, **horizontal scalability**, high availability, graceful behaviour when nodes join/leave, cache is **not** the source of truth (data loss is acceptable but must not cause outages).

**Estimates:** 1M ops/s peak, hot working set 1 TB, average value 1 KB. One node: about 100k ops/s and 64 GB RAM → at least **16 shards for capacity and 10 for throughput**; plan about 20 shards x (1 primary + 1 replica) = 40 nodes, leaving headroom for failure and skew.

**Architecture**

```
Clients (app servers) with a SMART CLIENT / local routing table  (or a proxy layer like twemproxy/Envoy)
      | hash(key) -> slot -> shard  (consistent hashing ring with vnodes, or 16,384 fixed slots as in Redis Cluster)
      v
  Shard 1: Primary --async replication--> Replica        Shard 2: Primary --> Replica   ...   Shard N
      ^                                                    Cluster manager / gossip: membership, failure detection, slot map, failover
```

**Single node internals (connects Part 4 L3 and L4)**

- **Hash table** for O(1) lookup (open addressing/chaining with incremental rehashing so resizing does not pause the server).
- **Eviction:** LRU via a doubly linked list + map (or approximate LRU by sampling, as Redis does, to avoid per-access list updates), LFU, TTL-first, random, or `noeviction` (reject writes when full); configurable per instance.
- **Expiration:** **lazy** (check on access) + **active** (periodically sample keys with TTLs; or a timing wheel/min-heap) so unread expired keys are reclaimed.
- **Memory management:** slab allocator (Memcached) or jemalloc to limit fragmentation; item size limits; compression optional.
- **Concurrency model:** a **single-threaded event loop** (epoll) executing commands serially (Redis) gives atomic commands without locks; or multi-threaded with sharded locks (Memcached). Network I/O threads parse/serialize while one thread executes.
- **Persistence (optional):** periodic snapshots and/or append-only log for warm restarts; pure caches skip it.

**Distribution design**

| Decision | Options and choice |
| --- | --- |
| **Partitioning** | consistent hashing with virtual nodes (minimal data movement on resize, H11) or fixed hash slots (explicit slot migration, easier ops) |
| **Routing** | smart client caches the slot map and gets redirects (`MOVED`) when stale; or a stateless proxy tier (simpler clients, extra hop) |
| **Replication** | primary-replica, **asynchronous** (fast, may lose the latest writes on failover: acceptable for a cache) |
| **Failure detection & failover** | heartbeats/gossip; majority of nodes agree a primary is down → promote its replica and update the slot map; clients refresh routing |
| **Resizing** | add a node, move only its share of slots/keys; during migration serve from the old owner and redirect |
| **Consistency** | AP-style: eventual between primary and replica; no cross-key transactions; single-key operations atomic |

```python
class ShardedCache:                                 # smart client using the consistent hash ring from H11 + replication to the next node
    def __init__(self, ring, nodes, replicas=1): self.ring, self.nodes, self.replicas = ring, nodes, replicas   # nodes: name -> client
    def put(self, key, value, ttl=None):
        owners = self.ring.get_replicas(key, 1 + self.replicas)        # primary first, then replica(s)
        self.nodes[owners[0]].set(key, value, ex=ttl)                  # synchronous to primary
        for n in owners[1:]: self._async(lambda n=n: self.nodes[n].set(key, value, ex=ttl))   # asynchronous to replicas
    def get(self, key):
        for n in self.ring.get_replicas(key, 1 + self.replicas):       # fall back to replicas if the primary is unreachable
            try:
                v = self.nodes[n].get(key)
                if v is not None: return v
            except ConnectionError: continue
        return None                                                    # a cache miss, never an error to the caller
```

**Problems to address (ties to H7):** **hot keys** (replicate the key to several nodes and pick randomly, add a client-side near cache with short TTL, or key suffix spreading); **thundering herd/stampede** (single-flight, TTL jitter, probabilistic early expiry); **cold start** after a node loss (gradual warming, replicas, rate-limited DB fallback to avoid overload); **big keys/values** (limit size, split, avoid blocking commands such as `KEYS *`); **cache penetration/avalanche** (negative caching, Bloom filter, jitter); **multi-tenant noisy neighbours** (memory quotas per tenant, separate clusters); **data consistency with the DB** (delete-on-write, TTL, versioning).

**Scaling and failures:** clients and proxies are stateless; shards scale by adding nodes and rebalancing slots; a lost primary → replica promoted within seconds (some recent writes lost); a lost shard with no replica → its keys miss and the database absorbs the load (protect it with rate limits and circuit breakers); network partitions: minority side stops accepting writes (or accepts and reconciles); multi-region: independent caches per region (do not replicate caches globally; replicate invalidation events if needed).

**Security:** network isolation (private subnets), authentication/ACLs and TLS between clients and nodes, disable dangerous admin commands, no sensitive data without encryption, limits on key and value sizes.

**Observability:** **hit ratio** (the key metric), evictions/s, expired keys, memory used and fragmentation, ops/s and latency percentiles per command, replication lag, connection count, slot/shard skew and hot keys, failover events.

**Trade-offs:** consistent hashing vs fixed slots; smart client vs proxy; async replication (speed) vs sync (safety); LRU vs LFU (recency vs frequency; scan resistance); single-threaded atomic simplicity vs multi-threaded throughput; cache as write-through/back vs cache-aside (Part 5, H7).

---

# REVISION SHEET

## R1. One-page checklist for any HLD answer

1. **Clarify:** features, scale (DAU, QPS), read/write ratio, latency, availability, consistency, durability, security; out of scope.
2. **Estimate:** QPS (avg/peak), storage per year x replication, bandwidth, cache size, servers.
3. **API:** endpoints, status codes, pagination, idempotency, auth.
4. **Data model:** entities, access patterns, SQL vs NoSQL, partition key, indexes.
5. **Architecture:** client → CDN → LB → gateway → stateless services → cache → DB (replicas/shards) → queue → workers → object storage. Walk a **write** and a **read** flow.
6. **Deep dive** the hard part: hot keys, fan-out, ID generation, ordering, uniqueness, double booking, hot partition, consistency.
7. **Failure and scale:** each component fails → what happens; retries/timeouts/circuit breakers; DR (RPO/RTO); backpressure; multi-region.
8. **Security** (authN/Z, TLS, PII, abuse), **observability** (RED/USE, SLOs, tracing), **trade-offs** and next steps.

## R2. Default technology choices (justify, do not just name)

| Need | Typical default |
| --- | --- |
| relational OLTP | PostgreSQL/MySQL with replicas; shard later |
| key-value at scale | DynamoDB / Cassandra / Redis |
| cache | Redis (cache-aside, TTL + jitter) |
| search | Elasticsearch/OpenSearch via CDC |
| queue/events | Kafka (log, replay, ordering per key) or SQS/RabbitMQ (task queue) |
| blobs | S3-style object storage + CDN |
| real-time | WebSocket gateway + Redis pub/sub or Kafka backplane |
| IDs | Snowflake / UUIDv7; base62 for short codes |
| coordination | etcd / ZooKeeper (locks, leader election); avoid unless necessary |
| workflows | Temporal/Step Functions or saga with outbox |
| observability | Prometheus + Grafana, OpenTelemetry, ELK/Loki |

## R3. Problem-to-concept map

| System | Key concepts to name |
| --- | --- |
| URL shortener | base62, ID generation (range allocator), cache-aside, 301 vs 302, KV store sharded by code |
| Rate limiter | token bucket, Redis Lua atomicity, fail-open, per-key sharding, 429 headers |
| Notification service | queues per priority/channel, retries + DLQ, idempotency, preferences, provider failover |
| Chat | WebSocket gateways, routing map, per-conversation sequence, persist-before-ack, presence TTL, offline push |
| File storage | chunking, content hash dedupe, presigned URLs, metadata vs block store, sync cursor, GC |
| News feed | fan-out on write vs read, hybrid for celebrities, timeline cache, cursor pagination |
| E-commerce | saga, reservation with TTL, atomic stock update, idempotent payments/webhooks, flash sale queue |
| Job scheduler | lease + SKIP LOCKED, at-least-once + idempotency, cron next-run, delay queue, DLQ |
| Booking | slot rows + unique constraint, hold with TTL, atomic conditional update, time zones, search eventual |
| Distributed cache | consistent hashing/slots, eviction + expiry, async replication, hot keys, stampede protection |

## R4. Numbers and formulas to keep in your head

- 1 day ≈ 10^5 s; 1M requests/day ≈ 12 QPS; peak = 2x to 5x average; 80/20 rule for caches.
- Availability: 99.9% ≈ 8.8 h/year; 99.99% ≈ 53 min/year; series multiplies, parallel redundancy `1 - (1 - a)^n`.
- Little's law `L = λ x W`; quorum `R + W > N`; error budget `(1 - SLO)` x period.
- Redis \~100k ops/s/node; DB \~1k to 10k QPS/node; Kafka hundreds of MB/s per broker; memory access 100 ns, SSD 100 us, datacenter RTT 0.5 ms, cross-continent RTT 100 to 150 ms.
- Base62 length 7 ≈ 3.5 trillion; Snowflake 41/10/12 bits ≈ 4096 IDs/ms/node.

## R5. Frequent follow-up questions (short answers)

1. **"What if this node/DB/region fails?"** replicas + automated failover, retries with backoff, circuit breakers, graceful degradation, RPO/RTO stated.
2. **"How do you handle hot keys/partitions?"** cache, key splitting/replication, better partition key, local caches, rate limits.
3. **"How do you guarantee no duplicates / no lost messages?"** at-least-once + idempotency keys + unique constraints + outbox.
4. **"Strong or eventual consistency?"** decide per data type; money/inventory/booking strong; feeds/counters/search eventual.
5. **"How would you scale 10x?"** measure the bottleneck, cache, replicas, shard, async, CDN, partition by access pattern.
6. **"How do you secure it?"** TLS, authN/Z (object-level), rate limiting, secrets management, PII minimization, audit logging.
7. **"How do you monitor it?"** SLOs on user-visible latency/errors, RED/USE dashboards, tracing, alerts with runbooks.
8. **"What are the trade-offs of your design?"** always name at least two alternatives and the reason you rejected them.

**Common mistakes:** jumping to boxes without requirements; skipping estimation; one giant database for everything; no failure discussion; ignoring idempotency and retries; caching without invalidation; "just use Kafka" with no reason; microservices on day one; forgetting security and observability; not driving the conversation.

---

*End of Part 5. Part 6 covers backend engineering and deployment (Docker, Kubernetes, CI/CD, cloud, environments), AI/ML and LLM fundamentals, project deep dives, resume and behavioural/HR preparation, and OA/interview strategy.*

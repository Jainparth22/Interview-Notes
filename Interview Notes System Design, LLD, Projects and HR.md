# Interview Notes: System Design, LLD, Projects and HR

This is the interview-round half of the master checklist: Section 4 (system design and HLD), Section 6 (LLD problems), Section 9 (project readiness) and Section 10 (placement logistics, behavioural and HR). The OA notes cover coding and SQL, and the technical-round notes cover CS fundamentals. This document leans on the projects you have actually built, so you can answer with evidence instead of theory.

## Part 1: How to run a design interview

A 45-minute design round goes well when you control the structure. Use the same skeleton every time:

1. **Clarify requirements (5 min).** Ask who the users are, the core features (functional), and the targets for scale, latency, availability and consistency (non-functional). Agree what is out of scope. Write the list down.
2. **Estimate scale (3 to 5 min).** Daily active users, requests per second, read-to-write ratio, storage per record, total storage, bandwidth. Round aggressively. The point is to find out which part of the system will hurt.
3. **API design (3 min).** The handful of endpoints that serve the core features, with request and response shapes.
4. **Data model (5 min).** Entities, relationships, primary access patterns, SQL or NoSQL and why, key indexes.
5. **High-level architecture (10 min).** Clients, load balancer, stateless app servers, cache, database, queue, workers, object storage, CDN. Draw one box-and-arrow diagram and walk a request through it end to end.
6. **Deep dive (10 min).** Pick the hardest component (usually the one the interviewer is nudging toward) and go deep: data structures, consistency, failure handling.
7. **Bottlenecks, failures, trade-offs (5 min).** What breaks first, how you scale it, what you would monitor, what alternatives you rejected and why.

Habits that score well: think aloud, state assumptions explicitly, name trade-offs (never present a choice as free), start simple and add complexity only when a requirement demands it, and tie every component to a requirement.

## Part 1 continued: Fundamentals

### Functional vs non-functional requirements, and API design

- **Functional:** what the system does (shorten a URL, send a message, book a slot). **Non-functional:** how well it does it: scalability, latency, availability, consistency, durability, security, cost, maintainability.
- Client-server basics: clients send requests over HTTP or WebSockets, stateless servers process them, state lives in databases and caches. Design APIs around resources, make writes idempotent where retries are possible, paginate lists, and version the API.

### Latency, throughput, capacity estimation

- **Latency:** time for one operation. **Throughput:** operations per second. Optimising one can hurt the other (batching raises throughput and latency).
- Useful numbers to remember: memory read around 100 ns, SSD read around 100 microseconds, a datacentre round trip around 0.5 ms, a cross-continent round trip around 150 ms, a disk seek around 10 ms. Memory is orders of magnitude faster than disk, and the network is slower than both.
- Handy scale facts: a day has about 86,400 seconds (round to 10^5). 1 million requests per day is about 12 requests per second. 100 million per day is about 1,200 per second. Peak is usually 2 to 5 times the average.
- Worked example, URL shortener: 100 million new URLs per month is about 40 writes per second. With a 100:1 read ratio, about 4,000 reads per second. Each record is about 500 bytes, so 100 million times 12 months times 5 years is 6 billion records, around 3 TB. That fits one sharded database, and the read volume suggests a cache.
- Also know: storage units (1 KB = 10^3 B, 1 TB = 10^12 B), bandwidth equals requests per second times payload size, and cache size equals the hot fraction (often 20 percent of data serves 80 percent of reads) times daily volume.

### Scaling and statelessness

- **Vertical scaling:** a bigger machine. Simple, has a ceiling and a single point of failure. **Horizontal scaling:** more machines behind a load balancer. Needs stateless services and shared state in external stores.
- **Stateless services:** no user-specific state kept in server memory between requests. Sessions go to Redis or a token (JWT), files go to object storage. Any server can answer any request, so you can add, remove and restart servers freely.

### Load balancing

- Spreads traffic across servers and removes unhealthy ones using health checks.
- Algorithms: round robin, weighted round robin, least connections, IP hash (sticky routing), consistent hashing (for caches and shards).
- **Layer 4** balancers route by IP and port (fast). **Layer 7** balancers route by URL, headers and cookies (flexible, can terminate TLS). Avoid sticky sessions where you can. Balancers must not be a single point of failure, so run them redundantly (active-passive or DNS plus anycast).

### Caching, Redis, CDN

- Cache at several levels: browser, CDN, API gateway, application (in-process and Redis), database buffer pool.
- Strategies and invalidation are covered in the technical-round notes (cache-aside, write-through, write-behind, TTLs, stampede protection). In an interview state the freshness requirement first, because it decides everything: a feed can be seconds stale, an account balance cannot.
- Eviction: LRU is the default answer, LFU for stable popularity. Watch hot keys (replicate or add a local cache) and cold starts (warm the cache).
- **CDN:** edge servers near users cache static content (images, video, JS) and sometimes API responses. Lowers latency and origin load. Control with cache headers (`Cache-Control`, `ETag`), invalidate by versioned file names or purge calls.

### Choosing a database

- Need transactions, joins, constraints, strong consistency: relational (PostgreSQL, MySQL). Huge key-value lookups or sessions: Redis or DynamoDB. Flexible documents: MongoDB. Massive write throughput and time series: Cassandra or a time-series store. Relationships as the main query: a graph database. Full-text search: Elasticsearch or OpenSearch as a secondary index, not the source of truth. Files and media: object storage.
- The strongest answer is usually polyglot: PostgreSQL as the system of record, Redis for hot reads, a search index for text queries, S3 for blobs. Justify each by its access pattern.

### Replication, read replicas, sharding

- **Replication** copies data to other nodes for availability and read scaling. Leader-follower: writes go to the leader, reads can go to followers. Asynchronous replication is fast but followers lag (stale reads, and writes can be lost on failover). Synchronous is safer and slower. Handle read-your-own-writes by reading from the leader briefly after a write.
- **Sharding** (horizontal partitioning) splits data across nodes when one machine cannot hold or serve it. Strategies: range (simple, hot spots), hash (even spread, range queries are hard, resharding is painful), directory or lookup table (flexible, a new dependency). The **shard key** decides everything: it should spread load evenly and keep common queries on one shard.
- Costs of sharding: cross-shard joins and transactions, rebalancing, operational complexity. Exhaust vertical scaling, indexing, caching and read replicas first.

### Indexing, partitioning, connection pooling

- Index design follows the query patterns (covered in the DBMS notes). Table **partitioning** splits one large table inside a database by range, list or hash, for pruning and easy retention (drop an old partition). Connection pooling reuses a bounded set of database connections, because each Postgres connection is expensive and too many collapses the database.

### Consistency models, CAP and PACELC

- **Strong consistency:** after a write, every read sees it. **Eventual consistency:** replicas converge given time, reads may be stale. In between: read-your-writes, monotonic reads, causal consistency.
- **ACID vs BASE:** ACID favours correctness for a single database. BASE (basically available, soft state, eventually consistent) favours availability at scale.
- **CAP theorem:** during a network partition you must choose between consistency (refuse or fail some requests) and availability (answer, possibly with stale data). It is not a choice of two out of three at all times, partitions are a fact of life. A bank ledger picks consistency, a social feed picks availability.
- **PACELC:** extends CAP. If there is a Partition choose A or C, **E**lse (normal operation) choose Latency or Consistency. It captures that even with no failure, stronger consistency costs latency.

### Consistent hashing

- Problem: with `hash(key) % N`, changing N remaps almost every key, wrecking caches. Consistent hashing places nodes and keys on a ring, and each key belongs to the next node clockwise. Adding or removing a node moves only about 1/N of the keys.
- **Virtual nodes:** each physical node appears many times on the ring, which evens the load and lets bigger machines take more. Used by distributed caches, Dynamo-style stores and some load balancers.

### Unique ID generation

- Auto-increment: simple, but a single point of failure and it leaks volume. UUID v4: no coordination, random 128 bits, bad index locality. UUID v7 or ULID: time-ordered, index friendly.
- **Snowflake style:** 64 bits made of a timestamp, a machine id and a per-millisecond sequence. Roughly sortable, generated locally, many IDs per millisecond. Issues to mention: clock skew and machine id assignment.
- For short URLs: a counter encoded in base62, pre-generated key ranges handed to each server, or a hash of the URL with collision handling.

### Blob storage and file uploads

- Store files in object storage (S3 or compatible), keep only metadata (owner, name, size, content type, object key) in the database. Never store large blobs in the relational database.
- **Large upload flow:** the client asks the API for a **pre-signed URL**, uploads directly to object storage (so your servers do not carry the bytes), then notifies the API, which records the metadata and triggers processing (virus scan, thumbnails) through a queue. For big files use multipart or resumable uploads. Serve downloads through a CDN, with signed URLs for private files.
- Durability comes from replication across zones, and lifecycle rules move old data to cheaper tiers.

## Part 2: Communication and service architecture

### Message queues, Kafka, pub-sub

- A **queue** decouples producers from consumers: the producer enqueues and moves on, workers consume at their own pace. Benefits: absorbs spikes (load levelling), isolates failures, enables retries, scales consumers independently. Costs: eventual consistency, ordering and duplicate-handling complexity, one more thing to operate.
- **Delivery semantics:** at-most-once (may lose), at-least-once (may duplicate, the common default, so make consumers idempotent), exactly-once (really at-least-once plus idempotent processing or transactions).
- **Kafka:** a distributed, partitioned, replicated append-only log. Topics are split into partitions, ordering is guaranteed only within a partition, so choose the partition key (for example `user_id`) to keep related events in order. Consumers in a **consumer group** share partitions, each partition is read by one consumer in the group. Messages are retained for a period, so consumers can replay. High throughput, durable, good for event streams and event sourcing.
- **RabbitMQ and SQS style queues:** a message goes to one consumer and is removed on acknowledgement. Better for task queues with per-message routing and simple retry. Use a **dead-letter queue** for messages that keep failing.
- **Pub-sub:** a publisher sends to a topic, every subscriber receives a copy. Use it for fan-out (one order event consumed by billing, inventory and notifications).

### REST, gRPC, GraphQL

- **REST:** resource oriented, over HTTP and JSON, cacheable, universal. Over-fetching and under-fetching are the common complaints.
- **gRPC:** binary protobuf over HTTP/2, strongly typed contracts, streaming, fast. Best for service-to-service calls inside a backend. Poor browser support without a proxy.
- **GraphQL:** the client asks for exactly the fields it wants from one endpoint. Good for varied clients and nested data. Watch for expensive queries (add depth and cost limits), N+1 resolvers (use batching) and harder HTTP caching.

### API gateway, microservices, service discovery

- **API gateway:** the single entry point that handles routing, authentication, rate limiting, TLS termination, request logging and aggregation. Keep business logic out of it.
- **Monolith vs microservices:** a monolith is simple to build, test, deploy and debug, with in-process calls and real transactions. Microservices allow independent deploys and scaling per team and domain, at the cost of network failures, distributed transactions, versioned contracts and operational overhead. Start with a well-structured (modular) monolith and split along clear domain boundaries only when team size or scaling needs justify it. This is also the honest answer for your own projects.
- **Service discovery:** services find each other by name rather than fixed addresses. Client-side discovery queries a registry (Consul, etcd), server-side discovery uses a load balancer or DNS (Kubernetes services). **Health checks** (liveness and readiness) keep unhealthy instances out of rotation.

### WebSockets, SSE, polling

- **Short polling:** the client asks repeatedly. Simple, wasteful. **Long polling:** the server holds the request open until there is data. Works everywhere, medium efficiency.
- **Server-Sent Events:** one-way server-to-client stream over HTTP, auto-reconnect, great for notifications and live feeds. **WebSockets:** full-duplex persistent connection, for chat, collaboration and games. They are stateful, so scaling needs a pub-sub layer (Redis or Kafka) so a message reaches the server that holds the recipient's socket, plus sticky or connection-aware routing.
- **Graceful degradation:** fall back from WebSocket to HTTP polling when the socket fails. You built exactly this in TaskFlow.

### Background jobs and task queues

- Covered in depth in the technical-round notes. For design interviews state the pattern: the API validates and persists the request, enqueues a job and returns `202 Accepted` with a job id, a worker processes it, and the client polls a status endpoint or receives a push. Add retries with backoff, idempotent handlers, a dead-letter queue, a scheduler for periodic work, and visibility metrics (queue depth, age of oldest message).

## Part 2 continued: Reliability, security and distributed systems

### Rate limiting

- Purpose: protect against abuse and overload, and enforce fair use or pricing tiers. Return `429` with a `Retry-After` header.
- **Token bucket:** a bucket refills at a steady rate up to a capacity, each request takes a token. Allows bursts, the most common choice. **Leaky bucket:** a smooth, constant output rate. **Fixed window counter:** simple, but a burst across a window boundary can double the limit. **Sliding window log:** accurate, memory heavy. **Sliding window counter:** a good compromise.
- Distributed implementation: keep counters in Redis with atomic operations (`INCR` plus `EXPIRE`, or a Lua script so check and update are one atomic step, otherwise you reintroduce a race). Decide the key (user, API key, IP, route), and decide what to do if Redis is down. Failing open favours availability, failing closed favours protection. Your appointment system fails open, say that you chose it deliberately and why.

### Idempotency

- An operation is idempotent if repeating it has the same effect as doing it once. Clients retry after timeouts, so any non-idempotent write must carry an **idempotency key**. The server stores `(key, response)` and returns the saved response for repeats. Make the key lookup and the write atomic (a unique constraint on the key), otherwise two simultaneous retries both proceed.

### Retries, timeouts, backoff, jitter

- Always set timeouts, since a missing timeout turns a slow dependency into an outage. Retry only idempotent or keyed operations and only on transient errors.
- **Exponential backoff** (wait 1 s, 2 s, 4 s, ...) stops retries from hammering a struggling service. **Jitter** (randomise the wait) stops thousands of clients from retrying in lockstep, the thundering herd. Cap the maximum wait and the attempt count, and retry budgets prevent retry storms.

### Circuit breakers and fault tolerance

- **Circuit breaker:** wraps calls to a dependency. **Closed:** calls pass, failures are counted. **Open:** after a threshold of failures, calls fail immediately without touching the dependency, giving it time to recover. **Half-open:** after a cool-down, a few trial calls test recovery, success closes the circuit, failure reopens it.
- Companions: bulkheads (isolate resource pools so one dependency cannot exhaust everything), fallbacks and graceful degradation (cached or heuristic answers), load shedding, and timeouts. Your TaskFlow ML-client circuit breaker and the appointment system's heuristic fallback are both direct examples.

### Redundancy, high availability, disaster recovery

- **Availability** is uptime. 99.9 percent allows about 8.8 hours of downtime per year, 99.99 percent about 53 minutes. Reach it by removing single points of failure: multiple app instances across availability zones, replicated databases with automatic failover, redundant load balancers.
- **Active-active** (all serve traffic) versus **active-passive** (standby takes over on failure). **RPO** (how much data you can afford to lose) and **RTO** (how long you can be down) drive backup frequency and failover design. Test restores, an untested backup is a hope, not a backup.

### Distributed transactions: Saga and the outbox

- Two-phase commit across services is blocking and fragile. A **Saga** splits a business transaction into local transactions, each with a **compensating action** if a later step fails (reserve stock, charge payment, and if the charge fails, release the stock). **Choreography:** services react to each other's events (simple, hard to follow). **Orchestration:** a coordinator tells each service what to do (explicit and easier to reason about).
- **Transactional outbox:** write the state change and an outbox row in the same local transaction, and a relay process publishes outbox rows to the broker, retrying until acknowledged. It guarantees that an event is published if and only if the change committed. Consumers must still be idempotent.

### Distributed locks and leader election

- A distributed lock gives mutual exclusion across machines, typically `SET key value NX PX ttl` in Redis with a unique value so only the owner releases it. Always set a TTL so a crashed owner cannot hold it forever, and be aware of the danger: a paused process can lose its lock and still act. Use **fencing tokens** (a monotonically increasing number checked by the resource) for correctness-critical cases. ZooKeeper or etcd give stronger guarantees.
- Often the better answer is to avoid the lock entirely: use an atomic database update, a unique constraint, or optimistic concurrency. Your slot booking does precisely this with a compare-and-swap on a status column.
- **Leader election:** nodes agree on one leader (through etcd, ZooKeeper, Raft-based systems) for work that must run once, such as a scheduler. Know the idea of Raft and Paxos at a high level: majority quorum, terms, and log replication.

### Authentication, authorisation, security

- **OAuth 2.0** lets a user grant an app limited access to their account without sharing a password. Authorization-code flow (with PKCE for public clients): the app redirects to the provider, the user consents, the app gets a code and exchanges it for an access token and often a refresh token. **OpenID Connect** adds identity on top (the ID token).
- **JWT:** structure, signature checks, short expiry and refresh tokens (covered in the technical notes). For service-to-service use mutual TLS or signed service tokens.
- Security checklist for any design: TLS everywhere, hashed passwords, least-privilege access, input validation, parameterised queries, rate limiting, secrets in a secrets manager (not code or images), encryption at rest for sensitive data, audit logs, and abuse prevention such as captchas and quotas.

### Concurrency control in design

- State which mechanism you would use and why: **atomic conditional update** (`UPDATE ... WHERE status = 'free'`, check the row count) for claiming a resource, **optimistic locking** with a version column for low-contention updates, **pessimistic locking** (`SELECT ... FOR UPDATE`) for high-contention critical sections, **unique constraints** to make duplicates impossible, and **serializable transactions with retry** when invariants span rows. CAS and TOCTOU are the vocabulary for explaining why check-then-act in application code is unsafe.

### Observability

- **Logs** (what happened, structured, with a request id), **metrics** (numbers over time: request rate, error rate, latency percentiles p50, p95, p99, saturation), **traces** (one request's path across services with timing). Use RED for services (rate, errors, duration) and USE for resources (utilisation, saturation, errors).
- **SLI** is a measurement (percentage of requests under 300 ms), **SLO** is the target (99.5 percent), **SLA** is the contractual promise with penalties. **Error budget** is the allowed failure, and spending it guides how fast you ship. Alert on symptoms users feel (error rate, latency) rather than on every cause, and attach runbooks.
- Bottleneck analysis: measure first, find the saturated resource (CPU, memory, database connections, disk I/O, network, lock contention), fix the biggest one, measure again. Mention p95 and p99 rather than averages, since tail latency is what users notice.

## Part 3: The ten HLD systems

For every system the same checklist applies: requirements, estimates, APIs, data model, architecture and data flow, caching and async, consistency, bottlenecks and failures, security, trade-offs. Below is the core idea and the sharp points for each, so you can fill in the rest on the whiteboard.

### 1. URL shortener

- **Requirements:** create a short link, redirect fast, optional custom alias and expiry, click analytics. Read-heavy (about 100:1), redirects must be low latency and highly available.
- **API:** `POST /urls {long_url, alias?, ttl?}` returns `{short_url}`. `GET /{code}` returns a `301` or `302` redirect. Use `302` if you need every click to pass through you for analytics, `301` lets browsers cache it.
- **Data:** `urls(code PK, long_url, user_id, created_at, expires_at)`. Key-value access pattern, so a relational table with an index on `code`, or a key-value store, both work.
- **Code generation:** base62 of a unique counter (7 characters gives 62^7, about 3.5 trillion codes). Avoid a single counter bottleneck by giving each app server a pre-allocated range, or use a Snowflake id. Hashing the URL needs collision handling.
- **Scale:** Redis cache in front of the database for hot codes (cache-aside), replicas for reads, shard by `code` hash. Analytics events go to a queue and are aggregated asynchronously, never in the redirect path.
- **Failure and abuse:** rate limit creation, scan for malicious URLs, expiry cleanup by a background job.

### 2. Rate limiter

- **Requirements:** limit requests per user, key or IP, low added latency, correct under many servers, configurable per route.
- **Architecture:** a middleware or gateway layer calls a shared Redis. Token bucket implemented in a Lua script that reads tokens, refills based on elapsed time, decrements and returns allow or deny in one atomic step.
- **Data:** key `rl:{user}:{route}` holding `(tokens, last_refill_ts)` with a TTL. **Scale:** shard Redis by key, local in-memory pre-checks to reduce calls. **Failure:** decide fail-open versus fail-closed, sync across regions loosely.
- **Trade-offs:** strict global accuracy costs latency, per-node approximate limits are cheap but allow some overage.

### 3. Notification service

- **Requirements:** send email, SMS and push from many internal services, respect user preferences, scheduled and bulk sends, retries, no duplicates, delivery tracking.
- **Flow:** services call `POST /notifications` (with an idempotency key). The service validates, checks preferences and templates, writes to the database (outbox) and enqueues to per-channel queues (email, SMS, push). Channel workers call providers (SES, Twilio, FCM) with retry and backoff, and update delivery status. Provider webhooks report bounces and opens.
- **Sharp points:** priority queues (a one-time password outranks a newsletter), rate limits per provider, dead-letter queue, de-duplication, quiet hours, template versioning, fallback to a second provider.

### 4. Chat application

- **Requirements:** one-to-one and group chat, delivery and read receipts, online presence, history, media, low latency.
- **Connection layer:** WebSocket gateways hold user connections. A session map in Redis tracks which gateway holds which user. A message goes: sender gateway, chat service (persist, assign a per-conversation sequence number), look up recipients, route through pub-sub to the recipient's gateway, push down the socket. Offline users get a push notification and fetch on reconnect.
- **Data:** messages in a wide-column or partitioned store keyed by `(conversation_id, sequence)` for fast range reads of history. Users and conversation metadata in a relational database. Media in object storage plus CDN.
- **Sharp points:** ordering through per-conversation sequence numbers, at-least-once delivery with client-side de-duplication by message id, group fan-out (write per member for small groups, read-time for huge channels), presence through heartbeats with a TTL.

### 5. File-storage service

- **Requirements:** upload, download, share and sync large files reliably, versioning, permissions.
- **Design:** metadata service (database: files, folders, versions, ACLs) separate from block storage (object store). Files are split into chunks (for example 4 MB), each content-hashed, so uploads resume and identical chunks are de-duplicated. Upload through pre-signed URLs, then commit the metadata. Downloads go through a CDN with signed URLs.
- **Sync:** clients keep a local state and ask for changes since a cursor, a notification channel tells them when to pull. Conflicts produce a versioned copy rather than silent overwrites.
- **Sharp points:** chunk de-duplication, replication across zones, virus scanning in an async pipeline, quotas, lifecycle tiers.

### 6. News feed

- **Requirements:** a personalised, mostly reverse-chronological or ranked feed, fast reads, followers with very different counts.
- **Fan-out on write (push):** when a user posts, write the post id into each follower's feed list in Redis. Reads are instant, writes are expensive for accounts with millions of followers. **Fan-out on read (pull):** compute the feed at read time by merging followees' posts, cheap writes, slow reads. **Hybrid:** push for normal users, pull for celebrities, merging at read time.
- **Data:** posts table (sharded by author), follow graph, per-user feed cache holding the latest few hundred post ids, hydrated from a post cache. Ranking service scores candidates. Pagination by cursor.
- **Sharp points:** the celebrity problem, cache invalidation on delete, eventual consistency is acceptable, media through CDN.

### 7. E-commerce backend

- **Services:** catalogue and search (Elasticsearch fed from the catalogue database), cart (Redis or database), inventory, orders, payments, shipping, notifications.
- **Hard part, checkout:** reserve stock, charge payment, confirm the order. Use a saga with compensations (release stock if payment fails), idempotency keys on payment calls, and the outbox pattern for events. Prevent overselling with an atomic conditional stock decrement (`UPDATE stock SET qty = qty - :n WHERE sku = :s AND qty >= :n`) and a reservation with a TTL.
- **Consistency:** orders and payments are strongly consistent in a relational database. Catalogue and search can be eventually consistent. Cache product pages and serve images through a CDN.
- **Spikes (sales):** queue the orders, rate limit, pre-warm caches, a waiting room for flash sales. You can connect this to your InventoryPro design, where stock changes only on order state transitions inside atomic updates.

### 8. Job queue and task scheduler

- **Requirements:** run jobs now, later, or on a cron schedule, with retries, priorities, at-least-once execution, and visibility into status.
- **Design:** a jobs table or broker queue holds the job, payload, status, run time and attempt count. Workers pull jobs and take a lease (visibility timeout), a job whose lease expires is picked up by another worker. A scheduler (leader-elected, so it does not run twice) scans for due jobs every few seconds, or uses a time-ordered structure such as a Redis sorted set scored by run time.
- **Sharp points:** idempotent handlers, exponential backoff, dead-letter queue, priority queues, rate-limited workers, concurrency limits per job type, and monitoring of queue depth and job age. Your Celery Beat plus worker setup is the working example.

### 9. Appointment-booking system

This is the system you have already built, so treat it as your strongest design answer.

- **Requirements:** providers publish slots, patients book, no double booking, reminders and follow-ups, calendar sync, handle abandoned bookings.
- **Core problem, double booking:** two users click the same slot together. A check-then-book in application code has a TOCTOU window. The fix is an atomic conditional update that doubles as a state machine: `UPDATE slots SET status = 'held' WHERE id = :id AND status = 'free'`, and success means exactly one row changed. Lock-free, works on PostgreSQL and SQLite.
- **Two-phase hold:** free, then held (a time-limited reservation, such as 10 minutes, while the user confirms or pays), then booked. A periodic job reclaims expired holds so abandoned bookings return to the pool.
- **Async and integrations:** reminders and emails through a queue with an outbox for at-least-once delivery, calendar sync through OAuth with encrypted refresh tokens, an AI provider behind an interface with a heuristic fallback on timeout.
- **Scale and failure:** cache read-heavy availability queries (invalidate on state change), rate limiting that fails open, partition slots by provider or date, and monitor hold-expiry lag.

### 10. Distributed cache

- **Requirements:** low-latency key-value reads and writes, horizontal scalability, high availability, eviction, optional persistence.
- **Design:** nodes partitioned using consistent hashing with virtual nodes, a client library or proxy routes keys to nodes. Each partition has a replica for failover. Eviction by LRU (a hash map plus doubly linked list), TTL expiry via lazy deletion on access plus a background sweep.
- **Sharp points:** hot keys (replicate them or add a local cache), cache stampede protection, replication being asynchronous (a failover can lose recent writes), rebalancing when nodes join, consistency between cache and database, memory fragmentation, and monitoring hit ratio.
- Compare with Redis Cluster (hash slots) and Memcached (client-side sharding) to show you know real systems.

### Checklist to rehearse for each system

- Can I state the requirements and estimates in under 3 minutes?
- Can I draw the architecture and walk one write and one read through it?
- Can I name the single hardest problem and two ways to solve it?
- Can I say what fails first at 10 times the load and what I would do about it?
- Can I name two alternatives I rejected and why?

## Part 4: Low-level design (LLD)

### The approach that works every time

1. **Clarify requirements and scope** (5 min). Ask which features are in, what the actors are, expected scale, and whether concurrency matters. Write down 4 to 6 use cases.
2. **Identify entities** from the nouns, and behaviours from the verbs. Decide who owns each responsibility (single responsibility).
3. **Sketch the class diagram:** attributes, methods, relationships (inheritance, composition, interfaces). Keep it small.
4. **Walk the core flows** through the classes: for a parking lot, park a vehicle and exit with payment. If a flow feels awkward, the design is wrong.
5. **Apply patterns where they earn their place:** Strategy for swappable rules (pricing, allocation), State for lifecycles, Observer for notifications, Factory for creating variants, Singleton only for genuinely single resources. Naming a pattern without a reason scores badly.
6. **Extensibility:** show how a new vehicle type, payment method or rule is added without editing existing classes (open/closed).
7. **Concurrency and edge cases:** two users claiming the same resource, invalid input, capacity limits, expiry, failure midway. Mention how you would test.

Code quality signals interviewers look for: clear names, small methods, interfaces for variation points, no god classes, enums for fixed sets, validation at boundaries, and dependencies injected rather than constructed inside.

### Reference implementation 1: LRU cache with fixed capacity

A hash map for O(1) lookup plus a doubly linked list for O(1) recency updates. Most recent at the head, eviction from the tail.

```python
import threading

class _Node:
    __slots__ = ('key', 'val', 'prev', 'next')
    def __init__(self, key=None, val=None):
        self.key, self.val, self.prev, self.next = key, val, None, None

class LRUCache:
    def __init__(self, capacity: int):
        if capacity <= 0:
            raise ValueError('capacity must be positive')
        self.cap, self.map = capacity, {}
        self.head, self.tail = _Node(), _Node()          # sentinels
        self.head.next, self.tail.prev = self.tail, self.head
        self.lock = threading.Lock()

    def _unlink(self, n):
        n.prev.next, n.next.prev = n.next, n.prev

    def _push_front(self, n):
        n.prev, n.next = self.head, self.head.next
        self.head.next.prev = n
        self.head.next = n

    def get(self, key):
        with self.lock:
            n = self.map.get(key)
            if n is None:
                return -1
            self._unlink(n); self._push_front(n)
            return n.val

    def put(self, key, val):
        with self.lock:
            n = self.map.get(key)
            if n:
                n.val = val
                self._unlink(n); self._push_front(n)
                return
            if len(self.map) >= self.cap:
                lru = self.tail.prev
                self._unlink(lru); del self.map[lru.key]
            n = _Node(key, val)
            self.map[key] = n
            self._push_front(n)
```

Python shortcut to mention: `collections.OrderedDict` with `move_to_end` and `popitem(last=False)` does the same in a few lines. Know both. Follow-ups: thread safety (the lock above, or sharded locks for throughput), making it generic, and LFU.

### Reference implementation 2: cache with TTL expiration

Two expiry mechanisms together: **lazy** (check on access) and **active** (a background sweep or a min-heap of expiry times).

```python
import time, heapq, threading

class TTLCache:
    def __init__(self, default_ttl=60.0, clock=time.monotonic):
        self.ttl, self.clock = default_ttl, clock
        self.data, self.heap = {}, []                  # key -> (value, expires_at); (expires_at, key)
        self.lock = threading.Lock()

    def put(self, key, value, ttl=None):
        exp = self.clock() + (ttl if ttl is not None else self.ttl)
        with self.lock:
            self.data[key] = (value, exp)
            heapq.heappush(self.heap, (exp, key))

    def get(self, key, default=None):
        with self.lock:
            item = self.data.get(key)
            if item is None:
                return default
            value, exp = item
            if exp <= self.clock():                     # lazy expiry
                del self.data[key]
                return default
            return value

    def sweep(self):                                   # call from a background thread
        now = self.clock()
        with self.lock:
            while self.heap and self.heap[0][0] <= now:
                exp, key = heapq.heappop(self.heap)
                item = self.data.get(key)
                if item and item[1] == exp:             # ignore stale heap entries
                    del self.data[key]
```

Points to make: use a monotonic clock (wall clock can jump), inject the clock so tests need no `sleep`, stale heap entries appear when a key is overwritten (hence the equality check), and combine with LRU for a bounded TTL cache.

### The other LLD problems: design sketches

**Parking lot**

- Entities: `ParkingLot`, `Floor`, `ParkingSpot` (type: compact, regular, large, EV; status), `Vehicle` (type), `Ticket` (entry time, spot, vehicle), `Payment`.
- Patterns: **Strategy** for spot allocation (nearest to entrance, first free, by floor) and for pricing (hourly, flat, peak). **Factory** for vehicle and spot creation.
- Flow: park, find a spot that fits the vehicle type, mark it occupied atomically, issue a ticket. Exit: compute the fee from the pricing strategy, take payment, free the spot.
- Edge cases: lot full, lost ticket, a spot claimed by two entry gates at once (atomic claim or a lock per floor), wrong vehicle type for a spot.

**Splitwise**

- Entities: `User`, `Group`, `Expense` (payer, amount, participants, split type), `Split`, a balance sheet (who owes whom).
- Patterns: **Strategy** for split types (equal, exact amounts, percentages, shares), each validating that the splits sum to the total.
- Store money as integer minor units (paise or cents) or `Decimal`, never floats, and decide where the rounding remainder goes.
- Maintain `balance[a][b]` net amounts. **Simplify debts:** compute each person's net balance, put creditors in a max-heap and debtors in another, repeatedly settle the largest debtor against the largest creditor. That minimises the number of transactions.

**Elevator system**

- Entities: `Elevator` (current floor, direction, state, request set), `Request` (external from a floor with a direction, internal destination), `Controller` or `Dispatcher`, `Door`.
- Patterns: **State** (idle, moving up, moving down, door open, maintenance). **Strategy** for dispatching a request to a car (nearest car going the same way, least loaded).
- Scheduling: the SCAN or elevator algorithm. Keep serving stops in the current direction (sorted sets of up-stops and down-stops), reverse when there are none ahead.
- Edge cases: overload, emergency stop, several cars, a request for the current floor.

**Library management**

- Entities: `Book` (title, ISBN) versus `BookItem` (one physical copy with a barcode and status), `Member`, `Librarian`, `Loan` (issue date, due date, return date), `Reservation`, `Fine`.
- Rules: max books per member, loan period, renewals, reservation queue per title, fine = overdue days times a rate. Search by title, author, subject (an index per field).
- Patterns: **Observer** to notify a member when a reserved book is back, **Strategy** for fine calculation. Keep inventory changes atomic so one copy is never issued twice.

**Movie-ticket booking**

- Entities: `Theatre`, `Screen`, `Seat`, `Movie`, `Show` (movie, screen, time), `ShowSeat` (seat status for one show: available, locked, booked), `Booking`, `Payment`, `User`.
- Core problem: two users select the same seat. Same solution as slot booking: atomic `available -> locked` with a lock expiry (for example 5 minutes), then `locked -> booked` on payment, and a sweeper releases expired locks.
- Patterns: **State** for seat or booking status, **Strategy** for pricing (weekday, premium seats), **Observer** for confirmations. Search shows by city, movie and date.

**Food delivery**

- Entities: `Customer`, `Restaurant`, `MenuItem`, `Cart`, `Order`, `OrderItem`, `DeliveryAgent`, `Payment`, `Address`.
- **State** machine for the order: placed, accepted, preparing, ready, picked up, delivered, cancelled, with only legal transitions allowed.
- **Strategy** for agent assignment (nearest available, least loaded) and for delivery fee. **Observer** for status notifications to the customer, restaurant and agent.
- Edge cases: restaurant rejects, no agent available (retry or widen the radius), payment failure, cancellation windows, refund logic.

**Vending machine**

- Entities: `VendingMachine`, `Inventory` (slot code to product, count, price), `Product`, `Coin` or note denominations, `Transaction`.
- Pattern: **State** with `Idle`, `HasMoney`, `Dispensing`, `OutOfService`. Each state implements `insert_money`, `select_product`, `dispense`, `refund`, and illegal actions in a state raise or are ignored. This is the textbook State example.
- Change-making: a greedy algorithm works for canonical coin systems, otherwise DP. Handle insufficient funds, sold out, and no change available.

**Logging framework**

- Entities: `Logger` (named, hierarchical), `LogLevel` enum (DEBUG, INFO, WARN, ERROR), `LogRecord`, `Handler` or `Appender` (console, file, remote), `Formatter` (plain, JSON).
- Patterns: **Strategy** for formatters and handlers, **Chain of responsibility** or a list of handlers where each filters by level, **Singleton** (or a registry) for the logger manager, **Observer** in effect for multiple sinks.
- Concerns: thread safety, asynchronous logging through a queue so callers never block on I/O, log rotation, lazy message formatting, and context such as a request id.

**Notification framework**

- Entities: `Notification`, `Channel` interface (`EmailChannel`, `SmsChannel`, `PushChannel`), `Template`, `UserPreference`, `NotificationService`.
- Patterns: **Strategy** per channel, **Factory** to resolve a channel, **Observer** or an event bus to trigger notifications from domain events, **Decorator** for retry or rate-limit behaviour around a channel.
- Concerns: preferences and quiet hours, retries with backoff, idempotency, priority, templating, and adding a new channel without touching existing code.

**Task scheduler**

- Entities: `Task` (id, callable, run\_at, interval or cron, priority, retries), `Scheduler`, `WorkerPool`, `TaskStore`.
- Design: a min-heap ordered by `run_at` plus a condition variable. The scheduler thread sleeps until the earliest task is due (or is woken when an earlier one arrives), then hands the task to a worker pool. Recurring tasks are re-queued with the next run time.
- Patterns: **Command** (a task is an object), **Strategy** for retry policy, **Observer** for completion callbacks. Concerns: cancellation, missed runs after downtime, ordering of equal times, task timeouts, and persistence for restarts.

### LLD answer template to practise

For each of the 12 problems, rehearse in 20 minutes: requirements list, class diagram, the two core flows as code or pseudo-code, one pattern with its reason, one concurrency issue with its fix, and one extension (a new type or rule) shown without editing existing classes.

## Part 5: Project interviews

In most interviews the project round decides the result, because it is the one place you control the material. The rule from your checklist is the right one: you must be able to explain every significant line you claim to have written. Interviewers pick a decision and keep asking why until you run out of answers, so prepare to the third 'why'.

### The template for every project

Prepare each of these in writing, in your own words:

- **Problem and users:** who needs it and what hurt before it existed (two sentences).
- **Two-minute walkthrough:** about 20 seconds on problem and users, 40 seconds on architecture and data flow, 40 seconds on the single hardest problem and how you solved it, 20 seconds on the result and what you would improve next. Practise it out loud with a timer.
- **Architecture drawing:** one diagram you can sketch from memory in two minutes, then walk one request through it.
- **Why each technology:** for each tool, the alternative you considered and the reason you chose this one. 'It was popular' is a bad answer, 'it fit this access pattern' is a good one.
- **One real bug:** symptom, how you found it, root cause, fix, and what you changed so it cannot return. Pick one you actually hit, because follow-ups will expose a borrowed story.
- **Bottlenecks and measured improvements:** what was slow, how you measured (tool, load, before and after), what you changed. If your resume has a percentage, be ready to say exactly how you measured it.
- **Cross-cutting questions:** security, scalability, concurrency, failure handling, testing, deployment.
- **What you would change:** two honest limitations and how you would fix them. Interviewers trust candidates who know their own weak spots.
- **Housekeeping:** the live link works, the README has setup steps, screenshots and an architecture diagram, and the repo is clean.

### Project cards from what you have told me

These cards contain only facts you have stated about each project. The 'fill in' lines are things only you know, and they are the parts interviewers care about most.

**PPA V2 Placement Portal** (Flask, Celery, Redis, Vue.js 3, JWT, 40+ REST APIs)

- **One-line pitch:** a placement portal for students, companies and admins, built to remove manual admin work.
- **Draw:** Vue 3 client, Flask API with JWT and RBAC middleware, service and repository layers, PostgreSQL, Redis cache, Celery workers for async jobs.
- **Decisions to defend:** stateless JWT plus RBAC instead of server sessions (and how you handle revocation and expiry), cache-aside with Redis (what you cache, TTLs, how you invalidate), Celery to take slow work off the request path, layered architecture so business logic is independent of data access.
- **Numbers to be able to justify:** p95 latency down 40 percent, 25 percent better responsiveness under concurrent load, 30 percent less admin effort (you described this one as an estimate, so say so honestly).
- **Likely deep dives:** what happens on a cache miss storm, how RBAC is enforced per endpoint and per record, what happens to a Celery task if a worker dies mid-task, how you would add resume analysis as an async task, how you would scale to 10 times the users.
- **Fill in:** your real bug story, and the load test or profiling you used for the latency numbers.

**TaskFlow** (the deliberately over-engineered task manager)

- **One-line pitch:** a task manager built to put distributed systems, database engineering, algorithms, MLOps and GenAI into one production-grade codebase.
- **Draw:** Flask API, Redis plus Celery workers, PostgreSQL with write and read models (CQRS), Celery Beat ETL into `weekly_reviews`, an ML service behind a circuit breaker, an embeddings and RAG path, WebSocket with HTTP-polling fallback.
- **Decisions to defend:** cache-aside with a `SETNX` mutex to stop stampedes, CQRS (why separate OLTP writes from OLAP reads, and the staleness you accept), optimistic locking with a `version` column (versus pessimistic locking), DAG dependencies with 3-colour DFS for cycle detection and Kahn's algorithm for ordering, XGBoost to correct time-estimation bias with weekly PSI drift checks and retraining above 0.20, idempotency keys, the outbox pattern, circuit breaker states, and the audit log.
- **Be ready for the 'is this over-engineering' challenge.** Your honest answer is that it is deliberate, as a learning vehicle, and you can say which parts you would drop for a real product (CQRS and event sourcing add cost, and a small app rarely needs them) and which you would keep (idempotency, optimistic locking, indexes).
- **Likely deep dives:** write the cycle-detection code live, explain a lost update and how the version column prevents it, explain PSI and why 0.20, what the circuit breaker does to a user request while open, how semantic search with `all-MiniLM-L6-v2` and cosine similarity works and how you would index it at scale.
- **Fill in:** which of the nine domains you understand deepest, so you steer the interview toward it.

**Healthcare Appointment Follow-up Manager** (FastAPI, PostgreSQL, Redis, Celery worker and Beat, Docker Compose, Railway)

- **One-line pitch:** an appointment and follow-up system, built in 18 hours as a company assignment, with double-booking prevention and AI-assisted follow-ups.
- **Draw:** FastAPI app, PostgreSQL slots table with the status state machine, Redis for caching and rate limiting, Celery worker and Beat, the AI provider interface (Gemini by default, swappable by environment variable), Google Calendar OAuth sync, email outbox.
- **Decisions to defend:** the lock-free compare-and-swap booking (`UPDATE ... WHERE status = 'free'`) and why it beats check-then-book, the free to held to booked hold with a 10-minute reservation and a 60-second sweep, fail-open Redis, outbox for at-least-once emails, a heuristic fallback on LLM timeout, Fernet-encrypted refresh tokens, the swappable provider protocol.
- **Great story for deadline and prioritisation questions:** you shipped a working, deployed system in 18 hours. Be ready to say what you cut and what you protected first.
- **Likely deep dives:** prove there is no double booking under 100 simultaneous requests (describe the concurrency test), what if the worker is down when holds expire, what are the downsides of failing open, how does the calendar sync handle conflicts, how would you move this from one database to many shards.
- **Fill in:** the order you built things in during those 18 hours, and one thing you would redo.

**InventoryPro** (Next.js, Prisma, Neon PostgreSQL, Vercel, NASSCOM hackathon)

- **One-line pitch:** an inventory management system for small and medium enterprises, built for a hackathon.
- **Decisions to defend:** JWT checked at the Edge Middleware with `jose` (fast, runs before the origin, and why `jose` rather than a Node-only library), Prisma with pooled connections on serverless Postgres, stock changed only on order state transitions (Sales Order Dispatch deducts, Purchase Order Completed adds) and inside atomic updates so counts reconcile, a custom lightweight CSS design system rather than a component library for a flicker-free dark mode, Master-Detail layout.
- **Likely deep dives:** what happens if two dispatches hit the same stock simultaneously, how you prevent negative stock, what a cold start does to latency on Vercel and Neon, how you would add multi-warehouse support.
- **Fill in:** what you shipped within hackathon time, and the demo flow you would show.

**ML projects:** text-classification pipeline, Smart MCQ Solver, Resume Analyzer

- **Text classification pipeline:** stacking ensemble on a dataset of 200,000+ records, 0.826 Macro F1. Be ready for: why Macro F1, class distribution, how you built the stack without leaking (out-of-fold predictions for the meta-model), baselines you beat, error analysis, and why stacking over a single boosted model.
- **Smart MCQ Solver:** frozen RoBERTa features, a BiLSTM with attention, and LightGBM stacking for a Kaggle-style competition. Be ready for: why freeze the encoder (cost, overfitting on small data), what attention added, how you validated, leaderboard versus local score gaps.
- **Resume Analyzer:** deployed on Hugging Face Spaces, skill matching with embeddings, planned as a Celery task inside PPA V2. Be ready for: how skills are extracted and matched, how you evaluate quality without labels, latency, and privacy of resume data.
- Mention your Hugging Face open-source contribution wherever it fits, with exactly what you changed and why it was accepted.

### Coding questions to expect from your own projects

- Write cycle detection and a topological sort (TaskFlow's DAG).
- Implement an atomic conditional update and a retry loop, or an LRU cache (Redis eviction, caching).
- Write a SQL query for a report your system needs (tasks per user per week, utilisation of slots per provider).
- Implement a token bucket, or a debounced retry with exponential backoff and jitter.
- Design a schema for a feature you did not build yet, and say which indexes it needs.

## Part 6: Behavioural and HR rounds

### Tell me about yourself (60 seconds)

Use three beats: who you are now, what you have built that is relevant, what you want next. Put it in your own voice and cut anything you would not say to a person across a table. A draft you can bend:

'I'm Parth, a final-year computer science student at VIT Bhopal, and I'm also doing the data science degree at IIT Madras alongside it. I enjoy backend and applied ML work. My biggest projects are a placement portal on Flask, Celery and Redis, and a task manager where I tried to build real distributed-systems ideas into one codebase: caching with stampede protection, optimistic locking, idempotent retries and a drift-monitored ML model. I also built a healthcare booking system in 18 hours for an assignment, where I prevented double booking with an atomic compare-and-swap. I've contributed to an open-source project on Hugging Face too. I'm looking for an SDE or ML role where I can keep working on backend systems that have to be correct under load, and this role looks like exactly that.'

Adjust the last sentence to each company's job description. Do not recite grades unless asked.

### Five STAR stories to prepare

STAR means Situation, Task, Action, Result. Spend most of your time on Action (what you did, with 'I' rather than 'we') and finish with a concrete Result plus what you learned. Prepare five stories that can be bent to many questions:

1. **Delivering under a tight deadline:** the 18-hour appointment system. What you prioritised, what you cut, how you kept it working and deployed.
2. **A hard technical problem:** a concurrency bug or a performance bottleneck from your projects, with how you diagnosed it.
3. **Initiative or going beyond the task:** the Hugging Face open-source contribution, or building TaskFlow as a learning vehicle.
4. **A mistake or failure:** a real one, with ownership, the fix and the change in habit. Avoid disguised strengths ('I work too hard').
5. **Teamwork, disagreement or helping someone:** a time you disagreed, persuaded with evidence or compromised, or helped a peer get unstuck.

Write each as five or six bullet points, not a script. Learn the bullets, not the sentences, so it sounds like you.

### Common questions and how to approach them

- **Strengths:** pick two that the role needs and back each with a specific example (backend fundamentals with concurrency, shipping end to end).
- **Weaknesses:** a real, non-fatal one, plus what you are doing about it. For example, a tendency to over-engineer (which is visible in TaskFlow) and how you now ask 'what is the simplest thing that meets the requirement' first.
- **Why this company or role:** one specific thing about their product, scale or engineering problems, tied to something you have done. Read their engineering blog or recent launches the night before.
- **Why should we hire you:** the overlap between what they need and what you have demonstrated, in three sentences.
- **Where do you see yourself in five years:** growing into someone who owns a system and mentors others, in the same domain as the role. Do not say you want their manager's job.
- **Conflict with a teammate, handling pressure, handling criticism:** use a STAR story, stay factual, show you listened and changed something.
- **Relocation, shifts, joining date, bond or service agreement:** know your constraints before the round and answer plainly.
- **Expected CTC:** do not guess. Research the range for that role and level (company pages, Glassdoor, AmbitionBox, levels.fyi, seniors who joined). For campus offers the band is usually fixed, so say you are looking for the standard package for the role and care most about learning and growth. If pushed in an off-campus process, give a researched range, not a single number, and say you are flexible for the right team.

### Two good questions to ask the interviewer

Pick two that you genuinely care about and that show you thought about the work:

- What does the first six months look like for someone in this role, and how is success measured?
- What is the hardest engineering problem the team is working on right now?
- How does the team handle on-call, testing and code review?
- What would make someone excellent in this role versus just good?

Do not ask anything the careers page already answers, and do not open with salary or leave.

## Part 7: Getting hired

### Application materials

- **Resume:** one page, a single column, standard headings (Education, Skills, Projects, Experience, Achievements), plain fonts and no tables or images so applicant tracking systems parse it. Each project gets two or three bullets that follow 'did X using Y, which resulted in Z', with numbers only where you can defend them. Match keywords to each job description honestly. Export a PDF and name it clearly.
- **LinkedIn:** a headline that says what you do and want ('Backend and ML engineer | Python, FastAPI, Redis, Celery'), a short About section, projects featured with links, and the same story as your resume.
- **GitHub:** pin your best four repositories, each with a README that has a one-line purpose, a screenshot or demo link, an architecture diagram, setup steps, and a short 'design decisions' section. Remove half-finished experiments or mark them archived.
- **Live deployments:** at least two projects with working links, checked the day before every interview.
- **One language you are strongest in** (Python) for coding rounds, with enough of a second (C++ or Java) to read it.
- **Target list:** shortlist companies and roles, grouped by how well your profile fits (SDE, backend, ML or data), and read each job description for its real requirements before you prepare for that company.

### Referrals and applications

- Referrals raise your chance of being seen, but they do not replace the interview. Prepare a short, specific message for each senior or contact and make it easy to say yes:

'Hi \[name\], hope you're well. I'm applying for the \[role\] opening at \[company\] (link). I've worked on \[one relevant thing, one line\]. Could you refer me, or point me to the right person? I've attached my resume and can send a short summary of the role fit if that helps. Thanks either way.'

- Keep a **tracker** (a spreadsheet or Notion table) with company, role, link, date applied, referral status, stage, next date, deadline, interviewer notes and follow-up date. Check career pages on a fixed rhythm, and apply early because many roles close once enough applications arrive.
- After every interview, write down every question you could not answer fully within an hour, and fix it before the next one. This mistake log is the highest return habit on the whole list.
- Send a short thank-you or follow-up within a day, and a polite nudge after the date they promised has passed.

### Mock interviews

- Do at least three with seniors, one each for coding, technical fundamentals plus projects, and design. Ask them to interrupt you, because real interviewers do. Record yourself once for the project walkthrough and watch it back.

### Offers and negotiation

- Compare offers by **role and scope, team and manager, learning curve, tech and domain, compensation, location and flexibility, and growth** (promotion and company stability). Compensation means total, not just fixed: variable pay, joining bonus, stock, benefits, notice or bond terms.
- Be honest and polite when negotiating: thank them, state that you are excited, give your reason (a competing offer, market data), and ask whether there is flexibility. Never invent an offer. Get the final terms in writing before you decline anything else.

## Part 8: Weekly rhythm and interview-day checklist

The last line of your checklist is the best advice in it: prioritise from the job description instead of mastering every advanced topic before applying. A sustainable weekly rhythm:

| Block | Frequency | What |
| --- | --- | --- |
| DSA | 5 days, about 90 min | 2 problems by pattern, then revisit last week's mistakes |
| Timed OA | 1 per week | 90 minutes, three problems, log failures |
| CS fundamentals | 3 days, 45 min | One OS, DBMS or networks topic, answered aloud |
| SQL | 2 days, 30 min | 4 to 5 queries, window functions and gaps-and-islands |
| Design and LLD | 2 days, 60 min | One HLD or LLD problem end to end, on paper |
| Projects | 1 day, 45 min | Walkthrough out loud, one deep-dive question per project |
| Applications | 2 days, 30 min | Tracker, referrals, tailoring the resume |
| Mock interview | 1 per week | Rotate coding, technical, design |

**Night before an interview:** read the job description and company product, skim your mistake log and project cards, open every live link, charge your laptop, test audio and video, and sleep.

**During the interview:** repeat the question back, clarify, give the brute force before the optimised version, think aloud, test your code with an example and an edge case, state complexity, and ask for a hint instead of going silent for five minutes.

**Afterwards:** write down the questions, mark the ones you missed, send the follow-up, update the tracker.

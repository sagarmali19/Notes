# System Design — Interview-Ready Notes

---

## 1. What Is System Design & Why It Matters

**Definition:** `System = Components + Common Goal`. A system is a set of components working together toward a shared purpose.

**Key interview soundbite:** "Anyone can write code that works. System design is what makes it work for millions of people at once."

**Why interviewers test it:** Writing code that works for 10 users is trivial. Making it work for millions of concurrent users (WhatsApp, Instagram, Amazon) requires deliberate architectural decisions — that's the system design round, even if not labeled as such.

### The Alien Bank Analogy (mental model to recall components fast)
A single cashier serving customers maps directly onto system design vocabulary:

| Bank Problem | Fix Applied | System Design Term |
|---|---|---|
| Cashier is slow (10 min/customer) | Train cashier to be faster | **Code optimization (LLD/DSA)** |
| More customers arriving | Bigger desk, cash-counting machine | **Vertical scaling** (upgrade the single unit) |
| Customers still wait too long | Add a second counter | **Horizontal scaling** (add more units) |
| Two counters don't share data → double withdrawal possible | Central record book both counters check | **Centralized database** |
| One counter overloaded, other idle | A person who routes customers by queue length | **Load balancer** |

Mapping: Customers = **requests/traffic**, Counter = **server**, Cashier's work = **application code**, Central record = **database**, Router person = **load balancer**.

---

## 2. Data-Intensive vs Compute-Intensive Applications

**Interview trick question:** "How would you fix latency in this app?" — Answer depends entirely on which category it falls into. Picking the wrong lever wastes money and doesn't fix the problem.

### Data-Intensive
- **Bottleneck:** moving/storing/retrieving large volumes of data, not calculation.
- **Symptoms/worries:** slow DB reads, safe storage, concurrent access by many users, server/DB/network failure handling.
- **Fix with:** better databases, caching, replication, sharding, CDNs, data consistency strategies — **not** CPU/GPU upgrades.
- **Examples:** Instagram feed, WhatsApp messaging (2–10M messages/day), bank transactions, analytics dashboards, log processing.

### Compute-Intensive
- **Bottleneck:** the amount of data is small, but processing it is heavy.
- **Symptoms/worries:** how fast can we compute, can tasks run in parallel, can we reduce compute cost, CPU vs GPU choice.
- **Fix with:** parallelization, GPUs, better algorithms.
- **Examples:** image/video processing, ML model training, simulations (flight simulators), cryptography.

### Key nuance
A single app can be **both**, depending on feature: YouTube is data-intensive for serving videos but compute-intensive for generating recommendations.

**Rule of thumb to say in interview:** *"If time is lost in data movement, it's data-intensive. If time is lost in computation, it's compute-intensive."*

---

## 3. Functional vs Non-Functional Requirements

Always state this framework early in a system design interview — it shows structured thinking.

**Functional Requirements (WHAT the system does)** — every feature/action a user can take.
> Example (Amazon): register, login, browse/search/filter products, add to cart, apply coupon, place order, pay, track order.

**Non-Functional Requirements (HOW WELL the system performs)** — quality attributes, usually with measurable targets.

| NFR | Example Definition |
|---|---|
| Scalability | Handle 1M daily active users; scale horizontally/vertically |
| Performance / Latency | Response time < 200ms; track P90/P95/P99 |
| Availability | 99.9% uptime, governed by SLAs/SLOs |
| Reliability | No data loss |
| Security | Authentication, authorization, encryption |
| Fault Tolerance | Handle 10x traffic spikes (Black Friday), recover from failures |
| Observability | Monitoring & logging in place |

**Interview tip:** Always clarify both categories with the interviewer *before* drawing any diagram — it's the "smart way" to start, versus jumping straight to boxes and arrows.

---

## 4. DNS (Domain Name System)

**Problem it solves:** Machines communicate via IP addresses, not names. There are 350M+ domains — too many to store in a browser or a single server (and IPs change over time), so a distributed lookup system is needed.

### Resolution Flow (memorize this sequence)
1. Browser/OS asks the **DNS Resolver** (via ISP).
2. Resolver hits a **Root Server** (13 logical root servers worldwide, named A–M, run by ~12 orgs). Root server doesn't know the site — it knows which **TLD** server to go to (.com, .net, .in, etc.).
3. Resolver hits the **TLD server** → returns the **Authoritative Name Server** for the domain.
4. Resolver hits the **Authoritative Name Server** → returns the actual IP address.
5. Browser makes the request to that IP.

**Caching layers (3 places):** DNS Resolver cache, OS cache, Browser cache — so steps 1–4 only run on the *first* lookup.

**Zone:** A domain + all its subdomains (e.g., `docs.example.com`, `courses.example.com`) managed together under one authoritative name server.

**Domain vs Subdomain:** `example.com` is the domain; `docs.example.com` is a subdomain under it.

---

## 5. APIs — The Communication Layer

**Why an application layer exists between client and DB:** Users shouldn't need to know SQL/table structure. The app layer hides complexity and exposes **endpoints**.

### 5 Types of APIs (be ready to compare all 5)

| Type | Format | Best For | Notes |
|---|---|---|---|
| **REST** | JSON | Most modern apps | Simple, low overhead, industry default |
| **SOAP** | XML | Legacy systems | Bulkier than JSON; still seen in older enterprise systems |
| **GraphQL** | Query language over single endpoint | Flexible client-driven data fetching | One endpoint, client specifies exactly what data it needs (vs REST's multiple fixed endpoints) |
| **gRPC** | Protocol Buffers (binary) | Internal microservice-to-microservice comms | Smaller payload than JSON/XML → very fast; built by Google; low latency |
| **WebSockets** | Persistent bidirectional channel | Real-time: chat, live notifications, quizzes | Unlike request/response APIs, the **server can push data** without the client asking first |

**Interview line for gRPC:** "Used internally between microservices where low latency matters more than human readability."

**Interview line for WebSockets:** "Once the channel is established, either side can initiate — eliminates the need for constant client polling."

---

## 6. REST API — Deep Dive (very commonly tested)

### Anatomy of a Request
- **Method + Path** = the endpoint. E.g. `GET /api/v1/users`
- **Path**: used for unique identifiers only (IDs, slugs) — never sensitive data (it's exposed in logs/URLs).
- **Query Params**: used for filtering, sorting, searching (`?sort=asc&q=java`) — never sensitive data.
- **Body**: used for sensitive data (passwords) and full payloads (create/update).

### HTTP Methods
| Method | Purpose | Example |
|---|---|---|
| GET | Retrieve data | `GET /users/1` |
| POST | Create new entity | `POST /users` with body |
| PUT | **Full replace** of an entity | Sending partial data via PUT nulls out missing fields! |
| PATCH | **Partial update** — only changed fields | Preferred for updating a single field like username |
| DELETE | Remove entity | `DELETE /users/1` |

**Common interview gotcha — PUT vs PATCH:** If a user object has `{id, name, username, age}` and you PUT with only `{username}`, the other fields get wiped/defaulted. PATCH updates only what's sent. **Always prefer PATCH for partial updates.**

### Nesting vs Filtering
- **Nesting** (`/blogs/1/comments`) → use when there's a **clear, direct relationship** between two entities.
- **Filtering** (`/comments?postId=1`) → use when the relationship is **complex** or you need pagination/multi-attribute filtering (color, price range, etc.). Query params are preferred over body for filtering.

### HTTP Status Codes (know these cold)
| Code | Meaning | When |
|---|---|---|
| 200 | OK | Successful request |
| 201 | Created | New entity successfully created (POST) |
| 204 | No Content | Success, but nothing to return (e.g., DELETE) |
| 301 / 302 | Redirect | Permanent / Temporary |
| 400 | Bad Request | Validation error / malformed body |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Authenticated but not allowed (e.g., no course access) |
| 404 | Not Found | Resource doesn't exist |
| 500 | Internal Server Error | Backend bug — don't expose details to user, log it instead |

### Response Body Best Practice
**Never return a raw array.** Always wrap responses in an object:
```
// Bad
[ {...}, {...} ]

// Good
{ "users": [ {...}, {...} ], "count": 2 }
```
**Why:** Wrapping gives you flexibility to add metadata (counts, pagination cursors) later without breaking existing clients.

---

## 7. SQL vs NoSQL

### SQL (RDBMS)
- Structured **tables** with **rows** (records) and **columns** (attributes); relationships enforced via keys.
- Engines: PostgreSQL, MySQL.

**Constraints (know all 5):**
| Constraint | Purpose | Example |
|---|---|---|
| UNIQUE | No duplicate values | Username |
| NOT NULL | Field is mandatory | First name |
| PRIMARY KEY | Unique row identifier | User ID |
| CHECK | Custom validation rule | Password length ≥ 8, phone = digits only |
| FOREIGN KEY | Links to another table's primary key | `posts.author_id → authors.id` |
| DEFAULT | Fallback value when none given | New user role = "student" |

**Relationship types (know all 4 with examples):**
- **One-to-Many**: one author → many blog posts.
- **Many-to-One**: many posts → one author (reverse view of above).
- **Many-to-Many**: students ↔ courses (needs a **junction table**, e.g. `student_courses(student_id, course_id)`).
- **One-to-One**: a `content` table storing metadata, pointing to exactly one row in a `videos`/`audios`/`blogs` table via foreign key.

**Scaling SQL is hard** because relationships must be maintained across servers when you scale horizontally — this is NoSQL's main selling point.

### NoSQL
Advantages: easy to scale (horizontal/vertical), **schema-less** (documents can have different fields), fewer/no relationships to maintain, generally faster reads.

**4 Types of NoSQL databases (know examples for each):**
| Type | Structure | Use Case | Example DBs |
|---|---|---|---|
| **Key-Value** | Unique key → any value (JSON/string/blob) | Caching, sessions, cookies | Redis |
| **Columnar** | Read column-wise instead of row-wise | Analytics/aggregation (e.g., avg of one column across millions of rows) | Google BigQuery, Amazon Redshift, Databricks |
| **Graph** | Nodes (entities) + Edges (relationships), both can have properties | Pattern detection, recommendation, fraud detection | Neo4j-style (Gremlin, SPARQL, Cypher) |
| **Document** | JSON documents, flexible schema, no relationships | Logging, user profiles, mixed content types | MongoDB, CouchDB |

**Real company examples (great for interviews):**
- Netflix → Cassandra (user activity)
- Amazon → DynamoDB (scaling)
- Meta/Facebook → HBase (messaging, large-scale storage)
- Uber → MongoDB (flexible real-time data)
- Twitter/X → Redis (caching, timelines)

### When to pick which
- **SQL:** consistency-critical domains — payments, transactions, anything needing strict relationships.
- **NoSQL:** scalability/speed-critical, unstructured or fast-changing data, availability > strict consistency.

**One-liner:** *"When consistency > availability, pick SQL. When availability/scalability > strict consistency, pick NoSQL."*

---

## 8. Caching

**Problem it solves:** Repeatedly hitting the DB for the same data (e.g., homepage course listings) adds latency. Example from transcript: 3 DB tables + 1 DNS call × 6 courses = **24 requests per page view** — multiplied by thousands of users, this breaks the system.

**Definition:** A smaller, faster memory layer (front-end, back-end, or DB-level) storing frequently accessed data.

### Key Terms
- **Cache Hit:** requested data found in cache → fast response.
- **Cache Miss:** data not in cache → must query DB (slower) → then usually populate cache.
- **TTL (Time To Live):** how long an entry stays in cache before expiring.
- Cache is intentionally **smaller** than the DB (else lookups get slow again) and must be **refreshed** as "hot" data changes (e.g., trending course changes from Java to AI).

### Caching Strategies (very frequently asked — know all 4)

| Strategy | Read Path | Write Path | Use Case |
|---|---|---|---|
| **Read-Through Cache (RTC)** | App reads only from cache; on miss, cache itself fetches from DB and stores it | Writes go directly to DB (cache untouched) | General read-heavy data |
| **Write-Through Cache (WTC)** | — | Every write goes to cache first, then synced to DB | Data that must always be fresh, e.g., stock prices |
| **Write-Around Cache (WAC)** | Reads go through cache (miss → fetch from DB → populate cache) | Writes go **directly to DB**, bypassing cache | Twitter-style: importance of data decided at *read* time, not write time |
| **Write-Back Cache (WBC)** | Reads served from cache | Writes go to cache first (fast ack to user), then **asynchronously** flushed to DB | High-write-throughput apps like Swiggy/Zomato order-status updates — trades consistency for speed |

**Interview tip:** WBC trades consistency for speed (data might be lost if cache crashes before DB sync) — mention this trade-off explicitly.

### Cache Eviction Policies (know all 6)
| Policy | Full Form | Removes |
|---|---|---|
| **LRU** | Least Recently Used | Data not accessed recently (e.g., iPhone 11 searches after iPhone 17 launches) |
| **MRU** | Most Recently Used | Data just used and unlikely to be needed again (e.g., already-applied coupon, already-streamed video segment) |
| **LFU** | Least Frequently Used | Data accessed rarely overall (e.g., one-time plant search vs. frequent gadget searches) |
| **FIFO** | First In First Out | Oldest inserted entry, regardless of usage |
| **LIFO** | Last In First Out | Most recently inserted entry (stack-like) |

---

## 9. Load Balancers

**Two core responsibilities:**
1. **Choose which server** handles the next request (via an algorithm).
2. **Health-check** servers (passive: just observe traffic; active: send test/probe requests).

### Load Balancing Algorithms (know all 6 + trade-offs)

| Algorithm | How It Works | Pros | Cons |
|---|---|---|---|
| **Round Robin** | Requests distributed sequentially (S1→S2→S3→S1...) | Simple, equal distribution | Ignores server capacity differences (weak server gets same load as powerful one) |
| **Weighted Round Robin** | Round robin but higher-capacity servers get proportionally more requests | Fixes RR's capacity blindness | Requires maintaining/updating weights |
| **Geo-Based** | Routes based on user's location to nearest server | Lower latency for global apps | VPNs mislead routing; needs regional DB replication; complex failover |
| **Least Connections** | Sends request to server with fewest active connections | Good for long-lived/sticky sessions (WebSockets, chat) | Ignores request weight/complexity; still needs constant connection-count updates |
| **Least Time (Response Time)** | Sends to server with lowest average response time | Best for latency-sensitive apps (trading, search) | Complex to calculate; sensitive to traffic spikes |
| **IP Hash** | Hash of client IP maps deterministically to a server | Keeps a user's session on the same server (session stickiness) | Hard to scale — adding/removing a server reshuffles the whole hash mapping; largely replaced by JWT-based stateless auth |

**Modern reality:** Real systems combine multiple algorithms (hybrid), not just one.

### Health Check Parameters
- **Interval** — how often to check (e.g., every 5s).
- **Timeout** — how long to wait before flagging as unhealthy.
- **Threshold** — number of failed/successful calls before marking unhealthy/healthy (e.g., 20 consecutive failures = alarm).

---

## 10. Replication

**Purpose:** Keep copies of data across multiple nodes to avoid single point of failure, increase availability, keep data close to users (lower latency), and increase read throughput.

### Single-Leader Replication
- All **writes** go to one **leader**; leader propagates to **followers**.
- **Sync replication:** leader waits for all followers to confirm before acking the user. ✅ Consistent, easy failover (any follower can become leader) ❌ Slow, blocks resources — **rarely used in practice**.
- **Async replication:** leader acks the user immediately, propagates in background. ✅ Fast, non-blocking ❌ Followers may briefly serve **stale data** (consistency trade-off).
- **Adding a new follower:** take a full snapshot from an existing node, apply it, then apply the delta of changes since the snapshot, then connect it live.
- **Leader failure:** elect a new leader from the follower with the most up-to-date data (via timestamps); reconfigure other followers to point at it.

### Multi-Leader Replication
- Multiple data centers each have their own leader+followers; leaders sync with each other.
- ✅ Better fault tolerance, supports collaborative concurrent writes (e.g., Google Docs-style).
- ❌ **Write conflicts** are possible (two leaders accept different updates to the same record).

**Conflict resolution strategies (know all 3):**
1. **Last Write Wins (LWW)** — timestamp-based, most recent write survives.
2. **Higher ID/Priority Wins** — leader with higher assigned ID's value wins, regardless of timing.
3. **Manual resolution** — like a Git merge conflict; surface both versions to the user/app to decide.

### Leaderless Replication
- No leader — client sends read/write requests to **multiple nodes directly** (used by Amazon Dynamo, Cassandra, Riak).
- **Quorum** determines success: for `N` total nodes, need **more than N/2** to acknowledge a write/read.
- Reads might return **stale data** from lagging nodes — quorum majority resolves which value is "truth."

---

## 11. Partitioning (Sharding)

**Purpose:** Even replication can't help if data volume itself is too large for one node, or indexes get too slow to search — so data is **split** (partitioned) across nodes.

**Two rules of a valid partition scheme:**
1. Combining all partitions must reconstruct the full dataset (no data loss).
2. Data should be **evenly distributed** to avoid a "hot" node.

### Partitioning Approaches
| Approach | How | Risk |
|---|---|---|
| **By Key Range** | e.g., user IDs 1–50,000 → Partition 1; 50,001–100,000 → Partition 2 | Can create a **hotspot** if traffic isn't evenly distributed by key (e.g., all US users happen to land in one partition) |
| **By Hash of Key** | Hash function maps ID → partition number/range | More even distribution, but still can hotspot; harder to reason about ranges |

**Hotspot:** a partition receiving disproportionately more traffic than others — dangerous because it can overload and crash that node.

### Secondary Indexes Under Partitioning
- **Local Secondary Index (per partition):** each partition keeps its own index (e.g., by color). Used by Cassandra, Elasticsearch. ❌ Still must query *every* partition even if most return "no match."
- **Global Secondary Index (centralized):** one central index maps key → which partition holds it. ✅ Faster reads (only look up right partition) ❌ Slower/more complex writes (must update the central index too).

---

## 12. CAP Theorem

**You can only guarantee 2 of 3 in a distributed/partitioned system:**
- **C — Consistency:** every read gets the latest write.
- **A — Availability:** every request gets *a* response (even if not the latest data).
- **P — Partition Tolerance:** system keeps working despite network partitions between nodes.

| Combination | Meaning | When It Applies |
|---|---|---|
| **CA** | Consistent + Available | Only possible with a **single node/DB** — no partition exists to tolerate |
| **CP** | Consistent + Partition-tolerant | Sacrifice availability — node with stale data refuses to respond until synced. Used for **banking/financial transactions** |
| **AP** | Available + Partition-tolerant | Sacrifice consistency — always respond, even with stale data. Used for **social media** (Instagram: a slightly stale like-count is fine) |

**Since real distributed systems always need Partition Tolerance (P) once they scale beyond one node, the real trade-off in practice is C vs A.**

**Interview soundbite:** *"Banking = CP (can't show wrong balance). Instagram = AP (a few seconds of staleness is fine, but the app must stay up)."*

---

## 13. Message Queues

### Sync vs Async (foundational distinction)
- **Sync:** caller waits for a response before continuing. Required when there's a **dependency** (e.g., inventory must update before confirming an order) or in **financial/security-critical** flows.
- **Async:** fire-and-forget; caller moves on immediately. Used for **broadcast** actions: emails, SMS, notifications, OTPs (30s+ timers exist because the system handles it async).

### What a Message Queue Does
Sits between a **producer** (publisher) and a **consumer** (subscriber) as a buffer/broker:
1. **Holds** requests until they can be processed.
2. **Delivers** requests to consumers (push or pull).
3. **Handles failures/retries** so the producer doesn't have to.
4. **Prevents duplicate processing.**
5. Manages **ordering** and **prioritization**.

### Ordering Approaches
- **Strict FIFO:** if one message fails, the whole queue can stall — risky.
- **Unordered Queue:** failures are skipped/retried independently; the queue keeps moving — generally preferred.
- **Priority Queue:** each message gets a priority score; higher-priority messages are served first regardless of arrival order.

### Delivery Models
- **Pull-based:** consumer decides when to fetch (pulls when free).
- **Push-based:** the queue actively pushes messages to the consumer as they arrive.

### Key Failure-Handling Concepts
- **DLQ (Dead Letter Queue):** messages that repeatedly fail processing are routed here instead of blocking the main queue — allows investigation without stalling the system.
- **Poison Message:** a message that can never be successfully processed (invalid/corrupt) — wastes consumer resources if retried blindly; should be moved to DLQ.
- **Duplicate handling:** once a message is successfully processed, it must be removed/marked so it isn't processed twice (important for non-idempotent operations like charging a payment).

### When to Use / Avoid Message Queues
✅ Use for: async workflows, decoupling services, analytics/logging pipelines, load-leveling (acts like a load balancer for backpressure), deferred/scheduled processing (e.g., nightly report generation).

❌ Avoid for: low-traffic systems (queues cost money/infra), real-time systems needing immediate synchronous response, any flow requiring an acknowledgment/confirmation the caller must wait on.

---

## 14. Faults & Errors — 3 Categories

| Category | Nature | Examples | Mitigation |
|---|---|---|---|
| **Hardware** | Random, unpredictable, mostly inevitable | Disk full, power failure, network cable damage, server overheating, DB crash | Redundancy — multiple servers/DBs so one failure doesn't take down the system |
| **Software** | Deterministic — same steps reproduce the same bug | Bad exception handling, missed edge cases, config mismatches between dev/prod, merge conflicts, slow queries | Careful code review, thorough testing, consistent environment configs |
| **Human** | Unpredictable, but arguably most powerful/critical component | Mistakes in judgment, bad quick-fixes instead of root-cause fixes | Good engineering discipline, root-cause fixes over patches, accountability (even AI-written code needs human review) |

---

## 15. Monitoring & Observability

Split into **API Monitoring** and **Machine (Hardware) Monitoring**.

### API Monitoring
- **Throughput:** requests handled per second vs. capacity (alert before hitting the ceiling).
- **Error Codes:** track volume of 4xx/5xx responses; alert on spikes; maintain logs for RCA (root cause analysis).
- **Health Checks:** track 200-OK rate over time (passive/active, as covered in Load Balancers).
- **Latency via Percentiles (P-values):** Don't use plain averages — they hide outliers.
    - **P50:** 50% of requests complete within this time (median).
    - **P90 / P95 / P99 / P99.9:** the time within which 90%/95%/99%/99.9% of requests complete.
    - **Interview example:** if P50 = 4s but P90 = 12s, something is wrong with the slower 10% of requests — investigate and optimize those specifically.

### Machine (Hardware) Monitoring
- **CPU usage** — alert at a threshold (e.g., 75%).
- **Memory usage** — alert before capacity is exhausted (e.g., 90%).
- **Disk I/O** and **Network usage** — same alerting principle.

---

## 16. Worked Example: Designing a Video Streaming System (YouTube/Netflix-style)

**Interview approach demonstrated:** Clarify scope first (here: only video delivery pipeline — no login/ratings). Never add components "because you know them" — every component adds cost.

### Core Concepts
- A video = sequence of image **frames** (measured in **FPS** — 30 or 60 typically) + audio.
- **Never download the whole file before playing** — instead split into **segments/chunks** and stream them progressively.
- Protocols: **RTMP** (Real-Time Messaging Protocol) and **RTSP** (Real-Time Streaming Protocol), both over **TCP** (needed because video requires ordered, reliable delivery).

### Adaptive Bitrate Streaming (ABR) — the key technique
1. Source video is transcoded into **multiple resolutions** (4K, 1080p, 720p, 480p, 240p).
2. Each resolution is split into **time-based segments** (e.g., every few seconds).
3. Client continuously monitors network throughput and **switches resolution segment-by-segment**:
    - Network throughput > current segment bitrate → **upgrade quality** on the next segment.
    - Network throughput < current segment bitrate → **downgrade quality** to avoid buffering.
4. This is why players show "Auto / 1080p / 720p / 480p" — "Auto" runs this ABR logic automatically.

### System Components (in pipeline order)
```
Source Server → CDN (raw video) → Transcoding Service (creates resolutions)
→ Priority Queue → Worker Services (one per resolution: 1080p, 720p, 480p...)
→ Priority Queue (results) → Distributed CDN (region-specific: US, India, etc.)
→ Client Device (pulls appropriate segment/resolution)
```
- **Transcoding Service:** converts source video into multiple resolution variants.
- **Priority/Message Queues:** decouple transcoding from delivery; allow retry/failure handling.
- **Workers:** each handles converting to one specific resolution (parallelizable).
- **Distributed CDN:** places segments close to users geographically to cut latency (ties back to Geo-based load balancing).

### Back-of-the-Envelope Math (a good habit to demonstrate in interviews)
- Source: 50GB, 20-minute, 4K/60fps video.
- Segments: 20 min × 60 = 1200 seconds → 1200 segments (1 per second, for simplicity).
- Per-segment size at various resolutions (50GB ÷ 1200, 20GB ÷ 1200, etc.) summed across all resolutions ≈ **72MB total per second** of content across all quality variants.
- Multiply by concurrent viewers to size storage/bandwidth needs (e.g., 100 users → ~87.5GB fits on one server; 100,000 users → needs multiple servers, queues, and caching — this is the point where you'd bring in load balancers, replication/partitioning, and CDNs).

**Interview closing tip:** There's no single "correct" diagram — interviewers evaluate your **trade-off reasoning**, not a memorized answer. Always narrate *why* each component is added.

---

## Quick-Reference Cheat Sheet (last-minute review)

- **Scaling:** Vertical (bigger machine) → Horizontal (more machines) → needs Load Balancer + shared DB.
- **DB choice:** SQL = consistency/relationships. NoSQL = scale/speed/flexibility.
- **Cache strategies:** RTC (read-heavy), WTC (always-fresh writes), WAC (importance decided at read time), WBC (high write throughput, async DB sync).
- **Eviction:** LRU, MRU, LFU, FIFO, LIFO.
- **LB algorithms:** Round Robin, Weighted RR, Geo-based, Least Connections, Least Time, IP Hash.
- **Replication:** Single-leader (sync/async), Multi-leader (+conflict resolution), Leaderless (+quorum).
- **Partitioning:** by key range or hash; watch for hotspots; local vs global secondary indexes.
- **CAP:** pick 2 — CA (single node only), CP (banking), AP (social media).
- **Message Queues:** sync vs async, FIFO/priority, DLQ for poison messages, push vs pull.
- **Faults:** Hardware (random) / Software (deterministic) / Human (unpredictable but critical).
- **Monitoring:** throughput, error codes, health checks, P90/P99 latency, CPU/memory/disk/network.
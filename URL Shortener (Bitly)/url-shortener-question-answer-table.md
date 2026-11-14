# URL Shortener Interview — Question-by-Question Comparison

Your answers are paraphrased from the voice record. Model answers show one consistent design; other designs can also be correct. **✓** = sound answer, **△** = incomplete, **✗** = incorrect, **—** = not independently demonstrated. Repeated clarification questions are grouped together.

## 1. Requirements and scale

| Interview question | Your answer | Strong model answer |
|---|---|---|
| Who creates links, and who can open them? | ✓ Logged-in users create; anyone can redirect. | Creation requires authentication. Redirects are public. Analytics access is restricted to the link owner. |
| Which features belong in the first version? | ✓ Create links, redirect, optional expiration and analytics. Defer custom aliases and management. | Same scope. Before designing, clarify whether repeated destinations may receive different codes. Assume yes; retries of the same creation request return the same code. |
| What should analytics show? | △ Click counts and redirect speed. | Count successful redirect responses, not unique visitors or confirmed destination loads. Show latency measured at our service boundary. Confirm acceptable freshness and event loss. |
| Does latency include destination page loading? | ✓ No; it ends when our service responds. | Measure from request arrival to the redirect response at our service boundary. Destination loading and client network time are outside this target. |
| What daily volume are we designing for? | △ Revised figures; finally one million redirects normally and ten million on a busy day. Initially one thousand creations daily. | Lock those numbers before proceeding: one million normal-day redirects, ten million busy-day redirects and one thousand creations per day. |
| How do you convert daily volume into RPS? | ✗ Initially mixed daily totals and peak RPS. | Divide by 86,400. One million/day is **11.6 average RPS**; ten million/day is **115.7 average RPS**. One billion/day would be about **11,574 RPS**. |
| What is the busiest-second traffic? | △ Around 117 RPS, calculated from a busy-day total. | That calculation gives a busy-day average. Assume a separate burst factor: for example, ten times that average gives about **1,200 peak RPS**. State that this is an assumption to validate. |
| What latency and availability targets apply? | △ p95 below 50 ms, p99 below 100 ms and 99.99% availability. | Keep those targets. Define availability as successful eligible redirect requests over a stated window. Separate system errors from valid not-found or expired-link responses. Explain the deployment needed to support the target. |

## 2. High-level architecture and storage

| Interview question                                     | Your answer                                                                                            | Strong model answer                                                                                                                                                                                                                 |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| What are the main components?                          | △ Gateway, rate limiter, load balancer, creation/redirect/analytics services, Redis, SQL and replicas. | Start with multiple application instances behind a load balancer, PostgreSQL for links, Redis for mapping lookups, and Kafka plus an analytics worker and analytics database. Give each component a clear responsibility.           |
| Why do we need microservices?                          | △ Redirects need independent scaling, so a monolith is insufficient.                                   | This workload does not establish that a monolith is insufficient. Start with a modular application and separate analytics worker. Split services when measured workload, deployment needs or team ownership justify it.             |
| What costs does a service split introduce?             | ✓ More complexity, network latency, tracing and consistency challenges.                                | Those are the main costs. Also account for deployment coordination, timeouts and partial failures. Accept them only when the isolation or independent scaling benefits are useful.                                                  |
| Why use SQL, and do we need sharding?                  | △ SQL supports consistency; suggested it cannot hold many links. No initial sharding.                  | PostgreSQL fits transactional creation, unique constraints and indexed code lookups. At one thousand creations daily, we add only 365,000 rows annually. Estimate storage and measure database limits before sharding.              |
| Walk through a redirect on a cache miss.               | △ Mostly described cached links and analytics. Later said read the database and refill Redis.          | Look up Redis, then PostgreSQL on a miss. Check existence and expiration. Cache valid mappings with a bounded TTL, return a temporary redirect, and hand the click to a bounded asynchronous publisher.                             |
| How do you keep analytics writes away from link reads? | ✗ Write to a follower of the same primary and synchronize later.                                       | Use a separate analytics database with its own writable primary. This isolates analytics writes. A physical standby of the link database is not a separate writable store.                                                          |
| Can a PostgreSQL physical standby accept those writes? | ✗ Assumed another write replica could do so; later requested clarification.                            | No. A standard physical standby is read-only. Application writes go to the primary or to another independently writable database.                                                                                                   |
| How does database failover work?                       | △ Mentioned quorum, consensus, vector clocks, Merkle trees, election and fencing.                      | Choose a concrete failover system that detects failure, promotes an eligible standby and fences the old primary. State the recovery time and possible data loss from asynchronous replication. These terms are not interchangeable. |

## 3. Analytics, retries and correctness

| Interview question | Your answer | Strong model answer |
|---|---|---|
| How do you record clicks? | △ Send events to Kafka and batch database updates. Suggested persistence is unnecessary. | Publish events with a stable event ID, code, event time and processing duration. Workers aggregate them asynchronously. Choose Kafka acknowledgments, replication and retention according to the accepted loss policy. |
| What if every analytics database node is down for thirty minutes? | ✓ Keep events in the queue and retry. Initially focused on database election. | Continue redirects. Retain unprocessed events in Kafka, avoid committing progress past failed work, retry with backoff and alert on lag and retention headroom. Catch up after recovery. |
| A worker saves a click, then crashes before committing Kafka progress. What happens? | ✗ Proposed atomicity as sufficient; also said approximate counts are acceptable. | Kafka may replay the event. Insert its stable ID and increment the count in one database transaction. If the ID already exists, skip the increment. Commit Kafka progress after the database commit. |
| Why is an atomic count update insufficient? | ✗ All-or-nothing was expected to prevent duplicate processing. | Atomicity prevents partial changes inside one transaction. It does not stop a second transaction from incrementing again. A durable event identity and uniqueness check provide replay protection. |
| Can two clicks share a timestamp? | △ Suggested versions or accepting inaccuracies. | Yes. Timestamps are not unique identities. Give each distinct click a distinct event ID and preserve that ID across retries. Accepting approximation is a separate product decision. |
| May analytics be approximate? | ✓ Yes; redirects are the main product. | Yes, if the product owner agrees. Define freshness and loss tolerance. Deliberately dropping events during an exceptional outage is an explicit trade-off; accidental timestamp-based merging is an avoidable source of error. |

## 4. Short-code creation and concurrency

| Interview question | Your answer | Strong model answer |
|---|---|---|
| How do you generate short codes and guarantee uniqueness? | △ Pre-generate seven-character Base62 codes, store a used flag and enforce a unique constraint. | Your pool approach is valid. At this creation rate, random generation on demand is simpler: generate a code, enforce database uniqueness, and retry collisions with a limit. Seven Base62 characters give about **3.5 trillion** possible codes; collisions remain possible. |
| Two instances select the same unused code. What prevents duplicate assignment? | ✓ Lock the row in a transaction; another requester waits or retries and checks the used flag. | Select and lock an eligible row inside the transaction, recheck eligibility, assign the URL and commit. A waiting allocator must not reuse a now-allocated row. SKIP LOCKED can help allocate another available row instead of waiting. |
| What if the pre-generated pool runs out? | — Not asked directly. | Define replenishment thresholds and an exhaustion policy. Either generate more codes safely or return a controlled temporary error. Do not block requests indefinitely. |
| A creation commits but its response is lost. What happens on client retry? | ✓ Use a client-generated idempotency key and return the original result. | Scope the key to the creator. Store a request fingerprint and resulting code. Same key and payload return the previous result; different payload with the same key returns a conflict. |
| The service crashes between saving the link and saving the idempotency result. Is there a gap? | ✓ Clarified both records are in one transaction. | With both records in one database transaction, they commit together or roll back together. A crash after commit leaves the result recoverable; a crash before commit leaves no committed creation. |
| How does the backend answer a retry after commit? | ✓ Retrieve the existing successful result using the idempotency key. | Read the committed creator-scoped idempotency record and return the original short URL without allocating another code. Enforce key uniqueness to handle concurrent retries. |

## 5. Expiration, redirects and caches

| Interview question | Your answer | Strong model answer |
|---|---|---|
| How do you avoid redirecting an expired cached link? | △ Store expiration in Redis; later clarified the service checks it. | Persist expiration in PostgreSQL and include it in cached values. Check it before serving either path. Cache for no longer than the remaining link lifetime. |
| Redis expires the entry, but PostgreSQL still contains the row. What happens? | △ Initially said redirect and refill cache; later clarified expiration is checked. | Read the row and check its expiration. If expired, return **410 Gone** under this product policy. Otherwise refill the cache and redirect. Existence alone does not mean validity. |
| Which redirect status would you choose? | △ 302 because 301 can bypass analytics through browser caching. | Use **302 Found** for this design. A cached permanent redirect can bypass our service on later visits. A 302 still needs an explicit response caching policy. |
| Which header prevents browsers and CDNs from storing redirects? | — Did not know; requested the answer. | Send **Cache-Control: no-store**. This controls response storage; Redis can still cache the internal URL mapping. |
| Redis goes down. What is the fallback? | △ Local cache, database reads, read replicas and CDN fallback. | Use short cache timeouts and bypass Redis. Bound database concurrency, protect connections and shed excess load if needed. Optional local cache needs memory limits and expiration checks. |
| What about reading from a replica? | △ Recognized some write/replication latency. | Replica lag can make a newly created link appear missing. Initially use the primary for cache misses, or define a deliberate fresh-link fallback strategy before enabling replica reads. |
| How can a CDN be a fallback after choosing no-store? | △ It can cache hot links during an outage. | It cannot automatically serve responses it was instructed not to store. CDN caching requires an explicit alternate policy or edge lookup design, including expiration and analytics behavior. Avoid adding it as an unexplained fallback. |
| If a CDN serves the redirect, how do you count the click? | △ Scheduled worker reads CDN data into analytics. | If the CDN supports suitable request logs or edge events, ingest those records. Define event identity, delivery delay, retries and counting rules so origin requests and edge events are not double-counted. |

## 6. Outages, monitoring and security

| Interview question | Your answer | Strong model answer |
|---|---|---|
| Kafka is down. Continue redirects or reject requests? | ✓ Continue redirects; analytics is secondary. | Continue redirects. Use short publication timeouts and bounded buffering. Analytics failure must not consume unlimited memory or hold redirect requests indefinitely. |
| Every fallback buffer is full. What do you do next? | △ Initially added more caches; ultimately chose to drop the event. | Drop new analytics events under the agreed loss policy, increment a dropped-event counter and continue redirects. Buffer limits are real limits, not a reason to invent another unbounded fallback. |
| What proves redirects are healthy while analytics is failing? | ✗ Initially cache hit/miss metrics. | Track redirect success rate and latency separately from publication failures, dropped events, consumer lag and database write errors. Cache effectiveness alone cannot show analytics delivery. |
| Which counts would you reconcile? | △ Proposed reconciliation but could not define the two counts; requested help. | Compare successful redirect responses with persisted click events or aggregates over equivalent windows, allowing for processing delay. Explain any edge-served traffic and known loss. |
| What URL validation happens at creation? | △ Format validation, creator limits, fraud detection and SQL injection blocking. | Parse with a URL library, allow only HTTP/HTTPS destinations, require a valid host and enforce length limits. Authenticate and rate-limit creation. Use parameterized SQL independently. |
| What is a URL scheme? | ✗ Answered REST/API rather than destination schemes. | It is the part before the colon, such as **https**. REST describes an API style; it is not a URL scheme. |
| What security issues remain for HTTP/HTTPS links? | — Not explored. | Valid HTTPS links can still be phishing or malware destinations. Provide abuse reporting and takedown controls. If the backend fetches destination content, add SSRF protections and network restrictions. |

## 7. Questions the interview did not fully cover

These are preparation additions, not mistakes demonstrated by your answers.

| Question | Your answer | Strong model answer |
|---|---|---|
| What APIs are required? | — Not developed. | **POST /v1/links** with authentication and an idempotency key; **GET /{code}** for redirects; **GET /v1/links/{code}/analytics** with ownership checks. |
| What schema and indexes would you use? | △ Code uniqueness and used flag discussed; no complete schema. | Links: code primary key, destination, creator ID, creation time and expiration. Creation requests: unique creator/key pair, payload fingerprint and resulting code. Analytics: event IDs for deduplication and code/time-bucket aggregates. |
| How do you test correctness? | — Not explored. | Test collisions, concurrent allocation, duplicate creation retries, crashes around commit, cache-hit/miss expiration, replayed events and dependency outages. |
| How do you validate capacity? | — Not explored. | Load-test the assumed peak and hot-link distribution. Measure latency, database connections and cache-miss load, including Redis failure. Scale based on measured bottlenecks. |
| How do you prevent auxiliary storage from growing forever? | — Not explored. | Define retention for Kafka, processed event IDs and idempotency records. Align deduplication retention with the replay window and idempotency retention with supported client retries. |

## Answer structure to practice

Use **decision → mechanism → trade-off**. For example:

> I will keep serving redirects during a Kafka outage. The publisher has a bounded buffer and drops events when it fills. This preserves redirect availability while accepting measurable analytics loss.

## Five highest-priority fixes

| Priority | Practice target | Success criterion |
|---|---|---|
| 1 | Traffic estimates | Convert daily volume correctly and state a separate burst assumption. |
| 2 | PostgreSQL replication | Explain primary writes, read-only standbys, lag, promotion and fencing without mixing mechanisms. |
| 3 | Retry correctness | Explain why transactions, idempotency keys and event IDs solve different problems. |
| 4 | Cache behavior | Explain expiration, response caching and Redis fallback as one consistent design. |
| 5 | Communication | State one definite design, walk through its steps, then name its limitation. |

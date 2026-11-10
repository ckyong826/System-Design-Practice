The key points are seperated by component to explain, i think just like thinking as software engineer, we only understand how each component working behind, then only can design a good system whatever situations. (base knowledge is important for high level design which is essential in this AI era. Code is cheap now.)

## Table of contents

- [[#DNS and CDN edge]]
- [[#Load Balancer]]
- [[#API Gateway]]
- [[#Rate Limiter]]
- [[#Microservices and Distributed System]]
- [[#Cache]]
- [[#Relational DB & Transaction]]
- [[#Replication, Consistency and Failover]]
- [[#Sharding]]
- [[#Message Queue]]
- [[#Transactional Outbox & Durable Inbox]]
- [[#Idempotency & State Machines]]

# DNS and CDN edge
## Used in?
 file downloads, news-feed media, ticket-event pages, chosen short-link redirects(hot links)
## Characteristic
- DNS map domain name to a destination.
- CDN is a geographically distributed cache between users and an origin
- Why CDN? Distance adds network delay, CDN help caching hot item to reduce repeat traffic to the server.
- DNS discovery happends before a connection is established, often using cached answers.
- CDN sits on HTTP delivery path, ahead of object storage or an application origin.
- CDN use for popular images, static assets, public downloads.
- Be careful : Dont publicly cache secret or personalized responses in CDN.
- How request work? The edge computes a cache key and checks stored response. If miss : fetch origin and returns result. Then update cache with TTL limits.
- What if data inconsistent? versioned object names help to resolve outdating cache.
## Case Study
### A popular photo in the news feed (hot asset)
1. A post references immutable object photos/p123/v1.jpg .
2. The first nearby reader causes an origin fetch.
3. Later readers use that edge’s cached image rather than repeat the origin transfer.
4. The feed JSON and permissions still come from authorized application APIs.
5. If the image changes, publish v2.jpg instead of silently replacing a permanently cached v1.jpg .
![[Pasted image 20261106161806.png]]
## Trade Offs
### Longer TTL
- Pro : More cache hits and less server traffic
- Con : Changes take longer to propogate
### Versioned immutable objects
- Pro : Simple, predictable cache behavior
- Con : Old version need to get cleared constantly
### Private CDN Delivery
- Pro : Edge speed with restricted access
- Con : Signed access, cache isolation and protected origin configuration
==A private CDN can require signed URLs/cookies and restrict direct origin access. A private origin must not become public merely to improve cache hits.==
## Fallback (Failure Solutions)
### DNS provider down 
-> New resolutions may fail. Cache answers may still work
Solution : Reduntant authoritative service, monitoring and tested DNS changes but failover is not instant.
### CDN down
-> Edge request fail
Solution : fallback an alternative delivery endpoint with authorization and origin capacity.
### Origin down
-> Cache misses fail for CDN
Solution : Serve cache content with policy. (Failed open) Report missing content rather than invent success. (Failed closed) We can do hybrid here. Block for important data access, only allow public asset to get served with cache.
### Sensitive Cached response exposed
-> Privacy incident.
Solution: Disable unsafe cache path.
# Load Balancer
Basically a traffic distributor for multi instance microservices especially for distributed system.
## Characteristic
- Chooses a backend from a pool.
- Why needed? One server has limited capacity and may fail. This help client use a changing instance as failover solution.
- Balancer will selects the target , application instances report readiness
- Where is it placed? At the public entry point, between gateway and application pools.
- When needed? Multiple instances, rolling deployment, fault isolation justify it.
- How?
	- Round robin -> assign one by one (suits similar short req)
	- Weighted Routing ->  distribute based on server assigned weight (support different capacities based on server)
	- Least Connections -> Assign to inactive server more (help uneven long lived connections)
	- Health checks stop routing to failed instances after ping pong failed.
	- Connection draining(wait server complete request only shut down) lets existing work finish.
- These way help to maintain a workable balancer to block sudden high traffic to a server instance.
## Case Study
### A chat gateway crashes
1. Three gateways hold different users’ WebSocket connections.
2. The balancer distributes new connections; it does not move an existing socket to a different gateway.
3. Gateway B crashes. Its sockets break; health checks eventually exclude it.
4. Clients reconnect with jitter, authenticate again, and register new gateway sessions.
5. Durable message history supplies missed messages after reconnection.
![[Pasted image 20261106163346.png]]
## Trade offs
### Layer 4 (route traffic based on IP level)
- Pro : Protocal flexibility and limited inspection
- Con : Less application aware routing
### Layer 7 (route traffic at HTTP/HTTPs level)
- Pro : Host/path routing, TLS termination, richer policy
- Con : More processing and configuration
### Stick sessions (Ensure connection stick with same user and instance)
- Pro : Simpler access to local session state
- Con : Uneven load and harder failover : not a durability strategy
## Failure Fallbacks
### One backend Down
-> some in-flight request failed
Solution : 
- Health checks detect Server B is unhealthy.
- Load balancer stops sending new requests to B.
- Clients may retry failed requests on A or C.
### Balancer Down
-> healthy backend unreachable
Solution : 
- Deploy redundant load balancers.
- Use multiple Availability Zones (AZs).
- Use a managed multi-AZ load balancer, such as AWS ALB.
==Avoid Single Point of Failure==
### All instance Down
-> Product specific behavior
Solution: 
- Instead of blocking all traffic, ALB still routes requests to unhealthy targets.
- Some targets might still be capable of processing requests despite failing health checks.
- If all servers are actually broken, requests will still fail.
### Deployment closes many sockets
-> reconnect surge
Solution : 
- Connection Draining: Allow existing requests to finish before shutdown.
- Staggered Rollout: Deploy one server at a time instead of restarting all servers together.
- Admission Control: Limit how many requests or connections a recovering server accepts.
==AWS ALB use **fail-open** routing when every target in group is down.==
# API Gateway
 file sharing, private chat, merchant payment APIs, and authenticated booking.
## Characteristic
- What belongs here? Routing, req-size limits, authentication, coarse authorization, quotas and req identifiers.
- Authentication answers who you are here, authorization answers what this identity may do to this reousrce.
- How check?
	- Validate token signature, issuer, audience, expiry and relevant claims.
## Case Study
###  Bob downloads Alice’s shared file
1. Bob presents a valid identity token.
2. The file service loads the file and sharing relationship.
3. It verifies that Bob may download the current version.
4. Only then does it issue a short-lived, object-specific signed URL.
5. Bob downloads directly from storage. The signed URL is a capability; anyone holding it may have the permitted access until it stops being valid.
![[Pasted image 20261106165332.png]]
## Trade offs
### Locally Validated Signed Tokens (e.g., JWT)
The API Gateway verifies the JWT signature locally without calling the authentication server for every request.
- Gain: Fast, reduces dependency on the identity service.
- Cost: Hard to revoke tokens immediately. Signing key rotation must also be managed.
Example: User logs out, but their JWT may remain valid until it expires.

### Central Policy / Permission Lookup
The API Gateway contacts a central authorization service to check whether a user has permission.
- Gain: Always gets the latest permissions.
- Cost: Adds network latency and dependency on another service.
Example: An admin removes a user's access. The next permission check can immediately reject their request.

### Cached Permissions
The API Gateway stores permission decisions temporarily instead of checking the central service every time.
- Gain: Faster responses and fewer authorization service calls.
- Cost: Permission changes may not take effect immediately.
Example: Permission cache lasts 5 minutes. A revoked user might retain access for up to 5 minutes if the stale permission is still accepted.

## Failure Fallback
### 1. Identity Provider Down
Problem: Authentication service (e.g., Auth0, Cognito) is unavailable.
- New users cannot log in or refresh tokens.
- Existing JWTs may still work if the API Gateway can validate them locally.
- Fallback: **Continue accepting valid, unexpired signed tokens**
Important: Never skip authentication just because the identity provider is down.

### 2. Permission Store Down
Problem: The database or service storing user permissions is unavailable.
- API Gateway cannot verify whether a user has access.
- Fallback: **Deny protected operations (fail closed).**
- Public endpoints can remain accessible.
Example: If the gateway cannot verify whether a user is an admin, reject the admin request rather than allowing it.

### 3. Signing Key Rotation Issue
Problem: The identity provider changes its JWT signing key, but the API Gateway hasn't received the new public key.
- Valid JWTs may fail signature verification.
- Fallback: **Use overlapping key rotation, keeping old and new keys available during transition.**
- Monitor key expiry and refresh the trusted keys
Important: Never disable signature verification to fix the issue.

### 4. Internal Service Compromised
Problem: A hacker gains control of one backend service and attempts to access other internal services.
```
API Gateway
   |
Service A (Compromised)
   |
   ├── Service B  ← Unauthorized access attempt
   └── Database C ← Unauthorized access attempt
```
Fallback / Protection:
- **Service Authorization: Verify that Service A is allowed to access Service B.**
- **Least Privilege: Each service receives only the permissions it needs.**
- **Isolation: Restrict network communication between services.**
- **Credential Rotation: Revoke or replace compromised credentials.**

# Rate Limiter
Rate limiting controls how fast a client asks. Admission control controls how much work the system accepts.

### Used in
 Rate Limiter design, ticket waiting rooms, provider quotas, login protection, and reconnection storms.
 
## Characteristics
- Limiting req per user, endpoint, identity or operation cost.
- Waiting room is an additional admission mechanism to hold demand outside the service. (like for ticket booking system, we will suddenly have a peak 10x-100x traffic at a time)
- Why not simply autoscale?
	- downstream, db connections or 1 popular seat have hard limit.

## Case study
### A concert launch (Ticket Booking)
1. One million visitors browse cacheable event information.
2. Per-user limits slow abusive refresh loops.
3. A waiting room admits checkout users at a rate the booking system can sustain.
4. Signed, expiring admission tokens are bound to the intended user/event and verified by booking APIs.
5. The database still decides whether a seat can be held. Admission does not reserve inventory.
![[Pasted image 20261106170350.png]]

## Trade Offs
Assume the limit is 100 requests per minute.
### 1. Fixed Window
Counts requests within fixed intervals (e.g., 12:00–12:01).
- Gain: Simple and fast.
- Cost: Users can send 100 requests at 12:00:59 and another 100 at 12:01:00 — 200 requests in 2 seconds. -> Lead to req burst

### 2. Sliding Log
Stores timestamps of every request and counts requests within the last 60 seconds.
- Gain: Accurate rolling-window enforcement.
- Cost: High memory usage because every request timestamp is stored.

### 3. Sliding Counter 
Uses counters from current and previous time windows to estimate the rolling request count
- Gain: Lower memory usage than Sliding Log.
- Cost: Approximate count, not perfectly accurate.

### 4. Token Bucket (Normally we choose this)
Tokens refill at a fixed rate. Each request consumes one token.
Example: Bucket holds 100 tokens, refilling at 10 tokens/second.
- Gain: Allows short bursts while controlling the long-term average rate.
- Cost: Requires atomic token updates and careful clock handling.
==Normally we use Redis Lua script for atomic tokens updates to save each user token bucket in redis.==
*I would implement a distributed Token Bucket using Redis to store per-user bucket state and Lua scripts to atomically calculate refills, check token availability, and deduct tokens. This ensures consistent rate limiting across multiple API Gateway instances.*

### 5. Leaky Bucket 
Requests enter a queue and are processed at a fixed rate.
Example: Even if 100 requests arrive simultaneously, process only 10 requests per second.
- Gain: Smooth, predictable traffic.
- Cost: Introduces waiting time; requests may be rejected when the queue is full.

## Failure Fallback
### 1. Shared Rate Limiter Unavailable (Redis Down)
Problem: API Gateways cannot access Redis to check the global rate limit.
- Effect: Cannot coordinate how many requests users are allowed.
- Fallback: Use small local limits for low-risk requests, but reject sensitive operations if limits cannot be enforced.
Example: Allow limited product browsing, but restrict payment attempts.

### 2. Local Fallback on Many Gateways
Problem: Each API Gateway independently enforces its own limit when Redis is down.
Example:
- 3 API Gateways
- Each allows 100 requests/minute
- Total possible = 300 requests/minute, instead of 100.
- Fallback: Set conservative local limits based on the expected number of gateways.
    

### 3. Waiting Room Unavailable
Problem: A waiting room normally controls how many users can enter checkout, but it goes down.
- Effect: Too many users may enter checkout simultaneously, overloading the system.
- Fallback: Stop admitting new checkout sessions while allowing browsing and existing checkout status checks.
Example: During a flash sale, users can still view products, but new checkout attempts are temporarily paused.

### 4. One Identity Becomes a Hot Key
Problem: One user or API key sends extremely high traffic, causing its Redis key to overload one Redis shard.
- Effect: Redis becomes a bottleneck.
- Fallback:
    - Local prechecks: Block obvious excessive requests before reaching Redis.
    - Leased budgets: Give each gateway a small allocation of tokens from a shared global budget to consume locally.
Trade-off: Reduced Redis traffic, but rate limiting may become less precise, depending on the implementation.

### 5. Allowed Budget Exhausted
Problem: A user consumes all available tokens.
- Effect: Requests exceed the permitted rate.
- Fallback: Return HTTP 429 Too Many Requests, ideally with a `Retry-After` header indicating when to retry.
Example:
```
HTTP/1.1 429 Too Many Requests
Retry-After: 5
```
This tells the client to retry after 5 seconds.

# Microservices and Distributed System
Application Services execute business logic, while Service Discovery helps services locate available instances.
**Key Concept:** Stateless application services allow any healthy instance to handle requests, making horizontal scaling and failover easier.

### Used in
Microservices, distributed systems, URL shorteners, payment services, message services, booking systems, and e-commerce applications.
## Characteristics

### 1. Stateless Application Services
Stateless means no irreplaceable business/session state exists only inside one application instance.
- Business data is stored in shared storage (DB, Redis, etc.).
- Any healthy instance can process incoming requests.
- If one instance crashes, another can take over.
- Local caches are acceptable if they can be rebuilt.
**Example:**
- Server A handles User 1's request.
- Server A crashes.
- Server B handles User 1's next request using shared database/session data.
**Why Stateless?**
- Easier horizontal scaling.
- Easier rolling deployments.
- Better fault tolerance.
- No dependency on one specific application instance.

### 2. Microservices vs Modular Monolith
**Modular Monolith:**
- Application contains multiple business modules in one deployment.
- Example: User, Order, and Payment modules in one application.
- Simpler deployment and communication.
**Microservices:**
- Split business capabilities into independently deployable services.
- Example: User Service, Order Service, Payment Service.
- Services communicate through HTTP/gRPC or messaging.
**When should we split services?**
- Different scaling requirements.
- Independent team ownership.
- Different deployment cycles.
- Fault isolation requirements.
**Important:** Microservices introduce network calls, latency, and distributed-system complexity. Don't split services without clear benefits.

### 3. Horizontal vs Vertical Scaling
**Horizontal Scaling (Scale Out)**
Add more application instances.
Example:
- 2 servers → 5 servers.
- Load Balancer distributes requests among them.
Gain:
- Better scalability and fault tolerance.
- Individual instances can fail without taking down the whole service.
Cost:
- More infrastructure and coordination.
- Shared dependencies like databases may become bottlenecks.

**Vertical Scaling (Scale Up)**
Increase CPU, RAM, or resources of an existing server.
Example:
- 2 CPU / 4GB RAM → 8 CPU / 16GB RAM.
Gain:
- Simple, fewer instances to manage.
Cost:
- Hardware limits.
- Larger machines can be expensive.
- Does not automatically remove single points of failure.
**Normally:** Prefer horizontal scaling for stateless, high-traffic services, while considering vertical scaling when appropriate.

### 4. Service Discovery
Service Discovery tells callers where currently available application instances are located.
Example:
Payment Service has 3 instances:
- 10.0.0.1:8080
- 10.0.0.2:8080
- 10.0.0.3:8080
Instead of hardcoding these addresses, callers use a logical name such as `payment-service`.
Service Discovery resolves that name to available endpoints.
**Common approaches:**
- DNS-based discovery (e.g., Kubernetes Services).
- Service registries (e.g., Consul).
- Platform-managed load balancing.
**How it works:**
1. New application instance starts.
2. Instance initializes dependencies and passes readiness checks.
3. Platform or registry makes the instance discoverable.
4. Load Balancer or client routes traffic to the ready instance.
5. Unhealthy instances are removed from routing after detection.

**Important:** Small systems can use managed load balancers and internal DNS without building a custom service registry.

## Case Study
### Scaling a URL Shortener Redirect Service
Assume:
- One redirect instance can safely handle 2,000 requests/sec.
- Peak traffic is 10,000 requests/sec.
Minimum instances:
10,000 / 2,000 = 5 instances.
But 5 instances provide no spare capacity at peak traffic.
We should provision additional instances for failure tolerance, deployment, and traffic spikes.
![[Pasted image 20261107103620.png]]

**How it works:**
1. Load Balancer distributes requests across healthy redirect instances.
2. Each instance is stateless and reads URL mappings from shared storage or a disposable local cache.
3. When traffic increases, horizontal autoscaling adds instances.
4. New instances warm up and pass readiness checks before receiving traffic.
5. If an instance crashes, traffic is redirected to remaining healthy instances.
6. Analytics events are processed asynchronously so they do not slow down redirects.

**Important:**
- Autoscaling application instances does not automatically scale the database.
- Limit DB connection pools and concurrent requests.
- More instances must not create unlimited DB connections.

## Trade Offs
### 1. Modular Monolith
Multiple business modules run in one deployable application.
- Gain: Simpler deployment, fewer network calls and failure modes.
- Cost: Cannot independently deploy or scale individual modules easily.

### 2. Microservices (Separate Services)
Each service can be deployed and scaled independently.
- Gain: Independent scaling, deployment, and fault isolation.
- Cost: Network latency, service discovery, observability, distributed transactions, and versioning complexity.

### 3. Local Process State
Store temporary data directly in application memory.
Example: In-memory cache.
- Gain: Very fast access without network calls.
- Cost: Data disappears on restart and may differ between instances.
**Normally:** Use local memory for disposable caches, not the only copy of important business data.

### 4. Horizontal Scaling
Add more application instances.
- Gain: More throughput and better fault tolerance.
- Cost: More infrastructure and increased load on shared dependencies.

### 5. Vertical Scaling
Upgrade resources of an existing instance.
- Gain: Simple scaling without introducing more instances.
- Cost: Hardware limits, potentially higher cost, and no automatic redundancy.

## Failure Fallback
### 1. One Application Process Crashes
Problem: One instance crashes while processing requests.
- Effect: In-flight requests may fail or have an unknown outcome.
- Fallback:
  - Load Balancer stops routing new traffic to the failed instance after detecting failure.
  - Restart or replace the instance.
  - Retry only when the operation is safe or idempotent.
**Example:** A payment request times out after the server processes it.
Retrying without an idempotency key could create duplicate payments.
**Solution:** Use stable operation IDs or idempotency keys to prevent duplicate business operations.

### 2. Service Discovery Unavailable
Problem: Services cannot retrieve updated endpoint information.
- Effect: New service lookups may fail.
- Fallback:
  - Temporarily use last-known healthy endpoints.
  - Continue health checking cached endpoints.
  - Apply bounded cache expiry.
  - Avoid routing to expired or untrusted service identities.
**Example:** Service registry goes down, but Order Service still knows previously discovered Payment Service endpoints.
It may continue using validated endpoints temporarily.

### 3. Autoscaling Cannot Respond Fast Enough
Problem: Traffic suddenly increases 10x, but new instances take time to start.
- Effect: Latency increases and requests may fail.
- Fallback:
  - Maintain minimum spare capacity.
  - Pre-provision capacity for predictable events.
  - Use admission control or rate limiting.
  - Queue or reject excess work safely.
**Important:** Autoscaling is not instantaneous.

### 4. Too Many Database Connections
Problem: Horizontal scaling adds too many application instances, exhausting database connections.
Example:
- 100 application instances.
- Each instance opens 50 DB connections.
- Total potential connections = 5,000.
But the database may support far fewer.
- Effect: DB becomes overloaded, causing timeouts and failures.
- Fallback:
  - Limit connection pool sizes.
  - Use a connection proxy/pooler where appropriate.
  - Limit in-flight queries.
  - Queue briefly or reject excess requests.
**Important:** Application capacity is limited by downstream dependencies, not just CPU.

# Cache
## Used in
hot short links, feed timelines, post objects, folder listings, routing/presence, and rate-limit counters. The last two have different loss tolerances from ordinary content caches.
## Charactertistics
- L1 cache is local to an application
- L2 cache is like Redis, shared across processes
- Reduce repeated db work n network latency.
- application defines key, TTLs, invalidation, and source of truth reads.
- Cache Aside work? Read cache first, on a miss, read db and popolate cache. On write, commit authoritative changes and update caches with a defined race handling strategy.
## Case Study
### A Viral Post becomes hot key.
1. Feed caches store post IDs rather than copying the entire body into every follower’s timeline.
2. Feed service fetches post bodies from an object cache, ideally in batches.
3. One viral post overloads its single Redis shard despite a high overall hit ratio.
4. Short-lived L1 copies distribute those reads across application instances.
5. Deleted/private posts still pass current visibility checks before presentation.
![[Pasted image 20261107103917.png]]

## Trade Offs
### Cache Aside
Pro : Simple, selective population
Con : Miss latency

### Write Through
Pro : Cache updated on write path
Con : Extra write dependency, still needs coorect failure semantics

### Write behind 
Write to cache first, then async update db
Pro : Fast write
Con : Data loss risk, unless pending writes are durably protected(event or kafka)

### L1+L2
Pro : Protecs shared cache from hot reads
Con : More copies and difficult invalidation

### Cache Stampede
Problem: Popular cache expires → thousands of requests hit the DB simultaneously.
Solutions:
- Singleflight: Only one request queries DB; others wait for the result.
- TTL Jitter: Randomize expiration times to prevent simultaneous expiry.
- Cache Warming: Preload popular data before traffic spikes.
- Bounded DB Fallback: Limit concurrent DB queries to prevent overload

### Cache Penetration
Problem: Requests for nonexistent IDs repeatedly miss Redis and hit the DB.
Solution:
- Negative Caching: Cache `NOT_FOUND` with a short TTL (e.g., 30 seconds).
- Invalidation: Remove `NOT_FOUND` when the ID is created to prevent stale 404 responses.

### Cache Avalanche
Problem: Many cache keys expire simultaneously, or Redis goes down, overwhelming the DB.
Solutions:
- TTL Jitter to spread expiration.
- Redis high availability.
- Rate limiting to protect the DB.

## Failure Fallback
### 1. Redis Down
Problem: Redis becomes unavailable, causing requests to hit the database directly.
- Effect: DB may become overloaded.
- Fallback: Limit concurrent DB queries, return smaller responses, serve eligible stale L1 cache data, or reject excess requests.
    
### 2. One Hot Key
Problem: Thousands of requests access the same popular key, overloading one Redis node.
- Effect: One Redis node becomes a bottleneck.
- Fallback:
    - L1 Cache: Store popular data in application memory.
    - Hot-Key Replication: Replicate popular data across multiple cache nodes
Important: Adding more Redis shards doesn't necessarily help because the hot key may still reside on one shard.

### 3. Stale Content After Delete
Problem: Data is deleted from the DB but remains in Redis.
- Effect: Users can still access deleted content.
- Fallback:
    - Cache Invalidation: Remove cached data after deletion.
    - Tombstone: Mark the record as deleted and check its visibility before returning it.
Example: A deleted short URL should no longer redirect users.

### 4. Cache Refill Races with Update
Problem: An old database read finishes after a newer update and repopulates Redis with stale data.
Example:
```
1. Request A reads old value from DB.
2. Request B updates DB and invalidates Redis.
3. Request A writes old value into Redis. ❌
```
- Effect: Redis contains outdated data.
- Fallback: Use version-aware cache writes to prevent older values from overwriting newer ones, with carefully designed invalidation.

### 5. Cache Contains the Only Business Record
Problem: Important business data exists only in Redis.
- Effect: If Redis loses data, the business record is permanently lost.
- Fallback: Store durable business data in a persistent database and use Redis as a disposable cache.
Example: Don't store the only copy of payment records or orders in Redis.

### 6. Redis Replication (Asynchronous by Default)
Problem: Redis Primary may crash before its latest writes are replicated.
- Effect: Some acknowledged writes may be lost during failover.
- How it works:
    - Primary receives a write and returns success.
    - Primary asynchronously replicates data to replicas.
    - If Primary crashes before replication completes, the new Primary may not contain that write.
- Fallback: Use durable database storage for critical business records; Redis should not be the sole payment authority.
Example: A payment balance updated only in Redis could become incorrect after failover.

### What about Redis WAIT?
`WAIT 1 1000` waits up to 1 second for one replica to acknowledge preceding writes on the same connection.
- Gain: Reduces the risk of losing recent writes.
- Cost: Adds latency and can time out.
- Limitation: Does not guarantee zero data loss or strong consistency, even if the replica acknowledges the write.

### Redis Lua Script (Atomic Operations)
Problem: Multiple requests updating the same Redis key concurrently can cause race conditions.

Example:
- Token Bucket has 1 token remaining.
- Request A reads 1 token.
- Request B reads 1 token.
- Both requests consume the same token. ❌

Solution: Execute read → check → deduct atomically using a Redis Lua script.
- When to use: Rate limiting, distributed counters, conditional updates, and inventory counters.
- Pros:
  - Atomic execution prevents race conditions.
  - Multiple Redis operations in one request.
  - Consistent update logic across application instances.
- Cons:
  - Long scripts block other Redis operations.
  - Harder to debug.
  - Atomicity does not guarantee durability.
  - Redis Cluster requires script keys to share a hash slot.
- Fallback:
  - Redis down: Use conservative local limits or reject sensitive operations.
  - Script error: Reject or retry safely based on operation semantics.
  - Redis failover: Reconcile critical data with durable storage.

# Relational DB & Transaction
MySQL, PostgreSQL
## Characteristic
- transaction provide All-Or-Nothing, used to protect concurrency.
- Lock, check update.
- How do keep transaction safe?
	- Keep it short, use suitable index, check affected row counts, handle deadlock or serialization retries.

## Case Study
### 2 Debit purchases race
Assume an issuer account has RM150 available. Two requests each try to reserve RM100.
1. Each request has its own stable network/operation identity.
2. In one short transaction, establish idempotency, conditionally reserve funds, and store the authorization
decision.
3. The balance condition is re-evaluated under database concurrency control, so both cannot consume the same
RM100.
4. The accepted request commits its hold and decision together.
5. A retransmission reads the recorded decision rather than creating another hold.
![[Pasted image 20261107133650.png]]

## PostgreSQL Transaction Isolation Levels
Isolation controls how concurrent transactions read and update data.

### 1. Read Uncommitted
- Allows dirty reads in standard SQL (reading uncommitted data).
- PostgreSQL treats it as Read Committed.
- Use: Rarely used.

### 2. Read Committed (Default)
- Each statement sees the latest committed data at its start.
- Gain: Good performance and concurrency.
- Cost: Non-repeatable reads and phantom reads.
- Use: Normal CRUD and atomic conditional updates.

### 3. Repeatable Read
- Transactions see a consistent snapshot throughout.
- Gain: Prevents non-repeatable reads and phantom reads.
- Cost: Write skew is still possible; retries may be needed.
- Use: Consistent reporting or multi-query reads.

### 4. Serializable (Strongest)
- Transactions behave as if executed one at a time.
- Gain: Prevents concurrency anomalies, including write skew.
- Cost: Higher overhead and possible transaction retries.
- Use: Complex booking, financial, or inventory rules.

## Conditional Updates vs Serializable
### 1. Single-Row Condition
Problem: Two users try purchasing the last item.
- Solution: Atomic conditional UPDATE prevents overselling.
    UPDATE inventory
    SET stock = stock - 1
    WHERE id = 1 AND stock > 0;
- Works safely under Read Committed for this condition.

### 2. Cross-Row Condition
Problem: Two transactions update different rows but jointly violate a business rule.
- Example: Two doctors independently go off duty, leaving nobody available.
- Solution: Explicit locks, database constraints, or Serializable transactions with retries.

### Serializable Transaction Retry
Problem: Two concurrent transactions may violate business rules if both commit.
**Example: Booking the last available seat**
1. Transaction A reads available seats = 1.
2. Transaction B reads available seats = 1.
3. Both attempt to book the seat.
4. PostgreSQL detects a serialization conflict and aborts one transaction.
5. The application retries the aborted transaction.
6. Retried transaction sees the updated state and rejects the booking because no seats remain.
**How it works:**
- Run transaction with SERIALIZABLE isolation.
- PostgreSQL detects serialization anomalies.
- One transaction may fail with SQLSTATE `40001`.
- Application rolls back and retries the entire transaction using fresh reads.
**Gain:** Prevents concurrency anomalies and preserves business invariants.
**Cost:** Extra latency and retries under high concurrency.
**Fallback:**
- Retry the whole transaction with bounded attempts and backoff.
- If retries fail repeatedly, return an error or ask the client to retry later.
- Avoid irreversible external side effects inside retried transactions.
**Important:** Serializable does not automatically retry transactions. The application must implement retry logic.

## Trade Offs
### 1. Optimistic Version Check (Optimistic Locking)
Assume conflicts are rare. Update only if the record version hasn't changed.
- Example: `UPDATE seats SET status='booked', version=version+1 WHERE id=1 AND version=5`
- Gain: No long-held locks; good performance with low contention.
- Cost: Concurrent updates may fail and require retries.
- Use: Profile updates, low-contention inventory changes.

### 2. Short Pessimistic Row Lock
Lock a row before making changes using `SELECT ... FOR UPDATE`.
- Example: Two users booking the same seat. One waits until the other transaction releases the lock.
- Gain: Prevents competing transactions from modifying the row simultaneously.
- Cost: Lock waiting, contention, and potential deadlocks.
- Use: High-contention booking or inventory operations.
- Important: Keep transactions short and acquire locks in a consistent order.

### 3. Serializable Transaction
PostgreSQL ensures committed transactions behave as if executed sequentially.
- Gain: Strong protection for complex multi-row business rules.
- Cost: Transactions may abort with SQLSTATE `40001`.
- Fallback: Retry the entire transaction with bounded retries and backoff.
- Use: Complex financial, booking, or inventory invariants.

### 4. Read Replica for Display
Use database replicas to handle read-only requests.
- Example: Display available concert seats using a replica.
- Gain: Reduces read traffic on the primary database.
- Cost: Replication lag means displayed information may be outdated.
- Use: Product listings, dashboards, seat availability display.
- Important: Actual booking/payment decisions must use authoritative, sufficiently current data.

**Interview Takeaway:**
- Optimistic Locking → Best when conflicts are rare.
- Pessimistic Locking → Useful when contention is high.
- Serializable → Complex concurrency rules involving multiple rows.
- Read Replica → Scale display reads, not authoritative booking decisions.

## Failure Fallback
### 1. Connection Lost During Commit
Problem: Database connection drops while committing a transaction.
- Effect: Application doesn't know whether the transaction succeeded or failed.
- Fallback: Query the database using a durable operation ID or idempotency key to check the transaction status.
- Important: Never assume rollback just because the connection was lost.

### 2. Primary Database Fails
Problem: Primary DB crashes or becomes unavailable.
- Effect: Writes stop temporarily, and some reads may fail.
- Fallback:
  - Promote a healthy replica through controlled failover.
  - Fence the old primary to prevent split-brain (two writable primaries).
  - Understand replication lag and possible data loss.
- Important: Failover doesn't automatically guarantee zero data loss.

### 3. Deadlock / Serialization Failure
Problem: Concurrent transactions conflict, causing PostgreSQL to abort one.
- Effect: Transaction fails and its changes are rolled back.
- Fallback:
  - Retry the entire transaction with bounded retries and backoff.
  - Use a stable operation ID to prevent duplicate business operations.
- Example: PostgreSQL deadlock `40P01` or serialization failure `40001`.

### 4. DB Unavailable for Financial Decision
Problem: Database is unavailable during payment or balance verification.
- Effect: System cannot confirm whether sufficient balance exists or whether a transaction is valid.
- Fallback: Reject or defer the financial decision until authoritative data is available.
- Important: Never approve payments using stale Redis cache data.

**Interview Takeaway:**
- Unknown commit → Check using operation ID.
- Primary failure → Controlled failover + fencing.
- Deadlock → Retry entire transaction.
- Financial DB unavailable → Fail closed; never trust stale cache.


# Replication, Consistency and Failover
Replication creates copies of data across multiple servers. Consistency defines what data clients can see. Failover determines which surviving server takes over when one fails.

### Used in
Distributed databases, payment systems, booking systems, distributed KV stores, and high-availability applications.

## Characteristics
### 1. Replication
Store copies of data across multiple servers or availability zones.
- Gain: High availability, fault tolerance, and read scalability.
- Cost: Replication lag, additional infrastructure, and consistency complexity.
- Important: Replication is NOT backup. Accidental deletion may also replicate to other nodes.

### 2. Synchronous vs Asynchronous Replication
**Asynchronous Replication**
- Primary acknowledges writes before replicas confirm receiving them.
- Gain: Lower write latency.
- Cost: Replication lag and possible acknowledged-write loss during failover.
- Use: Read-heavy applications where some lag is acceptable.

**Synchronous Replication**
- Primary waits for configured replica acknowledgments before confirming success.
- Gain: Stronger durability guarantees.
- Cost: Higher write latency; writes may block if required replicas are unavailable.
- Use: Critical financial or booking data where write durability matters.
- Important: Actual guarantees depend on acknowledgment level and failover configuration.

### 3. Quorum (Important)
Quorum means requiring a certain number of replicas to respond before considering a read or write successful.
Example: 3 replicas (A, B, C).
- N = 3: Total replicas.
- W = 2: Write requires 2 acknowledgments.
- R = 2: Read requires 2 replica responses.
Rule: R + W > N
**Example:**
1. Client writes balance = 200.
2. Replica A and B store v2; C still has v1.
3. W = 2, so the write succeeds.
4. Client reads from B and C (R = 2).
5. System compares versions and can identify v2 as newer.
**Why Quorum?**
- Read and write groups overlap.
- Helps detect stale replicas.
- Allows some node failures while maintaining required acknowledgments.

**Important:**
- R + W > N guarantees overlap for fixed replica sets.
- It does NOT automatically guarantee strong consistency.
- Correct version ordering, concurrency handling, and protocol rules are still required.
- Some databases use majority consensus rather than configurable R/W quorums.

### 4. Consistency Models
**Strong Consistency (Linearizability)**
- Reads after a completed write see that write or a newer value.
- Gain: Latest authoritative data.
- Cost: More coordination and potentially higher latency.
- Use: Balances, inventory reservation, payment decisions.

**Eventual Consistency**
- Replicas may temporarily return stale values but converge when updates stop and communication recovers.
- Gain: Better availability and lower coordination overhead.
- Cost: Users may temporarily see outdated data.
- Use: Feed views, analytics, noncritical display data.

## Case Study
### 3 Replicas During a Network Failure
Assume N = 3, W = 2, R = 2.
    Replica A → v2 ✅
    Replica B → v2 ✅
    Replica C → v1 (Lagging)
1. Client writes v2 to replicas.
2. A and B acknowledge the write.
3. Write succeeds because W = 2.
4. A read contacts B and C.
5. Version comparison identifies v2 as newer.
6. Read repair may update C to v2.
If only C remains reachable:
- W = 2 cannot be satisfied.
- New authoritative writes must stop under this fixed-quorum policy.

## Trade Offs
### 1. Leader-Based / Consensus Replication
One authorized leader orders writes; a commit protocol determines when writes are safe.
- Gain: Ordered writes and strong consistency when supported by the protocol.
- Cost: Leader election overhead; minority partitions cannot commit authoritative writes.
- Use: Strongly consistent databases and coordination systems.

### 2. Dynamo-Style Leaderless Replication
Multiple coordinators can accept writes under the store's configured rules.
- Gain: Higher write availability during some failures.
- Cost: Concurrent versions, stale reads, and conflict resolution.
- Use: Highly available distributed KV workloads.

### 3. Asynchronous Read Replicas
Primary sends committed changes to replicas afterward.
- Gain: Lower write latency and scalable reads.
- Cost: Replica lag and possible data loss during promotion.
- Use: Product listings, feeds, reporting.

## Conflict and Repair Techniques

### 1. Last-Write-Wins (LWW)
Keep the value considered newest based on an ordering rule.
- Gain: Simple conflict resolution.
- Cost: Concurrent updates may be silently lost; clocks can be unreliable.

### 2. Vector Clocks
Track causal relationships between versions.
- Gain: Detect concurrent updates.
- Cost: Additional metadata and conflict resolution logic.
- Example: Two users modify the same shopping cart concurrently.

### 3. Read Repair
Reading detects stale replicas and triggers synchronization.
- Gain: Gradually fixes inconsistent replicas.
- Cost: Additional read/repair work.

### 4. Hinted Handoff
Temporarily store a failed replica's intended writes elsewhere, then deliver them when it recovers.
- Gain: Improves write availability.
- Cost: Temporary divergence and storage overhead.

### 5. Sloppy Quorum
Use other reachable nodes when preferred replicas are unavailable.
- Gain: Better availability during failures.
- Cost: Fixed-set R/W intersection may no longer hold.

### 6. Merkle Trees
![[Pasted image 20261108132719.png]]
Compare hash trees to identify different data ranges.
- Gain: Efficient replica synchronization.
- Cost: Additional computation; does not decide which business value is correct.

### 7. Gossip Protocol
![[Pasted image 20261108132739.png]]
Nodes exchange membership and failure information.
- Gain: Decentralized failure detection.
- Cost: Information propagates gradually.
- Important: Gossip is not consensus.

## Failure Fallback
### 1. One Replica Fails
Problem: A follower crashes or becomes unreachable.
- Effect: Reduced redundancy.
- Fallback: Continue only if required acknowledgments remain possible; replace and resynchronize the replica.

### 2. Leader Fails
Problem: Primary/leader becomes unavailable.
- Effect: Writes temporarily pause.
- Fallback:
  - Elect or promote an eligible replacement.
  - Fence the old leader to prevent split-brain.
  - Resume writes only after safe leadership is established.

### 3. Majority Unavailable
Problem: Only 1 of 3 consensus nodes remains reachable.
- Effect: Cannot safely commit new writes.
- Fallback: Stop authoritative writes until quorum is restored. (Fail closed)
- Important: Never allow two independent leaders to create conflicting authoritative histories.

### 4. Async Replica Promoted Too Early
Problem: Replica is promoted before receiving recent acknowledged writes.
- Effect: Previously successful writes may disappear.
- Fallback: Use explicit durability policies, appropriate synchronous replication, and reconciliation.

### 5. Network Partition Between Regions
Problem: Regions cannot communicate.
- Effect: Cannot safely coordinate ownership or writes.
- Fallback: Route writes to the valid owner or reject/defer requests requiring coordination.
- Important: Do not blindly allow both regions to become independent writers.

## CAP Theorem (Important)
During a network partition, a distributed system cannot guarantee both linearizable consistency and availability for every request.

### CP (Consistency + Partition Tolerance) 
Financial system
- Prioritizes strong consistency.
- May reject/block writes when quorum is unavailable.
- Example: Majority-consensus-backed database.

### AP (Availability + Partition Tolerance)
Amazon DynamoDB
- Prioritizes successful responses during some partitions.
- May return stale data or accept conflicting writes.
- Example: Dynamo-style eventually consistent KV design.

**Important:**
- CAP is a trade-off during network partitions, not simply choosing two out of three features.
- CAP availability differs from uptime SLA.
- Different operations in one system may make different choices.

## System Design Interview Takeaway
- Replication → Creates redundant copies.
- Synchronous → Stronger durability, higher latency.
- Asynchronous → Faster writes, possible replication lag.
- Quorum → Minimum required acknowledgments.
- R + W > N → Read/write group overlap, not automatic linearizability.
- Strong Consistency → Critical authoritative decisions.
- Eventual Consistency → Stale reads acceptable.
- Consensus → Coordinates safe leadership and committed state.
- Failover → Promote valid replacement and fence old leader.
- CAP → Consistency vs availability during network partitions.

**Remember:** Replication improves availability, but durability and consistency depend on the replication protocol, acknowledgment policy, and failover process.

# Sharding
Used in: chat history, KV storage, rate-limit keys, notification inboxes, feed timelines, and eventually large booking datasets.
## Characteristics 
- A subset of the dataset assigned to a storage group
- A partition key maps the record or query to subset
- used to spread data volume, write throughput.
- More expensive than adding index or remove unnecessary queries
- Choose a partition key to match common operations like chat_id for conversation history, user_id for feed reads
- How are records distributed, hash, range, directory based routing, or consistent hashing.
## Case Study
### conversation history grows too large
1. Route messages by chat_id so a conversation’s common reads stay local.
2. Add time buckets, such as (chat_id, month) , to bound partition size.
3. A read for recent messages touches recent buckets, not every historical shard.
4. One extremely active conversation can still be a hot write partition.
5. Splitting that conversation further requires an explicit ordering/merge strategy; more shards alone do not preserve a total message order.
![[Pasted image 20261108133730.png]]

## Database Sharding / Partitioning Trade-offs
### 1. Hash Partitioning
Distributes data based on a hash function, e.g., `hash(userId) % N`.
- Gain: Usually distributes keys evenly across shards.
- Cost: Range queries are difficult; popular keys can still cause hot spots.
- Use: User profiles, distributed KV stores.

### 2. Range Partitioning
Divides data based on value ranges, e.g., dates or user IDs.
- Example: Shard A (1–1000), Shard B (1001–2000).
- Gain: Efficient range queries and ordered scans.
- Cost: Sequential inserts may overload one shard (hot partition).
- Use: Time-series data, transaction history.

### 3. Consistent Hashing + Virtual Nodes
Maps keys and servers onto a hash ring. Virtual nodes give each physical server multiple positions on the ring.
- Gain: Adding/removing servers only remaps a portion of keys; virtual nodes improve key distribution.
- Cost: Rebalancing complexity, replica placement, and hot keys.
- Use: Distributed caching, Dynamo-style KV stores.

### 4. Co-locating Related Records
Store related records on the same shard, usually using a shared partition key.
- Example: All orders belonging to the same tenant stored together.
- Gain: Faster local queries and easier single-shard transactions.
- Cost: A very popular tenant/event may overload one shard.
- Use: Multi-tenant applications, booking systems.

### 5. Redis Cluster (Important)
Redis Cluster uses **16,384 hash slots**, not a traditional consistent-hashing ring.
- How: `HASH_SLOT = CRC16(key) mod 16384`.
- Each Redis Cluster node owns a subset of slots.
- Adding/removing nodes requires migrating slots.
- Hash Tags: Keys with the same `{tag}` are mapped to the same slot.

Example:
- `user:{123}:profile`
- `user:{123}:session`
- Both keys belong to the same hash slot.
- Gain: Enables related multi-key operations within one hash slot.
- Cost: Putting too many popular keys in one slot can overload its owner node.
- Important: Multi-key Lua scripts in Redis Cluster must operate on keys in the same hash slot.


## Sharding / Partitioning Failure Fallback
### 1. One Shard Owner Fails
Problem: A shard's primary node crashes or becomes unavailable.
- Effect: Part of the keyspace cannot be accessed.
- Fallback: Promote a healthy replica using the database's failover protocol.
- Example: Shard A fails → Replica A becomes the new primary.

### 2. Routing Map Stale
Problem: Application uses outdated shard routing information.
- Effect: Requests are sent to the wrong shard.
- Fallback: Refresh the routing map or follow validated redirects, then retry with a bounded limit.
- Example: Key moves from Shard A to B, but the application still sends requests to A.

### 3. Rebalancing Loses Ownership Clarity
Problem: During shard migration, both old and new nodes may accept writes incorrectly.
- Effect: Missing, duplicated, or conflicting writes.
- Fallback:
  - Controlled Cutover: Transfer ownership at a defined point.
  - Ownership Version/Fencing: Reject writes from outdated owners.
  - Reconciliation: Verify and repair data after migration.

### 4. Hot Key / Hot Event
Problem: One popular key or event receives too many requests.
- Effect: One shard becomes overloaded while others remain underutilized.
- Fallback:
  - L1 Cache: Serve popular reads from application memory.
  - Admission Control: Limit incoming writes.
  - Workload Isolation: Separate high-traffic workloads.
  - Key Redesign: Distribute traffic across multiple keys where correctness permits.
- Important: Adding more shards may not solve a single hot key.

### 5. Cross-Shard Operation Required
Problem: One business transaction needs to update data across multiple shards.
- Effect: A local database transaction cannot guarantee atomic updates across independent shards.
- Fallback:
  - Co-location: Store related data on the same shard when possible.
  - Distributed Transaction (2-Phase-Commit): Coordinate atomic commits across supported shards.
  - Saga Pattern: Execute local transactions with compensating actions on failure.
- Important: A Saga provides eventual business consistency, not atomicity.

# Message Queue
like kafka or RabbitMQ
Used in: notification pipelines, chat group delivery, news-feed fan-out, search indexing, and URL click analytics.
## Characteristics
- system that accepts events or job and let consumers process them indenpendently and asynchronously. 
- A work queue distribute tasks
- Help to decouple request latency from slow fan out, absorb temporary spikes, let independent consumers recover.
- Producers emit durable business events; Consumer groups perform tasks.
- Partition by required ordering key, persist event logs and commit progress for replay and metrics logs. 
- Reply or retry may repeat work, so consumers still need idempotency.
## Case Study
### Group chat fan out
1. Persist one accepted group message and its delivery intent.
2. Publish a delivery event keyed by chat_id .
3. A worker determines the eligible recipient/device set and creates delivery work.
4. It crashes after inserting some inbox rows but before committing its broker position.
5. Replay repeats the event; uniqueness on message-recipient/device work prevents duplicate logical insertion.
![[Pasted image 20261108142243.png]]

## Trade offs
### 1. Durable Log (Kafka)
Stores events in a replicated, persistent log. Consumers can read and replay events independently.
- Gain: Event replay, high throughput, and independent consumer groups.
- Cost: Partition management, retention limits, and operational complexity.
- Use: Event-driven systems, analytics, order events, and data pipelines.
- Important:
  - Ordering is guaranteed within a partition, not across all partitions.
  - Kafka transactions can coordinate Kafka offsets and output records.
  - External DB writes and payments still require idempotency.
  - Configure replication, `acks=all`, and `min.insync.replicas` for durability.

### 2. Managed Work Queue (AWS SQS / RabbitMQ)
Producers send tasks to a queue; workers consume and process them.
- Gain: Simple asynchronous task processing, retries, and worker scaling.
- Cost: Ordering, deduplication, and replay depend on queue type and configuration.
- Use: Email sending, image processing, background jobs.
- Important: Workers should be idempotent because messages may be delivered more than once.

### 3. Redis Pub/Sub
Publish messages to currently connected subscribers.
- Gain: Fast, lightweight real-time notifications.
- Cost: At-most-once delivery; disconnected subscribers permanently miss messages.
- Use: Live notifications, cache invalidation hints, disposable events.
- Important: Not suitable as the only durable chat/message storage.

### 4. Direct Synchronous Call (HTTP / gRPC)
One service directly calls another and waits for its response.
- Gain: Immediate result, simple request-response flow.
- Cost: Caller depends on downstream latency and availability.
- Use: Authentication checks, immediate payment responses, real-time queries.
- Fallback: Timeouts, circuit breakers, and safe retries.

## Failure Fallback
### 1. Broker Temporarily Unavailable
Problem: Kafka or message broker goes down, preventing event publishing.
- Effect: New events cannot be delivered to consumers.
- Fallback:
  - Transactional Outbox: Store events in the database alongside business transactions.
  - Retry publishing when the broker recovers.
  - Reject or pause new operations if durable backlog capacity is exhausted. (Failed Close)
- Important: Never lose a committed business event just because Kafka is down.

### 2. Consumer Crashes
Problem: Consumer crashes while processing a message.
- Effect: Work may be incomplete, or the outcome may be unknown.
- Fallback:
  - Replay messages from the last safely committed offset.
  - Use idempotency keys to prevent duplicate processing.
- Example: Payment consumer crashes after charging but before committing its offset. Replaying must not charge twice.

### 3. Backlog Exceeds Retention
Problem: Consumers process messages too slowly, and Kafka deletes old messages after the retention period.
- Effect: Unconsumed events may be permanently lost.
- Fallback:
  - Monitor consumer lag and alert before retention expires.
  - Increase consumer capacity or retention period.
  - Rebuild missing events from a durable source if available.

### 4. Poison Message Blocks a Partition
Problem: One invalid message repeatedly fails processing.
- Effect: Later messages in the same partition cannot progress if strict ordering is enforced.
- Fallback:
  - Retry with a bounded limit and backoff.
  - Send failed messages to a Dead Letter Queue (DLQ).
  - Define how to handle ordering before skipping the failed message.
- Important: Skipping a message may break business ordering requirements.

### 5. Hot Partition
Problem: One Kafka partition receives significantly more traffic than others.
- Effect: Processing bottleneck even when other consumers are idle.
- Fallback:
  - Distribute messages using better partition keys.
  - Increase partitions if ordering requirements permit.
  - Scale consumers after improving partition distribution.
- Important: Adding consumers alone won't speed up one partition within the same consumer group.

# Transactional Outbox & Durable Inbox
Outbox means “I committed this change and must publish its consequence.” Inbox means “I durably accepted this incoming event and must process it safely.”
	Basically, just means we save a event row in db, then scheduled job or observer will read the unprocessed event then publish to broker. Consumer then process it and run the related logic asynchronously.
## Used in: 
payments, file changes, posts, chat delivery, search indexing, and notifications.
## Characteristics
### 1. What — What Problem Does It Solve?
Problem: Dual Write — updating the database and publishing an event are two separate operations.
- Effect: DB update succeeds, but event publishing fails, causing inconsistent systems.
- Solution: Save business data and outbox event in the same DB transaction.

### 2. Why — Why Not Publish Directly?
Problem: Application may crash between DB commit and event publishing.
- Publish Before Commit: Event published, but DB transaction rolls back.
- Publish After Commit: DB commits, but application crashes before publishing.
- Solution: Transactional Outbox guarantees business data and event intent are committed together.

### 3. Who — Who Handles Each Step?
- Business Service: Saves business data + outbox event atomically.
- Outbox Relay: Reads committed outbox events and publishes to Kafka/SQS.
- Consumer: Processes events and deduplicates using event IDs.
- Inbox Pattern: Consumer stores verified incoming event IDs to prevent duplicate business effects.

### 4. Where — Where Is Atomicity Guaranteed?
- Producer DB: Business row + outbox row commit together.
- Consumer DB: Processed event ID + business changes commit together.
- Important: Outbox does NOT guarantee atomicity across different databases or external payment providers.

### 5. When — When Should We Use CDC?
CDC (Change Data Capture) reads committed database changes, often from the transaction log.
- Gain: Publishes changes without repeatedly polling the outbox table.
- Use: High-volume event publishing, data synchronization, and maintaining read projections.
- Important: Raw database changes are not automatically meaningful business events.

### 6. How — How Does Recovery Work?
1. Business service commits business data + outbox event.
2. Relay reads unpublished outbox events.
3. Relay publishes event to the message broker.
4. Broker acknowledges successful publication.
5. Relay marks the event as published.
6. If relay crashes before step 5, it republishes the same event ID.

## Case Study
### a post saves but fan-out publication fails
1. One transaction inserts post:P9 and event:E9, type=PostCreated .
2. The API acknowledges after the required durable commit.
3. Kafka is unavailable; E9 remains pending.
4. The relay publishes E9 after recovery.
5. A fan-out consumer atomically inserts its deduplication record and local feed work.
6. If E9 arrives twice, the second attempt does not create a second logical insertion.
![[Pasted image 20261109014755.png]]

### Durable webhook inbox
Stores incoming webhook events in a database before processing them asynchronously, preventing lost or duplicate payment processing.

**How it works:**
1. Payment provider sends a webhook (e.g., Payment Success).
2. Verify signature using the original request body and check timestamp/replay rules.
3. Save event in an Inbox table with a unique provider/account/event ID.
4. Return HTTP 2xx after successful durable storage.
5. Background worker reads unprocessed events and updates payment status.
6. Duplicate events are ignored using the unique event ID.

**Example:**
    Stripe → Webhook API
                  |
            Verify Signature
                  |
            PostgreSQL Inbox
                  |
             Return HTTP 200
                  |
            Background Worker
                  |
            Update Payment Status
            
## Trade offs
### 1. Poll Outbox Table (Scheduled Job)
Background worker periodically reads unpublished events from the database and publishes them to Kafka.
- Gain: Simple implementation, easy to understand and debug.
- Cost: DB polling overhead, publication delay, and outbox cleanup.
- Use: Small/medium applications with moderate event volume.
- Example: Scheduled job checks pending events every 5 seconds.

### 2. CDC Relay (Change Data Capture)
Tools like Debezium read committed changes from the database transaction log and publish events to Kafka.
- Gain: Near-real-time event publishing without repeatedly polling the outbox table.
- Cost: Connector management, transaction-log retention, schema changes, and operational complexity.
- Use: High-volume event-driven systems.
- Important: CDC can still produce duplicate events; consumers need idempotency.

### 3. Durable Inbox
Stores incoming events in the database before processing them asynchronously.
- Gain: Survives receiver crashes, supports retries and deduplication.
- Cost: Extra storage, deduplication logic, and processing backlog.
- Use: Payment webhooks, external callbacks, reliable event consumers.
- Example: Stripe webhook → Inbox DB → Background Worker.

### 4. Direct Publish Only
Application commits business data, then publishes an event directly to Kafka.
- Gain: Simple, fewer components, lower operational overhead.
- Cost: Application crash between DB commit and publishing can permanently lose the event.
- Use: Noncritical events where occasional event loss is acceptable.
- Important: Not recommended for critical payment or order events.

## Key Difference Between Outbox & Inbox
==Bascially Outbox event is saved in Producer DB but Inbox event is saved in Consumer DB==

|                 | Outbox               | Inbox                              |
| --------------- | -------------------- | ---------------------------------- |
| Direction       | Outgoing             | Incoming                           |
| Purpose         | Reliable publishing  | Reliable receiving/processing      |
| Event source    | Our application      | External system/broker             |
| Worker action   | Publish event        | Process event                      |
| Main protection | Lost outgoing events | Lost/duplicate incoming processing |

## Failure Fallback
### 1. Relay Down
Problem: Outbox worker crashes or stops publishing events.
- Effect: Business DB commits succeed, but events are not delivered to Kafka.
- Fallback: Restart worker and process pending events.
- Important: Monitor backlog size and oldest pending event age.

### 2. Publish Succeeds, Mark Fails
Problem: Event successfully publishes to Kafka, but worker crashes before marking it as published.
- Effect: Same event may be published again.
- Fallback: Use stable event IDs and idempotent consumers to prevent duplicate business effects.
- Example: Event published twice, but payment is processed only once.

### 3. Dedup Marker Saved Without Business Effect
Problem: Consumer marks an event as processed before completing the business operation.
- Effect: On retry, the event may be skipped forever even though the operation never completed.
- Fallback: Save the processed event ID and business changes in the SAME DB transaction.
- Example: Marking Payment Success as processed without updating the order status.

### 4. CDC Log Position Lost / Expired
Problem: CDC worker loses its checkpoint or required database log entries expire.
- Effect: Some events may be missing from Kafka or downstream projections.
- Fallback: Rebuild using a controlled database snapshot + replay available changes.
- Important: Reconcile downstream data against the source of truth.

### 5. Backlog Storage Fills
Problem: Outbox events accumulate because Kafka or the relay remains unavailable.
- Effect: Database storage may run out, preventing new transactions.
- Fallback: Throttle or reject new operations before storage is exhausted.
- Important: Never silently discard committed business events to free space.

# Idempotency & State Machines
Idempotency answers “is this the same operation?” A state machine answers “is this
transition allowed now?”
## Used in: 
payment creation/capture/refund/payout, seat checkout, file commits, chat sends, and notification delivery.
## Characteristic
- Idempotency key is used to identify is the operation same due to network retry or replay or any duplicate process to avoid duplicate process. -> duplicate payment is crictical
- upstream normally creates a atomically unique key or frontend.

## Case Study
###  payment succeeded but the response was lost
1. Client submits operation pay:order-42 .
2. The payment service saves the payment operation and a stable PSP-operation key.
3. PSP performs the effect, but the response times out.
4. Mark the outcome UNKNOWN / pending verification—not definitively failed.
5. Query/retry according to the provider’s original-operation contract.
6. Return the same verified operation result when the client retries.
![[Pasted image 20261109020100.png]]

## Tradeoffs 
### 1. Durable Uniqueness + State (Recommended)
Store idempotency keys and processing status in a durable database with a UNIQUE constraint.
- Gain: Prevents duplicate business operations even after crashes or restarts.
- Cost: Extra DB storage and concurrency handling.
- Use: Payments, orders, bookings.
- Example: Same payment request ID received twice → Return the existing result instead of charging again.

### 2. Cache-Only Deduplication (Redis)
Store processed request IDs in Redis with TTL.
- Gain: Fast, low latency.
- Cost: Cache eviction, failover, or TTL expiry may allow duplicate processing.
- Use: Noncritical operations where occasional duplicates are acceptable.
- Important: Don't rely on cache-only deduplication for financial transactions.

### 3. Long Dedup Retention
Keep idempotency records for an extended period.
- Gain: Prevents duplicates during long retry or event replay windows.
- Cost: More storage, cleanup, and privacy management.
- Use: Payment webhooks, financial events, delayed retries.
- Important: Retention should cover the expected retry and replay period.

### 4. One Generic Payment Status
Represent all payment operations using one status (e.g., PENDING, SUCCESS, FAILED).
- Gain: Simple implementation and UI display.
- Cost: Cannot accurately represent separate authorization, capture, refund, and payout operations.
- Fallback: Track each payment operation separately with its own ID and status.
- Example: Payment captured successfully, but refund is still pending. A single SUCCESS status cannot represent both correctly.

## Idempotency & Deduplication Failure Fallback
### 1. Two Identical Requests Race
Problem: Two requests with the same idempotency key arrive simultaneously.
- Effect: Both might execute the same operation, causing duplicate payments or orders.
- Fallback:
  - Use a UNIQUE constraint on the scoped idempotency key.
  - Atomically claim the operation before processing.
  - Second request waits or returns the first request's result.
- Example: Two payment requests arrive together → Only one payment is executed.

### 2. Same Key, Different Payload
Problem: Client reuses an idempotency key with different request data.
- Effect: Same key represents two conflicting operations.
- Fallback: Store and compare a request payload hash; reject mismatches.
- Example: Key `pay123` originally requests RM100, but retry requests RM200 → Reject.

### 3. Provider Outcome Unknown
Problem: Payment provider times out after receiving a charge request.
- Effect: Payment may have succeeded, risking double charges.
- Fallback:
  - Preserve the original idempotency key.
  - Query provider using the original operation ID.
  - Retry only when safe under provider guarantees.
- Important: Never generate a fresh key just because a payment timed out.

### 4. Dedup Store Unavailable
Problem: Database storing idempotency records becomes unavailable.
- Effect: Cannot verify whether the operation was already processed.
- Fallback: Reject or defer sensitive operations until the database recovers.
- Important: Never bypass idempotency checks for critical payments.

### 5. Provider Has No Idempotency Support
Problem: External provider cannot deduplicate requests or reliably query previous outcomes.
- Effect: Retrying an unknown operation may execute it twice.
- Fallback:
  - Avoid automatic retries when the outcome is uncertain.
  - Reconcile using provider records or manual investigation.
  - Explicitly accept a weaker delivery guarantee.
- Important: Local idempotency cannot fully prevent duplicate external effects without provider cooperation.

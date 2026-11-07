The key points are seperated by component to explain, i think just like thinking as software engineer, we only understand how each component working behind, then only can design a good system whatever situations.



## Table of contents

- [[#DNS and CDN edge]]
- [[#Load Balancer]]
- [[#API Gateway]]
- [[#Rate Limiter]]
- [[#Microservices and Distributed System]]
- [[#Cache]]
- [[#Relational DB & Transaction]]

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
Compare hash trees to identify different data ranges.
- Gain: Efficient replica synchronization.
- Cost: Additional computation; does not decide which business value is correct.

### 7. Gossip Protocol
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
- Fallback: Stop authoritative writes until quorum is restored.
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
- Prioritizes strong consistency.
- May reject/block writes when quorum is unavailable.
- Example: Majority-consensus-backed database.

### AP (Availability + Partition Tolerance)
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

The key points are seperated by component to explain, i think just like thinking as software engineer, we only understand how each component working behind, then only can design a good system whatever situations.

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

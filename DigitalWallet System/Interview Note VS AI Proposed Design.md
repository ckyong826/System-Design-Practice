# Digital Wallet System Design — Interview Review
Prepared for your interview tomorrow. Based on your mock interview, including your revisions and requests for help.

Your answers below are faithful paraphrases, not word-for-word quotations. Repetitions are consolidated; substantive proposals and changes of direction are preserved. Answers supplied by the interviewer are identified separately. There is no single perfect architecture: the right answer is a coherent design whose guarantees you can defend.

## Read this first: tomorrow’s priorities

1. **Start simple:** stateless wallet service instances, a load balancer, and one PostgreSQL primary. Add replicas for history reads when justified. Microservices and sharding are optional developments, not starting requirements.
2. **Explain one transfer completely:** identity validation → idempotency → ordered account locks → funds check → balance updates and ledger records → commit → response.
3. **Protect the invariants:** no overdraft, no duplicate application, equal debit and credit, no partial transfer, no loss of acknowledged transfers within the agreed failure model.
4. **Match reads to requirements:** fresh balances from the primary; history may tolerate a few seconds of lag.
5. **Name a failure and explain recovery:** especially commit succeeds but response is lost.
6. **Measure before adding complexity:** traffic estimates do not prove that a database needs sharding.

Your strongest independent answers were row locking, idempotency with database uniqueness, payload validation, and deriving the sender identity in the backend. Spend tonight connecting those ideas into one precise flow.

## 1. Requirements and estimation: your answers beside stronger answers

| Topic | Your proposed answer | Stronger interview answer / how to improve |
|---|---|---|
| Scope | Check balance, transfer money, view history. | “I’ll focus on internal wallet-to-wallet transfers, balance checks, and paginated history. Are deposits, withdrawals, fees, FX, and external payment providers excluded?” This makes the atomicity boundary clear. |
| Consistency | Asked whether to use strong or eventual consistency. | Good clarification. “Transfers must be atomic and prevent overdrafts. Balance reads must reflect previously completed transfers. History can lag a few seconds.” Specify observable behavior rather than only a consistency label. |
| Availability | Suggested 99.9%; accepted 99.95%. | “We target 99.95% monthly availability, but reject or pause transfers when correctness cannot be established. Is that trade-off acceptable?” Distinguish a target from a guarantee. |
| DAU | Asked for DAU; accepted 10 million registered and 2 million daily active users. | Good question. Also clarify peak concentration, geographic scope, and whether transfers are internal. Avoid collecting numbers without explaining which design decision they influence. |
| Write traffic | Five transfers per active user daily; 10 million transfers/day. | Correct assumption-based calculation: 2 million × 5 = 10 million/day; divide by 86,400 ≈ 116 TPS average. Say explicitly that “transaction” means a business transfer, not one SQL statement. |
| Peak writes | Assumed ten times average: about 1,160 TPS. | Reasonable working assumption. “I’ll plan for roughly 1,200 peak transfers/sec and validate the peak multiplier against real traffic.” It is an assumption, not measured capacity. |
| Read traffic | Suggested 10,000 peak RPS; later assumed 50 reads per active user/day. | 2 million × 50 = 100 million reads/day ≈ 1,157 average RPS, or 11,574 at 10× peak. Round to 12,000. Explain whether the 50 combines balance and history reads. Fifty may be high for manual actions; automated refresh behavior could support it. |
| Latency | Initially interpreted the targets as P95 under 200 ms and P99 under 500 ms. | Both agreed targets were P95: balance <200 ms, transfer <500 ms. Restate each endpoint’s percentile and whether it measures completed transfer or merely request acceptance. |
| Monthly storage | Said 10 million × 30 = 3 billion; assumed 500 bytes per transfer. | Arithmetic correction: 300 million records × 500 bytes = 150 GB/month in decimal units. Check powers of ten aloud. |
| Annual storage | Estimated 1.5 TB after the monthly correction. | Approximately 1.8 TB using twelve 30-day months, or 1.825 TB using 365 days. This is only the assumed raw representation. Two ledger entries, indexes, WAL, replication, backups, and retention add storage. |
| Whether estimates are needed | Asked whether storage estimation was necessary. | “I’ll estimate storage to judge retention and partitioning needs, and throughput to plan a load test.” Estimates should support a decision, not consume the entire interview. |

The DAU, consistency behavior, availability target, latency targets, and several calculations were supplied or clarified by the interviewer. Distinguish requirements you gathered from decisions you independently justified.

## 2. Architecture, concurrency, and idempotency

| Topic | Your proposed answer | Stronger interview answer / how to improve |
|---|---|---|
| Initial architecture | Client → load balancer → wallet system → database; mentioned an API gateway. | A useful baseline. Explain the transfer flow before adding components. An API gateway may centralize authentication/rate limits, but state its purpose if included. |
| Monorepo / monolith | Said “monorepo” for the initial service. | A monorepo is a source-code organization choice; a monolith is a deployment architecture. Say “one modular wallet service deployed as multiple stateless instances.” |
| Microservices | Proposed read and transfer services to handle high traffic. | “A single service can scale horizontally. I’ll split deployments only if independent scaling, ownership, or release isolation justifies the operational cost.” Traffic alone does not require microservices. |
| Independent scaling | Proposed larger write-service machines and more service instances behind a load balancer. | Valid capabilities, but benchmark first. More instances help application capacity and instance failures; they do not remove database lock contention or database I/O limits. |
| Extra complexity | Mentioned coordination, traffic distribution, and distributed logs. | Good starting trade-offs. Add network failures, tracing, deployment/version compatibility, and potentially distributed transactions if data ownership is split. |
| Concurrency | Pessimistic lock, check available funds, update inside a DB transaction, roll back on error. | Strong foundation. “Lock the relevant account rows, check funds under those locks, then update both balances and insert the transfer and ledger entries in one transaction.” Say row locks, not whole-table locks. |
| Transaction boundary | Said both sender and recipient belong in the transaction. | Include both balance updates, all ledger entries, transfer result, and idempotency result. A debit without a credit must never be a committed success. |
| Deadlocks | Proposed lock timeout, rollback, and backoff. | Useful recovery, but prevent the specific circular wait by locking account rows in ascending account-ID order for every transfer. Still handle database deadlock errors and bounded retries. Lock ordering was supplied after you requested help. |
| Idempotency | Check the key; return an existing result instead of executing again. | Good. Scope the key to the authenticated caller and operation. Persist a request fingerprint and stable outcome. A lookup is a fast path; uniqueness is the concurrency safeguard. |
| Simultaneous duplicate keys | Use a unique database constraint so only one insert succeeds. | Correct. Claim the key before money updates. Handle the conflicting request by fetching the committed result or giving a defined processing/retry response. Database conflict handling must respect transaction-abort behavior; do not assume every uniqueness error leaves the transaction usable. |
| Idempotency atomicity | Agreed the key and money changes should commit together. | Explain why: otherwise money could commit without a retry record, or a retry record could commit without money movement. Both create incorrect retry behavior. |
| Crash handling | Initially said a crash implies rollback; later explained returning the committed result on retry. | “Before commit, an abandoned transaction rolls back. After commit, it remains committed. If the outcome is unknown, retry the same key and retrieve the durable outcome.” A service crash does not imply rollback if commit already happened. |
| Changed payload with same key | Reject the request and require a new key. | Correct. Compare normalized business fields, including recipient, amount, and currency. A new key means a genuinely new intended operation; do not silently generate it for an uncertain previous outcome. |
| Duplicate prevention | Later said atomicity prevents a transfer applying twice. | Correction: atomicity prevents partial application within one transaction. Idempotency prevents retries from creating multiple complete transactions for one intent. |

## 3. Database and history design

| Topic | Your proposed answer | Stronger interview answer / how to improve |
|---|---|---|
| Balance schema | Balance/account ID, user ID, available balance, current balance; one account per user. | Start with wallet ID, owner ID, currency, and one clearly defined posted balance. Add available/held amounts only if the product includes reservations. Define their relationship and update rules. |
| Current vs available | Current balance not yet reconciled; available balance after deductions; nightly scheduler. | This was unclear. A common model is posted balance including completed entries, with available balance adjusted for holds. Reconciliation verifies correctness; it should not be required to make a completed internal transfer accurate. For this scope, omit holds unless needed. |
| User lookup | Unique user ID, foreign key to user, additional read index. | Uniqueness is reasonable for one wallet per user. PostgreSQL already creates an index for a unique constraint; an identical extra index is redundant. For multiple currencies, uniqueness may instead be owner + currency. |
| Monetary type | Decimal/numeric; later numeric in the smallest unit. | Either fixed-scale `NUMERIC` or integer minor units is defensible. Choose one explicitly. Example: `BIGINT` cents for a single two-decimal currency. `NUMERIC` and integer minor units are different choices; the earlier interviewer incorrectly treated your numeric proposal as necessarily integer storage. |
| Floating point | Asked for the explanation. | Binary floating point approximates many decimal fractions, which can introduce rounding discrepancies. Exact representation plus explicit rounding/overflow rules avoids these errors. [PostgreSQL numeric types](https://www.postgresql.org/docs/current/datatype-numeric.html). |
| Currency | Asked whether a currency column was needed. | One currency was accepted for scope. You can still store currency explicitly for clarity and prevent accidental mismatches. Multiple currencies require scale rules and FX design; do not silently add them. |
| Application-only funds checks | Asked whether the service could enforce nonnegative balances without DB checks. | Keep the funds check under a row lock or in a conditional update, and also use a DB constraint as defense against buggy/direct writers. Database enforcement protects the invariant across write paths. |
| ABS | Proposed taking the absolute value. | Incorrect: it changes an invalid negative amount into invented positive funds. Reject the invalid write instead. |
| Nonnegative constraint | Initially “positive numeric type,” then NOT NULL and balance >0; revised to ≥0. | Use both `NOT NULL` and `CHECK (balance_minor >= 0)`. Nullability and value range are separate. Transfers need `amount_minor > 0`; wallet balances must allow zero. |
| Ledger | Credit/debit users and amounts linked to a transaction ID. | Your paired debit/credit row can be made valid with explicit invariants. A clearer extensible alternative is one immutable entry per wallet per transfer: wallet ID, transfer ID, signed amount, currency, timestamp. Entries for an internal transfer sum to zero. Do not present a normal row-level CHECK as enforcing a cross-row sum. |
| Transfer record | Initially one user/account ID, amount, timestamp, transaction ID. | Include sender wallet, recipient wallet, positive amount, currency, status, timestamp, and stable transfer ID. Specify rejection for self-transfers or define their behavior. |
| Sender/recipient history | Added sender and receiver after prompting; optional action type. | Correct revision. Explain that one transfer appears as outgoing for the sender and incoming for the recipient; direction is relative to the viewer. |
| History index | Proposed sender + receiver + created-at composite index. | Use separate indexes `(sender_wallet_id, created_at DESC, transfer_id DESC)` and `(recipient_wallet_id, created_at DESC, transfer_id DESC)` for the two access paths. One sender-leading index is poorly matched to arbitrary incoming transfers. [PostgreSQL multicolumn indexes](https://www.postgresql.org/docs/17/indexes-multicolumn.html). |
| Cursor | Named cursor pagination but could not explain its contents. | Use a stable ordering by timestamp and unique transfer ID. Encode the last pair into the cursor; query strictly older pairs for descending order. Apply the boundary to both incoming and outgoing queries, then merge. |
| Page merging | Supplied by interviewer. | Fetch up to the page size from each branch, merge under the same total order, deduplicate if needed, then retain one page. Rejecting self-transfers avoids one common overlap. A cursor is a position marker, not automatically a snapshot of all future changes. |

## 4. Caching, replicas, failover, and sharding

| Topic                             | Your proposed answer                                                                                   | Stronger interview answer / how to improve                                                                                                                                                                                                                                                                                                                                        |
| --------------------------------- | ------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Balance cache                     | Redis reads; refresh after successful transfer.                                                        | Database commit and cache refresh are separate operations. A crash between them leaves a stale hit. For this interview’s strict freshness requirement, read balances from the primary. Keep cache out of the money-authorizing path.                                                                                                                                              |
| Cache failure classification      | Called the stale-cache scenario cache penetration; proposed DB fetch on a miss.                        | This is a stale hit, not a miss. Cache penetration normally describes repeated requests for nonexistent data reaching the backing store. Name the exact state before choosing a remedy.                                                                                                                                                                                           |
| Periodic refresh                  | Suggested every 15 minutes / constant scheduler refresh.                                               | This bounds some staleness but does not provide strong consistency. Faster refresh still leaves a gap. Only offer it if the product explicitly accepts that gap.                                                                                                                                                                                                                  |
| Client/CDN history cache          | Cache first response; refresh after write.                                                             | Client caching is possible with short refresh intervals or notifications plus refresh. Incoming transfers happen without that client writing. Financial history must never enter an unprotected shared cache; CDN caching would require careful private access and cache-key controls. Omit CDN caching from the initial design.                                                  |
| Always fetch history              | Reverted to reading the DB every time.                                                                 | Reasonable baseline. Use indexes and pagination; add history replicas when needed. Define how the client refreshes to discover incoming transfers.                                                                                                                                                                                                                                |
| Replicas for balances and history | Proposed both reading from replicas.                                                                   | History may tolerate lag; balances may not. A transfer can commit before an asynchronous replica applies it. Route strict balance reads to the primary unless you implement a clearly justified replica freshness protocol.                                                                                                                                                       |
| Quorum                            | Proposed N=3 and R+W>N, reading versions and choosing newest.                                          | This is not a drop-in PostgreSQL primary/standby design. Overlap reasoning needs precise membership, versions, read/write protocols, and failure assumptions; overlap alone does not establish full linearizability. Do not mix Dynamo-style mechanisms with PostgreSQL streaming replication without an implementation plan.                                                     |
| Replica acknowledgements          | Eventually agreed the update must be visible for replica reads.                                        | Receiving, durable flushing, and applying WAL are distinct. Synchronous durability need not mean immediate query visibility. PostgreSQL `remote_apply` waits for application on the required synchronous standbys; arbitrary other replicas can still lag. [PostgreSQL WAL settings](https://www.postgresql.org/docs/current/runtime-config-wal.html).                            |
| One replica fails                 | Proposed election / sloppy quorum using another reachable node.                                        | A healthy primary stays primary. Stop routing reads to the failed replica. Transfer availability depends on required synchronous acknowledgements and remaining eligible standbys. Do not silently weaken durability just to keep writes available.                                                                                                                               |
| Primary fails                     | Proposed election and choosing the latest node.                                                        | Use a managed/failover system with safe leader coordination, eligible replica selection, replay, and fencing. PostgreSQL streaming replication alone is not a complete automatic leader-election mechanism.                                                                                                                                                                       |
| Split brain                       | Suggested pausing transfers and ping checks.                                                           | Pausing is a valid correctness-first response. Ping cannot distinguish a crash from a partition. Fence the former writer before allowing a new one to accept transfers; fail closed when writer exclusivity cannot be established.                                                                                                                                                |
| No lost acknowledged transfers    | Suggested monitoring; requested answer.                                                                | Require suitable synchronous durability before success, then promote a replica known to contain all acknowledged commits. Monitoring helps observe lag; it does not create durability. State the failure model and accept reduced availability when required copies are unavailable. [PostgreSQL standby replication](https://www.postgresql.org/docs/current/warm-standby.html). |
| Sharding                          | Proposed user-ID sharding for balances, ledger, transactions; consistent hashing for easier expansion. | User sharding colocates one user’s data, not both parties in a transfer. Consistent hashing reduces remapping but does not solve migration consistency, cross-shard atomicity, or hot wallets. Defer sharding until measurements justify it.                                                                                                                                      |
| Redis distributed lock            | Proposed locking across databases.                                                                     | It can coordinate writers but cannot atomically commit two independent databases. A crash between debit and credit remains possible.                                                                                                                                                                                                                                              |
| Two-phase commit                  | Described RPC to another shard and committing after its response.                                      | Both participants must durably prepare; the coordinator durably records the decision; participants recover and follow it. Prepared participants can block while the decision is unavailable. A successful ordinary RPC is not a durable prepare.                                                                                                                                  |
| Message queue for throughput      | Proposed asynchronous buffering, accepting more latency.                                               | A queue absorbs bursts; it does not increase sustained database capacity. If arrivals exceed processing, backlog grows. Define accepted/pending versus completed, bounded backlog, idempotent consumers, and status retrieval. This changes the original synchronous completion behavior.                                                                                         |
| Single-primary capacity           | Assumed too many records and heavy I/O require sharding.                                               | Record count alone does not prove failure at 1,160 TPS. Benchmark realistic data and transactions. Inspect P95 latency, throughput, lock waits, connections, CPU, disk, WAL, and replication lag before choosing a remedy.                                                                                                                                                        |

## 5. Security, testing, overload, and final reasoning

| Topic | Your proposed answer | Stronger interview answer / how to improve |
|---|---|---|
| Sender authorization | Backend derives sender from authenticated token; client supplies recipient. | Strong answer. Derive ownership from validated identity, not client input. Apply equivalent authorization to balance and history endpoints. |
| Token authenticity | Proposed MFA / second password / biometrics. | Step-up authentication can help transfer security, but does not replace validating the existing access token. For JWTs, verify with trusted keys and permitted algorithms, then validate required issuer, audience, expiry and other applicable claims. [JWT security guidance](https://www.rfc-editor.org/rfc/rfc8725.html). |
| Logout | Redis token blacklist. | Useful revocation strategy. Check revocation after authenticity validation; expire blacklist entries when tokens expire. Not being blacklisted does not prove validity. Stolen valid tokens are a separate threat from fabricated tokens. |
| Incorrect money movement | Suggested nightly reconciliation after interviewer explanation. | Reconcile ledger against balances, match debit/credit totals, detect duplicate application, and investigate mismatches. Nightly reconciliation is a useful backstop; add earlier alerts for critical invariant violations. Define the ledger as authoritative and repair through controlled, auditable procedures. |
| Concurrent transfer test | Repeated the locking implementation; requested expected assertions. | Use real concurrent DB transactions: sender starts at 100, two transfers of 80 to distinct recipients with distinct keys. Expect one success, one insufficient-funds rejection, sender 20, exactly one recipient +80, and one complete balanced transfer in the ledger. |
| Hot wallet | Recognized serialization and slower transfers; suggested shorter synchronous code and fewer network calls. | Correct diagnosis and useful optimization. Still protect shared resources: per-wallet concurrency admission before DB connection acquisition, short lock timeouts, bounded waiting, and a global cap. |
| Timeout / fail closed | Suggested timing out hot-wallet requests. | Good partial answer. Release resources by rolling back timed-out transactions, define retryable outcomes, and prevent too many waiting requests from taking all connections in the first place. |
| Rate limiter | IP-based token bucket in Redis, Lua for atomic updates. | Token bucket and atomic updates are reasonable. IP alone penalizes shared networks and misses distributed load. Use authenticated-user limits for fairness, wallet limits for shared-row contention, and global protection. |
| Rate-limit revisions | Switched to user ID, then wallet/balance ID. | Good adaptation. Explain complementary limits rather than replacing one dimension with another. Rate limits control arrival rate; concurrency limits control in-flight work. |
| Retry response | Suggested 204 / 402 / 403, backoff; later learned 429. | 429 suits rate limiting; include Retry-After where appropriate. 503 can indicate temporary service overload. 204 means success with no body, 402 is not the rate-limit status, and 403 is authorization refusal. |
| Retry synchronization | Requested explanation. | Use capped exponential backoff with random jitter, a retry budget, and the original idempotency key. Honor server retry guidance. Fixed identical delays synchronize another burst. |
| Biggest remaining risk | Listed addressed concerns and concluded the design was okay. | Identify a specific unresolved mechanism. Example: “Cross-shard atomicity is still unresolved, so I would remove sharding from the initial design. Safe primary failover remains a dependency whose durability and fencing guarantees must be verified.” |

## 6. A coherent reference answer you can say aloud

### Opening

“I’ll scope this to internal transfers in a single currency, balance checks, and paginated history. I’ll exclude FX, fees, deposits, and withdrawals unless you want them included. My key invariants are no overdrafts, no partial transfers, no duplicate application, and balanced ledger entries.”

### Estimates and starting architecture

“With two million daily active users and five transfers each, we get about 116 average TPS and 1,160 at a ten-times peak. Fifty combined balance/history reads per user gives roughly 12,000 peak RPS. I’ll start with stateless wallet instances behind a load balancer and PostgreSQL as the transactional store. I’ll benchmark this design before sharding.”

```text
Client → load balancer / optional API gateway → stateless wallet service instances
                                               ├─ transfers → PostgreSQL primary
                                               ├─ balances  → PostgreSQL primary
                                               └─ history   → primary initially;
                                                              replicas if needed

PostgreSQL primary → eligible synchronous standby for durability/failover
                  → optional additional history replicas
```

“Redis can support rate limiting. It is not the source of truth for money. History may use replicas because a few seconds of lag is acceptable; current balance reads use the primary.”

### Minimal data model

| Table | Important fields | Key rules |
|---|---|---|
| wallets | wallet ID, owner ID, currency, balance in minor units | Primary key; owner/currency uniqueness if required; balance NOT NULL and ≥0 |
| transfers | transfer ID, sender wallet, recipient wallet, amount, currency, status, created time | Amount >0; valid wallet references; self-transfer policy |
| ledger_entries | entry ID, transfer ID, wallet ID, signed amount, currency, created time | Immutable posted entries; one debit and one credit for this simple transfer type |
| idempotency_requests | caller ID, operation, key, normalized request fingerprint, outcome, transfer ID | Unique caller + operation + key; outcome commits atomically with money changes |

Use one consistent sign convention: for example, sender entry −80 and recipient entry +80. Cross-row balancing is enforced by the transactional posting operation and verified through reconciliation. A row-level CHECK alone cannot validate the sum of arbitrary rows.

An auditable rejection can be stored separately from posted ledger entries. Rejection must not create a posted debit or credit. Keep permissions narrow so application or maintenance writers cannot bypass the posting rules casually.

### Complete transfer flow

1. Validate authentication, derive the sender in the backend, and validate recipient, positive amount, currency, and limits.
2. Begin a database transaction and claim the caller-scoped idempotency key using database uniqueness.
3. If an equivalent committed request already exists, return its outcome. If its payload differs, reject the key reuse. Define handling for a still-running concurrent request.
4. Lock both wallet rows in ascending wallet-ID order. Keep the transaction short and avoid external network calls while holding locks.
5. Check sufficient funds under the lock. For insufficient funds, record the defined rejected outcome without posting money movement.
6. For success, debit the sender, credit the recipient, create the transfer, and insert matching ledger entries.
7. Persist the idempotency outcome in that transaction and commit with the required durability policy.
8. Return success only after commit. If the response is lost, a retry with the same key returns the saved outcome.

“I would test this with concurrent overspending, simultaneous duplicate requests, opposite-direction transfers, and crashes around commit. Row locks prevent concurrent overspending; uniqueness and idempotency prevent duplicate business operations.”

### Failure and scaling position

“If the primary fails, only a suitably caught-up eligible standby may be promoted, and the old writer must be fenced. If we cannot establish safe writer ownership or required durability, transfers pause. History may remain available under its weaker consistency contract.”

“I’ll load-test realistic data and skewed traffic. For a hot merchant, limit admitted concurrency before database connections are acquired. Use user, wallet, and global limits, short timeouts, bounded queues, and retry guidance with jitter. A queue can absorb bursts, but cannot fix a sustained processing deficit.”

## 7. API and query examples to understand

These were not fully developed in the mock interview. They are suggested additions, not answers you gave.

```http
POST /transfers
Authorization: Bearer <access-token>
Idempotency-Key: <client-generated-key>

{
  "recipient_wallet_id": "...",
  "amount_minor": 8000,
  "currency": "MYR"
}
```

MYR is illustrative, not a requirement from the mock. Use the chosen currency’s minor-unit scale. The sender is derived from validated identity.

```http
GET /wallet/balance
GET /wallet/transactions?limit=20&cursor=<opaque-cursor>
```

Define successful completion versus pending explicitly. Reject invalid inputs, enforce ownership, return a stable transfer ID, and document retry behavior. A same-key replay returns the saved semantic outcome according to the API contract.

```sql
-- Independent access paths for outgoing and incoming history:
CREATE INDEX transfer_sender_history
ON transfers (sender_wallet_id, created_at DESC, transfer_id DESC);

CREATE INDEX transfer_recipient_history
ON transfers (recipient_wallet_id, created_at DESC, transfer_id DESC);

-- Outgoing branch for a page after the cursor:
SELECT transfer_id, sender_wallet_id, recipient_wallet_id,
       amount_minor, currency, created_at
FROM transfers
WHERE sender_wallet_id = :wallet_id
  AND (created_at, transfer_id) < (:cursor_time, :cursor_id)
ORDER BY created_at DESC, transfer_id DESC
LIMIT 20;
```

Run the equivalent recipient branch with the same cursor boundary, merge in the same total order, and keep twenty. Omit the cursor predicate for the first page. These are query examples, not a complete production schema or migration.

## 8. Corrections to the mock interview itself

- Some interviewer follow-ups revealed substantial parts of the answer. Use this document for learning, but do not interpret repeating those supplied solutions as independent mastery.
- Your numeric-in-minor-units answer was prematurely restated as integer storage. Pick and name the actual type yourself.
- The replication discussion mixed generic quorum ideas with PostgreSQL-specific behavior. For tomorrow, defend primary reads and explicit synchronous durability first. Do not claim a generic quorum formula alone makes your PostgreSQL design strongly consistent.
- Saying “one success” in the overspending test assumes both requests are valid, distinct, processed without unrelated failures, and the recipients differ from the sender. State those test conditions.
- The raw 500-byte estimate does not necessarily describe the full transfer plus ledger storage. The annual figure is a planning baseline, not a complete capacity calculation.
- There was no demonstrated need to shard at the estimated traffic. Equally, no benchmark established that one primary definitely meets it. The correct statement is that capacity must be measured.

## 9. Tonight’s focused preparation

Do not try to master every advanced topic overnight. Prioritize explanations you can reproduce without help.

| Time budget | Practice | Completion criterion |
|---|---|---|
| 35 minutes | Rehearse the complete transfer flow and draw four tables | Explain every step and its invariant without notes |
| 25 minutes | Four failures: duplicate request, commit then lost response, simultaneous overspending, opposite-direction deadlock | State durable outcome, retry behavior, and prevention for each |
| 20 minutes | Balance primary reads vs history replicas; cache crash gap | Explain why periodic refresh cannot satisfy fresh balance reads |
| 20 minutes | Exact money, CHECK vs NOT NULL, history indexes, cursor ties | Give one concrete example of each |
| 15 minutes | Token validation, ownership, rate limits, jitter | Explain the purpose of each without mixing them |
| 20 minutes | One uninterrupted mock answer | Requirements → estimates → simple design → one complete transfer → failures → measured scaling |
| 10 minutes | Review remaining weak spots | Write down uncertainties; do not invent guarantees |

Then stop and sleep. Tomorrow, briefly rehearse the transfer flow and failure cases rather than introducing new infrastructure.

## 10. How to sound stronger tomorrow

| Instead of | Say |
|---|---|
| “We have a lot of traffic, so microservices.” | “I’ll start with horizontally scaled stateless instances; I’ll split services if independent ownership or scaling justifies the cost.” |
| “It should be strongly consistent.” | “A balance read after a successful transfer must reflect that transfer, so I’ll read from the primary.” |
| “The transaction will roll back if the service crashes.” | “Before commit it can roll back; after commit it survives. An uncertain response is resolved with the same idempotency key.” |
| “We use a lock so it is atomic across databases.” | “A lock coordinates access. Cross-database atomicity needs a separate commit protocol.” |
| “We need sharding because there are many records.” | “I’ll measure throughput, latency, lock contention and I/O before deciding whether sharding is necessary.” |
| “I’m not sure, give me the answer.” | “I haven’t implemented that mechanism. My required guarantee is X; I would keep the simpler design until I can justify the additional failure modes.” |
| “Everything is handled.” | “The largest unresolved risk is X. My initial design avoids it by Y, with the trade-off Z.” |

Use this answer pattern: **decision → guarantee → failure case → trade-off**. Take a short pause, then give one complete explanation. Clear reasoning matters more than naming advanced components.

## Reference notes

The suggested architecture and interview phrasing are a synthesis for this specific exercise. Primary documentation supporting the implementation details:

- [PostgreSQL numeric types](https://www.postgresql.org/docs/current/datatype-numeric.html)
- [PostgreSQL multicolumn indexes](https://www.postgresql.org/docs/17/indexes-multicolumn.html)
- [PostgreSQL WAL and synchronous commit settings](https://www.postgresql.org/docs/current/runtime-config-wal.html)
- [PostgreSQL standby replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [JWT Best Current Practices, RFC 8725](https://www.rfc-editor.org/rfc/rfc8725.html)

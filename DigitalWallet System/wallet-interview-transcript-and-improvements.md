# Digital Wallet Interview — Question-by-Question Comparison

Your answers are paraphrased from the available voice record. Model answers show one consistent design; other designs can also be correct. **✓** = sound answer, **△** = incomplete, **✗** = incorrect, **—** = not independently demonstrated. Repeated clarification questions are grouped together. Where you revised an answer, the row describes that revision.

## 1. Requirements and estimates

| Interview question              | Your answer                                                | Strong model answer                                                                                                                  |
| ------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| What features are in scope?     | ✓ Balance checks, transfers, history.                      | Internal wallet transfers, fresh balance checks, and paginated history. Exclude deposits, withdrawals, fees, and FX unless required. |
| Strong or eventual consistency? | ✓ Asked which was required.                                | Transfers must be atomic and prevent overspending. Balance reads reflect completed transfers; history may lag a few seconds.         |
| Availability target?            | △ Suggested 99.9%; accepted 99.95%.                        | Target 99.95%. Temporarily reject transfers when safe money movement cannot be established.                                          |
| How many users?                 | ✓ Asked for DAU.                                           | Good clarification: use 2M daily active users out of 10M registered, and clarify geographic scope if relevant.                       |
| Average transfer traffic?       | △ Five transfers per active user; needed calculation help. | 2M × 5 = 10M/day. Divide by 86,400: approximately 116 TPS.                                                                           |
| Peak traffic?                   | ✓ Ten times average: 1,160 TPS.                            | Correct working assumption: approximately 1,200 peak TPS, subject to validation.                                                     |
| Read traffic?                   | △ Proposed 10K RPS, then fifty reads per user daily.       | 2M × 50 = 100M reads/day: approximately 1,160 average RPS and 12K peak at 10×. Clarify balance/history proportions.                  |
| Latency targets?                | △ Confused endpoint targets with P95/P99.                  | P95 balance under 200 ms; P95 completed transfer under 500 ms.                                                                       |
| Monthly storage?                | ✗ Initially said 3 billion transfers/month.                | 10M × 30 = 300M. At 500 bytes each: 150 GB raw/month.                                                                                |
| Annual storage?                 | ✗ Estimated 1.5 TB.                                        | Approximately 1.8 TB using twelve thirty-day months, before additional records, indexes, replicas, backups, and logs.                |

## 2. Architecture, concurrency, and idempotency

| Interview question | Your answer | Strong model answer |
|---|---|---|
| Initial architecture? | △ Client → load balancer → wallet → DB. | Good baseline: stateless wallet instances and PostgreSQL primary. Explain a complete transfer before adding components. |
| Monorepo or monolith? | △ Used monorepo to describe one service. | Monorepo organizes source code. Say one modular service deployed as multiple instances. |
| Why microservices? | △ Traffic, availability, horizontal scaling. | One service can also scale horizontally. Split when independent ownership, release cycles, or scaling justify the added complexity. |
| How do you prevent overspending? | ✓ Pessimistic lock, check funds, update. | Lock the relevant account rows inside the transaction, check funds after acquiring locks, and update both balances atomically. |
| What belongs in the transaction? | △ Both sides. | Both balance updates, transfer record, ledger entries, and idempotency outcome commit together. |
| How do you prevent deadlocks? | △ Timeout, rollback, backoff. | Acquire account locks in ascending account-ID order. Retain timeouts and bounded retry for recovery. |
| What happens on service crash? | △ Initially said rollback. | Before commit, unfinished work rolls back. After commit, changes survive. Resolve uncertain outcomes using the same idempotency key. |
| How do retries avoid duplicates? | △ Store and check a key. | Scope the key to caller and operation; persist a request fingerprint and result atomically with money movement. |
| Two same-key requests arrive together? | ✓ Unique DB constraint. | Correct. Claim the key before updates; uniqueness admits one owner. Duplicate requests retrieve the outcome or receive a defined processing response. |
| Why commit idempotency and money together? | △ Agreed; explanation initially unclear. | Otherwise money can commit without a retry record, or a retry record without the money movement. |
| Commit succeeds, response lost? | ✓ Return saved success on retry. | Correct: retry the same key, return the durable result, and do not transfer again. |
| Same key, changed amount? | ✓ Reject after payload comparison. | Correct: reject conflicting recipient, amount, or currency. New keys represent new intended operations. |
| Does atomicity prevent duplicates? | ✗ Later suggested yes. | Atomicity prevents partial transactions. Idempotency prevents duplicate business operations. |

## 3. Database design

| Interview question | Your answer | Strong model answer |
|---|---|---|
| Balance table fields? | △ User/account ID, available and current balances. | Wallet ID, owner ID, currency, exact balance. Add held/available amounts only if reservations are required. |
| Available versus current? | △ Definitions relied on later reconciliation. | Define them precisely or use one posted balance for this scope. Completed transfers update spendable money immediately; reconciliation verifies afterward. |
| User lookup index? | △ Unique user ID plus another index. | A PostgreSQL unique constraint already creates an index. Avoid an identical extra index. |
| Money type? | △ Decimal/numeric; later smallest units. | Choose explicitly: BIGINT minor units or fixed-scale NUMERIC. Define scale, rounding, and overflow handling. |
| Why avoid floating point? | — Asked for help. | Many decimal amounts are represented approximately, creating rounding discrepancies. Use exact arithmetic for the chosen representation. |
| Need currency? | △ Asked whether to include it. | Scope to one currency; explicit currency fields help prevent mismatched postings. Multiple currencies require additional rules. |
| Application-only funds check? | △ Initially preferred application checks. | Check funds under concurrency protection, and add a database nonnegative constraint as defense against invalid writers. |
| Can ABS prevent negatives? | ✗ Proposed ABS. | No: it changes an invalid negative balance into invented positive money. Reject invalid writes. |
| Nonnegative constraint? | ✓ Eventually NOT NULL plus CHECK ≥0. | Correct: NOT NULL rejects missing values; CHECK balance ≥0 rejects negatives. Transfer amounts must be strictly positive. |
| Transfer record? | △ Initially one account; later sender and receiver. | Transfer ID, sender wallet, recipient wallet, positive amount, currency, status, timestamp. Define self-transfer behavior. |
| Ledger record? | △ Debit/credit users and amounts linked to transaction. | One immutable entry per wallet per transfer is a clear model: transfer ID, wallet ID, signed amount, currency, timestamp. Internal postings sum to zero. |
| Reconciliation? | △ Nightly scheduler. | Compare ledger totals against balances; detect missing, duplicated, or unbalanced postings. Investigate with auditable repair, not silent overwriting. |

## 4. History queries and pagination

| Interview question | Your answer | Strong model answer |
|---|---|---|
| Transfer in both histories? | △ Added sender/receiver after prompting. | Query where the wallet is sender or recipient; show direction relative to the viewer. |
| History indexes? | ✗ Sender + receiver + timestamp in one index. | Separate sender + timestamp + transfer-ID and receiver + timestamp + transfer-ID indexes. |
| Why separate indexes? | — Asked for help. | Incoming transfers may have any sender. A sender-leading index is poorly matched to that lookup. |
| How do you merge results? | — Interviewer supplied explanation. | Fetch recent outgoing and incoming records, merge under one total order, deduplicate overlap if needed, keep twenty. |
| How do you paginate? | △ Named a cursor without its contents. | Encode the last timestamp and unique transaction ID; use the same ordering and strict boundary for subsequent pages. |
| Why include an ID? | — Asked for help. | It breaks timestamp ties so tied records are not skipped by a timestamp-only boundary. |

## 5. Caching and replicas

| Interview question | Your answer | Strong model answer |
|---|---|---|
| Balance read source? | △ Redis refreshed after transfers. | Read strict balances from the primary initially. Redis is not the money source of truth. |
| DB commits, cache refresh fails? | △ Miss handling and scheduler. | The stale value may remain a cache hit. Miss handling does not repair it. Avoid this cache on the strict read path. |
| Refresh every fifteen minutes? | ✗ Suggested for consistency. | That gives eventual freshness, not the agreed immediate reflection of committed transfers. |
| Cache penetration? | ✗ Used that term for stale data. | This is a stale hit. Diagnose the actual state before selecting the remedy. |
| Client-side history cache? | △ Refresh after a write. | Incoming transfers do not require that client to write. Use a defined refresh interval or notification plus refresh. |
| CDN history cache? | △ Proposed CDN, then client cache. | Omit shared CDN caching initially; private financial history needs carefully enforced access and cache isolation. |
| Replicas for balances/history? | △ Proposed both. | Async replicas can serve lag-tolerant history. Use primary for fresh balances unless a precise freshness protocol is implemented. |
| Does R+W>N solve it? | △ Proposed quorum and latest-version reads. | Not by itself. PostgreSQL replication needs explicit durability and read-routing semantics; a generic overlap formula is not a complete consistency protocol. |
| Replica acknowledgement means what? | — Needed explanation. | Distinguish receipt, durable storage, and application visible to queries. Choose the guarantee required for the operation. |

## 6. Failures and recovery

| Interview question | Your answer | Strong model answer |
|---|---|---|
| One replica down, primary healthy? | ✗ Election or sloppy quorum. | Keep the primary. Remove failed replica from reads. Continue writes only if required durability remains satisfied. |
| Primary fails? | △ Elect latest replica. | A failover system chooses an eligible replica, verifies required committed data, fences the old writer, and promotes safely. |
| Old primary isolated but running? | △ Pause transfers; ping checks. | Pausing is reasonable. Before resuming, fence the old primary so it cannot keep writing. Ping alone is insufficient. |
| Why not election alone? | — Asked for help. | Selecting a new leader does not stop the old one from accepting writes. |
| Preserve acknowledged transfers? | △ Suggested monitoring. | Require suitable synchronous durability before success and promote only a replica containing those commits. Monitoring observes; it does not create durability. |

## 7. Scaling and sharding

| Interview question | Your answer | Strong model answer |
|---|---|---|
| DB saturated, app CPU low? | △ Replicas/sharding; earlier larger app machines. | More application CPU does not remove a DB bottleneck. Measure queries, connections, locks, I/O, and write load first. |
| Why shard? | △ Many records and heavy I/O. | Benchmark realistic data and workload. Shard only when evidence justifies the operational and correctness cost. |
| Shard key? | △ User ID, consistent hashing. | User ID colocates one user’s data, not both parties’ wallets. Establish cross-shard transfer semantics before choosing it. |
| Consistent hashing benefit? | △ Easier migration on expansion. | It reduces remapping; safe migration, concurrent writes, and routing transitions still require a protocol. |
| Redis lock across shards? | ✗ Proposed for atomic transfer. | It coordinates access but cannot make independent database commits atomic. |
| Two-phase commit? | △ RPC, then commit after response. | Durable prepare at participants, durable coordinator decision, and recovery following that decision. Participants may block during failures. |
| Queue for more throughput? | △ Buffer and process asynchronously. | A queue absorbs bursts but not a sustained capacity deficit. Define pending/completed states, backlog limits, and idempotent consumers. |
| Validate capacity? | — Asked for help. | Load-test realistic volumes, read/write mix, and hot-key skew. Measure TPS, P95 latency, errors, lock waits, CPU, disk, and replication lag. |

## 8. Security and testing

| Interview question | Your answer | Strong model answer |
|---|---|---|
| Someone else’s wallet as sender? | ✓ Backend derives sender from token. | Correct: validate identity and derive owned wallet server-side. Do not trust client-supplied ownership. |
| Validate a token? | △ MFA, then blacklist. | For signed tokens, validate signature with trusted keys and allowed algorithms, required claims, and revocation. MFA is separate. |
| Blacklist absence proves authenticity? | — Asked for help. | No. A fabricated token may never be blacklisted; authenticity must be verified first. |
| Financial monitoring? | — Asked for help. | Monitor ledger/balance mismatches, unbalanced entries, duplicate application, negatives, and missing postings. Internal transfers conserve total funds. |
| Concurrent overspending test? | △ Repeated implementation. | Start at 100; two distinct concurrent transfers of 80. Assert one success, one insufficient-funds result, sender 20, one recipient credited, and one balanced posting. |
| Other tests? | — Not independently developed. | Same-key duplicates, changed payload, opposite-direction locking, crash before commit, and lost response after commit. |

## 9. Hot wallets, rate limits, and wrap-up

| Interview question | Your answer | Strong model answer |
|---|---|---|
| Popular merchant bottleneck? | ✓ Recognized hot-wallet contention. | Correct: transfers contend on the merchant balance row, increasing lock waits and limiting throughput. |
| Reduce hot-wallet impact? | △ Short transactions, fewer RPCs. | Good start. Cap per-wallet concurrency before taking DB connections, bound waiting, and reject excess. |
| Protect unrelated transfers? | △ Timeout / fail closed. | Per-wallet and global admission controls prevent hot-wallet waiters from consuming all shared connections. |
| Rate-limit dimension? | △ IP → user → wallet. | Complementary limits: user for fairness, destination wallet for contention, global for capacity. IP can supplement abuse protection. |
| Token bucket implementation? | ✓ Redis with atomic Lua. | Reasonable. Define refill/consume behavior, idle expiry, and what happens when the limiter is unavailable. |
| Rate versus concurrency limits? | △ Not distinguished. | Rate controls arrivals over time; concurrency controls in-flight work. Both can be necessary. |
| Status for rate limiting? | ✗ 204/402/403; later learned 429. | 429 Too Many Requests with retry guidance; 503 can indicate temporary service overload. |
| Retry strategy? | △ Backoff. | Capped exponential backoff, random jitter, bounded attempts, original idempotency key, and server guidance. |
| Why jitter? | — Asked for help. | Prevent synchronized retries from producing another traffic spike. |
| Biggest remaining risk? | △ Concluded most concerns were handled. | Name an unresolved mechanism: cross-shard atomicity is not established, so defer sharding; verify durability and fencing for failover. |



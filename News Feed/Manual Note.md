1. Reqruiement
	1. FR
		1. CRUD post
		2. Follow feature
		3. Pagination, unlimited scrolling
	2. NFR
		1. Read latency p95<200ms
		2. Write latency p95<500ms
		3. AVailability 99.99%
		4. Feed freshness
		5. Eventual consistency
		6. Scalability
2. Capacity Estimation
	1. DAU - 100M
	2. Post per user/day - 2
	3. Feed request per user per day 20
	4. avg follower count -500
	5. post wif images 20%
	6. avg image size 500kb
	7. Write thoughput = 2315 posts/sec
	8. Read 23k per sec
	9. Hidden problem : Fan out amplification
		1. if 200m daily post must be inserted into 500 flwers feed means that we have 100b feed insertions per day
	10. Storage : 200M x 500B = 100GB/day 
	11. Images : 200M x 20% x 500kb = 20TB/day
3. API 
	1. POST api/v1/posts
	2. PUT api/v1/users/follow/:userId
	3. GET api/v1/feed?limit=20&cursor=xyz
	4. Why cursor instead of OFFSET?
		1. Imagine the first page contains posts 100 to 81. While Bob is scrolling, someone publishes post 101.
		2. With `OFFSET 20`, items can shift between requests, causing duplicates or skipped posts.
		3. A cursor based on `(createdAt, postId)` lets us continue from the last item seen, even when new posts are published.
		4. For a hybrid feed, the cursor may also need source-specific positions and a snapshot boundary.
4. Data model
	1. Post DB, Feed Cache (save post ID only) , Object cache(object details by postID)
5. High Level Architecture
	1. ![[Pasted image 20261029122516.png]]
6. Deep Dive : 
	1. Fanout on read vs fanout on write
		1. ![[Pasted image 20261029122606.png]]
		2. Fanout Read
			1. adv : no duplicated entries, efifcient for inactive users, publishing is inexpensive
			2. disadv : expensive feed reads, multiple db queries, postntial slow merging when follow many acc
		3. Fanout Write
			1. adv: fast read ,predictable feed retriveal, minimal computation during normal feed requests
			2. Disadv : write amplification, expensive distribution to millions of ollowers
		4. ![[Pasted image 20261029123707.png]]
		5. Hybrid Fan out
			1. We do fanout on write for normal account(push to limited account), hot account can do fan out on read (follower pull the data)
		6. Cache Strategy
			1. why cache only ids?
				1. if cache post body into every feed would require big duplication data (500bytes x 1M = 500MB for each post body)
				2. if we only store 8 byte post id onl cost 8MB
				3. thus we split it into caching postID and object
			2. what if a viral post becomes a hot key?
				1. single cache shard holding this post may be overloaded
				2. Possible solutions :
					1. Replicate hot key accross post object accross multiple cache instances
					2. use local L1 caches in Feed Service instance for hot object
					3. Use request distribution and cache monitoring to detect hot partitions
		7. Pagination and Feed Freshness
			1. Use a stable cursor containting ordering information
			2. avoid offset based pagination
			3. deduplication id when merging push and pull
7. Tradeoff
	1. Kafka crash?
		1. retry unack events
	2. event deliver twice
		1. idempotent key
	3. Redis lose feed
		1. fetch from db again
	4. Fan out queue becomes overloaded
		1. autoscale workers
	5. post deleted
		1. filter from read, invalidate content cache, clean feed entries asynchronously
	6. user unfollows someone
		1. filter immediately at read time; clean feed asynchronously
	7. PostDB is down
		1. serve safe cached if possible ; fail writes that cannot be durable stored (consistent)
	8. DB saved post but kafka dont receive event
		1. Transactional Outbox Pattern
			1. Ina db transaction, save post and an outbox event
			2. commit both
			3. seperate publisher reads outbox and send event to kafka
			4. mark event published after successful delivery
			5. worker deduplicate event processing
		2. Avoid unsafe dual write between db and msg broker
	9. Monitoring
		1. Feed read p95, p99
		2. Kafka consumer lag and oldest unprocessed event age
		3. post to feed propagation latency
![[Pasted image 20261029131752.png]]
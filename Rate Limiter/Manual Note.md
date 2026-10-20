1. Requirement
	1. Functional
		1. Identify users by id, IP or api key
		2. limit requests based on configurable rules
		3. return proper error headers and status codes
	2. Non Functional
		1. 100M DAU
		2. 1M RPS
		3. availability > consistency, 
		4. Low Latency , minimal, dont affect normal api call (<10ms)
		5. scalable to 1M RPS
	3. Core Entities
		1. Request
		2. Client (IP, userID, API key)
		3. Rules 100rqs
	4. API Design & DB Design
		1. isRequestAllowed(clinetId,rulesId) -> {passes:boolean, remaining:number,resetTime : timestamp}
	5. High Level Design
		1. Where should rate limiter live?
			1. Gateway/Load Balancer
		2. How should we identify clients
		3. Maybe authenticated user having higer rate limit?
		4. We use a redis cluster to act as rate limiter, it offers fast in memory, TTL, atomic operations(LUA script)
		5. Redis can be scale horizontally
	6. Approach
		1. Fixed Window Counter
		2. Sliding Window Log
		3. Sliding window counter
		4. Token Bucket (Chosen)
			1. Key : rate_limit:user_123:posts 
			2. tokens = 1
			3. last_refill = 176000000
		5. Leaky Bucket
	7. Deep Dive 
		1. Race Condition
			1. What if we have 2 gateway and read write into one redis cluster? 2 requests allowed using may only 1 token?
			2. Solution : Redis Lua Script
				1. can make read, check, and write in one atomic operation, likely transaction
		2. How can we scale to 1M RPS?
			1. Redis Sharding![](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system%2C%20BlinkMacSystemFont%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20sans-serif%22%20font-weight%3D%22430%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(255%2C%20255%2C%20255\)%22%20viewBox%3D%220%200%20340%20200%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%22105%22%20y%3D%228%22%20width%3D%22130%22%20height%3D%2238%22%20rx%3D%228%22%20fill%3D%22%233b82f6%22%20fill-opacity%3D%22.13%22%20stroke%3D%22%233b82f6%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%2223%22%20fill%3D%22currentColor%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20font-weight%3D%22600%22%3EAPI%20Gateways%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%2237%22%20fill%3D%22currentColor%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%3EDetermine%20Redis%20key%3C%2Ftext%3E%3Cpath%20d%3D%22M170%2046%20V72%20M56%2072%20H284%20M56%2072%20V105%20M170%2072%20V105%20M284%2072%20V105%22%20stroke%3D%22%2394a3b8%22%20stroke-width%3D%221.6%22%20fill%3D%22none%22%2F%3E%3Crect%20x%3D%227%22%20y%3D%22105%22%20width%3D%2298%22%20height%3D%2264%22%20rx%3D%229%22%20fill%3D%22%23d97706%22%20fill-opacity%3D%22.10%22%20stroke%3D%22%23d97706%22%2F%3E%3Ctext%20x%3D%2256%22%20y%3D%22125%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20font-weight%3D%22600%22%20fill%3D%22currentColor%22%3EShard%201%3C%2Ftext%3E%3Ctext%20x%3D%2256%22%20y%3D%22144%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3ERedis%3C%2Ftext%3E%3Ctext%20x%3D%2256%22%20y%3D%22159%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22%2364748b%22%3EUsers%20A%E2%80%93H%3C%2Ftext%3E%3Crect%20x%3D%22121%22%20y%3D%22105%22%20width%3D%2298%22%20height%3D%2264%22%20rx%3D%229%22%20fill%3D%22%23d97706%22%20fill-opacity%3D%22.10%22%20stroke%3D%22%23d97706%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%22125%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20font-weight%3D%22600%22%20fill%3D%22currentColor%22%3EShard%202%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22144%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3ERedis%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22159%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22%2364748b%22%3EUsers%20I%E2%80%93P%3C%2Ftext%3E%3Crect%20x%3D%22235%22%20y%3D%22105%22%20width%3D%2298%22%20height%3D%2264%22%20rx%3D%229%22%20fill%3D%22%23d97706%22%20fill-opacity%3D%22.10%22%20stroke%3D%22%23d97706%22%2F%3E%3Ctext%20x%3D%22284%22%20y%3D%22125%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20font-weight%3D%22600%22%20fill%3D%22currentColor%22%3EShard%203%3C%2Ftext%3E%3Ctext%20x%3D%22284%22%20y%3D%22144%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3ERedis%3C%2Ftext%3E%3Ctext%20x%3D%22284%22%20y%3D%22159%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22%2364748b%22%3EUsers%20Q%E2%80%93Z%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22190%22%20text-anchor%3D%22middle%22%20font-size%3D%2211%22%20fill%3D%22%2364748b%22%3EConceptual%20partitioning%20by%20hashed%20user%20identity%3C%2Ftext%3E%3C%2Fsvg%3E)
			2. Redis Cluster already handle routing using 16384 hash slot accross primary nodes,so no need consistent hashing
			3. suppose a redis node can sustain 80k rps
			4. -> 1M/80000 = 12.5 redis replica works well for 1M RPS
			5. Failover : What if redis failed?
				1. Fail Open, allow request
				2. Fail Closed: reject request
				3. we can do hybrid here, for payments route or login we can do fail closed, some public api or search api can do fail open
![[Pasted image 20261015171316.png]]
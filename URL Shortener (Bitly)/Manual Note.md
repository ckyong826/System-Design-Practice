1. Requirment Gathering
	1. Functional Req
		1. URL shorting
		2. URL should be unique
		3. URL Redirection short url to long url
		4. Optional : Link Analytics
		5. Optional : customize url name
		6. Optional : 
	2. Non Functional
		1. Low Redirect Latency
		2. 100M DAU -> 1 user 10 click, 1B reads per day
		3. 10^9/10^5 = 10k RPS
			1. Hot URL may make it more in peak period
		4. 5B lifetime URLS
		5. **Clarify with your interviewer if you should run back-of-the-envelope usage calculations.**
			- Size per paste
			    - 1 KB content per paste
			    - `shortlink` - 7 bytes
			    - `expiration_length_in_minutes` - 4 bytes
			    - `created_at` - 5 bytes
			    - `paste_path` - 255 bytes
			    - total = ~1.27 KB
			- 12.7 GB of new paste content per month
			    - 1.27 KB per paste * 10 million pastes per month
			    - ~450 GB of new paste content in 3 years
			    - 360 million shortlinks in 3 years
			    - Assume most are new pastes instead of updates to existing ones
			- 4 paste writes per second on average
			- 40 read requests per second on average
2. API Design/DB Design
	1. POST /api/urls/shorten
		1. Req : long url
		2. Res : short url
	2. GET /api/urls/{shortUrl}
		1. Req : short url
		2. Res : Redirect 301 (Browser cache response)
		3. Res ; Redirect 302 (Temporary : useful for tracking clicks)
	3. DB Design (1kb for each record) x 5b => 5TB

| Attribute  | type    |
| ---------- | ------- |
| short_url  | varchar |
| long_url   | varchar |
| uid        | varchar |
| created_at | date    |
| used_count | int     |

1. High Level Design
	1. Using Hash function to hash long url to short url in Base62 (0-9,a-z,A-Z) 7 bytes -> supporting 62^7 => 3.5 trillion unique URLS enough for 5B urls need to be stored
	2. Hash function using CRC32, MD5, SHA-1
		1. but after hashed they would not get 7 bytes, might be longer -> hash collisions(duplicate if we only get first 7)
		2. Resolve it then we for loop? 
		3. It takes time and add latency in memory, if acceptable we can use this
		4. alternative : Bloom Filters 
	3. Map short url to Long Url
2. Deep Dives
	1. Unique Short URLs?![[Pasted image 20261014170553.png]]
		1. Counter
			1. Cons : 
				1. what if multiple instance of service? duplicate?
				2. what if server down? counter drop?
	2. URL Redirection![[Pasted image 20261014170824.png]]
	3. Hot Url problem
		1. Solve with using redis for frequent read url, caching
		2. DB read replica to handle READ operation, everytime write will write to read replica first, use back our quorum consessus
	4. Analytics?
		1. We can have another analytic service called by gateway after redirection is called
		2. then counter is saved to another DB replica to handle Write operation for counting (since this is very heavy to db, alot write operation)
		3. Every 60 sec we will update counts into Master DB
3. Complete Architecture
![[Pasted image 20261014171632.png]][[]]
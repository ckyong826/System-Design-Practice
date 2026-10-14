4 Step
1. Requirement Gathering
	1. Problem : Distributed application need a reliable way to generate unique IDs, if each machine generate uuid themselves may have duplicate ID for million or billions of orders
	2. Functional :
		1. Uniqueness (dont have 2 same id for same object)
		2. Numeric (only numbers)
		3. 64 bits for each id
		4. sortable by timestamp
2.  Non Functional Requiremnt
	1. Generate 10k ID/s second
		1. For example if we have DAU likely 1 million users
		2. 1m/86400=11SAU
		3. if like 11.11 event, we should support 	x100 1100SAU
		4. if 1 user avg order 5 in second, means we need 5500 ID per second
		5. so at least 10k ID/s is needed
	2. Low Latency to generate ID to services
3. Scale
	1. Support multiple services and servers and data centers
	2. Avoid Single Point of failure
4. High Level Design
	1. Multi Master Auto Increament
		1. actually giving different db offset and increament, then each db will not geenrate same ID
		2. nextID = previousID + numberOfServers
		3. but what if adding or removing server?
		4. timestamp is not synchornized between servers (might collide)
		5. complicated if server go much to manage
	2. UUIDv4
		1. each server can generate UUID themselves
		2. pros : 
			1. no centralized ID service
			2. easy to scale horizontally
			3. no per ID communcation required
		3. cons
			1. 128bits
			2. dont preserve creation time order
			3. Random id lead to less locality in db indexs
			4. UUID v4 is not sortable by timestamp
		4. Howeveer, UUID v7 actually include timestamp and time sortable, if 128 bit is acceptable, this is satisfied
	3. Ticket Server
		1. A centralized server to generate ID for other server and nodes
		2. simple solution
		3. but we have problem of
			1. Single point of failure
			2. Additional network latency
			3. as request volume increase, it is diffcult to scale horizontally, only vertically then would be bottle neck
	4. Twitter Snowflake
		1. contruct ID by important pieces of information
		2. Time + Machine ID + Sequence Number

| Approach                    | Distributed            | 64-bit | Time-ordered        |
|------------------------------|------------------------|--------|---------------------|
| Multi-master increment      | Yes                    | Yes    | Not globally        |
| UUIDv4                      | Yes                    | No     | No                  |
| Ticket server               | Centralized allocation | Yes    | By allocation order |
| Snowflake                   | Yes                    | Yes    | Approximately       |
| UUIDv7 (modern alternative) | Yes                    | No     | Approximately       |


5. API + Database Design
	1. n/a
6. Deep Dive + Failure Handling
	1. How Snowflake works?
		1. 64bit Snowflake
		2. ![[Pasted image 20261014134744.png]]
		3. 2 ^ 41 ms = we can support 70years afterward
		4. DC id and machine id make each machine generated snowflake ID identical
		5. 2^12 support 4096 id for each 1ms
		6. this means that 1 second we can generate 4 millions id per second for each machine, which is enough to fullfill our requirmenet
	2. Failure Handling
		1. What if clock moves backward for machine?
			1. might generate duplicate id
			2. we can just check if now < lastTimestamp used
			3. Fallback ->
				1. Wait for clock to catch up
				2. Reject ID generation temporarily
				3. alert operators
		2. What if 2 machines receive same machine id? 
			1. Unique worker assignment
			2. Zookeper to handle identity assignment
			3. a worker id should never be assigned to 2 active generators concurrently
		3. What if we rlly have >4096 request in 1 ms?
			1. add a latency to wait next ms
			2. or redesign bit allocation to support more sequence and less machine/data center
		4. what fi 2 server have different timestamp?
			1. how to solve this?
7. Final Architecture
![[Pasted image 20261014135503.png]]
8. Trade off + Improvements
	1. any can add?
Probabliy this is same = Design Distributed Cache 
1. Requirement Gathering (Probably this is asking from interviewer)
	1. Functional
		1. Retrieve value[[]]
		2. Insert Update 
		3. Delete pair
	2. Non Functional
		1. Scalability (Support scaling of thousands of node storing data)
		2. Low Latency p99, <10ms
		3. Symmetry, Data consistent, Same responsibility
		4. CAP
2. Data Storage
	1. Write ahead log (sequentianal writing the log in disk?)
	2. Memtable (Read black tree) -> guarantee crash durability -> Keeps data sorted and fast O log N reads
	3. SSTable (Sorted String Table) use fast sequential disk, got data block, index block 
	4. Compaction, We have alot SS tables, background wokrer merge files to remove deleted keys or any update
	5. Bloom Filter (avoid searching disk for a key dont exist) -> used to proved an key not existed, 
		1. Hash("abd") = Index 2,5,7
		2. Check in memory bit array, we found index2 == 0 from [0,1,0,0,1,1,0,1,0] Array of bits
		3. abd is not in file , this is O(1) in time complexity
3. API Design
	1. Using RPC Remote Procedure Call
		1. Abstract the messy complexities of network communication
		2. Execute actions on remote server as if calling a local function
	2. Get(key), Put(key, context,value), delete(key)
4. Deep Dive 1 : Partitioning & Replication Server 
	1. Consistent Hashing
		1. Fixed Consistent Hashing Ring, each request is saved to nearest node when running clodewise
		2. Using Distributed Randomly Virtual node to achieve perfect load balancing
		3. Probably this help in scability, in modulo hashing, it is difficult to add and remove server coz just chaning the ownership for certain hash range like 60-80, unlike modulo will affect most of keys
	2. Replication
		1. leader, candidate, follower
		2. it replicate to node clockwise in consistent hashing as well
5. Deep Dive 2: Quorum Consensus & Consistency
	1. Quorum -> solve the problem when system crash before slave node havent copy the data from master which make system inconsistency while sending success msg directly to user
	2.  Implementation 
		1. N : Total Replicas
		2. W : Write Quorum
		3. R : Read Quorum
		4. Golden Rule : W+R > N
	3. For example, DynamoDB Default prioritise speed and availability => Make sure eventual consistency 
	4. Increase R and W -> Increase latency
	5. Simply say,  we having Nodes [A,B,C]
		1. Eventual Consistency : 
			1. N=3, W=1, R=1 => 1+1<3
			2. A -Write-> v2
			3. B -> v1
			4. C -Read-> v1(Stale)
			5. Eventual Consistency reached but C still read stale data, user directly get success response but only one node having v2, and else node still having v1 (data inconsistency but very fast)
		2. Strong Consistency : 
			1. N=3, W=2, R=2 -> 4>3
			2. . A -Write-> v2
			3. B -Write-> v2
			4. C -Read-> v1(Return v2)
			5. for this, since W=2, we must write A and B, and when read R=2, even data is fetched from Node C, it should also read Node A/B and get latest data
			6. Guarenteed Latest Data, but slower , failes if 2 nodes die
6. Deep Dive 3 : Conflict Resolution
	1. Data race condition for eventual consistency
	2. LWW Last Write wins
		1. Always attach a physical timestamp to every write
		2. latest timestamp win
	3. use vector clock / versioning
		1. Baseline [ Cart: "Shoes" -> Vector: {N1:1,N2:0}]
		2. Concurrent writes (packet loss, or concurrently or firewall problem any else) : 
			1. Node 1 : Add shirt -> Cart: ["Shoes","Shirt"] Vector: [N1:2,N2: 0]
			2. Node 2 : Add hat ->Cart: ["Shoes","Hat"] Vector: [N1:1,N2: 1]
		3. Partition Heals : DB compares vectors and found conflict -> refuses to overwrite
		4. Resolution : Merge both to get Cart: ["Shoes","Hat", "Shirt"] Vector: [N1:2,N2: 1]
	4. Failure Detection
		1. Gossip Protocal(<100ms)
			1. What if master node is down but how can we let other nodes know?
			2. keep hearbeating between nodes is costly
			3. We can use Gossip Protocal : Like a virus spreading or rumor spreading, I tell u, u tell others -> A tell B and C, B and C each tell others nodes
		2. Hinted Handoff (Temp Failures
		3. Permanent Failures
			1. Node exchanges Merkle Tree
			2. ![[Pasted image 20261013173311.png]]![[Pasted image 20261013173238.png]]
			3. Keep hashing from leaves, then when we found different on root node, means differ, and can quickly identify the exact key which is wrong
7. Complete Architecture
	1.  Write API
		1. App Server Routing
		2. Coordination(RAFT)
		3. Paralle Dispatch
		4. appended into disk (write ahead log) then inserted into in memory memtable 
		5. ack and quorum between nodes
		6. Success Response
	2. Read API
		1. App server routing
		2. dispatch and local lookup
		3. quorum Read evaluation  
		4. if found problem, read repair and response
		5. get clean read, if conflict, vector clock help resolving
8. Additional Discussion Points
	1. Hot Partitions (hyper popular key)
		1. Mitigrate by dynamic increasing the replicaiton factor by hot keys
	2. Data center Awareness 
		1. should be available zone aware
		2. coodinator ensure that N replicase never placed on nodes in same data center


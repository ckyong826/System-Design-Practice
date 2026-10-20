1. Requirements
	1. Functional
		1. Upload a file
		2. Download a file
		3. Auto sync file accross devices
	2. Out of scope
		1. roll own blob storage
	3. Non Functional
		1. CAP - Availability >> consistency AP
		2. Low latency upload & Download (as low as possible)
		3. support large files as 50gb
			1. resumable uploads
		4. High data integrity (sync accuracy)
2. API / DB Design
	1. Core Entities
		1. File(raw bytes)
		2. File Metadata
		3. Users 
	2. POST /files -> 200
		1. body: File & FIleMetadata 
	3. GET /files/:fileId -> File & FileMetadata
	4. GET /changes?since={timestamp} -> fileIds[]
3. High Level Design
	1. User should able to upload file from aby device
		1. DynamoDB(noSQL) to save metadata
			1.   {
				    "id": "123",
				    "name": "file.txt",
				    "size": 1000,
				    "mimeType": "text/plain",
				    "uploadedBy": "user1"
				  }
		2. Bad solution: Upload file to single backend server in local file system
		3. Good Solution: Store file in Blob storage like Amazon S3.
			1. Unlimited file can be saved as it will handle the scaling for us
			2. if server down, we dont lose access to files
			3. feature like to auto delete old files and versioning to keep track of file changes if needed
			4. Tradeoff : complex, require upload a file twice (backend) and S3 once. Can be solved by allowing user to upload file directly to blob storage
		4. BEST : Upload file directly to blob storage
			1. faster and cheaper
			2. we can use presigned URLS to generate a URL that user can direct upload file
			3. after file is uploaded, blob storage service send a notification to backend to save metadata (push)
				1. POST /files/presigned-url -> PresignedUrl
						Request:
						{
						  FileMetadata
						}
![[Pasted image 20261016185727.png]]
	2. User should able to download a file from any device
		1. Bad : download through file server
			1. Double download, server once and S3 once
		2. Good : Download from S3
			1. Using presigned URL then user can use it to download directly![[Pasted image 20261016185849.png]]
			2. Tradoff : could be slow for a large user base, and limited in a single region, more far region take slower download time
		3. Great : CDN
			1. Reduce latency and serve file from server closest to the user
			2. We can generate a URL for user to download file from CDN for a limited time
			3. ![[Pasted image 20261016190007.png]]
	3. User should be able to share file with other user
		1. Bad : add a sharelist to metadata
			1. using index, but when getting the list of file shared would need to scan shareList attribute of every file which time consuming
		2. Good : Caching to speed up fetching sharelist
		3. Great : seperate table to store sharing details, can use UserId to partition for sharding as well
	4. User can auto sync file accross devices
		1. Local Trigger
			1. Every time user update file then will trigger sync to remote server
			2. Need a client side sync agent to monitor the local dropbox folder for changes
			3. if detect change, queue to upload locally
			4. "Last Write Wins" is used when 2 user edit same file
		2. Remote pull
			1. Polling : The client periodically ask server "has anything change?" , server check by updatedAt timestamp
				1. but this is slower to detect change and upload
				2. waste bandwidth if no changes
			2. Websocket : open connection to push noto when changes occur, 
				1. more complex
				2. real time updates
		3. Hybrid
			1. We use websocket + periodic polling as a fallback solution
![[Pasted image 20261016190515.png]]
4. Deep Dives
	1. How to support Large files
		1. Before deep dive, we need to know problems:
			1. Web server and client typically having timeout, if Post 50gb file, will easily exceed timeout
			2. Browser and server limited the size of a request payload
			3. Network interupption
			4. User experience : if they are blind to progress of upload, they have no idea how long it take
		2. Solution: 
			1. Chunking the file, we can break file into 5-10MB pieces and upload chunk by chunk
			2. Then we can do progress indicator for the user since it is based on chunks uploaded
			3. Resumable Uploads : we can save state of upload in db in File Metadata
				1. then user can resume uploading based on the chunks,
			4. **But how should we ensure this chunks field is kept in sync with the actual chunks that have been uploaded?**
				1. Good : Update based on client PATCH request
					1. The client takes the file, chunks it, and uploads the chunks directly to S3.
					2. S3 responds to each chunk upload with a success message.
					3. Upon success, the client sends a PATCH request to our backend to update the chunks field in the FileMetadata table.
					4. Challegnes: Security risk, user could send a PATCH request to mark all chunks as uploaded without uploading them, risk for inconsistent state
					5. we can use server side approach to ensure chunk field stays in sync with actual
				2. Server Side Chunk Veirification
					1. ETags
					2. Each chunk gets an ETag upon successfully upload.
					3. Backend can verify them by calling S3's ListParts API, it can validate multiple chunks at once
		3. S3 Multipart VS Delta Sync
			1. Delta sync is supported by chunk hashes to support auto deduplication
	2. Delta Synchronization
		1. if 4MB is edited from 100MB file, we can just reupload the blocks has been changed
		2. How server know which block chaned?
			1. Each block has a content hash, the hash is different is it is changed
		3. What if a byte is inserted at the beginning of the file
			1. Fixed Size Chunks : Insertion can shift every later chunk boundary
			2. Advanced : Content Defined Chunking (CDC) : use rolling hashes to choose chunk boundaries based on content, this is more efficient for insertions and deletions
	3. Real Time File Synhcornization
		1. How does Server know that there is a new version?
		2. Approach
			1. Short polling
				1. Simple but repeated request and slower updates
			2. Long polling
				1. Efficient for infrequent updates but need reconnect after each response
			3. Websocket
				1. Fast bidirectional communication but need to do persistent connection management
	4. Deep Dive 4 : Concurrent Updates and Conflict Resolution
		1. 2 Device commit update same file at same time or offline
		2. 409 Conflict
		3. Use Optimistic Concurrency Control
			1. UPDATE Files
				SET latest_version_id = :new_version
				WHERE file_id = :file_id
				AND latest_version_id = :expected_version;
		4. Client can create another conflicted copy if conflict
		5. It is safer than applying Last Write Wins
		6. Trade Off
			1. LWW is simple but may silently discard changes
			2. Version checking preserves data but requires conflict handling logic
			3. automatic merging may work for doc or text but not for binary files like image
	5. File Sharing and Permission
		1. When B requests a download:
			- Authenticate B.
			- Retrieve the file metadata.
			- Check B's permission.
			- Generate a short-lived signed download URL.
			- Allow B to download the file directly from object storage.
		2. Backend should never generate download URLs merely bcoz someone knows a fileID or chunk hash
		3. A presigned URL is effectively a bearer credential. Anyone who obtains it can generally use it until it expires, so keep lifetime short and limit access to the specific object or operation
		4. sharing permissions may be inherited by descendants that requires explicit rules to prevent unexpected access after moves or permission changes
	6. Failure Handling

		| Failure | Solution |
		|---|---|
		| Internet disconnects during upload | Resume missing chunks |
		| Client retries the same commit | Idempotency key |
		| Storage upload succeeds, DB commit fails | Retain pending state; retry/reconcile; clean up orphaned data |
		| Metadata saved but notification fails | Transactional outbox + retry |
		| WebSocket disconnects | Reconnect and fetch change log |
		| Duplicate Kafka event | Idempotent consumers |
		| Two devices edit the same version | Version check + conflicted copy |
		| Database instance fails | Replica failover and backups |
		| Object becomes unavailable/corrupted | Durable replication, checksum validation, recovery |
		| Sudden reconnect spike | Backoff with jitter, admission control |
		1. Why use a Transactional Outbox
			1. What is service crash when db commit succeeds? then kafka event is never published?
			2. Instead, save the metadata update and an outbox event in the same database transaction.
			3. An independent worker then publish the outbox event to kafka, retries failed deliveries and marks event as published.
			4. Kafka delivery and processing may still be at least once, so consumers should be idempotent.
7. Trade Offs
	1. PostgreSQL vs DynamoDB? (SQL vs NonSQL)
		1. PostgreSQL since we want transactional metadata updates, version checks, folder organization and permission relationships
		2. even DynamoDB is flexible for key value data but file storage app is not only storing file, some sub feature is important to support this as well
	2. Should we use redis?
		1. Selectively, can cache frequently accessed folder listings and user metadata
	3. Should we use CDN
		1. Hybrid
			1. Private files : Direct S3 downlaods by default
			2. Public and hot file : use CDN using signed access
			3. Sensitive docs: auth > caching
	4. Deduplication vs Privacy
		1. hashing content can save storage by avoiding duplicate chunks
		2. But cross user deduplication can leak information if implemented carelessly
		3. 

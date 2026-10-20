1. Requirement
	1. Funcitonal
		1. Deliver notiifications to users in real time based on topic which multiple users are subscribed to
			1. End User(SMS,email,push noti)
			2. Internal services(producer) : send noti req to system
			3. Operations : Bulk noti to large user segments
		2. Offline user still can fetch notification later
		3. At least once delivery, no notofication should be silently lost
		4. Notification is persisted and retrievable
		5. system track delivery status per notification
![[Pasted image 20261015230646.png]]
	2. Non Functional
		1. 100M DAU, 1B notification per day
		2. Read:Write , Write heavy for pipeline, read heavy for in app inbox
		3. Low latency, how many? <5 sec for transactional notifications, <30 s for normal
		4. Availability : 99.99% for transactional noti should not be lost
		5. Consistency : At least once delivery. Eventual Consistency
	3. Back of the Envelope
		1. Write QPS
			1. 1B notification pd
			2. 1B/86400s = 11600 QPS
			3. Peak x5 = 58k QPS
			4. if we have multiple channel? avg channel = 1.5
			5. 11.6k x 1.5 = 17.4k QPS, Peak = 87k QPS
		2. Read QPS
			1. 100M DAU, if 20% check once per day
			2. 20M per day
			3. 20M/86400 = 230 QPS
			4. Peak Read x5 = 1.15k QPS
		3. Data Storage
			1. each record is 500 byte (user_id, type, title, body, metadata, timestamp, read status)
			2. 1B x 0.5(fraction that go in app) X 500 bytes = 250GB /day
			3. if we persist 90days = 22.5TB
			4. We also need to persist delivery logs(for transactional or notificaiotn need to be retried)
				1. if log is about 200 bytes = 300GB/day = 9TB/ 3 month
			5. Bandwidth to 3 party Providers
				1. 500M pushes notifications /day x 1kb = 500GB/day = 46Mbps avg
				2. Emails are larger -10kb with HTML : 300M emails/day x 10kb = 3TB /day = 280Mbps
	4. API Design
			1. POST /api/v1/notifications
				1. { "user_id": "abc123", "type": "order_shipped", "priority": "high", "data": { "order_id": "789", "carrier": "FedEx" }, "channels": ["push", "email", "in_app"] (optional -- if omitted, use user preferences) }
				2. 202 : { "notification_id": "notif_xyz", "status": "queued" }
				3. 400 Invalid, 429: Rate limited
			2. POST /api/v1/notifications/bulk
			3. GET /api/v1/users/{user_id}/notificaionts?page={page}&limit=100
			4. PATCH /api/v1/notifications/{id}
			5. PUT /api/v1/users/{id}/notification-preferences
		2. High Level Architecture
			1. Notification API(entry point)
			2. Kafka (Message queue) : Decouples poducers, absorbs traffic spikes (peak 87k QPS), persist notification(durable) to avoid loss
			3. Notification Processing Workers : 
				1. Lookup preferences and device tokens
				2. Determine which channel to use
				3. Render the msg from template service
				4. apply rate limiting
				5. publish to channel specific queues
			4. Channel Specific Queues and Senders
				1. Seperate queues for push, email, SMS and in app
				2. Each has its own pool of sender workers to call external provider APIS like APN, FCM, Twilio
			5. In App Writer, write record to Inbox DB
			6. Template service, store notification templates per type per channel
			7. Delivery status tracker : a db to report the result based on the status to power retry logic and debugging, saving delivery log![[Pasted image 20261015234146.png]]
	5. Deep Dive
		1. Retry Logic
			1. Notification Data will be persisted as delivery log with status to do retry logic
		2. Deduplication, event IDs can be checked to avoid sending duplicate notificaions
		3. 
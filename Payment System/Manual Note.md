1) Requirement
	1) Functional
		1) Merchant can submit a payment req
		2) Customer can complete payment using a payment method
		3) Merchant/Customer can view the current status of any payment
		4) System should settle all authorized payments once daily
	2) Non FR
		1) Security - No confidential data leak
		2) Strong Consistency - no duplicate payment, exactly one processing
		3) Durability - 99.9999999& data durability
		4) High scalability - >= 10k QPS sustained throughput
2) API and DB Design
	1) DB
		1) Merchant
			1) merchant_id, name, api_key, status
		2) Payment
			1) payment_id,merchant_id,amount,currency,status, customer_info,created_at, payment_info(encryption)
		3) Transaction		
			1) transaction_id,payment_id,type,status,provider_id,actor_type,respond_code, processed_at
	2) API
		1) POST /payments/init
		2) POST /payments/{payment_id}/confirm
		3) GET /payments/{payment_id}
		4) POST /settlement_batches
	3) State Machine (this is important for payment since nees strict status)![[Pasted image 20261020134130.png]]
3) High Level Design
![[Pasted image 20261020140721.png]]
4) Deep Dive
	1) Security
		1) Risk 1 : Raw Payment Data flowing through ur application infra
			1) Credit card information passes thru public APIS
			2) fall under PCI DSS Level 1 Complicance
			3) leak of info
		2) Solution : Tokenization
			1) Replaced card details with a safe reference token to pass around freely
			2) How it works?
				1) Customers enter payment info into a secure Stripe SDK
				2) SDK sends payment details to payment vault
				3) vault returns a payment token e.g. pm_abc123
				4) token then passes to backend in confirm call
			3) Risk 2 : trust between inetnal services
				1) Risk is no mention of internal authentication, if one of service is compromised due to attack, it might affect other services as well
				2) Solution : mTLS mutual validate identity method
					1) each service has a unique identity certificate, issued by a trusted internal authority
					2) when one service makes a call, it presents its digital badge
					3) the receiving service validates the identity and check
					4) if valid, call proceeds, otherwise call blocks![[Pasted image 20261020141711.png]]
			4) Risk 3 : Updated Payment Table
				1) payment_info is replaced by payment_method_token : Tokenized reference to the customer's payment method
				2) never store raw payment data
	2) Payment Corectness & Exact Once
		1) Double authorizing or settling payment
			1) a retry may cause a card to be authorized twice
			2) a worker might crash after charging the card but before recording result
			3) settlement job might be re run accidentally causing same payment to be settled multiple times
			4) multiple workers might try to update same record at same time
		2) Strategy 
			1) Idenpotency key
			2) Transaction Outbox
			3) Enforce State Transitions in the Database 状态机
			4) Track settlement batches by a batch ID
			5) ![[Pasted image 20261020145614.png]]
			6) How outbox works?
				1) a Outbox table to decouple DB writes (Trigger by event type and payload)
				2) Ensuring at least once delivery and enabling retries without duplication
	3) Payment with webhook
		1) Why need webhook?
		2) polling for updates is efficient

Completed Architecture
![[Pasted image 20261020150139.png]]
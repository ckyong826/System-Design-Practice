1. Requirement
	1. FR 
		1. Driver location - Driver continuously share GPS location
		2. Nearby location - Riders can see available cars nearby
		3. Request ride - riders submit pickup and destination
		4. Driver matching - System selects a suitable available driver
		5. Trip management - Accept,arrive, start, complete, cancel
		6. Real time tracking - Rider watches the assigned driver appraoch
		7. Payment - Charge the rider when the trip completes
	2. NFR
		1. 99.99% Target
		2. <200MS nearby driver lookup
		3. Matching <2s
		4. GPS freshness update every 3-5 sec
		5. trip consistency : strong consistency
		6. location : Eventual consistency
		7. Scalabilty
2. Estimation
	1. 10M Rides per day
	2. 1M drivers at peak
	3. each driver sends a GPS update every 4 sec
	4. a trip record occupies 1kb
	5. 1M/4sec = 250k update/s
	6. 116 req/s : peak 5x = 580/s
3. API & DB Design
	1. Driver location update
		1. Websocket and update in json
			1. {
				  "type": "LOCATION_UPDATE",
				  "latitude": 3.1579,
				  "longitude": 101.7116,
				  "timestamp": 1793230000,
				  "sequence": 1024
				}
		2. websocket supports frequent bidirectional updates
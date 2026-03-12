# Architecture
## How would you handle conflicts?
### Playbook
1. Diagnose the conflict: 
    1. Disagreement of design principles and patterns
        1. Two options with tradeoffs
        2. Compare tradeoffs with ADR-style evaluation
    2. Priorities
        1. Product vs engineering priorities
        2. use business impact and risk framing
    3. People
        1. Egos, Ownership, Opinions
        2. depersonalize, Support it with evidence, data
2. Shift the conversation from “my design vs your design” to "Shared Guiding principles for problem resolutions"
    1. Mapping: Which design option best satisfies our principles
    2. Comparative analysis with Tradeoff Matrix
        1. Scalability
        2. Complexity
        3. Operation support
        4. Cost & timeline
        5. Alignment with long term architecture
3. Convert debate into ADR
    1. Context, Problem, Options, Tradeoffs
    2. Decision Evaluation
        1. A lot of conflict comes from people thinking on different timeline: Now Vs Future
        2. Do we want to solve all problems: speed, scale and stability now vs later

## How do we know the system is healthy?
### Checklist
- System-Level Indicators
    1. Latency : Spikes, increasing latency under load
    2. Traffic: Spiky pattern, frequent bursts, and outages
    3. Errors: 500/400, timeouts, retries
    4. Capacity Metrics: CPU, memory, I/Os, thread pools
    5. Availability, Throughput(Req/Sec)
- Business-Level Indicators
    1. Broken customer experience
    2. High support volume
    3. Data integrations are out of sync
    4. Notifications 

- Four characteristics:
    1. Customer Exp Impacted
    2. Business workflows are working
    3. Infrastructure is stable
    4. Performance Issues
- A system is healthy when
    1. Predictability: Behaves as expected under load
    2. Degrades gracefully under load
    3. Recovers automatically from failures
    4. Create actionable signals when something goes wrong
    5. Supports business workflows without interruption

## How does the system degrades gracefully?
1. Fail Fast Vs Fail Late
    1. Fail Late: Leaky system, timeouts and cascade failure 
    2. Fail Fast: Detect request early, reject request early and preserve capacity for healthy operations
2. Circuit Breakers:Prevent cascade failure
    1. The circuit opens, Calls stop being sent,
    2. The system returns a fallback response instead of outages
3. Load Shedding: protects core functionality
    1.  intentionally drops low‑priority traffic to protect high‑priority workflows. Write operations are more critical and then Read(GET, GETALL)
    2. Return 503 -Service Unavailable immediately by Gateway to inform client to retry
    3. Apply load shedding policies in API gateway, LB, or in API middleware to reject requests 
3. Generates actionable signals proactively
    1. Dependency health checks
    2. Resource consumptions/metrics Alerts
4. Autoscaling with guardrails
    1. Max scale limits
    2. Cooldown periods
    3. Predictive scaling

## How would you protect PIIs?
1. Replace PIIs with token/Ids wherever possible
1. Encrypt where we persists
    1. Apply Field level encryption
    2. Rotate encryption keys often
2. Protect PII in Transit
    1. TLS: HTTPS, MTLS for B2B
    2. No PII in query strings
3. No PII in logs and other side channels(like Full Story)
    1. Sanitize to drop
    2. Mask or hash

## Tell me about a time a provider sent bad data
### A B2B Bad Data Handling Playbook
#### What Counts as “Bad Data”
1. Contract Failure
    1. Invalid payload type, missing required field,invalid schema/format 
    2. Negatives, duplicates, invalid
2. Fail fast and isolate the bad payload
    1. Reject/deadletter/quarantine to prevent data corruption and downstream failures
3. Communicate with Provider
    1. Payload, correlationid, validation errors
    2. Setup alerts for invalid payloads
4. Decide severity level
    1. Critical: breaking business rules(like incorrect enums or identity)
    2. High: missing require fields
    3. Low: new optional fields
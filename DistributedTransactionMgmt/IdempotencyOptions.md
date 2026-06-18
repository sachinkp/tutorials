Best-effort only, no key storage
Treat POST as deterministic upsert and DELETE as no-op if already deleted.
Keep X-Idempotency-Key for tracing only.
Retries may still double-trigger downstream side effects.
Good when duplicate side effects are acceptable.
Push idempotency to database constraints
Use unique keys on natural business identity, such as dealer plus phoneType, dealer plus emailType, rep plus dealer plus roleType.
For retries, same write just updates same row.
No separate idempotency table needed.
Works well for your resource model, but does not protect non-DB side effects.
Concurrency-only model
Use If-Match and ETag for conflict protection.
Keep PUT strong, make POST and DELETE logically idempotent by route semantics.
Minimal overhead.
This protects state races more than network retry duplicates.
Very short-lived cache instead of durable storage
Use in-memory or Redis with tiny TTL such as 2 to 10 minutes.
Store only hashed key plus request fingerprint.
Lower cost than durable persistence.
Can miss duplicates after restart or failover.
Client-only retry discipline
Client reuses same key for retries and stops retrying after unknown timeout.
Server does no dedupe.
Lowest server overhead.
Highest risk of duplicate processing.
Practical recommendation for your current API:

Use option 2 as baseline, since your endpoints are naturally keyed by type and identity.
Add option 3 for race safety.
Keep option 4 only for the high-risk endpoints that trigger external side effects.
If you choose option 1 or 5, update the spec wording from required for deduplication to required for traceability, because true deduplication is no longer guaranteed.

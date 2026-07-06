# Multi tenant SaaS System


## How to structure any system design answer

Use this 3‑layer structure (keep it conversational):  Github

1. Problem & constraints (30–60s)• Clarify: tenants, scale, latency, data sensitivity, SLAs.
• Ask: “How many tenants?”, “Any high‑security tenants?”, “Read/write ratio?”, “Real‑time vs batch?”

2. High‑level architecture (3–5 min)• Core components:• API gateway / BFF
• Auth & tenant resolution
• App services (user, billing, reporting, etc.)
• Data layer (DB choice + tenancy model)
• Async workers (jobs, reports, emails)
• Observability (logs, metrics, traces)


3. Deep dives & trade‑offs (rest of time)• Tenancy model
• Isolation & security
• Scaling & noisy neighbors
• Failure modes & monitoring

## Isolate Tenant Data
“Design a multi‑tenant platform where each customer’s data is isolated. How do you handle auth, data partitioning, noisy neighbors, and scaling?”

* Option A — Shared DB, shared schema (most common)

• Design:• Single DB (e.g., PostgreSQL).
• Every table has tenant_id.
• Composite keys: (tenant_id, id) where possible.
• All queries include WHERE tenant_id = ?.

• Pros:• Simple to operate, cheaper.
• Easy to onboard new tenants.

• Cons:• Strong discipline needed on query filtering.
• One big DB can become a bottleneck.

• Mitigations:• Code review checklist: “Every query must filter by tenant_id.”  Github
• Automated tests that assert tenant isolation.
• Indexes on (tenant_id, important_field).


* Option B — Separate schema per tenant

• Design:• Same DB instance, but each tenant has its own schema.

• Pros: Better isolation, easier per‑tenant migrations.
• Cons: More complex to manage at scale.


* Option C — Separate DB per tenant

• Design:• Each tenant gets its own database.

• Pros: Maximum isolation; good for high‑risk or regulated tenants.
• Cons: Operational overhead (backups, migrations, monitoring per DB).  gitgood.dev +1

In the interview, pick A as default, mention B/C for special cases, and explain when you’d switch.


## Tenant Resolution and Auth
Explain how the system knows which tenant a request belongs to.

• Entry points:• Subdomain (tenantA.cloudforce.com)
• Request header (X-Tenant-Id)
• JWT claims (tenant_id inside token)

• Good pattern:• API gateway validates JWT.
• Extract tenant_id from token (never trust body/query).  Github +1
• Pass tenant_id to downstream services as part of context.
• Services never infer tenant from user input.

RBAC: roles per tenant (admin, member, read‑only), enforced at service layer.

## Handling noisy neighbors & scaling
What if one tenant is 1000× bigger than others?”  gitgood.dev +1

Key ideas:

• Rate limiting per tenant• At gateway: limit requests per tenant.
• Work queues per tenant• Long‑running jobs (reports, exports) go to queues.
• Use priority or separate queues for large tenants.
• Resource isolation• For very large or sensitive tenants, move them to separate DB or dedicated cluster.
• Sharding• If shared DB gets too big, shard by tenant_id range or hash.

Say explicitly:

“My default is shared DB with strong tenant filtering. For very large or regulated tenants, I’d move them to dedicated DBs or clusters.”


## API design for large data (reports, dashboards)

Cloudforce is about modernizing SaaS—expect reporting/export questions.  adamantcode.com

Patterns to mention:

• Pagination:• Cursor‑based pagination (next_cursor) instead of offset for large datasets.
```
Request
GET /items?cursor=abc123encryptedTokenString==
Response
{
  "data": [ ... 10 items ... ],
  "nextCursor": "abc123encryptedTokenString==" 
}

```
• Async exports:• Client requests export → job queued → user gets link when ready.

• Caching:• Per‑tenant cache (Redis) for common queries.

• Backpressure & retries:• Workers use exponential backoff. a header called ⁠x-ms-retry-after-ms⁠ in response
    * Backpressure is a defensive mechanism where a downstream system (the receiver) tells an upstream system (the sender) to slow down because it cannot handle the current rate of data or requests.


• Idempotent job design (safe to retry).

## Observability & failure modes
Senior/lead interviews care a lot about operational thinking.
Mention:
• Logging:• Every log line includes tenant_id, request_id.
• Metrics:• Per‑tenant latency, error rate, throughput.
• Tracing:• Distributed tracing across gateway → services → DB.
• Alerts:• Alert when one tenant’s traffic or errors spike.

## Final Answer
1. Clarify requirements• “Before I design, I’d like to confirm: number of tenants, data sensitivity, expected scale, and whether we have any high‑security or regulated customers.”

2. Propose architecture• “I’d use an API gateway with JWT‑based auth, resolve tenant_id at the edge, and pass it through to a set of services (user, billing, reporting). For data, I’d start with a shared PostgreSQL database where every table includes tenant_id and queries are strictly filtered by it. Long‑running tasks like reports go to an async worker tier.”

3. Explain tenancy trade‑offs• “Shared DB keeps operations simple and cheap, but requires discipline on query filtering. For very large or regulated tenants, I’d move them to separate databases or clusters to guarantee isolation.”

4. Address noisy neighbors & scaling• “To avoid one tenant impacting others, I’d enforce per‑tenant rate limits at the gateway, use per‑tenant or priority queues for heavy jobs, and monitor per‑tenant metrics so we can detect and isolate issues quickly.”

5. Close with security & observability• “Security-wise, tenant_id comes only from validated tokens, never from client input. Observability-wise, all logs and metrics are tagged with tenant_id and request_id, so we can debug issues per tenant and maintain strong isolation.”

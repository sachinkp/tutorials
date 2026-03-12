# Prompt
As a modernization architect I want to modernize the legacy auction registratiuon system for auction dealer and their representatives. The current legacy system is  power builder + sybase application to be modernized into a web system. We also want to use Salesforce CRM Suggest an architecture based on industry wide best practices

## Target architecture overview
At a high level:

- Channel layer:
    - Web app: React/Angular SPA for dealers & reps
    - API gateway / BFF: Per-channel BFFs (e.g., dealer-portal-bff) for optimized payloads and auth

- Domain services (microservices or well-modularized services):
    - Dealer Service: Dealer master, hierarchies, eligibility
    - Representative Service: Rep profiles, roles, delegation, permissions
    - Auction Registration Service: Registration lifecycle, rules, approvals, capacity
    - Inventory & Vehicle Service: Vehicles, VIN, condition, fees
    - Billing & Contracts Service: Fees, invoices, settlements
    - Notification Service: Email/SMS/push for confirmations, reminders

- Integration & data movement:
    - Salesforce Integration Service:
        - Outbound: publish dealer/rep/registration events to Salesforce
        - Inbound: consume Salesforce events/changes (e.g., account updates, contact changes)
    - Event backbone: Kafka/Event Hub for domain events (DealerCreated, RepAssigned, RegistrationApproved, etc.)
    - ETL/CDC from Sybase: For phased migration and coexistence

- Data stores:
    - Operational DBs: Postgres/SQL Server per service (or schema-per-service in a shared cluster to start)
    - Read models: Denormalized projections for portal queries (CQRS style)
    - Salesforce: System of engagement for CRM, sales, and relationship workflows

- Cross-cutting:
    - Identity & Access Management (OIDC/OAuth2, SSO)
    - API management (rate limiting, versioning, security)
    - Observability (logs, metrics, traces)
    - Feature flags, configuration, secrets

## Domain and bounded contexts
Think in terms of bounded contexts, not just services:
- Dealer Management Context
    - Dealer master, legal entities, locations, risk flags
    - Sync with Salesforce Account objects

- Representative Management Context
    - Reps, roles, delegation, access to auctions
    - Sync with Salesforce Contact (and possibly User or Partner constructs)

- Auction Registration Context
    - Registration lifecycle: draft → submitted → approved → active → cancelled
    - Rules: eligibility, credit, limits, auction capacity
    - Events drive notifications and CRM updates

- Engagement/CRM Context (Salesforce)
    - Opportunities, tasks, cases, campaigns
    - Not the system of record for registration logic, but the system of engagement

This gives you a clean split: core registration logic stays in your platform, while relationship, pipeline, and follow-up live in Salesforce.

## Salesforce integration patterns
**Use Salesforce as a peer system, not a database.**

### Core patterns:

- Event-driven sync (recommended):

    - From auction platform → Salesforce:
        - Publish events like DealerOnboarded, RepAddedToDealer, RegistrationSubmitted, RegistrationApproved
        - Salesforce Integration Service transforms and calls:
            - REST/Bulk APIs to upsert Accounts/Contacts/Custom Objects (e.g., Auction_Registration__c)

- From Salesforce → auction platform:
    - Use Platform Events / Change Data Capture (CDC) for Account/Contact changes
    - Integration service consumes and updates Dealer/Rep services

- API exposure:
    - Expose REST APIs from your platform for:
        - Dealer/rep lookup
        - Registration status

    - Salesforce can call these via Named Credentials / External Services if needed

- Data model alignment:
    - Dealer ↔ Account
    - Representative ↔ Contact
    - Registration ↔ Custom Object (Auction_Registration__c)
    - Maintain a correlation ID (e.g., External_Id__c) for each mapping

## Modernization strategy from PowerBuilder + Sybase
Given this is a critical system, I’d recommend a strangler-fig pattern with clear seams:

1. Front-door replacement first
    - Introduce the new web portal + API gateway in front of the legacy system.
    - Initially, some flows still call into legacy via a Legacy Adapter Service (wrapping PowerBuilder/Sybase via services or DB access).

2. Carve out domains incrementally
    - Start with read-heavy, low-risk domains:
        - Dealer/Rep lookup, basic profile views
    - Then move to new capabilities:
        - New registration flows, self-service features, notifications
    - Finally, replace core registration workflows and decommission corresponding legacy modules.

3. Data strategy
    - Use CDC or scheduled ETL from Sybase into new operational stores during coexistence.
    - Gradually flip system-of-record per domain:
    - e.g., Dealer master moves from Sybase → new Dealer Service → synced to Salesforce.

4. Coexistence
    - For a period, some registrations are still processed in legacy, others in the new system.
    - Use eventing + reconciliation reports to ensure consistency.

5. Technology stack (example, adjust to your org standards)
- Frontend:

    - React/Angular SPA, TypeScript
    - Component library aligned with your design system

- Backend:
    - .NET 8 / Java Spring Boot for services
    - REST + async messaging (Kafka/Event Hub)
    - BFFs for dealer portal and internal ops tools

- Data:
    - Postgres/SQL Server per service
    - Redis for caching hot reads (e.g., dealer/rep profiles)
    - Elasticsearch/OpenSearch for search-heavy use cases

- Salesforce integration:
    - Dedicated Integration Service (e.g., .NET/Java) with:
    - OAuth2 JWT Bearer flow for Salesforce auth
    - Bulk API for high-volume sync
    - Platform Events/CDC for inbound changes

- Platform & ops:
    - Kubernetes for deployment
    - API Gateway / APIM
    - Centralized logging (ELK/EFK), metrics (Prometheus), tracing (OpenTelemetry)

6. Logical architecture sketch (textual)
- Client:  
    - Dealer Portal SPA → Dealer Portal BFF

- Gateway & BFF:  
    - BFF → API Gateway → Domain Services

- Domain Services:
    - Dealer Service
    - Representative Service
    - Auction Registration Service
    - Inventory Service
    - Notification Service
    - Salesforce Integration Service

- Event Backbone:  
    - Domain Services ⇄ Event Bus (Kafka/Event Hub)

- Data:  
    - Each service → its own DB
    - Read models for portal queries

External Systems:
    - Salesforce (CRM)
    - Legacy PowerBuilder/Sybase (via Legacy Adapter during transition)

## High-level principle
- Microservices platform = system of record + business brain for auction registration.
- Salesforce = system of engagement for sales, relationship management, tasks, and visibility.

Salesforce should not own core registration rules, approvals, or eligibility. It should reflect them.

1. Clear ownership boundaries
**Microservices own**
- Dealer & Representative operational model
    - Dealer Service: canonical dealer entity for auction operations (status, eligibility, risk flags, limits).
    - Representative Service: reps, roles, delegation, which dealer(s) they can act for.

- Auction Registration lifecycle
    - Registration Service owns:
        - Creation, validation, eligibility checks
        - Approval workflow (including multi-step approvals)
        - State machine: Draft → Submitted → UnderReview → Approved → Rejected → Cancelled
        - Capacity rules, auction-specific constraints

- Approval workflow
    - Business rules: who can approve, SLAs, escalation logic.
    - Audit trail: who approved, when, under what conditions.
    - Notifications to approvers and applicants.

- Events & integration
    - Emits events like:
        - RegistrationSubmitted
        - RegistrationApproved
        - RegistrationRejected
        - DealerStatusChanged
    - Integration service translates these into Salesforce updates.

**Salesforce owns**
- CRM & engagement
    - Accounts, Contacts, Opportunities, Activities, Cases.
    - Follow-ups, tasks, campaigns, call logs.
- Sales visibility & collaboration
    - Sales reps see:
        - Which dealers are registered for which auctions.
        - Registration status (read-only from Salesforce perspective).
    - They can create tasks, notes, and opportunities based on that.

- Lightweight workflows
    - Optional: Salesforce flows that react to platform events (e.g., RegistrationApproved) to:
        - Create an Opportunity.
        - Create a Task for the account owner.

    - But they do not change the registration state in the auction platform.

2. Where the approval lives
Given your requirement:

“There is also an approval stage involved where authorized associate can approve the registrations before they flow into Salesforce.”

That’s a strong signal that:

- Approval must be completed in the microservices platform before Salesforce is updated.

- Salesforce should only see post-decision data (or at least clearly marked as “Pending Approval” if you choose to expose earlier).

**Recommended pattern**
- User submits registration
    - Dealer/rep uses the web portal → Registration Service.
    - Registration enters Submitted state.

- Approval workflow in platform
    - Registration Service:
        - Determines approver(s) based on rules (region, risk, auction type, etc.).
        - Emits RegistrationSubmitted event.
- Approval UI:
    - Could be:
        - A dedicated internal web app (Ops Portal) backed by the same services.
        - Or an “Approvals” section in the main portal for authorized associates.
    - Approver reviews, then:
        - Approve → state changes to Approved.
        - Reject → state changes to Rejected.
- Post-approval integration to Salesforce
    - When state changes to Approved:
        - RegistrationService emits RegistrationApproved event.
    - Salesforce Integration Service consumes it and:
        - Upserts Account (Dealer) and Contact (Rep) if needed.
        - Upserts a Auction_Registration__c custom object with status Approved.
        - Optionally creates an Opportunity or Task.
    - No direct approval in Salesforce
        - Salesforce users can see the registration and its status.
        - If they want to influence it, they:
            - Use comments/notes/tasks in Salesforce.
            - Or trigger a process that sends a signal back (e.g., “Request Additional Info”), which your platform can interpret—but final approval still happens in the platform.

3. Data model boundary
In the platform (microservices)
- Dealer Service
    - DealerId (internal)
    - ExternalDealerId (legacy, if needed)
    - SalesforceAccountId (optional, once linked)
    - Status, risk flags, limits, etc.

- Representative Service
    - RepId
    - DealerId
    - SalesforceContactId (optional)
    - Roles, permissions.

- Registration Service
    - RegistrationId
    - DealerId, RepId
    - Auction details (auction id, date, type)
    - State: Draft/Submitted/UnderReview/Approved/Rejected/Cancelled
    - Approval metadata: approver, timestamps, comments.

- In Salesforce
    Account
        External_Dealer_Id__c (maps to DealerId)
    Contact
        External_Rep_Id__c (maps to RepId)
    Auction_Registration__c
        External_Registration_Id__c (maps to RegistrationId)
        Status__c (mirrors platform state, but read-only from a process perspective)
        Key fields for sales visibility (auction, date, capacity, etc.)
Ownership rule: platform IDs are primary; Salesforce stores them as external references.

4. Integration flow with approvals in mind
- Outbound (platform → Salesforce)
    - Event source: RegistrationService
    - On RegistrationSubmitted:
        - Optional: send to Salesforce if you want early visibility as “Pending Approval”.
    - On RegistrationApproved:
        - Always send to Salesforce.
    - On RegistrationRejected:
        - Send to Salesforce to update status.

- Integration Service responsibilities:
    - Enforce idempotency (same event processed multiple times safely).
    - Handle mapping and error handling (e.g., Salesforce down → retry, DLQ).
    - Maintain correlation between platform IDs and Salesforce IDs.

- Inbound (Salesforce → platform)
Keep this minimal and controlled:
    - Allowed patterns:
        - Salesforce user flags an Account as “Do Not Register” → emits a Platform Event → Integration Service → updates DealerService (e.g., DealerStatusChanged).
        - Salesforce user requests follow-up or additional info → Platform Event → Notification in platform.

    - Not allowed:
        - Salesforce directly changing registration state (no “Approve in Salesforce” that bypasses platform rules).

5. UI/experience boundaries
- Dealer/Rep-facing portal (platform):
    - Registration creation, editing, submission.
    - Status tracking.
    - Approval outcomes.
- Internal associate portal (platform):
    - Work queue of registrations needing approval.
    - Detailed view with all operational data.
    - Approve/Reject actions.

- Salesforce UI:
    - Read-only view of registrations (status, key fields).
    - CRM actions: tasks, opportunities, notes, campaigns.
    - Optional deep link back to platform UI for detailed operational view.

## What Salesforce Should Not Do
To keep boundaries clean:
1. Salesforce should not approve or reject registrations
2. Salesforce should not own registration state
3. Salesforce should not enforce auction rules
4. Salesforce should not be the system of record for registrations

It is purely a consumer and engagement layer

- Platform (Microservices)
    - Owns registration lifecycle
    - Owns approval workflow
    - Emits events: RegistrationApproved, RegistrationRejected, etc.

- Salesforce Integration Service
    - Listens to events
    - Upserts Salesforce objects
    - Creates tasks/opportunities/cases
    - Triggers flows and automations

- Salesforce
    - Provides visibility
    - Drives CRM actions
    - Supports sales, service, and marketing
    - Enhances dealer engagement

## Sequence diagram (registration submission → approval → Salesforce sync)
```
    Participant Dealer/Rep User
    Participant Dealer Portal (Web App)
    Participant API Gateway / BFF
    Participant Registration Service
    Participant Dealer Service
    Participant Approval UI (Internal)
    Participant Event Bus
    Participant SF Integration Service
    Participant Salesforce

    Dealer/Rep User -> Dealer Portal (Web App): Start registration (auction, dealer, reps)
    Dealer Portal (Web App) -> API Gateway / BFF: POST /registrations (payload)
    API Gateway / BFF -> Registration Service: CreateRegistration(command)
    Registration Service -> Dealer Service: ValidateDealer(dealerId)
    Dealer Service --> Registration Service: DealerValid / DealerInvalid
    Registration Service -> Registration Service: Apply business rules, create Draft
    Registration Service --> API Gateway / BFF: RegistrationCreated(Draft)
    API Gateway / BFF --> Dealer Portal (Web App): 201 + registrationId

    Dealer/Rep User -> Dealer Portal (Web App): Submit registration
    Dealer Portal (Web App) -> API Gateway / BFF: POST /registrations/{id}/submit
    API Gateway / BFF -> Registration Service: SubmitRegistration(command)
    Registration Service -> Registration Service: Transition Draft -> Submitted
    Registration Service -> Event Bus: RegistrationSubmitted(event)
    Registration Service --> API Gateway / BFF: RegistrationSubmitted(response)
    API Gateway / BFF --> Dealer Portal (Web App): 200 + status=Submitted

    Approval UI (Internal) -> Registration Service: GET /registrations?status=Submitted
    Registration Service --> Approval UI (Internal): List of pending registrations

    Approval Associate -> Approval UI (Internal): Review & Approve
    Approval UI (Internal) -> Registration Service: POST /registrations/{id}/approve
    Registration Service -> Registration Service: Validate approver, rules
    Registration Service -> Registration Service: Transition Submitted -> Approved
    Registration Service -> Event Bus: RegistrationApproved(event)
    Registration Service --> Approval UI (Internal): 200 + status=Approved

    SF Integration Service -> Event Bus: Subscribe RegistrationApproved
    Event Bus --> SF Integration Service: RegistrationApproved(event)
    SF Integration Service -> Dealer Service: GET /dealers/{dealerId}
    Dealer Service --> SF Integration Service: Dealer details
    SF Integration Service -> Registration Service: GET /registrations/{id}
    Registration Service --> SF Integration Service: Registration details

    SF Integration Service -> Salesforce: Upsert Account (Dealer)
    Salesforce --> SF Integration Service: AccountId
    SF Integration Service -> Salesforce: Upsert Contact(s) (Reps)
    Salesforce --> SF Integration Service: ContactId(s)
    SF Integration Service -> Salesforce: Upsert Auction_Registration__c
    (External_Registration_Id__c, Status__c=Approved, lookups to Account/Contact)
    Salesforce --> SF Integration Service: Registration record Id

    Salesforce -> Salesforce: Trigger Flows/Automation
    - Notify Account Owner
    - Create Task/Opportunity/Case
    - Add to Campaign/Journey

    Salesforce --> SF Integration Service: 200 OK
    SF Integration Service -> Event Bus: RegistrationSyncedToSalesforce(event) (optional)
    Event Bus --> Registration Service: RegistrationSyncedToSalesforce(event) (optional)
    Registration Service -> Registration Service: Mark CRM sync status (optional)

```
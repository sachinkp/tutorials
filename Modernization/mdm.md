# MDM

## Use cases or pain points in wholesale auction registration:

• Same dealer registered under multiple names
• Same rep with multiple emails/phone numbers
• Dealers with multiple tax IDs or legacy IDs
• Salesforce, legacy systems, and microservices all having different versions of the same entity

Correct pattern:

Microservices → Reltio → Microservices → Salesforce

Flow:

1. Dealer/Rep enters data in the portal
2. Registration Service calls Reltio Match API
3. Reltio returns:• Exact match → use existing DealerId
• Probable match → flag for review
• No match → create new dealer/rep

4. Registration continues with the resolved identity
5. Approved registration flows to Salesforce with the Reltio Golden ID


5. Recommended Integration Pattern

At Registration Submission

Portal → Registration Service → Reltio Match API


Reltio returns:

• matchFound = true/false
• goldenRecordId
• confidenceScore
• survivorshipAttributes
• potentialDuplicates[]


Registration Service logic:

• If exact match → use golden record
• If probable match → route to manual review
• If no match → create new entity in Reltio


After Approval

RegistrationApproved → Event Bus → Salesforce Integration → Salesforce


Salesforce receives:

• Golden Dealer ID
• Golden Rep ID
• Clean, deduped data

1. Dealer Master (Golden Dealer Record)

Reltio should own the authoritative identity for:

• Legal business name
• DBA names
• Tax IDs (EIN)
• Addresses (HQ + branches)
• Phone numbers
• Emails
• Legacy system IDs
• Salesforce Account IDs
• Microservices Dealer IDs
• Reltio Golden ID


Why it belongs in Reltio:

• Dealers appear in multiple systems (legacy, Salesforce, microservices).
• Reltio resolves duplicates and merges identities.
• Reltio provides survivorship rules for “best version of truth”.


---

2. Buyer / Representative Master

Reltio should maintain the golden record for reps:

• Name
• Email(s)
• Phone(s)
• Role
• Dealer association(s)
• Salesforce Contact ID
• Legacy IDs


Why:

• Reps often use multiple emails or phone numbers.
• They may represent multiple dealers.
• Salesforce and legacy systems often have conflicting versions.


---

3. Dealer–Rep Relationship Graph

This is one of Reltio’s superpowers.

Reltio can maintain:

• Rep → Dealer relationships
• Dealer → Parent Dealer relationships
• Dealer → Location relationships
• Rep → Multiple Dealer relationships
• Dealer → Franchise relationships


Why:

• Relationships are cross‑system and change slowly.
• Salesforce is not a graph database.
• Microservices should not own identity relationships.


---

4. Data Quality Scores

Reltio automatically computes:

• Completeness score
• Validity score
• Consistency score
• Duplicate confidence score
• Survivorship score


Why:

• Approval workflows can use DQ score as a signal.
• Credit monitoring can use DQ score as a risk factor.
• Salesforce can display DQ score to account owners.


---

5. Enrichment Attributes

Reltio can store enriched data from:

• Dun & Bradstreet
• Experian
• LexisNexis
• Internal credit systems
• Third‑party risk providers


Examples:

• Business size
• Industry codes
• Risk indicators
• Fraud flags
• Bankruptcy indicators


Why:

• Microservices should not call external enrichment APIs directly.
• Salesforce should not own enrichment logic.
• Reltio is built for enrichment + survivorship.


---

6. Cross‑System Identity Resolution

Reltio should maintain the mapping:

System	ID	
Salesforce	AccountId / ContactId	
Microservices	DealerId / RepId	
Legacy	DealerNumber / RepNumber	
Reltio	GoldenRecordId	


Why:

• Prevents identity drift
• Enables clean integration
• Ensures Salesforce receives the correct identity
• Simplifies migration and modernization


---

7. Reference Data & Standardization

Reltio can standardize:

• Address formats
• Phone formats
• Country/state codes
• Dealer types
• Rep roles
• Risk tiers
• Business categories


Why:

• Microservices should not maintain reference data.
• Salesforce should not be the source of truth for reference data.
• Reltio ensures consistency across systems.


---

8. Identity Change History (Audit Trail)

Reltio keeps:

• Merge history
• Split history
• Attribute lineage
• Source system lineage
• Survivorship decisions


Why:

• Critical for compliance
• Critical for credit decisions
• Critical for auditability
• Microservices should not store identity lineage
• Salesforce cannot store identity lineage


---

9. Identity Events

Reltio can emit events like:

• GoldenRecordCreated
• GoldenRecordUpdated
• PotentialDuplicateDetected
• EntityMerged
• EntitySplit


Why:

• Microservices can react to identity changes
• Salesforce can update Account/Contact records
• Credit monitoring can adjust exposure based on identity merges


---

🔶 Summary: What Should Stay in Reltio

Domain	Should Stay in Reltio?	Why	
Dealer identity	Yes	Golden record, dedupe, survivorship	
Rep identity	Yes	Multi‑email, multi‑dealer relationships	
Dealer–Rep relationships	Yes	Graph relationships	
Data quality scoring	Yes	Approval + credit signals	
Enrichment attributes	Yes	External data integration	
Cross‑system ID mapping	Yes	Prevent identity drift	
Reference data	Yes	Standardization	
Identity lineage	Yes	Compliance + audit	
Registration lifecycle	No	Microservices own	
Approval workflow	No	Microservices own	
Credit monitoring	No	Microservices own	
CRM workflows	No	Salesforce owns	


---

🔷 Final Architectural Position

Reltio = Identity + Golden Record + Data Quality + Enrichment

Microservices = Operational Logic + Approvals + Credit + Registration

Salesforce = Engagement + CRM Workflows + Visibility

This is the cleanest, most scalable, enterprise‑grade boundary.

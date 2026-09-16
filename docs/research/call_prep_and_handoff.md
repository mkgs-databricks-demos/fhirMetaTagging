# PARATUS Phase 2 — Call Prep & BDR Handoff Brief

## The Situation in 30 Seconds

BDR (our SI partner) built a working Databricks prototype for a DHA program called **PARATUS** — a Periodic Health Assessment (DD Form 3024) data pipeline. They passed Phase 1. Phase 2 is a **conformance gate due to the Government this Friday**. BDR needs to demonstrate they can correctly apply FHIR R4 governance tags to the data — and their team doesn't know FHIR. Robyn Bollhorst (Enterprise AE) is asking for a healthcare SME to help them get the tagging right.

## What BDR Needs to Deliver by Friday

| Deliverable | What It Is | Record Count |
|---|---|---|
| **Tier 1 output** | Tag all 305 Government-provided records + generate missing Output elements | 30 core + 275 volume = 305 total |
| **Tier 2 output** | Generate 10+ brand-new PHA records from nothing | >= 10 (separate file, no overlap with Tier 1 IDs) |

**Format**: ndjson — one FHIR R4 Bundle per line, every resource carrying `meta.tag` governance tags.

## The Three Things BDR Must Get Right

### 1. Governance Tag Derivation
- Data ships completely untagged
- 21 tags across 5 categories defined in `tagging-taxonomy.json`
- 34-element schema in `pha-data-dictionary.xlsx` maps each DD 3024 element to FHIR resource/path and governance tags
- Tags go on individual resources using `meta.tag` with system `"pha-governance"`

### 2. Three Conformance Rules
1. AI_GENERATED + IMR_DETERMINANT must have PROVIDER_APPROVED in same Bundle
2. PHI_CLINICAL / PII_SENSITIVE must be on security-label-capable resources
3. HITL_GATED evaluated pass/fail independently

### 3. Instance Discriminator
- RiskAssessment, Communication, ServiceRequest, Provenance appear multiple times per Bundle
- Data dictionary's "Instance Discriminator" column tells how to distinguish them
- ResourceType + FHIR path alone is NOT sufficient

## What BDR Does NOT Need to Build
- No real AI/ML models — conformance-only evaluation
- No IL5 accreditation for Phase 2
- No external FHIR server — everything in BDR's Databricks environment
- All data is synthetic — no real PII/PHI

## Questions to Ask BDR on the Call
1. Have you reviewed pha-data-dictionary.xlsx and tagging-taxonomy.json yet?
2. How are you currently parsing the Bundles?
3. Have you opened a core-set file and looked at the Instance Discriminator cases?
4. What's your plan for the 11 Output elements?
5. Do the untagged input resources already have meta fields?
6. What's the timeline for the live demo?

## What We've Prepared
- Full L100-L300 design spec set (Genie Code reference)
- FHIR R4 tagging & governance reference document (27 linked sources)
- Call prep brief (this document)

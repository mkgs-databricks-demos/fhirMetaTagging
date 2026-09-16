# UC Page Glossary Candidates — FHIR Resource Types in PARATUS

## FHIR R4 Bundle
- **Definition**: A container resource that holds a collection of other FHIR resources. In PARATUS, each Bundle represents one Service member's complete PHA record. Type is "transaction" with an entry array of individual resources.
- **Business Context**: The unit of ingestion (one Bundle = one PHA record) and the unit of output (one ndjson line = one Bundle). Bundle-level tags (e.g., CORE_COVERAGE_SET) go on the Bundle's own `meta.tag`.
- **Data Usage**: `bundle_id` is the primary key for a PHA record throughout the pipeline. `bronze_bundles` has one row per Bundle; `silver_resources` has one row per resource within a Bundle.
- **Related Terms**: ndjson, entry, resourceType
- **Source**: Phase 2 Instructions PDF, Section 5.1

## Instance Discriminator
- **Definition**: A field-based or content-based signal used to distinguish multiple occurrences of the same FHIR resource type within a single Bundle. Defined in the data dictionary's "Instance Discriminator" column.
- **Business Context**: RiskAssessment, Communication, ServiceRequest, and Provenance each appear multiple times per Bundle with identical FHIR paths. The Instance Discriminator tells you which occurrence maps to which data dictionary element (and therefore which governance tags to apply). This is the most common source of tagging errors.
- **Data Usage**: `ref_discriminator_rules` materialized view. Used in the Gold join to match resources to the correct data dictionary row. Exact matching logic finalized after L200-B exploratory analysis.
- **Related Terms**: Data Dictionary, Tag Mapping, resource_type
- **Source**: Phase 2 Instructions PDF, Step 5 paragraph 4

## RiskAssessment
- **Definition**: A FHIR resource representing a clinical risk evaluation. In PARATUS, appears as Part A (intake risk), Part C1 (AI/ML MHA risk score), and Part C2 (composite multi-part risk). A multi-instance resource type requiring Instance Discriminator resolution.
- **Business Context**: The Part C2 RiskAssessment references Part A, B, and C1 assessments via `basis` — this is how the composite risk visualization and the AI-vs-Provider audit trail are reconstructed.
- **Data Usage**: `resource_type = 'RiskAssessment'` in Silver. Distinguished by Instance Discriminator (basis references, prediction presence). `RiskAssessment.basis` is a critical reference chain that must be preserved.
- **Related Terms**: Instance Discriminator, basis, ClinicalImpression
- **Source**: Phase 2 Instructions PDF, Step 4 paragraph 3

## Provenance
- **Definition**: A FHIR resource that describes the entities, agents, and processes involved in producing or changing another resource. In PARATUS, used for the AI/Provider Audit Trail, Authentication Method, and Digital Signature.
- **Business Context**: `Provenance.entity.what` references link AI-generated outputs back to their input basis — this is how the evaluation panel reconstructs the AI-vs-Provider audit trail. A multi-instance resource type.
- **Data Usage**: `resource_type = 'Provenance'` in Silver. Tier 1 has 3 instances per Bundle; Tier 2 has 2 (no Digital Signature). `entity[0].what.reference` is a critical reference chain.
- **Related Terms**: Instance Discriminator, AI/Provider Audit Trail, entity.what
- **Source**: Phase 2 Instructions PDF, Step 4 paragraph 3

## Element Categories (Input, Manually Generated, Output)
- **Definition**: Every element in the data dictionary falls into exactly one of three categories. Input and Manually Generated elements are present in the CDP data and require tagging. Output elements must be generated from scratch by the pipeline.
- **Business Context**: The 11 Output elements are the AI/ML-derived resources that BDR's pipeline must create. Input and Manually Generated elements are already in the Bundles — they just need tags applied.
- **Data Usage**: `source_category` column in `ref_dictionary_elements` and `gold_tagged_resources`. Output elements are generated in L200-F.
- **Related Terms**: Data Dictionary, Output Elements, Tag Derivation
- **Source**: Phase 2 Instructions PDF, Section 4 Overview

## meta.tag
- **Definition**: The FHIR R4 metadata element used for workflow, processing, and categorization tags. In PARATUS, governance tags are applied here (not in `meta.security`). Each tag is a Coding with `system` and `code`.
- **Business Context**: The entire conformance gate evaluates whether `meta.tag` arrays are correctly populated on every resource. The tag structure is `{"system": "pha-governance", "code": "<TAG_CODE>"}`.
- **Data Usage**: Tag injection happens in `gold_tagged_resources` via Photon-native string concatenation. The `tagged_resource_json` column contains the complete resource with `meta.tag` injected.
- **Related Terms**: pha-governance, governance_tags, tag_json_array
- **Source**: Phase 2 Instructions PDF, Section 5.2

### Tagging Strategy
- Priority: HIGH — BDR's team needs to understand these FHIR concepts to implement the pipeline correctly
- Implementation: Create as UC Pages in the `paratus.pha_pipeline` schema. Consider a UC Domain "FHIR Resource Types" to group them.

# UC Page Glossary Candidates — Governance Tags

## pha-governance (Tag System)
- **Definition**: The custom governance tag system used by PARATUS. Tags are applied to individual FHIR resources via `meta.tag` with `system: "pha-governance"`. 21 tags across 5 categories, defined in `tagging-taxonomy.json`.
- **Business Context**: This is NOT the standard FHIR `meta.security` / v3-Confidentiality system. It is a DHA-specific governance taxonomy for the PHA program.
- **Data Usage**: `governance_tags` column in Gold tables. `tag_json_array` column contains the pre-built JSON array for injection into the FHIR resource.
- **Related Terms**: meta.tag, Conformance Rules, Tagging Taxonomy
- **Source**: Phase 2 Instructions PDF, Section 5.2

## AI_GENERATED
- **Definition**: A governance tag indicating the resource was produced by an AI/ML capability, not by a human provider. Applied to all 11 Output elements.
- **Business Context**: Part of the Provenance/Human-in-the-Loop tag category. Triggers Conformance Rule 1: if an AI_GENERATED element reaches an IMR_DETERMINANT, a PROVIDER_APPROVED element must exist in the same Bundle's audit trail.
- **Data Usage**: `governance_tags LIKE '%AI_GENERATED%'` in conformance validation. Pre-computed as `is_ai_provider_chain_valid` boolean.
- **Related Terms**: PROVIDER_APPROVED, IMR_DETERMINANT, ADVISORY_ONLY
- **Source**: Phase 2 Instructions PDF, Step 5

## PROVIDER_APPROVED
- **Definition**: A governance tag indicating a human provider has reviewed and approved the resource. Applied to provider-downstream elements (Tier 1 only — not generated in Tier 2).
- **Business Context**: The counterpart to AI_GENERATED in the audit trail. Required by Conformance Rule 1 when AI_GENERATED + IMR_DETERMINANT are present.
- **Data Usage**: `bundle_has_provider_approved` boolean in `mv_bundle_flags`. Checked per-Bundle, not per-resource.
- **Related Terms**: AI_GENERATED, IMR_DETERMINANT, HITL_GATED
- **Source**: Phase 2 Instructions PDF, Step 5

## HITL_GATED (Human-in-the-Loop Gated)
- **Definition**: A governance tag indicating the element requires human-in-the-loop review before it can be acted upon. Evaluated pass/fail independently of surrounding technical quality (Conformance Rule 3).
- **Business Context**: HITL_GATED elements are the most strictly evaluated — they must be present and correctly tagged regardless of whether other elements have issues.
- **Data Usage**: `bundle_hitl_complete` boolean in `mv_bundle_flags`. Counts HITL_GATED resources per Bundle against expected count from data dictionary.
- **Related Terms**: PROVIDER_APPROVED, AI_GENERATED
- **Source**: Phase 2 Instructions PDF, Step 5 paragraph 3

## PHI_CLINICAL
- **Definition**: A sensitivity governance tag indicating the resource contains Protected Health Information of a clinical nature (diagnoses, lab results, medications, etc.).
- **Business Context**: Conformance Rule 2 requires PHI_CLINICAL elements to be carried on a resource/field capable of holding security labels. In the pipeline, this drives ABAC field-level redaction — element_value is masked for non-clinical users.
- **Data Usage**: `element_sensitivity = 'PHI_CLINICAL'` in `gold_resource_elements`. Drives the ABAC column mask via `USING COLUMNS (element_sensitivity)`.
- **Related Terms**: PII_SENSITIVE, ABAC, element_sensitivity
- **Source**: Phase 2 Instructions PDF, Step 5

## PII_SENSITIVE
- **Definition**: A sensitivity governance tag indicating the resource contains Personally Identifiable Information (names, identifiers, dates of birth, etc.).
- **Business Context**: Same conformance rule and ABAC behavior as PHI_CLINICAL. The distinction is clinical content vs. administrative identity data.
- **Data Usage**: `element_sensitivity = 'PII_SENSITIVE'` in `gold_resource_elements`. Same ABAC mask as PHI_CLINICAL.
- **Related Terms**: PHI_CLINICAL, ABAC, element_sensitivity
- **Source**: Phase 2 Instructions PDF, Step 5

## ADVISORY_ONLY
- **Definition**: A governance tag indicating the resource is advisory — it informs but does not determine an outcome. Applied to AI-generated risk flags, scores, and recommendations that have not yet been provider-approved.
- **Business Context**: Distinguishes AI outputs that are informational from those that drive IMR determinations. An ADVISORY_ONLY resource does not trigger Conformance Rule 1 on its own.
- **Data Usage**: Present in `governance_tags` column. Used in the Genie Agent instructions to explain that advisory resources are AI-generated suggestions, not final determinations.
- **Related Terms**: AI_GENERATED, IMR_DETERMINANT
- **Source**: Phase 2 Instructions PDF, Section 5.2

## IMR_DETERMINANT
- **Definition**: A governance tag indicating the resource contributes to or determines the Individual Medical Readiness classification. When combined with AI_GENERATED, triggers Conformance Rule 1.
- **Business Context**: The critical tag for the AI-to-Provider approval chain. An AI_GENERATED + IMR_DETERMINANT resource without a corresponding PROVIDER_APPROVED resource in the same Bundle is a conformance failure.
- **Data Usage**: `governance_tags LIKE '%IMR_DETERMINANT%'` in the `is_ai_provider_chain_valid` boolean computation.
- **Related Terms**: IMR, AI_GENERATED, PROVIDER_APPROVED
- **Source**: Phase 2 Instructions PDF, Step 5 paragraph 3

### Tagging Strategy
- Priority: CRITICAL — these are the tags the pipeline assigns and the conformance gate validates
- Implementation: Create as UC Pages in the `paratus.pha_pipeline` schema. Consider creating a UC Domain "PHA Governance Tags" to group them.

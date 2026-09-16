# FHIR R4 Tagging & Governance Requirements — DHA PARATUS Reference

## Context
BDR (SI partner) built a Databricks prototype called PARATUS for a DHA CSO phased downselect. Phase 2 requires a video demo showing the prototype can ingest a Data Contract Package, transform and governance-tag the data, and produce the output extract — all in accordance with FHIR R4 profile constraints.

## IMPORTANT: PARATUS Uses Custom pha-governance Tags, NOT Standard FHIR Security Labels

The Phase 2 Instructions PDF reveals that the PARATUS governance tagging scheme is a custom `pha-governance` tag taxonomy using `meta.tag` — NOT the standard `meta.security` / v3-Confidentiality system. The generic FHIR R4 reference material below is useful background, but the PARATUS-specific requirements in the L100-L300 design specs are what BDR actually needs to implement.

## FHIR R4 Resource-Level Tagging Mechanisms

| Element | Purpose | Cardinality | Type |
|---|---|---|---|
| `meta.profile` | Declares which profiles the resource conforms to | `0..*` | `canonical(StructureDefinition)` |
| `meta.security` | Security/privacy labels controlling handling or access | `0..*` | `Coding` |
| `meta.tag` | Workflow, processing, categorization, or business tags | `0..*` | `Coding` |

Key distinction: `meta.security` labels connect the resource to security policy. `meta.tag` values are for workflow and operational classification — applications are explicitly "not required to consider the tags when interpreting the meaning of the resource."

## v3-Confidentiality Code System (Standard FHIR — NOT used by PARATUS)

| Code | Display | Meaning |
|---|---|---|
| U | Unrestricted | No confidentiality protection required |
| L | Low | Altered or de-identified data |
| M | Moderate | Moderate harm risk |
| N | Normal | Standard protection for healthcare information |
| R | Restricted | Potentially stigmatizing information |
| V | Very Restricted | Highest protection |

Code system URI: `http://terminology.hl7.org/CodeSystem/v3-Confidentiality`

## PARATUS-Specific: pha-governance Tag Structure

Tags go on individual resources using `meta.tag` with system `"pha-governance"`:

```json
"meta": {
  "tag": [
    { "system": "pha-governance", "code": "PART_C1" },
    { "system": "pha-governance", "code": "AI_GENERATED" },
    { "system": "pha-governance", "code": "ADVISORY_ONLY" }
  ]
}
```

21 tags across 5 categories (defined in tagging-taxonomy.json):
- Source Part (PART_A, PART_B, PART_C1, PART_C2)
- Sensitivity (PHI_CLINICAL, PII_SENSITIVE)
- Provenance / Human-in-the-Loop (AI_GENERATED, PROVIDER_APPROVED, HITL_GATED)
- Determinant / Outcome (IMR_DETERMINANT, ADVISORY_ONLY)
- Operational / System (CORE_COVERAGE_SET)

## Sources Referenced

### HL7 FHIR R4 Core Specifications (v4.0.1)
- Resource Definitions: https://hl7.org/fhir/R4/resource.html
- Security Labels: https://hl7.org/fhir/R4/security-labels.html
- Profiling: https://hl7.org/fhir/R4/profiling.html
- Validation: https://hl7.org/fhir/R4/validation.html
- Provenance: https://hl7.org/fhir/R4/provenance.html
- Bundle: https://hl7.org/fhir/R4/bundle.html

### HL7 Terminology (THO)
- v3-Confidentiality CodeSystem: https://terminology.hl7.org/7.3.0/en/CodeSystem-v3-Confidentiality.html

### US Core Implementation Guide v9.0.0
- Patient Profile: https://www.hl7.org/fhir/us/core/STU9/StructureDefinition-us-core-patient.html
- Observation Profile: https://www.hl7.org/fhir/us/core/StructureDefinition-us-core-simple-observation.html
- Condition Profiles: https://www.hl7.org/fhir/us/core/StructureDefinition-us-core-condition-problems-health-concerns.html

### Defense Health Agency (DHA)
- PEO DHMS Publications: https://www.health.mil/Military-Health-Topics/Technology/PEO-DHMS?type=Publications
- MHS GENESIS Data Guide: https://www.health.mil/Reference-Center/Publications/2023/08/01/MHS-GENESIS-Data-Guide
- DHA Data Sharing Agreements: https://www.health.mil/Military-Health-Topics/Privacy-and-Civil-Liberties/Data-Sharing-Agreements

### PARATUS Phase 2 CDP
- Phase 2 Instructions.pdf — provided by Robyn Bollhorst via Slack DM, 2026-09-15

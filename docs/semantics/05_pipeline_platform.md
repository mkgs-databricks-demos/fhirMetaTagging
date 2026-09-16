# UC Page Glossary Candidates — Pipeline and Platform

## Conformance Rules (Three Rules)
- **Definition**: Three validation rules defined in `tagging-taxonomy.json` that are checked during conformance validation. (1) AI_GENERATED + IMR_DETERMINANT must have PROVIDER_APPROVED in same Bundle. (2) PHI_CLINICAL/PII_SENSITIVE must be on security-label-capable resources. (3) HITL_GATED elements evaluated pass/fail independently.
- **Business Context**: These are the specific checks the Government runs against the output extract. The pipeline enforces them as pre-computed boolean expectations on the Gold streaming tables.
- **Data Usage**: `is_ai_provider_chain_valid`, `is_sensitivity_placement_valid`, `is_hitl_complete` boolean columns on `gold_validated_resources`. Pipeline expectations EXPECT these booleans.
- **Related Terms**: AI_GENERATED, PROVIDER_APPROVED, HITL_GATED, PHI_CLINICAL
- **Source**: Phase 2 Instructions PDF, Step 5 paragraph 3

## element_sensitivity
- **Definition**: A derived column on `gold_resource_elements` that classifies each FHIR element's sensitivity level: PHI_CLINICAL, PII_SENSITIVE, or NONE. Derived from the data dictionary's Governance Tag(s) column — not a separate classification.
- **Business Context**: Drives the ABAC column mask on `element_value`. The Genie Agent sees `[REDACTED]` for sensitive elements and full values for non-sensitive elements, based on this column.
- **Data Usage**: `element_sensitivity` column in `gold_resource_elements`. Passed to the masking function via `USING COLUMNS (element_sensitivity)`.
- **Related Terms**: PHI_CLINICAL, PII_SENSITIVE, ABAC, gold_resource_elements
- **Source**: Derived from pha-data-dictionary.xlsx Governance Tag(s) column

## SCD1 / SCD2 (Slowly Changing Dimensions)
- **Definition**: SCD Type 1 overwrites the current value (no history). SCD Type 2 preserves historical versions with `__START_AT` and `__END_AT` timestamps. Both are implemented via AUTO CDC in the SDP pipeline.
- **Business Context**: Reference tables (tag mapping, taxonomy) use SCD1 for the current state the pipeline joins against, and SCD2 for audit history. When DHA updates the taxonomy, the pipeline picks up changes automatically with full version history.
- **Data Usage**: `ref_tag_mapping_current` (SCD1) is what Gold joins against. `ref_tag_mapping_history` (SCD2) enables point-in-time reproducibility: "What tags would this resource have received under the taxonomy version active on date X?"
- **Related Terms**: AUTO CDC, ref_tag_mapping_current, ref_tag_mapping_history
- **Source**: Pipeline design decision — L100 Cross-Cutting Pattern 7

## ABAC (Attribute-Based Access Control)
- **Definition**: Unity Catalog's GA mechanism for enforcing row-filter and column-mask policies based on governed tags. In PARATUS, used for field-level redaction on the Genie Agent's element-level Gold table.
- **Business Context**: The same governance tags that drive conformance validation also drive access control. The Genie Agent queries one table; UC masks sensitive element values transparently per user group. No separate redacted view needed.
- **Data Usage**: Governed tag `paratus.pha_pipeline.sensitivity` on the `element_value` column. Masking function `mask_element_value` uses `USING COLUMNS (element_sensitivity)` for per-row decisions.
- **Related Terms**: element_sensitivity, PHI_CLINICAL, PII_SENSITIVE, Genie Agent
- **Source**: Databricks ABAC GA (April 2026) — pipeline design decision L100 Cross-Cutting Pattern 6

### Tagging Strategy
- Priority: MEDIUM — important for understanding the pipeline architecture, less critical for the Friday demo
- Implementation: Create as UC Pages in the `paratus.pha_pipeline` schema

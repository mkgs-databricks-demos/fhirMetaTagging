# PARATUS PHA Pipeline — Genie Code Reference Set

## What This Is
Design specs, glossary, diagrams, and reference materials for the PARATUS PHA (Periodic Health Assessment / DD Form 3024) governance tagging pipeline on Databricks. Created for BDR's Phase 2 CDP Conformance Gate demo (due Friday).

## Document Inventory (31 files, ~168 KB)

### docs/design/
- **L100_L200_L300_design_specs_v2.md** — The complete system design: L100 (system overview / constitution), L200-A through L200-L (per-component designs), L300 (implementation specs). This is the primary Genie Code reference.

### docs/research/
- **fhir_r4_tagging_reference.md** — FHIR R4 tagging & governance requirements research with linked sources.
- **call_prep_and_handoff.md** — BDR call prep brief: what BDR built, what they need now, questions to ask.

### docs/semantics/ (UC Page Glossary Candidates — 26 terms across 5 subdomains)
- **01_dha_pha_domain.md** — DD Form 3024, PHA, IMR, MHA, Component Type, Unit/Command Hierarchy (6 terms)
- **02_cdp_conformance.md** — CDP, Conformance Gate, Tier 1, Tier 2, Core Coverage Set, Volume Set, ndjson (7 terms)
- **03_governance_tags.md** — pha-governance, AI_GENERATED, PROVIDER_APPROVED, HITL_GATED, PHI_CLINICAL, PII_SENSITIVE, ADVISORY_ONLY, IMR_DETERMINANT (8 terms)
- **04_fhir_resource_types.md** — FHIR Bundle, Instance Discriminator, RiskAssessment, Provenance, Element Categories, meta.tag (6 terms)
- **05_pipeline_platform.md** — Conformance Rules, element_sensitivity, SCD1/SCD2, ABAC (4 terms)

### docs/diagrams/ (10 Mermaid + 10 SVG)
- **01_pipeline_dag.md** — End-to-end pipeline DAG (Bronze → Silver → Gold → Output) — *embed in L100 Architecture*
- **02_deploy_sequence.md** — Two-bundle deployment sequence diagram — *embed in L300 Execution Order*
- **03_tag_derivation_flow.md** — Tag derivation join logic (Silver → Gold) — *embed in L200-D*
- **04_abac_redaction_model.md** — ABAC field-level redaction model — *embed in L200-E, L200-I*
- **05_dd3024_parts_flow.md** — DD Form 3024 PHA workflow (Parts A → C2) — *embed in L100 Domain Context*
- **06_conformance_rules_decision.md** — Conformance rules decision tree — *embed in L200-D, L200-L*
- **07_two_bundle_architecture.md** — Two-bundle DAB architecture — *embed in L100 DAB Bundle Structure*
- **08_scd_reference_flow.md** — SCD1 + SCD2 reference data flow — *embed in L200-A*
- **09_fhir_bundle_structure.md/.svg** — FHIR R4 Bundle structure (SVG) — *embed in L200-C, FHIR reference*
- **10_gold_table_split.md/.svg** — Two Gold tables and their consumers (SVG) — *embed in L100 Pattern 8, L200-D/E*

### fixtures/
- **output_element_templates.json** — 11 Output element definitions (placeholder — finalize after L200-B)
- **tier2_coverage_requirements.json** — Tier 2 coverage requirements from PDF page 9

## How to Use with Genie Code
1. Start with `docs/design/L100_L200_L300_design_specs_v2.md` — this is the constitution
2. Review `docs/diagrams/` for visual context — each diagram notes where it should be embedded
3. Follow the Execution Order in L300 (two-bundle deploy)
4. Run L200-B (exploratory analysis) FIRST before building the pipeline
5. Use `docs/semantics/` to create UC Pages for the Genie Agent's domain context
6. The design specs contain all SQL for every pipeline stage

## Key Design Decisions
- VARIANT + parse_json() everywhere (no struct schemas)
- Spark Declarative Pipelines (SDP) — fully streaming, Photon-native
- Auto Loader via cloud_files() with _metadata lineage
- Metadata-driven — zero hardcoded values
- Pre-computed booleans for expectations (no subqueries)
- ABAC for field-level redaction (GA since April 2026)
- SCD1 + SCD2 for reference data via AUTO CDC
- Two Gold tables: resource-level (production) + element-level (Genie Agent)
- Two-bundle DAB deployment: infrastructure first, experience layer second
- Schema + volumes as bundle resources
- Dashboard + Genie Agent serialized inline (no file_path, no parent_path)

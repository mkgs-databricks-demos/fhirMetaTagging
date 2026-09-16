# PARATUS PHA Pipeline — L100-L200-L300 Design Specs v2

## Genie Code Reference Set for BDR Phase 2 CDP Conformance Gate

**Version**: 2.0 — Complete rewrite incorporating all design decisions from 2026-09-16 session.
**Audience**: Genie Code (interactive build sessions), BDR engineering team, Databricks SA-FDE.
**Source of truth**: Phase 2 Instructions PDF (CDP Conformance Gate), supplemented by FHIR R4 research.

---

# L100: System Overview

### Purpose

This is the system-level constitution for the PARATUS PHA (Periodic Health Assessment, DD Form 3024) data pipeline on Databricks. It defines a fully streaming, metadata-driven, Photon-native SDP pipeline that ingests FHIR R4 Bundles, derives governance tags from reference data, enforces conformance rules inline, and produces both a production-ready streaming output and a government-mandated ndjson conformance file.

### What This Pipeline Must Deliver by Friday

| Deliverable | Format | Record Count |
|---|---|---|
| Tier 1 output | ndjson — one tagged FHIR R4 Bundle per line | ALL 305 (30 core + 275 volume) |
| Tier 2 output | ndjson — separate file | >= 10 new self-originated records |
| Live demo | Narrated walkthrough of >= 1 core record end-to-end | During Phase 2 demonstration |

Evaluation is **conformance-only**: structure + correct governance tagging. Clinical or analytical quality of Output elements is NOT evaluated.

### Architecture

```
REFERENCE PATH (metadata-driven rules engine):
  cloud_files(reference/*.json) -> bronze_taxonomy (ST) -> ref_taxonomy (SCD1 + SCD2)
  cloud_files(reference/*.xlsx) -> bronze_dictionary (ST) -> ref_tag_mapping (SCD1 + SCD2)
                                                            -> ref_dictionary_elements (MV)
                                                            -> ref_discriminator_rules (MV)

CLINICAL DATA PATH (streaming FHIR ingestion + tagging):
  cloud_files(core/) -> bronze_core (ST)  -+
  cloud_files(volume/) -> bronze_vol (ST) -+
                                           |
  bronze_bundles (ST) -> silver_resources (ST)
    -> gold_tagged_resources (ST) <- joins ref_tag_mapping (SCD1)
      -> gold_validated_resources (ST) <- joins mv_bundle_flags
        +-- -> Lakebase, Kafka, LTAP (production consumers)
        +-- -> gold_resource_elements (ST) <- joins ref_dictionary_elements
        |       +-- Genie Agent (ABAC masks element_value per sensitivity)
        |       +-- AI/BI Dashboard (mv_* materialized views)
        +-- -> output_bundles (MV — collect_list) -> ndjson file (government only)

ST = Streaming Table, MV = Materialized View
```

### Cross-Cutting Patterns

#### 1. VARIANT + parse_json() Everywhere
All FHIR data stored as VARIANT. No struct schemas, no spark.read.json() with schema inference. VARIANT preserves the full JSON tree, handles polymorphic FHIR fields, avoids schema evolution problems.

#### 2. Spark Declarative Pipelines (SDP) — Always Streaming
Every stage is a streaming table except: reference materialized views (loaded once, refreshed on change) and the final Bundle reassembly (collect_list requires batch aggregation). SDP manages checkpoints automatically.

#### 3. Auto Loader via cloud_files()
All file ingestion uses cloud_files() for idempotent, incremental processing with _metadata lineage capture.

#### 4. Metadata-Driven — Zero Hardcoded Values
The pipeline SQL is a fixed engine. The reference files are the program it runs. Tag assignments, discriminator rules, conformance checks, sensitivity classifications, and ABAC policies are all derived from tagging-taxonomy.json and pha-data-dictionary.xlsx. No hardcoded IN lists, no magic strings.

#### 5. Pre-Computed Booleans for Expectations
All conformance validation resolves as boolean columns in the SELECT (via joins to reference MVs), then expectations are trivial EXPECT(column_name). No subqueries in expectations — guaranteed to work on every runtime.

#### 6. ABAC for Field-Level Redaction (GA since April 2026)
UC governed tags + ABAC column-mask policies enforce field-level redaction on the Genie Agent's element-level Gold table. The Genie Agent queries one table; UC masks sensitive element values transparently per user group. The masking function uses USING COLUMNS to read element_sensitivity from the same row.

#### 7. SCD1 + SCD2 for Reference Data
Reference tables use AUTO CDC — SCD1 for current state (what the pipeline joins against), SCD2 for audit history (point-in-time reproducibility). When DHA updates the taxonomy or dictionary, drop new files in the Volume; the pipeline picks up changes automatically.

#### 8. Two Gold Tables — Different Consumers
- **gold_validated_resources**: Resource-level, production-grade terminal output. Full tagged FHIR resources ready for Lakebase, Kafka, LTAP, ndjson.
- **gold_resource_elements**: Element-level, Genie Agent optimized. Each FHIR field is a row with its own sensitivity classification. ABAC masks element_value per row. Non-sensitive structural fields (dates, terminology codes, status) always visible.

### Component Inventory

| Component | L200 | Type | Purpose |
|---|---|---|---|
| Reference File Ingestion | L200-A | Streaming + SCD | Ingest taxonomy JSON + data dictionary XLSX via cloud_files, feed SCD1 and SCD2 |
| Exploratory Analysis | L200-B | Notebook (one-time) | Validate assumptions against actual reference files before building pipeline |
| Clinical Data Ingestion | L200-C | Streaming | cloud_files -> Bronze -> Silver (exploded resources) |
| Tag Derivation Engine | L200-D | Streaming | Silver -> Gold: metadata-driven tag assignment + conformance booleans |
| Element-Level Gold | L200-E | Streaming | Gold -> element-level explode for Genie Agent with per-field sensitivity |
| Output Element Generation | L200-F | Streaming | Generate 11 AI-ML-derived Output elements as tagged FHIR resources |
| Bundle Reassembly | L200-G | Materialized View | collect_list -> ndjson Bundle packaging (government file only) |
| Tier 2 Record Factory | L200-H | Table | Generate >= 10 new PHA records from nothing |
| ABAC + Governance Setup | L200-I | Setup notebook | Governed tags, masking functions, ABAC policies |
| Genie Agent | L200-J | DAB resource | serialized_space inline in YAML (Bundle 2), queries gold_resource_elements |
| AI-BI Dashboard | L200-K | DAB resource | serialized_dashboard inline in YAML (Bundle 2) |
| Conformance Validation MVs | L200-L | Materialized Views | Pre-computed MVs powering dashboard + self-check |

### DAB Bundle Structure — Two-Bundle Deployment

The dashboard and Genie Agent require the tables they reference to exist in the schema before deployment. This mandates a **two-bundle approach**: Bundle 1 creates the infrastructure (schema, volume, pipeline, tables), Bundle 2 registers the experience layer (dashboard, Genie Agent) against those tables.

**Deploy order**: `dab deploy -t dev` on Bundle 1 -> run pipeline -> verify tables exist -> `dab deploy -t dev` on Bundle 2.

**Bundle 1: paratus-pha-pipeline (Infrastructure)**

```
paratus-pha-pipeline/
+-- databricks.yml
+-- resources/
|   +-- schema.yml                <- UC schema resource
|   +-- volumes.yml               <- UC volume resources (landing + output)
|   +-- pipeline.yml              <- SDP pipeline definition
+-- src/
|   +-- pipeline/
|   |   +-- 00_reference.sql      <- L200-A
|   |   +-- 01_bronze.sql         <- L200-C
|   |   +-- 02_silver.sql         <- L200-C
|   |   +-- 03_gold_tagging.sql   <- L200-D
|   |   +-- 04_gold_elements.sql  <- L200-E
|   |   +-- 05_output_gen.sql     <- L200-F
|   |   +-- 06_bundles.sql        <- L200-G
|   |   +-- 07_sinks.py           <- Kafka sink (Python — sinks are Python-only)
|   |   +-- 08_validation.sql     <- L200-L
|   +-- notebooks/
|       +-- 00_explore.py         <- L200-B: exploratory analysis
|       +-- 01_abac_setup.sql     <- L200-I
|       +-- 02_ndjson_write.py    <- government file write
+-- fixtures/
    +-- output_element_templates.json
    +-- tier2_coverage_requirements.json
```

**Bundle 2: paratus-pha-experience (Dashboard + Genie Agent)**

```
paratus-pha-experience/
+-- databricks.yml
+-- resources/
    +-- dashboard.yml             <- serialized_dashboard inline
    +-- genie.yml                 <- serialized_space inline
```

**Bundle 1 resources/schema.yml:**

```yaml
resources:
  schemas:
    pha_pipeline:
      catalog_name: ${var.catalog}
      name: pha_pipeline
      comment: "PARATUS PHA governance tagging pipeline"
      grants:
        - principal: account users
          privileges:
            - USE_SCHEMA
```

**Bundle 1 resources/volumes.yml:**

```yaml
resources:
  volumes:
    landing:
      catalog_name: ${var.catalog}
      schema_name: ${resources.schemas.pha_pipeline.name}
      name: landing
      volume_type: MANAGED
      comment: "CDP file landing zone — Auto Loader source"

    output:
      catalog_name: ${var.catalog}
      schema_name: ${resources.schemas.pha_pipeline.name}
      name: output
      volume_type: MANAGED
      comment: "ndjson output extracts for government submission"
```

---

# L200-A: Reference File Ingestion + SCD

### Overview
Ingest tagging-taxonomy.json and pha-data-dictionary.xlsx through the same Auto Loader pipeline as clinical data. These are the program the pipeline runs.

### Taxonomy (JSON -> VARIANT -> SCD1 and SCD2)

```sql
CREATE OR REFRESH STREAMING TABLE bronze_taxonomy
COMMENT 'Raw taxonomy JSON from landing zone'
AS SELECT
  _metadata,
  _metadata.file_name AS source_file,
  _metadata.file_modification_time AS version_timestamp,
  parse_json(value) AS taxonomy
FROM cloud_files(
  '/Volumes/paratus/pha_pipeline/landing/reference/',
  'text',
  map('pathGlobFilter', '*.json', 'wholetext', 'true')
);

CREATE OR REFRESH STREAMING TABLE ref_taxonomy_staged
COMMENT 'Individual tag definitions exploded from taxonomy JSON'
AS SELECT
  source_file,
  version_timestamp,
  variant_get(tag.value, '$.code', 'STRING') AS tag_code,
  variant_get(tag.value, '$.category', 'STRING') AS tag_category,
  variant_get(tag.value, '$.display', 'STRING') AS tag_display,
  variant_get(tag.value, '$.definition', 'STRING') AS tag_definition
FROM STREAM(bronze_taxonomy)
LATERAL VIEW variant_explode(taxonomy) tag;

CREATE OR REFRESH STREAMING TABLE ref_taxonomy_current;
CREATE FLOW ref_taxonomy_scd1
AS AUTO CDC INTO ref_taxonomy_current
FROM STREAM(ref_taxonomy_staged)
KEYS (tag_code) SEQUENCE BY version_timestamp
STORED AS SCD TYPE 1;

CREATE OR REFRESH STREAMING TABLE ref_taxonomy_history;
CREATE FLOW ref_taxonomy_scd2
AS AUTO CDC INTO ref_taxonomy_history
FROM STREAM(ref_taxonomy_staged)
KEYS (tag_code) SEQUENCE BY version_timestamp
STORED AS SCD TYPE 2;
```

### Data Dictionary (XLSX -> structured rows -> SCD1 and SCD2)

```sql
CREATE OR REFRESH STREAMING TABLE bronze_data_dictionary
COMMENT 'Raw data dictionary from XLSX — 34 elements'
AS SELECT
  _metadata,
  _metadata.file_name AS source_file,
  _metadata.file_modification_time AS version_timestamp,
  *
FROM cloud_files(
  '/Volumes/paratus/pha_pipeline/landing/reference/',
  'excel',
  map('pathGlobFilter', '*.xlsx', 'headerRows', '1')
);

CREATE OR REFRESH STREAMING TABLE ref_tag_mapping_current;
CREATE FLOW ref_tag_mapping_scd1
AS AUTO CDC INTO ref_tag_mapping_current
FROM STREAM(bronze_data_dictionary)
KEYS (element_id) SEQUENCE BY version_timestamp
STORED AS SCD TYPE 1;

CREATE OR REFRESH STREAMING TABLE ref_tag_mapping_history;
CREATE FLOW ref_tag_mapping_scd2
AS AUTO CDC INTO ref_tag_mapping_history
FROM STREAM(bronze_data_dictionary)
KEYS (element_id) SEQUENCE BY version_timestamp
STORED AS SCD TYPE 2;
```

### Derived Reference Views

```sql
CREATE OR REFRESH MATERIALIZED VIEW ref_dictionary_elements
COMMENT 'Element-level FHIR path mappings with sensitivity — drives Gold tables'
AS SELECT
  element_id,
  dd_3024_part,
  fhir_r4_resource AS resource_type,
  fhir_path_field AS fhir_path,
  concat('$.', regexp_replace(fhir_path_field, '^[A-Za-z]+\\.', '')) AS variant_path_expression,
  data_type AS element_data_type,
  cardinality,
  governance_tags,
  source_category,
  instance_discriminator,
  CASE
    WHEN governance_tags LIKE '%PHI_CLINICAL%' THEN 'PHI_CLINICAL'
    WHEN governance_tags LIKE '%PII_SENSITIVE%' THEN 'PII_SENSITIVE'
    ELSE 'NONE'
  END AS element_sensitivity,
  concat('[', array_join(
    transform(split(governance_tags, ','), t -> concat('{"system":"pha-governance","code":"', trim(t), '"}')),
    ','
  ), ']') AS tag_json_array
FROM ref_tag_mapping_current;

CREATE OR REFRESH MATERIALIZED VIEW ref_discriminator_rules
COMMENT 'Instance Discriminator rules — for multi-instance resource types'
AS SELECT element_id, resource_type, instance_discriminator AS discriminator_raw
FROM ref_tag_mapping_current
WHERE instance_discriminator IS NOT NULL AND instance_discriminator != '';

CREATE OR REFRESH MATERIALIZED VIEW ref_sensitivity_capable_resources
COMMENT 'Derived from data dictionary — not hardcoded'
AS SELECT DISTINCT resource_type
FROM ref_tag_mapping_current
WHERE governance_tags LIKE '%PHI_CLINICAL%' OR governance_tags LIKE '%PII_SENSITIVE%';
```

---

# L200-B: Exploratory Analysis of Reference Files

### Overview
**This runs FIRST, before building the rest of the pipeline.** A one-time notebook that ingests the reference files BDR provides and validates every assumption in this spec.

### What to Validate

```python
# 00_explore.py — Run interactively in Genie Code session

# STEP 1: Load and inspect tagging-taxonomy.json
# Q1: How many tags total? (Expected: 21)
# Q2: What are the 5 categories?
# Q3: What are ALL tag codes?
# Q4: Are there sensitivity codes beyond PHI_CLINICAL and PII_SENSITIVE?
# Q5: What are the three conformance rules — structured data or prose?
# Q6: What is the JSON structure? (array of objects? nested by category?)

# STEP 2: Load and inspect pha-data-dictionary.xlsx
# Q7: What are the exact column names?
# Q8: How many rows? (Expected: 34)
# Q9: Distinct values in Source column? (Expected: Input, Manually Generated, Output)
# Q10: Which elements are Output? (The 11 we must generate)
# Q11: Which resource types appear in multiple rows? (Need Instance Discriminator)
# Q12: What does Instance Discriminator column actually contain?
# Q13: Which elements have PHI_CLINICAL or PII_SENSITIVE?
# Q14: Can FHIR Path/Field values be converted to variant_get paths?

# STEP 3: Load and inspect a sample FHIR Bundle
# Q15: How many entry resources in one Bundle?
# Q16: What resource types are present?
# Q17: Do resources already have a meta field?
# Q18: Do resources already have meta.tag? (Should be empty)
# Q19: For multi-instance types: what fields distinguish instances?
# Q20: What does Organization.partOf look like?
# Q21: What does RiskAssessment.basis look like?

# STEP 4: Cross-validate dictionary against actual data
# Q22: Every resource_type in dictionary exists in sample Bundle?
# Q23: Every FHIR Path resolves to non-null in sample?
# Q24: Instance Discriminator signals actually distinguish instances?
# Q25: The 11 Output elements are truly absent from input?

# STEP 5: Document findings — update fixtures/ and pipeline SQL if needed
```

---

# L200-C: Clinical Data Ingestion (Bronze -> Silver)

```sql
CREATE OR REFRESH STREAMING TABLE bronze_core_bundles (
  CONSTRAINT valid_json EXPECT (bundle IS NOT NULL) ON VIOLATION DROP ROW
)
AS SELECT _metadata, _metadata.file_name AS source_file,
  parse_json(value) AS bundle, 'core' AS dataset_tier, current_timestamp() AS ingested_at
FROM cloud_files('/Volumes/paratus/pha_pipeline/landing/core/', 'text', map('wholetext', 'true'));

CREATE OR REFRESH STREAMING TABLE bronze_volume_bundles (
  CONSTRAINT valid_json EXPECT (bundle IS NOT NULL) ON VIOLATION DROP ROW
)
AS SELECT _metadata, _metadata.file_name AS source_file,
  parse_json(value) AS bundle, 'volume' AS dataset_tier, current_timestamp() AS ingested_at
FROM cloud_files('/Volumes/paratus/pha_pipeline/landing/volume/', 'text');

CREATE OR REFRESH STREAMING TABLE bronze_bundles (
  CONSTRAINT has_bundle_id EXPECT (bundle_id IS NOT NULL) ON VIOLATION DROP ROW,
  CONSTRAINT is_transaction EXPECT (bundle_type = 'transaction')
)
AS SELECT variant_get(bundle, '$.id', 'STRING') AS bundle_id,
  variant_get(bundle, '$.type', 'STRING') AS bundle_type,
  bundle, dataset_tier, source_file, _metadata, ingested_at
FROM (SELECT * FROM STREAM(bronze_core_bundles) UNION ALL SELECT * FROM STREAM(bronze_volume_bundles));

CREATE OR REFRESH STREAMING TABLE silver_resources (
  CONSTRAINT has_resource_type EXPECT (resource_type IS NOT NULL) ON VIOLATION DROP ROW,
  CONSTRAINT has_resource_id EXPECT (resource_id IS NOT NULL)
)
AS SELECT b.bundle_id, b.dataset_tier, b.source_file, b._metadata, b.ingested_at, entry_index,
  variant_get(entry.value, '$.resource.resourceType', 'STRING') AS resource_type,
  variant_get(entry.value, '$.resource.id', 'STRING') AS resource_id,
  entry.value:resource AS resource, entry.value:request AS request
FROM STREAM(bronze_bundles) b
LATERAL VIEW posexplode(b.bundle:entry) AS entry_index, entry;
```

---

# L200-D: Tag Derivation Engine (Silver -> Gold)

```sql
CREATE OR REFRESH STREAMING TABLE gold_tagged_resources (
  CONSTRAINT has_governance_tags EXPECT (is_tagged),
  CONSTRAINT known_resource_type EXPECT (is_known_resource_type)
)
AS SELECT sr.bundle_id, sr.dataset_tier, sr.source_file, sr._metadata, sr.entry_index,
  sr.resource_type, sr.resource_id, sr.resource,
  tm.element_id, tm.dd_3024_part, tm.governance_tags, tm.source_category, tm.tag_json_array,
  concat('{"meta":{"tag":', tm.tag_json_array, '},', substring(to_json(sr.resource), 2)) AS tagged_resource_json,
  (tm.governance_tags IS NOT NULL AND tm.governance_tags != '') AS is_tagged,
  (tm.element_id IS NOT NULL) AS is_known_resource_type
FROM STREAM(silver_resources) sr
LEFT JOIN ref_dictionary_elements tm
  ON sr.resource_type = tm.resource_type
  AND (tm.instance_discriminator IS NULL OR tm.instance_discriminator = ''
    OR ( /* Instance Discriminator — finalized after L200-B */ TRUE ));

-- Bundle-level flags
CREATE OR REFRESH MATERIALIZED VIEW mv_bundle_flags AS
SELECT bundle_id,
  bool_or(governance_tags LIKE '%PROVIDER_APPROVED%') AS bundle_has_provider_approved,
  (COUNT(CASE WHEN governance_tags LIKE '%HITL_GATED%' THEN 1 END)
   >= (SELECT COUNT(*) FROM ref_tag_mapping_current WHERE governance_tags LIKE '%HITL_GATED%')
  ) AS bundle_hitl_complete
FROM gold_tagged_resources GROUP BY bundle_id;

-- Validated Gold with conformance booleans
CREATE OR REFRESH STREAMING TABLE gold_validated_resources (
  CONSTRAINT sensitivity_valid EXPECT (is_sensitivity_placement_valid),
  CONSTRAINT ai_provider_valid EXPECT (is_ai_provider_chain_valid),
  CONSTRAINT hitl_complete EXPECT (is_hitl_complete)
)
AS SELECT g.*,
  (NOT (g.governance_tags LIKE '%PHI_CLINICAL%' OR g.governance_tags LIKE '%PII_SENSITIVE%')
   OR g.element_id IS NOT NULL) AS is_sensitivity_placement_valid,
  (NOT (g.governance_tags LIKE '%AI_GENERATED%' AND g.governance_tags LIKE '%IMR_DETERMINANT%')
   OR COALESCE(bf.bundle_has_provider_approved, FALSE)) AS is_ai_provider_chain_valid,
  COALESCE(bf.bundle_hitl_complete, FALSE) AS is_hitl_complete
FROM STREAM(gold_tagged_resources) g
LEFT JOIN mv_bundle_flags bf ON g.bundle_id = bf.bundle_id;
```

---

# L200-E: Element-Level Gold (Genie Agent Optimized)

```sql
CREATE OR REFRESH STREAMING TABLE gold_resource_elements (
  CONSTRAINT has_element_sensitivity EXPECT (element_sensitivity IS NOT NULL)
)
COMMENT 'Element-level FHIR data for Genie Agent — per-field sensitivity, ABAC-masked'
AS SELECT gv.bundle_id, gv.dataset_tier, gv.resource_type, gv.resource_id,
  gv.element_id, gv.dd_3024_part, gv.governance_tags, gv.source_category,
  de.fhir_path AS element_path, de.element_data_type, de.element_sensitivity,
  variant_get(gv.resource, de.variant_path_expression, 'STRING') AS element_value,
  gv.resource_type AS visible_resource_type,
  gv.resource_id AS visible_resource_id,
  variant_get(gv.resource, '$.status', 'STRING') AS visible_status,
  COALESCE(
    variant_get(gv.resource, '$.effectiveDateTime', 'STRING'),
    variant_get(gv.resource, '$.period.start', 'STRING'),
    variant_get(gv.resource, '$.recordedDate', 'STRING'),
    variant_get(gv.resource, '$.authoredOn', 'STRING')
  ) AS visible_date,
  variant_get(gv.resource, '$.code.coding[0].system', 'STRING') AS visible_code_system,
  variant_get(gv.resource, '$.code.coding[0].code', 'STRING') AS visible_code,
  variant_get(gv.resource, '$.code.coding[0].display', 'STRING') AS visible_code_display,
  gv.is_tagged, gv.is_sensitivity_placement_valid, gv.is_ai_provider_chain_valid, gv.is_hitl_complete
FROM STREAM(gold_validated_resources) gv
LEFT JOIN ref_dictionary_elements de ON gv.element_id = de.element_id;
```


---

# L200-F: Output Element Generation

### Overview
Generate the 11 AI/ML-derived Output elements that do not exist in the input data. Clinical/analytical quality is NOT evaluated — only structure and tagging.

> **Dependency**: Finalize after L200-B confirms the exact Output element rows in pha-data-dictionary.xlsx.

### The 11 Output Elements (from PDF Section 8)

| Element | FHIR Resource | Expected Tags | Part |
|---|---|---|---|
| Preliminary Risk Flag | Flag | PART_C1, AI_GENERATED, ADVISORY_ONLY | C1 |
| Notification/Reminder Log | Communication | PART_C1, AI_GENERATED | C1 |
| Completeness/Discrepancy Score | ClinicalImpression | PART_B, AI_GENERATED | B |
| Discrepancy Flags | Flag | PART_B, AI_GENERATED | B |
| Record Review Queue Priority | Task | PART_C1, AI_GENERATED | C1 |
| Record Review Throughput Metrics | MeasureReport | PART_C1, AI_GENERATED | C1 |
| MHA AI/ML Risk Score | RiskAssessment | PART_C1, AI_GENERATED, ADVISORY_ONLY | C1 |
| MHA AI Summarization | ClinicalImpression | PART_C1, AI_GENERATED, ADVISORY_ONLY | C1 |
| Composite Multi-Part Risk Visualization | RiskAssessment | PART_C2, AI_GENERATED | C2 |
| AI-Drafted Recommendation/Referral | ServiceRequest | PART_C2, AI_GENERATED, ADVISORY_ONLY | C2 |
| AI/Provider Audit Trail | Provenance | PART_C2, AI_GENERATED | C2 |

### Design

```sql
CREATE OR REFRESH STREAMING TABLE gold_output_elements
COMMENT 'Generated Output elements — 11 per Tier 1 Bundle, tags from data dictionary'
AS SELECT
  sr.bundle_id, sr.dataset_tier, oe.resource_type,
  concat(sr.bundle_id, '-', oe.element_id) AS resource_id,
  oe.element_id, oe.dd_3024_part, oe.governance_tags,
  'Output' AS source_category, oe.tag_json_array,
  concat(
    '{"resourceType":"', oe.resource_type, '","id":"', sr.bundle_id, '-', oe.element_id, '",',
    '"meta":{"tag":', oe.tag_json_array, '},"status":"final",',
    '"subject":{"reference":"Patient/', sr.patient_id, '"}}'
  ) AS tagged_resource_json,
  TRUE AS is_tagged, TRUE AS is_known_resource_type
FROM (
  SELECT DISTINCT bundle_id, dataset_tier,
    variant_get(resource, '$.id', 'STRING') AS patient_id
  FROM STREAM(silver_resources) WHERE resource_type = 'Patient'
) sr
CROSS JOIN (SELECT * FROM ref_dictionary_elements WHERE source_category = 'Output') oe;
```

> **Note**: Minimal placeholder JSON. Each Output element has a different FHIR structure — RiskAssessment.basis references and Provenance.entity.what references must be finalized after L200-B. See `fixtures/output_element_templates.json`.

---

# L200-G: Bundle Reassembly (Government ndjson Only)

```sql
CREATE OR REFRESH MATERIALIZED VIEW output_bundles
COMMENT 'Government conformance output only — not a production path'
AS SELECT bundle_id, dataset_tier,
  concat('{"resourceType":"Bundle","id":"', bundle_id, '","type":"transaction",',
    '"meta":{"tag":[{"system":"pha-governance","code":"',
    CASE WHEN dataset_tier = 'core' THEN 'CORE_COVERAGE_SET' ELSE 'VOLUME_SET' END,
    '"}]},"entry":[',
    array_join(collect_list(concat('{"resource":', tagged_resource_json, '}')), ','),
    ']}') AS bundle_json
FROM gold_validated_resources GROUP BY bundle_id, dataset_tier;
```


---

# L200-H: Tier 2 Record Factory

### Overview
Generate >= 10 complete, correctly-tagged PHA records from nothing.

### Coverage Requirements (PDF page 9)
- Component Type: >= 1 Active + >= 1 Reserve
- Part B Discrepancy: >= 1 with + >= 1 without
- Composite Risk: >= 1 each low, medium, high

### Out-of-Scope (Do NOT Generate)
Provider Notification, Behavioral Health Determination, Provider-Approved Recommendation/Referral, Digital Signature/Certification, IMR Category Determination, IMR Notification to Commander/Service Member. Impact: Communication 1 not 3, ServiceRequest 1 not 2, Provenance 2 not 3, Observation 1 not 2.

### Design

```sql
CREATE OR REPLACE TABLE tier2_patients AS
SELECT
  concat('tier2-pt-', lpad(CAST(row_number() OVER (ORDER BY 1) AS STRING), 4, '0')) AS patient_id,
  CASE WHEN row_number() OVER (ORDER BY 1) <= 6 THEN 'Active' ELSE 'Reserve' END AS component_type,
  CASE
    WHEN row_number() OVER (ORDER BY 1) IN (1, 7) THEN 'low'
    WHEN row_number() OVER (ORDER BY 1) IN (2, 3, 8, 9) THEN 'medium'
    ELSE 'high'
  END AS risk_tier,
  CASE WHEN row_number() OVER (ORDER BY 1) IN (3, 4, 9, 10) THEN true ELSE false END AS has_discrepancy
FROM range(12);
```

> **Note**: Full Bundle assembly reuses Tier 1 tag mapping and Output element templates. Tier 2 also generates 14 Input elements as synthetic FHIR resources — exact structure depends on L200-B. See `fixtures/tier2_coverage_requirements.json`.

---

# L200-I: ABAC + Governance Setup

```sql
-- 01_abac_setup.sql — run once after pipeline populates Gold tables

-- Governed tag (allowed_values finalized after L200-B)
CREATE TAG IF NOT EXISTS paratus.pha_pipeline.sensitivity
  WITH (allowed_values = ('phi_clinical', 'pii_sensitive', 'none'));

-- Tag the element_value column
ALTER TABLE paratus.pha_pipeline.gold_resource_elements
  ALTER COLUMN element_value
  SET TAGS ('paratus.pha_pipeline.sensitivity' = 'element_level');

-- Masking function — checks element_sensitivity per row
CREATE OR REPLACE FUNCTION paratus.pha_pipeline.mask_element_value(
  val STRING, element_sensitivity STRING
) RETURNS STRING
RETURN CASE
  WHEN is_account_group_member('pha_clinical_reviewers') THEN val
  WHEN element_sensitivity IN ('PHI_CLINICAL', 'PII_SENSITIVE') THEN '[REDACTED]'
  ELSE val
END;

-- Apply with USING COLUMNS for per-row sensitivity
ALTER TABLE paratus.pha_pipeline.gold_resource_elements
  ALTER COLUMN element_value
  SET MASK paratus.pha_pipeline.mask_element_value
  USING COLUMNS (element_sensitivity);
```


---

# L200-J: Genie Agent (Bundle 2)

### Overview
One agent, one table (`gold_resource_elements`), ABAC handles redaction. Deployed via `serialized_space` inline in YAML — no parent_path, no external file.

### Design
- One table for both personas — UC ABAC enforces the difference
- Variable-substituted identifiers: `${var.catalog}.${var.schema}.table_name`
- 6 sample questions (governance + clinical)
- 6 example SQL queries
- Instructions: domain context, tag semantics, conformance flags, redaction, counting rules

### YAML (resources/genie.yml in Bundle 2)

```yaml
resources:
  genie_spaces:
    pha_agent:
      title: "PARATUS PHA — Clinical Data & Governance Agent"
      description: "Ask questions about PHA records, governance tagging, conformance status, and clinical data."
      warehouse_id: ${var.warehouse_id}
      embed_credentials: false
      serialized_space: |
        {
          "version": 2,
          "config": { "sample_questions": [ ... ] },
          "data_sources": { "tables": [
            { "identifier": "${var.catalog}.${var.schema}.gold_resource_elements", ... },
            { "identifier": "${var.catalog}.${var.schema}.mv_pipeline_kpis", ... },
            { "identifier": "${var.catalog}.${var.schema}.mv_bundle_completeness", ... },
            { "identifier": "${var.catalog}.${var.schema}.mv_tag_distribution", ... },
            { "identifier": "${var.catalog}.${var.schema}.mv_rule1_ai_provider", ... }
          ]},
          "instructions": { "text_instructions": [ ... ], "example_question_sqls": [ ... ] }
        }
      permissions:
        - level: CAN_RUN
          group_name: users
```

> **Full serialized_space**: The complete ~4KB JSON with all sample questions, table definitions, text instructions, and example SQL was provided in the conversation record (2026-09-16). Copy it into the `serialized_space: |` block.


---

# L200-K: AI/BI Dashboard (Bundle 2)

### Overview
Conformance and pipeline health dashboard. Deployed via `serialized_dashboard` inline in YAML.

### Dashboard Pages

**Page 1 — Pipeline Health**: Total Bundles (305), Total Resources Tagged, Conformance Pass Rate, Core vs Volume pie, Resources by Type bar, Ingestion Timeline.

**Page 2 — Governance Tag Coverage**: Tags by Category stacked bar, Tag Distribution Heatmap (resource_type x tag_code), Untagged Resources table, Tag Coverage by DD 3024 Part.

**Page 3 — Conformance Rules**: Rule 1 AI->Provider chain, Rule 2 Sensitivity Placement, Rule 3 HITL_GATED Completeness, Expectation Pass/Fail Summary.

**Page 4 — Record Drill-Down**: Bundle Selector filter, Resource Inventory table, Tag Completeness check, Reference Chain view.

### Implementation
Build interactively in the Databricks UI against mv_* tables after Bundle 1 populates them, then export with `databricks bundle generate dashboard --existing-id <id>`. Paste into `resources/dashboard.yml` under `serialized_dashboard: |`.


---

# L200-L: Conformance Validation Materialized Views

### Overview
Pre-computed MVs powering the dashboard and self-check before government submission.

### Design

```sql
CREATE OR REFRESH MATERIALIZED VIEW mv_pipeline_kpis AS
SELECT COUNT(DISTINCT bundle_id) AS total_bundles, COUNT(*) AS total_resources,
  COUNT(DISTINCT resource_type) AS distinct_resource_types,
  SUM(CASE WHEN is_tagged THEN 1 ELSE 0 END) AS tagged_resources,
  SUM(CASE WHEN NOT is_tagged THEN 1 ELSE 0 END) AS untagged_resources,
  COUNT(DISTINCT CASE WHEN dataset_tier = 'core' THEN bundle_id END) AS core_bundles,
  COUNT(DISTINCT CASE WHEN dataset_tier = 'volume' THEN bundle_id END) AS volume_bundles
FROM gold_validated_resources;

CREATE OR REFRESH MATERIALIZED VIEW mv_tag_distribution AS
SELECT resource_type, dd_3024_part, trim(tag.value) AS tag_code, COUNT(*) AS resource_count
FROM gold_validated_resources
LATERAL VIEW explode(split(governance_tags, ',')) AS tag
GROUP BY resource_type, dd_3024_part, trim(tag.value);

CREATE OR REFRESH MATERIALIZED VIEW mv_rule1_ai_provider AS
SELECT bundle_id,
  bool_or(governance_tags LIKE '%AI_GENERATED%' AND governance_tags LIKE '%IMR_DETERMINANT%') AS has_ai_imr,
  bool_or(governance_tags LIKE '%PROVIDER_APPROVED%') AS has_provider_approved,
  CASE WHEN bool_or(governance_tags LIKE '%AI_GENERATED%' AND governance_tags LIKE '%IMR_DETERMINANT%')
    AND NOT bool_or(governance_tags LIKE '%PROVIDER_APPROVED%') THEN 'FAIL' ELSE 'PASS'
  END AS rule1_status
FROM gold_validated_resources GROUP BY bundle_id;

CREATE OR REFRESH MATERIALIZED VIEW mv_bundle_completeness AS
SELECT bundle_id, dataset_tier,
  COUNT(DISTINCT resource_type) AS actual_resource_types,
  (SELECT COUNT(DISTINCT resource_type) FROM ref_dictionary_elements) AS expected_resource_types,
  CASE WHEN COUNT(DISTINCT resource_type) >= (SELECT COUNT(DISTINCT resource_type) FROM ref_dictionary_elements)
    THEN 'COMPLETE' ELSE 'INCOMPLETE' END AS completeness_status
FROM gold_validated_resources GROUP BY bundle_id, dataset_tier;

CREATE OR REFRESH MATERIALIZED VIEW mv_rule3_hitl AS
SELECT bundle_id, COUNT(*) AS actual_hitl_count,
  (SELECT COUNT(*) FROM ref_tag_mapping_current WHERE governance_tags LIKE '%HITL_GATED%') AS expected_hitl_count,
  CASE WHEN COUNT(*) >= (SELECT COUNT(*) FROM ref_tag_mapping_current WHERE governance_tags LIKE '%HITL_GATED%')
    THEN 'PASS' ELSE 'FAIL' END AS rule3_status
FROM gold_validated_resources WHERE governance_tags LIKE '%HITL_GATED%' GROUP BY bundle_id;
```

---

# L300: Implementation Specs

### Volume Layout (Bundle-Managed)

Both volumes are declared as resources in Bundle 1 (resources/volumes.yml) and created on `dab deploy`.

```
/Volumes/${var.catalog}/pha_pipeline/landing/     <- MANAGED volume (Bundle 1)
+-- core/          <- 30 Bundle JSON files (user drops after deploy)
+-- volume/        <- pha-volume-set.ndjson (user drops after deploy)
+-- reference/     <- tagging-taxonomy.json + pha-data-dictionary.xlsx

/Volumes/${var.catalog}/pha_pipeline/output/       <- MANAGED volume (Bundle 1)
+-- tier1_output.ndjson    (written by 02_ndjson_write.py)
+-- tier2_output.ndjson    (written by 02_ndjson_write.py)
```

SDP manages Auto Loader checkpoints automatically — no user-specified checkpoint path.

### Execution Order (Two-Bundle Deploy)

**Phase 1: Infrastructure (Bundle 1 — paratus-pha-pipeline)**
1. L200-B: Run 00_explore.py — validate assumptions against actual reference files
2. `dab deploy -t dev` on Bundle 1 — creates schema, volumes, pipeline, notebooks
3. Drop CDP files into /Volumes/paratus/pha_pipeline/landing/
4. Run or trigger the SDP pipeline — populates Bronze -> Silver -> Gold -> MVs
5. L200-I: Run 01_abac_setup.sql — governed tags + ABAC policies
6. Verify: `SELECT COUNT(DISTINCT bundle_id) FROM paratus.pha_pipeline.gold_validated_resources` = 305

**Phase 2: Experience Layer (Bundle 2 — paratus-pha-experience)**
7. `dab deploy -t dev` on Bundle 2 — registers dashboard + Genie Agent (tables now exist)
8. L200-K: Dashboard available — open and verify
9. L200-J: Genie Agent available — test sample questions
10. Notebook: Run 02_ndjson_write.py — produce government ndjson files
11. Demo: Narrate one core record end-to-end with dashboard + Genie Agent open

### Open Questions (Resolved by L200-B)

1. Exact JSON structure of tagging-taxonomy.json — array? nested by category?
2. Exact column names in pha-data-dictionary.xlsx
3. Instance Discriminator column format — field paths? prose? structured rules?
4. Whether input resources already have meta fields (meta.profile, meta.lastUpdated)
5. Full list of 21 tag codes and 5 categories
6. Whether there are sensitivity codes beyond PHI_CLINICAL and PII_SENSITIVE
7. Conformance rule structure in taxonomy — structured data or prose?

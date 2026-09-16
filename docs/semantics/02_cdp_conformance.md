# UC Page Glossary Candidates — CDP and Conformance

## CDP (Contract Data Package)
- **Definition**: The Government-furnished package containing the synthetic PHA dataset, data dictionary, and tagging taxonomy. Delivered as three components: `pha-synthetic-dataset-UNTAGGED.zip`, `pha-data-dictionary.xlsx`, and `tagging-taxonomy.json`.
- **Business Context**: The CDP is the input to the Phase 2 conformance gate. BDR ingests the CDP, applies governance tags, generates Output elements, and produces the ndjson output extract.
- **Data Usage**: CDP files land in `/Volumes/paratus/pha_pipeline/landing/` and are ingested via Auto Loader (`cloud_files`).
- **Related Terms**: Conformance Gate, Tier 1, Tier 2
- **Source**: Phase 2 Instructions PDF, Section 3

## Conformance Gate
- **Definition**: The Phase 2 evaluation checkpoint where the Government validates the offeror's output extract against the CDP. Pass/fail, evaluated identically across all offerors. Includes automated schema/structural validation (post-demo) and manual governance-tag spot-check (during live demo).
- **Business Context**: This is what BDR must pass by Friday. The pipeline's entire purpose is to produce output that passes this gate.
- **Data Usage**: The `output_bundles` materialized view produces the ndjson file that the Government validates. The `mv_*` materialized views and pipeline expectations provide self-check before submission.
- **Related Terms**: CDP, Tier 1, Tier 2, Conformance Rules
- **Source**: Phase 2 Instructions PDF, Section 6

## Tier 1 (Tag Government-Provided Records)
- **Definition**: The first of two required exercises. Ingest 305 Government-provided synthetic records, derive and apply governance tags from first principles, generate missing Output elements, and produce the ndjson output extract covering ALL records.
- **Business Context**: Tests whether the capability can correctly read and apply the tagging schema against records the Government has already substantially built.
- **Data Usage**: 30 core records (individual JSON files) + 275 volume records (ndjson). All 305 must appear in the Tier 1 output extract.
- **Related Terms**: Tier 2, Core Coverage Set, Volume Set
- **Source**: Phase 2 Instructions PDF, Sections 2-7

## Tier 2 (Independent Generation)
- **Definition**: The second required exercise. Generate minimum 10 new synthetic PHA records from nothing — no Government-provided Patient, no pre-built Bundle. Must cover Active/Reserve, discrepancy/no-discrepancy, and low/medium/high risk.
- **Business Context**: Tests whether the capability can originate a complete, correctly tagged PHA record — the closer analog to real operational use.
- **Data Usage**: Separate ndjson output file. Patient IDs must not overlap with `known_record_ids.txt`. 5 Provider-downstream elements are explicitly out of scope.
- **Related Terms**: Tier 1, Coverage Requirements
- **Source**: Phase 2 Instructions PDF, Section 8

## Core Coverage Set
- **Definition**: The 30 curated FHIR Bundle files in `core/`, representing every combination of component type (Active/Reserve), completeness, and behavioral-health risk tier, plus two edge cases (Provider override, mobile-only Reserve).
- **Business Context**: This is what the evaluation panel observes during the live demo. BDR narrates selected records from this set.
- **Data Usage**: Ingested via `cloud_files` with `wholetext => true` (one file = one Bundle = one row). `dataset_tier = 'core'` in the pipeline.
- **Related Terms**: Volume Set, Coverage Matrix
- **Source**: Phase 2 Instructions PDF, Section 3

## Volume Set
- **Definition**: The 275 FHIR Bundles in `pha-volume-set.ndjson`, one JSON object per line. Same structure as core but at throughput scale, for testing bulk ingestion and dashboard roll-up.
- **Business Context**: Not manually reviewed record-by-record by the panel. Used to validate ingestion throughput, unit/command dashboard aggregation, and queue-prioritization behavior.
- **Data Usage**: Ingested via `cloud_files` line-by-line (no `wholetext`). `dataset_tier = 'volume'` in the pipeline.
- **Related Terms**: Core Coverage Set, ndjson
- **Source**: Phase 2 Instructions PDF, Section 3

## ndjson (Newline-Delimited JSON)
- **Definition**: A file format where each line is a complete, valid JSON object. Used for both the volume set input and the required output extract format. Consistent with the FHIR Bulk Data export convention.
- **Business Context**: The Government's conformance check requires the output in this exact format. A proprietary export or live-query-only interface will not be evaluated.
- **Data Usage**: Volume ingestion: `cloud_files` reads line-by-line. Output: `output_bundles` MV produces one Bundle JSON per row, written to file with `coalesce(1).write.text()`.
- **Related Terms**: FHIR Bundle, Output Extract
- **Source**: Phase 2 Instructions PDF, Section 5.4

### Tagging Strategy
- Priority: HIGH — these terms define the evaluation framework
- Implementation: Create as UC Pages in the `paratus.pha_pipeline` schema

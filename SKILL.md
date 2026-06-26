---
name: sennet-data-search
description: Translate natural language queries into ElasticSearch JSON for the SenNet Data Portal search API, execute queries via POST, summarize results, and optionally save to JSON/CSV files
---

# SenNet Data Search Skill

## Overview

The SenNet Data Portal provides search access into SenNet's data repository via:
- **Web UI**: https://data.sennetconsortium.org/
- **Search API**: https://search.api.sennetconsortium.org/ (thin Elasticsearch wrapper)

Four entity types are searchable: **Dataset**, **Sample**, **Source**, **Collection**.

**Authentication**: The Search API supports two modes:
- **Anonymous**: No token required. Queries are limited to entities with `status: "Published"`.
- **Authenticated**: Uses a Globus Bearer token in the `Authorization` header. Provides access to the full authorized set including consortium-level data.

To obtain a token: log into the [SenNet Data Portal](https://data.sennetconsortium.org/), click your user menu (top right), and select "Copy Globus Token".

Always ask the user whether they want to query anonymously or with a token.

**Indexes**: Call `GET /indices` to check available indexes. The default search index is `entities`. Other indices include `files`, `cell-types`, `senotypes`, and various log indices.

This skill translates natural-language search requests into properly structured ElasticSearch JSON queries, sends them to the Search API (with user consent), summarizes the results, and optionally saves them to disk.

---

## Workflow

```
User: "Find all published RNAseq datasets from Yale"
  │
  ▼
Agent parses NL → identifies entity type, filters, fields
  │
  ▼
Agent asks clarifying questions if ambiguous
  │
  ▼
Agent builds ES JSON query document
  │
  ▼
Agent presents the JSON query and asks: "Send this to the Search API? (y/N)"
  │
  ▼
If yes → POST to Search API → display results summary (first 10 / total)
If no  → respond with the JSON only
  │
  ▼
Agent asks: "Save results? (JSON/CSV/both/skip)"
  │
  ▼
If yes → write file(s) to disk
```

---

## 1. Query Analysis & Clarification

### Parse the user's natural language to extract:

| Component | Examples |
|---|---|
| **Entity type** | Dataset, Sample, Source, Collection |
| **Filters** | organ = "Liver", assay = "RNAseq", group = "Yale", dataset_type = "Histology", source_type = "Human" or "Mouse" |
| **Sort order** | "newest first", "by date" |
| **Pagination** | implicit (use default `size: 5000`) or explicit |

### Ask clarifying questions when:

1. **Ambiguous assay type** — there are several kinds of RNAseq or ATACseq:
   - "Do you mean all RNAseq variants, or a specific one like `Bulk RNAseq`?"
   - "CosMx" could mean `CosMx Proteomics` or `CosMx Transcriptomics`. Ask: "Did you mean CosMx Proteomics, CosMx Transcriptomics, or both?"
   - "Visium" could mean `Visium (no probes)`, `Visium (with probes)`, or `Visium HD`. Ask: "Which Visium variant do you mean?"

2. **Ambiguous entity type** — "find kidneys" could mean datasets about kidneys or samples. Default to **Dataset** unless context suggests otherwise.

3. **Ambiguous field scope** — see [[DEFAULT_FIELDS]] below. If unsure, ask: "Should I return all available fields, or just the defaults for this entity type?"

4. **Pagination** — if user seems to expect more than 5000 results, ask about pagination.

5. **Multi-assay dataset type** — if the user's query includes a multi-assay `dataset_type` (e.g., "10X Multiome", "SNARE-seq2", "Visium (no probes)"), inform the user:
   - "That's a multi-assay dataset type with component types: `{components}`. Would you like to search for the multi-assay parent datasets, or for specific component types?"

6. **Ambiguous group name** — when the user mentions a term that matches multiple groups in [[GROUP_NAME_VALUES]], list the matching options and ask which they want. This applies to:
   - **Yale** — matches `TMC - Yale University - Fan`, `TMC - Yale University - Dixit`
   - **Minnesota** — matches `TMC - University of Minnesota - Robbins`, `TMC - University of Minnesota - Niedernhofer`
   - **Buck** — matches `TMC - Buck Institute`, `TDA - Buck Institute`

### Source Type (Human vs Mouse) Clarification

SenNet contains data from both Human and Mouse sources (as well as Human Organoid and Mouse Organoid). The source type is a first-class filter that significantly narrows results.

**Rule**: If the user does not explicitly specify a source type (human, mouse, organoid, etc.) in their natural-language query, the agent MUST ask:

> "Are you interested in Human data, Mouse data, or both?"

This applies regardless of the entity type being searched (Dataset, Sample, Source, or Collection).

### Dataset Classification

A user may ask for "primary datasets" or "derived datasets" — these are defined by specific ES field conditions.

**Primary** — `creation_action` is `Create Dataset Activity` or `Multi-Assay Split`:
- **Single-assay**: `creation_action` == `Create Dataset Activity`
- **Multi-assay**: `creation_action` == `Create Dataset Activity` AND contains descendants with different `dataset_type`
- **Component**: `creation_action` == `Multi-Assay Split`

**Derived** — `creation_action` == `Central Process` AND `dataset_type` contains square brackets `[ ]` around the processing pipeline name.

Current multi-assay dataset types and their components:

| Multi-assay Type | Component Types |
|---|---|
| `10X Multiome` | ATACseq, RNAseq |
| `SNARE-seq2` | ATACseq, RNAseq |
| `Visium (no probes)` | Histology, RNAseq |

> **Default assumption**: Unless the user explicitly asks for Derived datasets, assume they want Primary datasets. Add `creation_action` filter for `Create Dataset Activity` or `Multi-Assay Split` by default when querying Datasets.

---

### [[VOCABULARY_MAPPINGS]]

The agent should use this table and the ontology API to translate natural-language terms to ES field values.

#### Organ Code ↔ Name Mapping

The ES `organ` field (on Sample and `origin_samples`) stores UBERON codes. Use this table for translation. The authoritative source is `GET https://ontology.api.hubmapconsortium.org/organs?application_context=SENNET` — the agent may fetch it dynamically for the most current data. The `rui_code` column shows the 2-letter code (same as HuBMAP) which users may also reference.

| UBERON Code | RUI Code | Organ Name |
|---|---|---|
| UBERON:0001013 | AD | Adipose Tissue |
| UBERON:0000178 | BD | Blood |
| UBERON:0001474 | BX | Bone |
| UBERON:0002371 | BM | Bone Marrow |
| UBERON:0000955 | BR | Brain |
| UBERON:0000948 | HT | Heart |
| UBERON:0001066 | ID | Intervertebral Disc |
| UBERON:0004538 | LK | Kidney (Left) |
| UBERON:0004539 | RK | Kidney (Right) |
| UBERON:0000059 | LI | Large Intestine |
| UBERON:0002107 | LV | Liver |
| UBERON:0002168 | LL | Lung (Left) |
| UBERON:0002167 | RL | Lung (Right) |
| UBERON:0000029 | LY | Lymph Node |
| UBERON:0001911 | MM | Mammary Gland |
| FMA:57991 | ML | Mammary Gland (Left) |
| FMA:57987 | MR | Mammary Gland (Right) |
| UBERON:0005090 | MU | Muscle |
| null | OR | Organoid |
| UBERON:0010000 | OT | Other |
| UBERON:0002119 | LO | Ovary (Left) |
| UBERON:0002118 | RO | Ovary (Right) |
| UBERON:0001264 | PA | Pancreas |
| UBERON:0001987 | PL | Placenta |
| UBERON:0002097 | SK | Skin |
| UBERON:0002240 | SC | Spinal Cord |
| UBERON:0002106 | SP | Spleen |
| UBERON:0002370 | TH | Thymus |
| FMA:54974 | LT | Tonsil (Left) |
| FMA:54973 | RT | Tonsil (Right) |
| UBERON:0003126 | TR | Trachea |

**Note**: The ES `organ` field uses UBERON codes (e.g., `UBERON:0002106`). When querying, use the UBERON code in the filter (e.g., `{"term": {"origin_samples.organ.keyword": "UBERON:0002106"}}`). The ES index also includes an `organ_hierarchy` field with the display name (e.g., "Spleen") for use in results display.

#### Other Vocabulary Mappings

| Natural Language Term | ES Field | ES Value(s) |
|---|---|---|
| primary dataset | creation_action | `Create Dataset Activity` or `Multi-Assay Split` |
| derived dataset | creation_action, dataset_type | `Central Process`, AND contains `[...]` in dataset_type |
| single-assay dataset | creation_action | `Create Dataset Activity` |
| multi-assay dataset | creation_action | `Create Dataset Activity` (has component descendants) |
| component dataset | creation_action | `Multi-Assay Split` |
| affiliation, group, lab, team, data provider, TMC | group_name (Dataset/Source), source.group_name (Sample) | *use wildcard partial matching* — the user may specify only part of the name (e.g., `"Yale"` matches `"TMC - Yale University - Fan"` and `"TMC - Yale University - Dixit"`) |
| Yale | group_name | `TMC - Yale University - Fan`, `TMC - Yale University - Dixit` |
| UConn | group_name | `TMC - UConn Health` |
| Pittsburgh | group_name | `TMC - University of Pittsburgh` |
| Columbia | group_name | `TMC - Columbia University` |
| Minnesota Robbins | group_name | `TMC - University of Minnesota - Robbins` |
| Minnesota Niedernhofer | group_name | `TMC - University of Minnesota - Niedernhofer` |
| Jackson Lab, JAX | group_name | `TMC - The Jackson Laboratory` |
| Mayo | group_name | `TDA - Mayo Clinic - Schafer` |
| Duke | group_name | `TMC - Duke University` |
| UCSD | group_name | `TMC - University of California San Diego` |
| WashU, Washington University | group_name | `TMC - Washington University` |
| PNNL | group_name | `TDA - Pacific Northwest National Laboratory` |
| Johns Hopkins, JHU | group_name | `TMC - Johns Hopkins University` |
| Buck | group_name | `TMC - Buck Institute`, `TDA - Buck Institute` |
| MIT | group_name | `TDA - Massachusetts Institute of Technology` |
| Michigan | group_name | `TDA - University of Michigan` |
| Mass General, MGH | group_name | `TDA - Massachusetts General Hospital` |
| source type human | source.source_type | `Human` |
| source type mouse | source.source_type | `Mouse` |
| source type organoid | source.source_type | `Human Organoid`, `Mouse Organoid` |
| human donor | source.source_type | `Human` |
| mouse model | source.source_type | `Mouse` |
| all RNAseq | dataset_type | ["RNAseq", "Bulk RNAseq", "RNAseq (with probes)", "scRNAseq (10x Genomics v2)", "scRNAseq (10x Genomics v3)", "snRNAseq (10x Genomics v2)", "snRNAseq (10x Genomics v3)", "sciRNAseq", "snRNAseq (SNARE-seq2)"] |
| all ATACseq | dataset_type | ["ATACseq", "Bulk ATACseq", "sciATACseq", "snATACseq-multiome", "snATACseq", "snATACseq (SNARE-seq2)"] |
| CosMx, CosMX | dataset_type | `CosMx Proteomics`, `CosMx Transcriptomics` |
| all Histology | dataset_type | ["Histology", "AB-PAS Stained Microscopy"] |
| all Visium | dataset_type | ["Visium (no probes)", "Visium (with probes)", "Visium HD"] |
| whole-genome survey | dataset_type | WGS |
| Nucleic acid and protein | metadata.analyte_class | `Nucleic acid + protein` |
| Collagen | metadata.analyte_class | `Collagen` |

---

### [[DATASET_TYPE_VALUES]]

The Search API's `dataset_type` field contains distinct values across published Datasets. Values with brackets `[...]` indicate a processing pipeline has been applied to a primary type. When a user provides a partial or inexact name (e.g., "CODEX"), scan both tables for matches. If a name matches entries in both tables, inform the user and ask whether they want primary, derived, or both. For common base types (ATACseq, RNAseq, Histology), consider offering a roll-up of all variants.

If the user asks what dataset types are available, reference the lists below as a quick answer. However, these values were captured at a point in time and may not reflect the latest data. Offer to run a fresh terms aggregation on `dataset_type.keyword` to get the definitive current list from the live API.

Alphabetical lists:

#### Primary Dataset Types (no brackets)

```
10X Multiome
ATACseq
CosMx Proteomics
CosMx Transcriptomics
DBiT-seq
GeoMx (NGS)
Histology
LC-MS
PhenoCycler
RNAseq
RNAseq (with probes)
Seq-Scope
STARmap
Visium (no probes)
Visium (with probes)
Visium HD
WGS
Xenium
```

#### Derived / Processed Variants (with `[...]`)

```
10x Multiome [Salmon + ArchR + Muon]
Histology [Image Pyramid]
PhenoCycler [DeepCell + SPRM]
RNAseq [Salmon]
Visium (no probes) [Salmon + Scanpy]
Xenium [SpatialData + Scanpy]
```

---

### [[GROUP_NAME_VALUES]]

The `group_name` field identifies data providers (TMCs, TDAs, etc.). Group names are free-form strings that may embed partial names — always use wildcard matching when querying (see [[VOCABULARY_MAPPINGS]]). If the user asks what groups are available, reference the list below but offer to run a fresh terms aggregation on `group_name.keyword` for the current set.

```
TMC - UConn Health
TMC - University of Pittsburgh
TMC - Columbia University
TMC - University of Minnesota - Robbins
TMC - The Jackson Laboratory
TDA - Mayo Clinic - Schafer
TMC - Duke University
TMC - University of California San Diego
TMC - University of Minnesota - Niedernhofer
TMC - Washington University
TDA - Pacific Northwest National Laboratory
TMC - Johns Hopkins University
TMC - Buck Institute
TMC - Yale University - Fan
TDA - Buck Institute
TMC - Yale University - Dixit
TDA - Columbia University
TDA - Massachusetts General Hospital
TDA - University of Michigan
TDA - Massachusetts Institute of Technology
```

---

### [[ANALYTE_CLASS_VALUES]]

The `metadata.analyte_class` field identifies what type of analyte was measured in an assay. If the user asks what analyte classes are available, reference the list below but offer to run a fresh terms aggregation on `metadata.analyte_class.keyword` for the current set.

```
RNA
Nucleic acid + protein
Protein
Peptide
Polysaccharide
DNA
Collagen
Chromatin
Nucleic acid and protein
```

---

### [[SOURCE_TYPE_VALUES]]

The `source.source_type` field (on Sample entities) or top-level `source_type` (on Source entities) identifies the type of biological source:

- `Human` — human tissue donor
- `Mouse` — mouse model
- `Human Organoid` — human-derived organoid
- `Mouse Organoid` — mouse-derived organoid

The agent MUST ask about source type if the user hasn't specified one (see [[Source Type (Human vs Mouse) Clarification]] above).

---

## 2. ElasticSearch Query Construction

### Default Query Template

When querying Datasets, the primary-dataset filter is applied by default (unless the user explicitly asks for Derived):

```json
{
  "query": {
    "bool": {
      "filter": [
        {"terms": {"creation_action.keyword": ["Create Dataset Activity", "Multi-Assay Split"]}}
      ]
    }
  },
  "_source": [...],
  "from": 0,
  "size": 5000
}
```

Remove the `creation_action` filter if the user explicitly wants Derived datasets.

### Filter Patterns

Use `term` for exact-match single values:

```json
{"term": {"entity_type.keyword": "Dataset"}}
```

Use `terms` for multi-value filters (OR logic within same field, exact match):

```json
{"terms": {"dataset_type.keyword": ["RNAseq", "ATACseq"]}}
```

Use `wildcard` for **partial string matching** — always use this for `group_name` since the DB value may embed the user's term:

For **Dataset/Source** entities:
```json
{"wildcard": {"group_name.keyword": "*Yale*"}}
```

For **Sample** entities (group_name is nested under `source`):
```json
{"wildcard": {"source.group_name.keyword": "*Yale*"}}
```

For multiple partial matches (OR logic), wrap `wildcard` clauses in a `bool` `should`:

```json
{"bool": {"should": [
  {"wildcard": {"group_name.keyword": "*Yale*"}},
  {"wildcard": {"group_name.keyword": "*UConn*"}}
]}}
```

Use `range` for numeric/date ranges (timestamps are milliseconds since epoch):

```json
{"range": {"published_timestamp": {"gte": 1672531200000, "lte": 1704067199000}}}
```

Use `match` for text search on analyzed fields:

```json
{"match": {"description": "liver senescence"}}
```

Use `term` for source type filtering:

```json
{"term": {"source.source_type.keyword": "Human"}}
```

Combine multiple filters in the `filter` array (AND logic):

```json
{
  "query": {
    "bool": {
      "filter": [
        {"term": {"entity_type.keyword": "Dataset"}},
        {"term": {"status.keyword": "Published"}},
        {"wildcard": {"group_name.keyword": "*Yale*"}}
      ]
    }
  }
}
```

**Anonymous queries** — if the user is not providing a token, add a filter for published status:

```json
{"term": {"status.keyword": "Published"}}
```

### Sorting (optional)

Add a `sort` clause when the user specifies ordering:

```json
{
  "sort": [
    {"published_timestamp": {"order": "desc"}}
  ]
}
```

### Field Selection via `_source`

When returning all fields: omit `_source` entirely or set `"_source": true`.

When returning specific fields: list them as an array of strings. Use dot notation for nested fields:

```json
"_source": ["uuid", "sennet_id", "entity_type", "group_name", "dataset_type", "origin_samples.organ", "published_timestamp"]
```

### Pagination

Use `from` for offset. Default `size` is 5000. If user needs more, ask if they want pagination.

---

### [[FIELD_MAPPINGS]]

The fields below are derived from the OpenAPI specification at `github.com/sennetconsortium/search-api/blob/main/search-api-spec.yaml` and confirmed via live ES queries. Field names and types correspond to the Elasticsearch index documents.

#### Common Fields (All Entity Types)

| Field | Type | Description |
|---|---|---|
| `uuid` | string | SenNet unique identifier (32-hex-digit) |
| `sennet_id` | string | Consortium-wide ID (SNT###.ABCD.###) |
| `entity_type` | string | Dataset, Sample, Source, Collection |
| `description` | string | Free-text description |
| `data_access_level` | string (enum) | `public` or `consortium` |
| `created_timestamp` | integer | Milliseconds since epoch (ms) |
| `created_by_user_displayname` | string | Creator display name |
| `created_by_user_email` | string | Creator email |
| `last_modified_timestamp` | integer | Milliseconds since epoch (ms) |
| `group_name` | string | Globus group display name (top-level on Dataset, Source, Sample) |
| `group_uuid` | string | Globus group UUID |
| `registered_doi` | string | Registered DOI |
| `doi_url` | string | DOI resolution URL |

#### Source-Specific Fields

| Field | Type | Description |
|---|---|---|
| `source_type` | string (enum) | `Human`, `Mouse`, `Human Organoid`, `Mouse Organoid` |
| `label` | string | Lab-provided de-identified name |
| `mapped_metadata` | object | Structured clinical data with fields like `age`, `sex`, `race`, `body_mass_index` |
| `metadata` | object | Raw UMLS-coded clinical data (`living_donor_data`, `organ_donor_data` arrays) |
| `protocol_url` | string | Protocols.io DOI URL |

The `mapped_metadata` object (when present) contains key-value pairs where each value is an object with:
- `value` — array of values (e.g., `[50.0]`)
- `unit` — unit string (e.g., `"years"`)
- `value_display` — display string (e.g., `"50 years"`)
- `group_display` — facet group (e.g., `"Demographics"`)
- `key_display` — display key (e.g., `"Age"`)

#### Sample-Specific Fields

| Field | Type | Description |
|---|---|---|
| `sample_category` | string (enum) | `Organ`, `Block`, `Section`, `Suspension` |
| `organ` | string | UBERON code (e.g., `UBERON:0002106` for Spleen). Only set when `sample_category` is `Organ` |
| `organ_hierarchy` | string | Human-readable organ name (e.g., `Spleen`) |
| `source` | object (Source) | Nested Source object the tissue came from |
| `source.group_name` | string | Source's group/lab name |
| `source.source_type` | string | Source type (Human, Mouse, etc.) |
| `source.uuid` | string | Source UUID |
| `source.mapped_metadata` | object | Source clinical data |
| `rui_location` | object | Sample location/orientation in ancestor organ (RUI tool) |
| `visit` | string | Visit ID for source/patient |
| `origin_samples` | array[Sample] | Origin tissue samples |
| `direct_ancestor_uuid` | string | UUID of direct ancestor |
| `direct_ancestor` | object | Direct parent entity |
| `submission_id` | string | Internal ID |
| `protocol_url` | string | Protocols.io DOI URL |
| `has_metadata` | string | "True" or "False" |
| `has_rui_information` | string | "True", "False", or "N/A" |
| `contains_data` | string | "True" or "False" |

#### Dataset-Specific Fields

| Field | Type | Description |
|---|---|---|
| `title` | string | Dataset title |
| `status` | string (enum) | New, Processing, QA, **Published**, Error, Hold, Invalid |
| `published_timestamp` | integer | Publication timestamp (ms since epoch) |
| `contains_human_genetic_sequences` | boolean | True if data has human genetic sequence info |
| `dataset_type` | string | Assay/data type (e.g., RNAseq, CODEX, Histology) |
| `creation_action` | string | `Create Dataset Activity`, `Central Process`, or `Multi-Assay Split` |
| `ingest_metadata` | object | Pipeline ingest metadata (assay-specific) |
| `metadata` | object | Ingested experimental data metadata |
| `metadata.analyte_class` | string | Analyte class (RNA, Protein, DNA, etc.) |
| `files` | object | Ingested data files |
| `contacts` | array[Person] | Main contact people |
| `contributors` | array[Person] | Contributors to dataset creation |
| `antibodies` | array[Antibody] | Antibodies used in assay |
| `direct_ancestors` | array[Sample\|Dataset] | Direct parent entity(ies) |
| `origin_samples` | array[Sample] | Origin tissue samples (each containing `organ`, `organ_hierarchy`, `source`) |
| `sources` | array[Source] | Source objects (for multi-source datasets) |
| `local_directory_rel_path` | string | Path on HIVE filesystem |
| `thumbnail_file` | object | Thumbnail file details |
| `registered_doi` | string | DOI (e.g., `10.60586/SNT854.VJMX.652`) |
| `doi_url` | string | DOI resolution URL |
| `provider_info` | string | Provider information |

#### Collection-Specific Fields

| Field | Type | Description |
|---|---|---|
| `title` | string | Collection title |
| `contacts` | array[Person] | Main contacts |
| `contributors` | array[Person] | Contributors (analogous to author list) |
| `datasets` | array[Dataset] | Datasets in the collection |
| `registered_doi` | string | Registered DOI |
| `doi_url` | string | DOI resolution URL |

#### ES Index Denormalized Fields

The Elasticsearch index also contains denormalized fields for efficient querying:

| Field | Type | Description |
|---|---|---|
| `ancestors` | array[{uuid, entity_type}] | All ancestor entities |
| `descendants` | array[{uuid, entity_type}] | All descendant entities |
| `immediate_ancestors` | array[{uuid, entity_type, sennet_id}] | Direct parent entities |
| `immediate_descendants` | array[{uuid, entity_type, sennet_id}] | Direct child entities |
| `origin_samples` | array[{uuid, entity_type, organ, organ_hierarchy}] | Origin tissue samples |
| `immediate_ancestor_ids` | array[string] | Direct parent UUIDs |
| `immediate_descendant_ids` | array[string] | Direct child UUIDs |

**Note on field name suffixes**: Use `.keyword` suffix for exact-match filtering on string fields (e.g., `entity_type.keyword`, `status.keyword`). Use `.keyword` when filtering with `term`/`terms` to avoid analyzed-field surprises. This applies to nested fields within arrays too — for example, filter on `origin_samples.organ.keyword` and aggregate on `origin_samples.organ.keyword`. Note that `_source` selection does not use `.keyword`; it's only needed for filtering and aggregations.

**⚠️ `group_name` usage by entity type**:
- **Dataset / Source**: Use `group_name.keyword` at the top level
- **Sample**: Use `source.group_name.keyword` (nested under `source`)
- Always use `wildcard` on the `.keyword` field for partial matching (e.g., `{"wildcard": {"group_name.keyword": "*Yale*"}}`)

---

### [[DEFAULT_FIELDS]]

Default `_source` fields to return per entity type when user doesn't specify. These are reasonable defaults — the agent may ask the user to confirm or customize.

| Entity Type | Default Fields |
|---|---|
| **Dataset** | sennet_id, group_name, dataset_type, origin_samples.organ, origin_samples.organ_hierarchy, status, published_timestamp, title |
| **Sample** | sennet_id, group_name, source.group_name, source.source_type, sample_category, organ, organ_hierarchy, created_timestamp |
| **Source** | sennet_id, group_name, source_type, label, created_timestamp |
| **Collection** | sennet_id, title, group_name, status |

**Note**: Source clinical data (`age`, `BMI`, `sex`, `race`) is accessible via the `source.mapped_metadata` nested object on Sample entities, or via `mapped_metadata` directly on Source entities. These are not top-level scalar fields — access them via dot notation in `_source` (e.g., `"source.mapped_metadata"`). When displaying, use `value_display` from the mapped_metadata object.

---

## 3. Executing the Query

### Authentication

Ask the user before sending:

> "Should I query anonymously (Published entities only) or with a Globus Bearer token?"

- **Anonymous**: No `Authorization` header. Only entities with `status: "Published"` are returned. Always add `{"term": {"status.keyword": "Published"}}` to the filter.
- **Authenticated**: Ask the user for their Globus token. Include `Authorization: Bearer <token>` in the request headers. The token can be obtained from the [SenNet Data Portal](https://data.sennetconsortium.org/) user menu.

### Discover indexes (optional, before querying)

To confirm which indexes are available:

```
GET https://search.api.sennetconsortium.org/indices
```

Response: `{"indices": ["entities", "files", "cell-types", "senotypes", ...]}`

- `entities` — full entity documents (default). Use for most queries.
- `files` — file-level index
- `cell-types` — cell type annotations
- `senotypes` — senescence phenotype annotations

Default to `entities` index unless the user specifies otherwise.

### Construct the POST request

- **URL**: `https://search.api.sennetconsortium.org/search` (uses `entities` index by default)
- **Alternative**: `https://search.api.sennetconsortium.org/{index_name}/search` for a specific index
- **Method**: POST
- **Headers**: `Content-Type: application/json` (plus `Authorization: Bearer <token>` if authenticated)
- **Body**: the ElasticSearch JSON query constructed above

### Handle the response

Standard ES response:

```json
{
  "took": 45,
  "timed_out": false,
  "hits": {
    "total": {
      "value": 142,
      "relation": "eq"
    },
    "hits": [
      {
        "_index": "...",
        "_id": "...",
        "_source": { ... }
      }
    ]
  }
}
```

Interpret as a standard ES response:
- `hits.total.value` — total matching documents
- `hits.hits[]` — the returned documents (up to `size`)
- `timed_out` — flag if query timed out

### Special Response Behaviors

- **303 (S3 Redirect)**: If the response payload exceeds ~10 MB, the API returns a 303 with a redirect URL. Follow the redirect to retrieve the full results from S3. Inform the user: "Large result set — retrieving from S3 redirect."
- **504 (Gateway Timeout)**: The API has a 30-second max query/response time. If you get a 504, suggest the user narrow their query or use pagination with smaller `size` values.

### Error handling

- If the API returns an error (4xx/5xx), display the error message and the JSON query so the user can inspect or try manually.
- If the response doesn't match expected ES format, display what was received and ask for guidance.

---

## 4. Result Summarization

After receiving results, display to the user:

```
Found 142 results. Showing first 10:

1. [SNT###.ABCD.###](https://data.sennetconsortium.org/dataset?uuid={_id}) | Title: "..." | Organ: Liver | Assay: RNAseq | Group: TMC - Yale University - Fan
2. [SNT###.ABCD.###](https://data.sennetconsortium.org/dataset?uuid={_id}) | Title: "..." | Organ: Liver | Assay: RNAseq | Group: TMC - Yale University - Fan
...

Total: 142 results (showing first 10 of 5000 per-page limit)
```

For each hit, show key identifying fields (sennet_id as a portal link, title, organ via organ_hierarchy, assay, group). Use the hit's `_id` as the UUID in the portal URL: `https://data.sennetconsortium.org/dataset?uuid={_id}`. Truncate long titles.

For non-Dataset entities, link to the general search page: `https://data.sennetconsortium.org/search?uuid={_id}`.

Offer to answer specific follow-up questions about the results.

---

## 5. Saving Results to Files

After displaying results, ask:

```
Save results to file?
[j] JSON only
[c] CSV only
[b] Both JSON and CSV
[n] No, skip
```

### JSON File

- Filename: `sennet-{entity_type}-{YYYYMMDD-HHmmss}.json`
- Content: the full ElasticSearch response object as pretty-printed JSON (the raw response from the API)
- Write to current working directory

### CSV File

- Filename: `sennet-{entity_type}-{YYYYMMDD-HHmmss}.csv`
- Content specification:
  - Extract each hit's `_source` object as a row
  - Flatten nested objects using dot-notation for column names (like Python's `pandas.json_normalize()`)
    - Example: `{"source": {"mapped_metadata": {"age": {"value": [50]}}}}` → column name `source.mapped_metadata.age.value`
  - Array fields: serialize to a JSON string within the cell (e.g., `["item1", "item2"]`)
  - `null` / missing values: empty cell
  - Use comma as delimiter
  - Quote fields containing commas, newlines, or double-quotes (standard CSV quoting)
  - Include a header row with column names

---

## Appendix: Quick Reference

### Endpoints
- **Web Portal**: https://data.sennetconsortium.org/
- **Dataset URL**: https://data.sennetconsortium.org/dataset?uuid={uuid}
- **Search API**: https://search.api.sennetconsortium.org/
- **GET /indices**: https://search.api.sennetconsortium.org/indices
- **POST /search**: https://search.api.sennetconsortium.org/search
- **POST /{index}/search**: https://search.api.sennetconsortium.org/{index_name}/search
- **Ontology API**: https://ontology.api.hubmapconsortium.org/organs?application_context=SENNET

### Source of Truth
The API specification is maintained at:
`github.com/sennetconsortium/search-api/blob/main/search-api-spec.yaml`

Field schemas in this skill are derived from that spec and confirmed via live ES queries. If the search behavior doesn't match expectations, consult the spec for the most current definitions.

### Default query parameters
| Param | Value |
|---|---|
| `size` | 5000 |
| `from` | 0 |
| Display to user | First 10 hits + total count |

### File output defaults
| Format | Pattern |
|---|---|
| JSON | `sennet-{entity_type}-{timestamp}.json` |
| CSV | `sennet-{entity_type}-{timestamp}.csv` |

### CSV flattening
- Nested objects → dot-separated column names
- Arrays → JSON-stringified cell values
- null → empty cell

### Getting a Token
1. Navigate to https://data.sennetconsortium.org/
2. Click `Login` (top right)
3. After logging in, click your user menu (top right)
4. Select `Copy Globus Token`

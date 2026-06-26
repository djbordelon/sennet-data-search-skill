# SenNet Data Search Skill

A skill for [opencode](https://opencode.ai) that translates natural language queries into ElasticSearch JSON, sends them to the [SenNet Data Portal Search API](https://search.api.sennetconsortium.org/), summarizes results, and optionally saves them to JSON/CSV.

## Quick Start

```
"find datasets from Yale"
```

The skill parses the query, identifies entity type and filters, builds the ES query, asks for confirmation before sending, displays results with portal links, and offers to save to file.

## Features

- Natural language to ES JSON query construction
- Supports **Dataset**, **Sample**, **Source**, **Collection** entity types
- Anonymous (published-only) or Globus-authenticated queries
- Automatic organ code (UBERON) to name translation
- Multi-assay dataset type handling (10X Multiome, SNARE-seq2, Visium)
- Primary vs. derived dataset classification
- Group name wildcard matching for data providers
- Result summarization with portal links
- JSON and CSV export with nested field flattening

## Prerequisites

- [opencode](https://opencode.ai) CLI
- (Optional) [Globus Bearer token](https://data.sennetconsortium.org/) for authenticated queries

## Usage

Load the skill in opencode and describe what you want:

```
opencode> find RNAseq datasets from Yale
```

The skill will guide you through query construction, execution, and saving results.

## Learn More

- [SKILL.md](./SKILL.md) — full skill workflow, query templates, field mappings, and vocabulary tables
- [SenNet Search API Spec](https://github.com/sennetconsortium/search-api/blob/main/search-api-spec.yaml)
- [SenNet Data Portal](https://data.sennetconsortium.org/)

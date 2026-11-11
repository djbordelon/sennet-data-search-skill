# SenNet Data Search Skill

Ask the [SenNet Data Portal](https://data.sennetconsortium.org/) questions in plain English. This [OpenCode](https://opencode.ai) skill turns natural-language requests into Elasticsearch queries, shows you the query before running it, summarizes the results, and can export them to JSON or CSV.

## Why use it

The SenNet portal supports rich search, but programmatic access means knowing entity types, UBERON organ codes, source types, group names, and Elasticsearch syntax. This skill closes that gap:

- Describe the data you need in ordinary language.
- Let the agent resolve ambiguous assays, Visium/CosMx variants, and group names.
- Confirm whether you want human, mouse, organoid, or combined results.
- Review the generated query before anything is sent.
- Get a concise result summary with portal links.
- Export the full response for downstream analysis.

It is useful for exploratory questions, repeatable cohort discovery, and quick audits without leaving the conversation.

## Who it is for

- **Researchers** scoping SenNet data by tissue, assay, source type, or group.
- **Data curators** auditing published datasets, sources, samples, and collections.
- **Analysts and program staff** producing counts and exports for reports.
- **Developers** prototyping or debugging SenNet search queries.

## What you can search

| Area | What the skill understands |
|---|---|
| **Entity types** | Dataset, Sample, Source, Collection |
| **Common filters** | Organ, dataset type, group name, source type, analyte class, status, and creation action |
| **Source types** | Human, Mouse, Human Organoid, and Mouse Organoid |
| **Dataset shape** | Primary vs. derived datasets; single-assay, multi-assay parent, and component datasets |
| **Vocabulary** | UBERON organ codes, RUI codes, assay variants, group names, and controlled terms |
| **Indexes** | `entities` by default; optional `files`, `cell-types`, `senotypes`, and log indexes |

The skill treats human/mouse/organoid source type as a first-class filter, maps UBERON and RUI organ codes, and handles 10X Multiome and other multi-assay parent/component relationships.

## Example questions

| Ask | What happens |
|---|---|
| "Find human RNAseq datasets from Yale." | Asks whether you mean the Fan or Dixit Yale group, then filters by source type and assay. |
| "Compare human and mouse kidney dataset counts." | Resolves the organ and builds separate source-type counts. |
| "Find CosMx Transcriptomics datasets." | Distinguishes CosMx Transcriptomics from CosMx Proteomics. |
| "Show Visium HD datasets from Buck." | Resolves the Visium variant and the Buck TMC/TDA group. |
| "Find primary datasets from University of Minnesota and export CSV." | Clarifies Robbins vs. Niedernhofer, then exports the matching rows. |
| "Find cell-type annotations for lung." | Queries the `cell-types` index and returns matching annotations with organ and dataset context. |

## How it works

1. Parse the request into entity type, filters, sort order, and pagination.
2. Ask clarifying questions when a term is ambiguous.
3. Build an Elasticsearch query document.
4. Show the JSON and ask for permission before sending it.
5. POST to the SenNet Search API and summarize the first 10 hits plus the total count.
6. Offer to save the full response as JSON, CSV, or both.

The skill never sends a query or writes a file without confirmation.

## Install

Clone the repository into your OpenCode skills directory:

```bash
mkdir -p ~/.config/opencode/skills
git clone https://github.com/djbordelon/sennet-data-search-skill.git \
  ~/.config/opencode/skills/sennet-data-search
```

If you keep the repository elsewhere, symlink its `SKILL.md` instead:

```bash
mkdir -p ~/.config/opencode/skills/sennet-data-search
ln -s /path/to/sennet-data-search-skill/SKILL.md \
  ~/.config/opencode/skills/sennet-data-search/SKILL.md
```

OpenCode discovers the skill on its next launch.

## Use

Ask naturally in OpenCode:

```text
Find human RNAseq datasets from Yale.
```

The skill activates when your request matches its description, or you can invoke it explicitly by name: `sennet-data-search`.

## Authentication

- **Anonymous** — no token required. Queries are limited to entities with `status: "Published"`.
- **Authenticated** — use a Globus Bearer token to access the full authorized set, including consortium-level data.

To get a token:

1. Log in at [data.sennetconsortium.org](https://data.sennetconsortium.org/).
2. Open your user menu in the top-right corner.
3. Select **Copy Globus Token**.

The skill asks which mode you want before sending a query.

## Results and exports

Results are summarized as a compact list with SenNet IDs, portal links, titles, organs, assays, and groups. The total match count is shown alongside the first page of hits.

Exports use the full API response:

- `sennet-{entity_type}-{timestamp}.json` — pretty-printed raw response.
- `sennet-{entity_type}-{timestamp}.csv` — flattened `_source` rows with nested fields in dot notation.

## Scope and limitations

- **Metadata search, not data download.** The skill finds and describes datasets; it does not download primary data files.
- **Source type matters.** SenNet includes human, mouse, and organoid data; the skill asks for clarification when the request does not specify one.
- **Published-only by default for anonymous use.** Authenticated queries can reach the full authorized set.
- **Point-in-time vocabularies.** Field mappings and value lists are snapshots. `SKILL.md` contains the current mappings, but the live index is authoritative.
- **API limits.** The default page size is 5,000, queries have a 30-second gateway limit, and very large payloads may return a 303 redirect to S3.
- **Index choice matters.** Use `entities` for most searches; `files`, `cell-types`, and `senotypes` expose different record types.

## Learn more

- [SKILL.md](./SKILL.md) — full workflow, field mappings, vocabulary tables, result handling, and CSV rules.
- [SenNet Data Portal](https://data.sennetconsortium.org/)
- [SenNet Search API](https://search.api.sennetconsortium.org/)
- [Search API specification](https://github.com/sennetconsortium/search-api/blob/main/search-api-spec.yaml)
- [SenNet organ ontology](https://ontology.api.hubmapconsortium.org/organs?application_context=SENNET)
- [OpenCode](https://opencode.ai)

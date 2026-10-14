# Research Copilot High-Level Technical Design Plan

**Document status:** Draft v0.1  
**Source requirements:** [Requirements and Specifications.md](Requirements%20and%20Specifications.md)  
**Source vision:** [Agentic AI Project Vision Document.md](Agentic%20AI%20Project%20Vision%20Document.md)  
**Last reviewed:** 2026-09-20

## 1. Design objective

This design defines the system-level architecture for an evidence-traceable research copilot. The architecture must support the current Paper Scientist MVP while allowing later research, experiment, authoring, and review capabilities to be added without changing the source-evidence contract.

The central design rule is:

> Source evidence is authoritative. Every derived interpretation, model response, experiment, and manuscript claim remains traceable to its inputs and review state.

## 2. Architecture principles

1. **Evidence before generation:** retrieval and source locations precede model-generated summaries or claims.
2. **Derived data is disposable:** source PDFs and immutable extraction records are never overwritten by model output.
3. **Human approval at consequential boundaries:** accepting metadata, promoting an interpretation to a claim, and executing an experiment require review or explicit approval.
4. **Replaceable integrations:** model providers, embedding providers, scholarly APIs, storage engines, and execution backends are adapters, not core-domain dependencies.
5. **Structured state over conversation history:** persistent research records are the system of record; chat is an interface to that state.
6. **Local-first development:** the initial system works with the repository's paper collection and can operate without external scholarly services.
7. **Reproducibility by default:** each derived artifact records the source version, processing configuration, and producing operation.
8. **Fail per artifact:** a failure for one paper, model call, or external service does not destroy unrelated successful work.

## 3. Scope by release

### 3.1 Release 1: Paper Scientist MVP

The first release contains:

- PDF discovery and ingestion;
- page-level text extraction;
- stable paper identity and manifest management;
- structured metadata generation and validation;
- evidence and provenance records;
- keyword and semantic retrieval interfaces;
- citation-grounded summaries;
- cross-paper comparison;
- human review states;
- validation reports, structured logs, and progress reporting.

The first researcher-facing interface is a single-user Streamlit dashboard. It is local-first, consumes the application services directly, and remains replaceable by a future API/client presentation layer.

The frontend boundary is the versioned FastAPI contract under `/api/v1`. Streamlit uses the same client contract as other frontends; it may use the local transport for development or the HTTP transport when `RESEARCH_API_URL` is configured. Frontends must not access the filesystem or domain services directly.

### 3.2 Future releases

| Release | Capability | Main architectural addition |
| --- | --- | --- |
| 2 | Research Scientist | Scholarly adapters, literature graph, research-question and hypothesis services |
| 3 | Experimental Scientist | Versioned experiment service, controlled execution backend, result and statistics stores |
| 4 | Author and Reviewer | Manuscript projection, claim verifier, reviewer services, reproducibility audit |

Future releases consume the same paper, evidence, provenance, review, and audit contracts established by Release 1.

## 4. Logical architecture

```text
+---------------------+       +----------------------+
| Researcher interface| ----> | Application/API      |
| CLI, notebook, UI   |       | workflow coordinator |
+---------------------+       +----------+-----------+
                                       |
             +-------------------------+-------------------------+
             |                         |                         |
             v                         v                         v
+--------------------+      +---------------------+      +------------------+
| Ingestion pipeline |      | Research services   |      | Validation/audit |
| discover, hash,    |      | retrieve, summarize,|      | schema, links,   |
| extract, normalize  |      | compare, review     |      | logs, progress   |
+---------+----------+      +----------+----------+      +--------+---------+
          |                            |                         |
          v                            v                         v
+--------------------+      +---------------------+      +------------------+
| Source/evidence    |      | Structured research |      | Audit/artifact   |
| store              |      | store               |      | store            |
| PDFs, pages, refs  |      | metadata, claims,  |      | reports, logs,   |
|                    |      | reviews, graph      |      | run manifests    |
+--------------------+      +----------+----------+      +------------------+
                                       |
                                       v
                            +-----------------------+
                            | Integration adapters  |
                            | LLM, embeddings,      |
                            | scholarly APIs,       |
                            | execution backends   |
                            +-----------------------+
```

The application/workflow layer coordinates operations but does not become the owner of paper text or evidence. Domain services read and write through repository interfaces so storage can evolve without changing business rules.

## 5. Major components

### 5.1 Researcher interface

Responsibilities:

- submit or select paper files;
- start and monitor processing runs;
- inspect page-level evidence;
- review metadata, summaries, comparisons, and claims;
- accept, reject, or request evidence;
- view validation results and project progress.

The first implementation may be a CLI or notebook-friendly API. A web interface is an interchangeable presentation layer, not a prerequisite for the domain model.

### 5.2 Application/API layer

Responsibilities:

- authenticate and authorize operations when a service boundary exists;
- validate commands and configuration;
- start resumable jobs;
- expose operation status and artifacts;
- enforce approval gates;
- translate domain results into CLI, notebook, or UI responses.

This layer must not contain PDF parsing, prompt construction, or provider-specific logic.

### 5.3 Workflow coordinator

Responsibilities:

- sequence pipeline stages;
- create operation IDs and correlation metadata;
- persist checkpoints;
- retry transient failures with bounded policies;
- resume incomplete work;
- publish per-paper and batch progress.

The coordinator should initially be an in-process Python service. A queue or workflow engine can be introduced when processing volume requires it.

### 5.4 Ingestion and extraction pipeline

Stages:

1. discover configured PDF inputs;
2. calculate a content hash and assign or recover a stable `paper_id`;
3. detect duplicates and previously processed content;
4. extract page text using the PDF adapter;
5. write an immutable extraction record;
6. update the manifest and processing status;
7. emit validation and failure events.

The existing `scripts/extract_pdf.py` is the initial extraction implementation. Its page-preserving output is compatible with the source/evidence boundary and should be wrapped rather than duplicated.

### 5.5 Metadata and evidence service

Responsibilities:

- create structured metadata drafts from retrieved paper content;
- normalize model output into the versioned metadata contract;
- create page-linked evidence records;
- represent missing or uncertain fields explicitly;
- maintain evidence category and review status transitions.

The service must reject or flag output that cannot be linked to a source record. It must not infer that an absent field means a paper did not contain the information.

### 5.6 Retrieval service

Responsibilities:

- keyword search over page text and metadata;
- semantic search through an embedding adapter;
- return stable source locations with every result;
- record query, filters, provider version, and result identifiers;
- support reproducible retrieval for summaries and comparisons.

The initial implementation should provide keyword retrieval first. Semantic retrieval is added behind the same interface after evaluation data is available.

### 5.7 Research analysis services

MVP services:

- single-paper summarizer;
- cross-paper comparison service;
- review and approval service.

Future services:

- literature graph builder;
- research-gap detector;
- hypothesis and experiment proposal service;
- statistical analysis service;
- manuscript and reviewer services.

All services consume typed records and return drafts plus provenance, rather than writing directly to source storage.

### 5.8 Model orchestration adapter

The model adapter owns:

- provider and model selection;
- prompt templates and prompt versioning;
- structured response parsing;
- token and rate limits;
- bounded retries;
- cost and usage telemetry;
- model-call audit records.

The core system depends on a provider-neutral interface such as:

```text
complete(request, response_schema, provenance_context) -> structured_response
```

MiniMax may be the initial provider, but provider-specific request formats remain inside the adapter.

### 5.9 Validation and audit service

Validation runs should cover:

- JSON Schema compliance;
- required fields and status values;
- paper and artifact ID uniqueness;
- page-number ranges;
- evidence links;
- manifest consistency;
- source hash and processing metadata;
- review-state transitions;
- progress checklist consistency.

The service produces machine-readable reports and concise human-readable summaries. Blocking validation errors prevent acceptance but do not delete drafts.

## 6. Data and storage architecture

### 6.1 Storage tiers

| Tier | Contents | Authority |
| --- | --- | --- |
| Source store | Original PDFs and source metadata | Authoritative, immutable after ingestion |
| Extraction store | Page text and extraction run metadata | Derived, reproducible from source |
| Research store | Metadata, evidence, claims, reviews, comparisons | Derived, versioned, reviewable |
| Retrieval index | Keyword indexes and embeddings | Rebuildable acceleration layer |
| Audit store | Logs, validation reports, run manifests, usage records | Operational evidence |
| Artifact store | Future notebooks, results, figures, models, manuscripts | Derived research outputs |

The initial filesystem layout may continue to use `Research Papers/`, `data/processed/`, `data/metadata/`, `schemas/`, and `scripts/`. A database can be added behind repository interfaces without changing the logical records.

### 6.2 Record relationships

```text
Paper
  1 ---- * Page
  1 ---- * MetadataVersion
  1 ---- * EvidenceRecord
  1 ---- * OperationRun

EvidenceRecord
  * ---- 1 SourceLocation
  * ---- 0..1 Claim

Claim
  * ---- * EvidenceRecord
  1 ---- * ReviewDecision

OperationRun
  1 ---- * DerivedArtifact
```

### 6.3 Immutability and versioning

- Source PDFs are immutable once their content hash is recorded.
- Extraction records are replaced only by a new version linked to a new processor/configuration version.
- Metadata, claims, comparisons, and reviews are append-versioned rather than overwritten.
- Search indexes can be deleted and rebuilt from authoritative records.
- Every generated artifact includes its input record IDs and operation ID.

## 7. Primary data flows

### 7.1 Ingestion flow

```text
PDF input
  -> discover
  -> hash and identify
  -> extract pages
  -> validate extraction
  -> write extraction record
  -> update manifest
  -> report status
```

### 7.2 Metadata flow

```text
Paper pages
  -> retrieve relevant passages
  -> model produces structured draft
  -> normalize and validate schema
  -> attach evidence references
  -> researcher reviews
  -> accepted metadata version
```

### 7.3 Summary flow

```text
Researcher question
  -> query and filters
  -> retrieval with source locations
  -> model synthesis constrained to retrieved text
  -> citation/provenance validation
  -> draft response
  -> optional researcher acceptance
```

### 7.4 Future experiment flow

```text
Accepted hypothesis
  -> versioned experiment specification
  -> approval gate
  -> controlled execution backend
  -> result/artifact capture
  -> statistical analysis
  -> experiment ledger
  -> evidence-backed interpretation
```

## 8. Interfaces and contracts

### 8.1 Internal interfaces

| Interface | Responsibility |
| --- | --- |
| `PaperSource` | Discover PDFs and read source metadata. |
| `TextExtractor` | Convert one PDF into page-preserving extraction records. |
| `PaperRepository` | Store and retrieve paper, page, and manifest records. |
| `MetadataRepository` | Version structured metadata and evidence records. |
| `SearchIndex` | Index and retrieve source passages and metadata. |
| `ModelProvider` | Produce validated structured responses from model requests. |
| `ReviewRepository` | Record review decisions and state transitions. |
| `ArtifactRepository` | Store derived outputs and their lineage. |
| `Validator` | Run contract and integrity checks. |
| `OperationLog` | Record lifecycle events, durations, and failures. |

### 8.2 External interfaces

External services are accessed only through adapters:

- MiniMax or another LLM provider;
- embedding provider;
- OpenAlex and Semantic Scholar;
- vector index;
- Google Colab or notebook execution backend;
- optional Git repository metadata and artifact storage.

Adapters must define timeouts, retries, rate limits, credential handling, and a fallback/error contract.

## 9. Deployment and runtime topology

### 9.1 Development topology

```text
Developer workstation
  +-- Python application and CLI/API
  +-- Local filesystem source and derived stores
  +-- Local search index
  +-- Optional external model API
  +-- Automated tests and validation scripts
```

This topology matches the current repository and avoids introducing infrastructure before the data contracts are tested.

### 9.2 Service topology for later scale

```text
Client/UI
   |
API service
   |
Job/workflow queue ---- Worker processes
   |                         |
Metadata database       Extraction/model workers
   |                         |
Object/artifact store  Search/vector index
   |
External provider adapters
```

The migration from development to service topology is an operational change, not a domain-model rewrite.

## 10. Security, privacy, and safety design

- Secrets are loaded from environment or secret management and excluded from source and generated artifacts.
- Source papers and research records remain local by default; external transmission is explicit and logged.
- Model prompts contain only the minimum required passages and project context.
- Tool execution is disabled in the MVP and must require explicit approval in future releases.
- Future code execution runs in an isolated, resource-limited environment with filesystem and network policies.
- User roles and approval records are required before shared or production deployment.
- Audit records must not contain raw credentials or unnecessary sensitive source content.

Credential lifecycle: local development reads `RESEARCH_MODEL_API_KEY` and related settings from the process environment. `.env.example` contains names only; real `.env` files are ignored. Production deployments should use the hosting platform's secret manager and inject the same environment variables at process start. Credentials must never be stored in paper metadata, drafts, operation logs, Streamlit session artifacts, or Git history.

## 11. Reliability and observability

Every operation emits:

- `operation_id`;
- operation type and stage;
- paper/project identifier;
- input and output artifact IDs;
- processor or provider version;
- start/end time and duration;
- status and failure reason;
- retry count.

Reliability mechanisms:

- idempotent stage writes;
- checkpointed batch processing;
- bounded retries for transient provider failures;
- dead-letter or failed-item reports;
- atomic manifest updates;
- preservation of successful outputs;
- validation before state promotion.

## 12. Quality gates and architecture decisions

### 12.1 Release gates

Release 1 cannot be accepted until:

1. source and extraction records validate for the pilot paper set;
2. manifest generation is deterministic and resumable;
3. metadata and evidence records pass schema and provenance validation;
4. summaries and comparisons expose source locations;
5. provider failures and malformed papers are isolated;
6. review states and audit records are persisted;
7. project progress agrees with the requirements document's Section 10 checklist.

### 12.2 Architecture decisions to record

| ID | Decision | Initial direction |
| --- | --- | --- |
| ADR-01 | Source of truth | Immutable source and extraction records; derived research records are versioned. |
| ADR-02 | Processing model | Local in-process workflow first; queue/workers later if required. |
| ADR-03 | Model integration | Provider-neutral adapter; MiniMax as an initial option. |
| ADR-04 | Search strategy | Keyword retrieval first, semantic retrieval behind a replaceable interface. |
| ADR-05 | Storage strategy | Filesystem-compatible repositories first; database optional behind interfaces. |
| ADR-06 | Execution safety | No arbitrary code execution in MVP; approval and sandbox required later. |
| ADR-07 | Progress source | Requirements document Section 10 remains the project completion source of truth. |

## 13. High-level implementation sequence

1. Stabilize the paper, page, manifest, and metadata contracts.
2. Add repository interfaces around the current filesystem artifacts.
3. Add hashing, status tracking, resumability, and validation.
4. Implement metadata and evidence generation with a provider adapter.
5. Implement keyword retrieval, then evaluate semantic retrieval.
6. Add summaries, comparisons, review states, and audit reports.
7. Evaluate the MVP against a fixed paper set and record performance baselines.
8. Decide whether Phase 2 requires a database, queue, external scholarly APIs, or a web interface.
9. Extend the same contracts into literature graph, experiment, manuscript, and reviewer services.

## 14. Design checklist

These items describe completion of the high-level design. They are not counted by the requirements document's project progress calculation until corresponding implementation work is added to its Section 10 plan.

- [ ] Confirm MVP logical architecture and component boundaries.
- [ ] Confirm source, extraction, research, retrieval, audit, and artifact storage tiers.
- [ ] Confirm paper, evidence, metadata, review, and operation contracts.
- [ ] Record the initial architecture decisions as ADRs.
- [ ] Define repository interfaces for the filesystem-first implementation.
- [ ] Define model-provider adapter contract and failure behavior.
- [ ] Define retrieval result contract and reproducibility fields.
- [ ] Define approval gates for metadata, claims, and future execution.
- [ ] Define release gates and pilot evaluation protocol.
- [ ] Review the design against all requirements and update the traceability matrix.

## 15. Traceability to requirements

| Design area | Requirements covered |
| --- | --- |
| Ingestion and extraction pipeline | FR-01, FR-02, FR-06, FR-14, NFR-01, NFR-03 |
| Metadata and evidence service | FR-03, FR-04, FR-05, FR-11, FR-13, NFR-02 |
| Retrieval and research analysis | FR-07, FR-08, FR-09, FR-10, NFR-05 |
| Validation and audit | FR-12, FR-13, FR-15, NFR-01, NFR-06 |
| Interfaces and adapters | FR-10, NFR-05, NFR-08 |
| Security and execution boundaries | NFR-04, NFR-07 |
| Deployment and reliability | FR-14, NFR-01, NFR-03, NFR-06, NFR-08 |
| Release and progress gates | FR-15 and MVP acceptance strategy |

## 16. Design change policy

A high-level design change must identify the affected component, interface, data contract, requirement, release gate, and architecture decision. Changes that affect source authority, provenance, approval gates, or execution safety require review before implementation.

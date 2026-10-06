# Research Copilot Requirements and Specifications

**Document status:** Draft v0.1  
**Source:** [Agentic AI Project Vision Document.md](Agentic%20AI%20Project%20Vision%20Document.md)  
**Last reviewed:** 2026-10-06

## 1. Purpose

This document turns the project vision into an implementable, testable specification for an evidence-traceable research copilot.

The product goal is:

> Help a researcher conduct work whose methodology, evidence, reproducibility, and claims can withstand peer review.

The system is not intended to autonomously invent a publishable contribution or replace researcher approval. Every important output must distinguish literature facts, observations, hypotheses, experiments, results, interpretations, and claims.

## 2. Product scope

### 2.1 MVP scope: Phase 1, Paper Scientist

The MVP accepts a collection of research-paper PDFs and produces a structured, citation-grounded research workspace containing:

- source files and stable paper identifiers;
- page-level extracted text;
- structured paper metadata;
- claims with page-level evidence;
- searchable paper content;
- paper summaries and cross-paper comparisons;
- validation reports for provenance, schema compliance, and extraction completeness.

The MVP must support a human researcher in understanding and comparing papers. It does not execute arbitrary code, make unreviewed research claims, or generate a submission-ready manuscript.

### 2.2 Later scope

The following capabilities are planned but are not MVP acceptance criteria:

- literature graph construction and external scholarly search;
- research-gap and hypothesis generation;
- controlled experiment specification and Colab execution;
- statistical analysis and experiment ledger;
- evidence-backed manuscript generation;
- reviewer agents, claim verification, and reproducibility audit.

### 2.3 Out of scope for the MVP

- unrestricted autonomous access to external services;
- autonomous publication or submission;
- unreviewed deletion or mutation of source evidence;
- treating an LLM response as evidence without a source reference;
- guaranteeing novelty, acceptance, or publication in a particular venue.

## 3. Users and operating assumptions

| Actor | Responsibility |
| --- | --- |
| Researcher | Supplies papers, reviews extracted metadata, approves interpretations, and starts later research actions. |
| System operator | Configures model/API credentials, storage, limits, and execution permissions. |
| Research copilot | Extracts, organizes, retrieves, compares, and explains evidence while preserving provenance. |
| Reviewer/auditor | Inspects source-to-claim links, validation reports, experiment records, and generated artifacts. |

Assumptions:

- The pilot begins with the existing research-paper collection in `Research Papers/`.
- Python is the primary implementation language.
- A local-first workflow is preferred for source files and derived artifacts.
- Model providers, scholarly APIs, vector stores, and Colab are replaceable integrations behind interfaces.
- Human approval is required before an interpretation becomes a project claim and before any experiment is executed.

## 4. System boundaries

```text
Researcher
    |
    v
Workspace/API
    |
    +--> Ingestion and extraction --> Source/evidence store
    |
    +--> Retrieval and comparison --> Structured research store
    |
    +--> Model orchestration ------> Draft outputs with citations
    |
    +--> Validation and audit ------> Reports and project progress
```

The source/evidence store is authoritative for paper text and locations. Model output is derived data and must never overwrite source evidence.

## 5. Functional requirements

### FR-01: Paper ingestion

The system shall accept PDF files from a configured workspace and assign each file a stable `paper_id` that remains unchanged when derived files are regenerated.

**Acceptance:** An ingestion run reports accepted, rejected, duplicate, and failed files; each accepted paper has a source path, content hash, page count, and ingestion status.

### FR-02: Page-level extraction

The system shall extract text page by page and preserve the original page number for every extracted page.

**Acceptance:** For every successfully processed PDF, the output contains `source_file`, `page_count`, and an ordered `pages` collection. Extraction failures identify the paper and failure reason without stopping unrelated papers.

### FR-03: Structured paper metadata

The system shall represent, at minimum, title, authors, abstract, research problem, method, datasets, metrics, main results, limitations, and evidence references.

**Acceptance:** Metadata is validated against a versioned JSON Schema. Missing or uncertain fields are represented explicitly and are not silently fabricated.

### FR-04: Evidence and provenance

The system shall associate each extracted or generated claim with a source type, stable source identifier, page number where applicable, and provenance status.

**Acceptance:** A user can navigate from a claim to the exact paper and page supporting it. Claims without sufficient evidence are labelled `unverified` or `unsupported`.

### FR-05: Evidence categories

The system shall distinguish at least `literature_fact`, `observation`, `hypothesis`, `experiment`, `result`, `interpretation`, and `claim`.

**Acceptance:** Records cannot be promoted between categories without recording the actor, timestamp, and supporting evidence or decision.

### FR-06: Manifest and inventory

The system shall maintain a machine-readable manifest of papers, source files, processed files, page counts, and processing status.

**Acceptance:** Rebuilding the manifest is deterministic for unchanged inputs and excludes the manifest itself from the paper list.

### FR-07: Search and retrieval

The system shall support keyword and semantic retrieval over extracted paper content and structured metadata.

**Acceptance:** A retrieval result includes paper ID, page number or metadata path, matching text, and relevance information sufficient to reproduce the result.

### FR-08: Paper summary

The system shall generate a structured summary of an individual paper using only retrieved source material and shall expose citations for material claims.

**Acceptance:** The summary contains no uncited factual assertion about the paper when an applicable source location exists.

### FR-09: Cross-paper comparison

The system shall compare selected papers across shared dimensions including problem, dataset, method, baseline, evaluation metrics, results, and limitations.

**Acceptance:** The comparison identifies missing fields and conflicting reports instead of treating missing information as a negative result.

### FR-10: Model orchestration

The system shall call the configured language model through an adapter that separates prompts, model responses, tool calls, retries, limits, and audit records.

**Acceptance:** A model provider can be replaced without changing ingestion, storage, or validation modules. Failed calls are retryable and auditable.

### FR-11: Human review

The system shall provide a review state for generated metadata, comparisons, and claims: `draft`, `reviewed`, `accepted`, `rejected`, or `needs_evidence`.

**Acceptance:** Accepted research outputs retain the reviewer identity, review time, and version of the underlying source data.

### FR-12: Validation and audit

The system shall validate schemas, provenance links, required fields, duplicate IDs, page-number ranges, and manifest consistency.

**Acceptance:** A validation run exits non-zero when configured blocking errors exist and produces a report with error, warning, and passed counts.

### FR-13: Reproducible processing

The system shall record the input hash, processing version, configuration, model identifier where applicable, timestamp, and output path for each derived artifact.

**Acceptance:** A derived artifact can be traced to the exact source and processing configuration that produced it.

### FR-14: Failure handling

The system shall preserve partial successful work and report failures at paper and operation level.

**Acceptance:** One malformed PDF or model failure does not erase existing derived outputs or prevent independent papers from being processed.

### FR-15: Progress reporting

The system shall calculate project completion from the checked items in the Project Plan in this document.

**Acceptance:** Progress is reported as `completed items / total items` and as a percentage rounded to one decimal place. Only checklist items in Section 10 are counted.

## 6. Data specifications

### 6.1 Paper record

```json
{
  "paper_id": "stable-string",
  "source_file": "paper.pdf",
  "content_hash": "sha256:...",
  "page_count": 1,
  "pages": [
    {
      "page_number": 1,
      "text": "..."
    }
  ],
  "status": "extracted",
  "processing": {
    "processor_version": "...",
    "processed_at": "..."
  }
}
```

### 6.2 Structured metadata record

```json
{
  "schema_version": "1.0",
  "paper_id": "stable-string",
  "title": "...",
  "authors": ["..."],
  "abstract": "...",
  "research_problem": "...",
  "method": "...",
  "datasets": ["..."],
  "metrics": ["..."],
  "main_results": ["..."],
  "limitations": ["..."],
  "evidence": [
    {
      "claim": "...",
      "source_type": "paper",
      "paper_id": "stable-string",
      "page_number": 1,
      "status": "verified"
    }
  ],
  "review": {
    "status": "draft",
    "reviewer": null,
    "reviewed_at": null
  }
}
```

### 6.3 Naming and compatibility decision

The canonical metadata field for recording paper limitations is `limitations`. It is required for structured metadata records and is represented as an array of strings. Page-level extraction records remain independently readable because they store source text and do not depend on structured metadata fields.

### 6.4 Status values

- Processing: `queued`, `processing`, `extracted`, `failed`, `validated`.
- Review: `draft`, `reviewed`, `accepted`, `rejected`, `needs_evidence`.
- Evidence: `verified`, `unverified`, `unsupported`, `contradicted`.

## 7. Non-functional requirements

### NFR-01: Reproducibility

The same source files and pinned configuration shall produce equivalent structured outputs, apart from explicitly recorded timestamps and nondeterministic model fields.

### NFR-02: Traceability

Every user-visible factual statement derived from a paper shall be traceable to a source record and location, or clearly labelled as an interpretation or hypothesis.

### NFR-03: Reliability

Batch processing shall be resumable and shall not discard successful outputs when an individual item fails.

### NFR-04: Security and privacy

API keys shall be supplied through environment or secret management and shall never be committed to the repository or stored in generated paper content. External execution requires explicit user approval.

### NFR-05: Extensibility

Storage, model provider, embedding provider, scholarly search provider, and execution backend shall be replaceable through small interfaces.

### NFR-06: Observability

Each pipeline operation shall emit structured logs containing operation ID, paper/project ID where applicable, status, duration, and failure reason.

### NFR-07: Usability

A researcher shall be able to inspect source evidence before accepting generated metadata or claims. Errors shall identify the affected artifact and an actionable next step.

### NFR-08: Performance baseline

The system shall process papers independently and expose progress during batch runs. MVP performance targets will be set after measuring the current paper collection on the development machine.

## 8. Acceptance strategy

The MVP is accepted when:

1. all MVP functional requirements have an automated or documented acceptance check;
2. the existing paper collection can be ingested without losing page boundaries;
3. generated metadata passes the corrected versioned schema or is reported with actionable validation errors;
4. at least one paper summary and one cross-paper comparison include navigable evidence references;
5. a failed paper does not prevent successful papers from appearing in the manifest;
6. the project progress calculation agrees with the checked items in Section 10;
7. source files, derived records, and configuration are distinguishable and reproducible.

## 9. Open decisions

| ID | Decision | Owner | Due before |
| --- | --- | --- | --- |
| OD-01 | Choose SQLite, PostgreSQL, or filesystem-first storage for structured records. | Researcher | Filesystem-first for the local MVP; database remains a later option. |
| OD-02 | Choose local embedding/search implementation and define offline behavior. | Researcher | Keyword search and deterministic local fallback; remote embeddings are optional. |
| OD-03 | Select the initial model provider and record cost/rate limits. | Researcher | MiniMax is the first provider; credentials remain optional in local mode. |
| OD-04 | Define the exact claim/evidence graph representation. | Researcher | Provenance implementation |
| OD-05 | Define the first pilot research domain and evaluation set. | Researcher | All 10 current papers form the fixed pilot set. |
| OD-06 | Decide whether schema migration uses a breaking version or compatibility alias. | Researcher | Versioned corrected schema is the current contract. |
| OD-07 | Choose the first researcher-facing interface and draft persistence policy. | Researcher | Streamlit dashboard; versioned drafts persist locally and never overwrite source evidence. |

## 10. Project plan and progress tracking

This is the only checklist in this document used for project completion. Count every `- [x]` and `- [ ]` item in this section only.

### Current baseline: ingestion foundation

- [x] PDF page-text extraction exists in `scripts/extract_pdf.py`.
- [x] A paper manifest can be generated from processed JSON files.
- [x] Existing processed records validate against a versioned metadata schema.
- [x] Use the versioned and corrected paper metadata schema.
- [x] Automated tests cover extraction and manifest generation.

### Phase 1: Paper Scientist MVP

- [x] Define the repository data layout and artifact lifecycle.
- [x] Add stable content hashes and processing metadata.
- [x] Implement ingestion status and resumable batch processing.
- [x] Version and correct the paper metadata schema.
- [x] Normalize extracted paper records to the paper contract.
- [x] Implement structured metadata extraction.
- [x] Implement evidence record creation with page references.
- [x] Add schema and provenance validation.
- [x] Add deterministic manifest validation.
- [x] Add keyword retrieval with page-level results.
- [x] Add semantic retrieval behind a replaceable interface.
- [x] Add citation-grounded single-paper summaries.
- [x] Add cross-paper comparison tables.
- [x] Add human review states and review metadata.
- [x] Add structured operation logs and failure reports.
- [x] Add configuration and secret handling.
- [x] Evaluate the MVP against a fixed pilot paper set.
- [x] Document MVP usage and recovery procedures.

### Phase 2: Research Scientist

- [x] Add external scholarly search adapters.
- [x] Build a literature graph for papers, methods, datasets, citations, and claims.
- [x] Detect repeated limitations, under-tested assumptions, and comparison gaps.
- [x] Generate research questions with supporting evidence.
- [x] Generate hypotheses with predictions and alternative explanations.
- [x] Add adversarial novelty and feasibility checks.
- [x] Add researcher approval gates for research proposals.

### Phase 3: Experimental Scientist

- [x] Define a versioned experiment specification.
- [x] Generate controlled experiment code or notebooks.
- [x] Add explicit execution approval and sandbox limits.
- [x] Integrate a controlled Colab or notebook execution backend.
- [x] Record datasets, seeds, code versions, hardware, parameters, and artifacts.
- [x] Create an experiment ledger containing successful and failed runs.
- [x] Add statistical analysis with uncertainty, effect size, and test assumptions.
- [x] Propose follow-up experiments from observed results.

### Phase 4: Author and Reviewer

- [x] Generate manuscript sections from accepted evidence and results only.
- [x] Attach provenance to manuscript claims.
- [x] Add claim verification and unsupported-claim reporting.
- [x] Add methodology, novelty, statistics, writing, and reproducibility reviews.
- [x] Add revision tracking and reviewer response records.
- [x] Add a final reproducibility audit.
- [x] Add export to a documented manuscript format.

### Progress calculation

Current progress: **45 / 45 items complete (100.0%)**.

The progress number must be updated whenever a project-plan checkbox changes. A future automation should parse only Section 10 and calculate:

```text
progress_percent = completed_items / total_items * 100
```

A checked item means the implementation and its acceptance check are complete, not merely that work has started.

## 11. Traceability matrix

| Requirement group | Project-plan coverage | Primary verification |
| --- | --- | --- |
| FR-01 to FR-06 | Ingestion foundation and Phase 1 | Unit tests, schema validation, manifest validation |
| FR-07 to FR-10 | Phase 1 retrieval and orchestration | Retrieval fixtures, adapter contract tests, citation checks |
| FR-11 to FR-15 | Phase 1 review, audit, and progress | Workflow tests, validation reports, checklist parser |
| NFR-01 to NFR-08 | All phases | Reproducibility fixtures, security review, logs, pilot benchmark |

## 12. Specification change policy

Requirements changes must update the requirement ID, acceptance criteria, affected project-plan items, and traceability matrix entry. A completed checkbox must be reopened if its acceptance criteria materially change.

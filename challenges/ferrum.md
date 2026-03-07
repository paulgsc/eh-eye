# Ferrum — Local LLM Orchestration and Evaluation Engine
## Six-Month Engineering Roadmap: Q2–Q3 2025

Version: 0.1.0  
Status: Approved for Execution  
Company: Ironthread Systems (fictional, open-source AI infrastructure)  
Team: 8 engineers — 4 Rust backend, 2 ML, 1 infra, 1 frontend (TS)  
Signal: *"we want someone with experience building AI/LLM workflows, not just connecting APIs"*

---

## Signal Interpretation

### Surface Reading
The signal rejects thin integration work: a backend that calls an LLM endpoint and
returns the string response. It asserts that the value-producing work lives in the
system surrounding the model call, not in the call itself.

### Structural Gap
A system that only wraps an LLM API cannot: replay a workflow deterministically for
regression testing, route a partial result through a validation and repair loop, persist
intermediate agent state across a crash, attribute a wrong answer to a specific
retrieval failure vs. a model failure, or enforce a cost budget across a multi-step
reasoning chain. These are not missing features — they are structural impossibilities
in a thin-wrapper architecture. Any one of them requires building an execution substrate.

### Capability Cluster

| Competency | Architectural Requirement |
|---|---|
| Prompt orchestration | Versioned template registry with injection, diffing, and replay |
| RAG pipeline | Chunking, embedding, vector retrieval, reranking — each as a replaceable stage |
| Structured output enforcement | Schema validation, LLM-driven repair, output lineage |
| Tool-using agents | Typed function registry, sandboxed execution, bounded iteration |
| State and memory | Serializable workflow state, crash recovery, session continuity |
| Evaluation harness | Offline benchmark corpus, scoring pipeline, CI integration |
| Observability | Per-step trace, token accounting, failure classification |
| Human-in-the-loop | Confidence routing, review queue, approval audit log |
| Cost management | Model router, semantic cache, token budget enforcer |

### Domain Restatement
*How do you build a multi-step LLM execution engine that is deterministically replayable,
model-agnostic, and fully evaluable without depending on proprietary infrastructure or
live model access?*

---

## System Definition

### System Name
`ferrum` — A local-first LLM workflow orchestration and evaluation engine

### Problem Statement
Teams building production AI features consistently discover that the model call is not
the engineering problem. The engineering problem is the substrate around it: how to
define workflows as composable graphs rather than imperative scripts; how to validate
and repair structured outputs without manual intervention; how to retrieve relevant
context reliably at query time; how to know when the system is wrong; and how to keep
inference costs bounded as query volume scales. These problems are solved by building
an execution engine — not by selecting a hosted orchestration SaaS, which relocates the
problem without solving it and binds the team to vendor decisions. Ferrum is that engine:
a local-first, model-agnostic LLM workflow runtime with a native evaluation loop, built
in Rust with a minimal Python surface for ML-specific operations.

### Technical Mandates

| ID | Mandate | Rationale |
|----|---------|-----------|
| M1 | All workflow executions must be deterministically replayable from a serialized trace | Regression testing against fixed model snapshots requires replay without live inference |
| M2 | All orchestration logic must run without network access | Eliminates SaaS dependency; makes CI feasible on air-gapped infrastructure |
| M3 | The engine must be model-agnostic at the API boundary | Coupling to a single inference backend makes model upgrades a refactoring event |
| M4 | Every inter-node state transition must be serializable to a stable schema | Crash recovery and audit logging require deserializable intermediate state |
| M5 | Evaluation must run offline against versioned, immutable corpora | Live model variance must not pollute benchmark results |
| M6 | Tool execution must be sandboxed with explicit capability grants | Agent workflows that call external systems must not do so without auditable authorization |
| M7 | Prompt templates must be versioned and diffable independently of service code | Prompt regressions must be detectable without redeploying the engine |
| M8 | Token cost must be tracked per workflow node, not per session | Cost attribution at the node level is required for optimization and budget enforcement |

### Stack

**Rust (orchestration, retrieval, serving — primary)**
```
tokio                  — async runtime; all I/O-bound pipeline stages run on tokio tasks
axum                   — HTTP API for workflow submission and result retrieval
sqlx + sqlite          — workflow state persistence; postgres-compatible schema
serde / serde_json     — all inter-component serialization
tracing                — structured per-span observability throughout the engine
petgraph               — DAG representation of workflow graphs
tantivy                — full-text index for keyword-augmented retrieval
qdrant-client          — vector database client (local qdrant instance)
jsonschema             — output schema validation
candle                 — local tensor inference (Hugging Face models)
```

**Python (ML operations only — no orchestration logic)**
```
sentence-transformers  — embedding generation for ingestion and query pipelines
llama-cpp-python       — local GGUF model serving (inference backend)
datasets               — evaluation corpus loading and management
ragas                  — RAG evaluation metrics (answer relevance, faithfulness)
```

**TypeScript (evaluation dashboard and CLI only)**
```
vite + react           — evaluation results dashboard
tanstack-query         — async result polling
zod                    — frontend schema validation mirroring Rust types
ink                    — terminal UI for workflow submission CLI
```

**Prohibited**
```
OpenAI / Anthropic / any paid LLM API   — use llama-cpp-python or candle
Pinecone, Weaviate Cloud, or hosted VDB  — use local qdrant
LangChain, LlamaIndex, LlamaStack        — build the orchestration; do not wrap it
Any proprietary or gated dataset         — use HuggingFace Hub open corpora only
Hosted observability SaaS (Datadog etc.) — use OpenTelemetry with local Jaeger
```

---

## Fiscal Calendar

```
H1 Execution Window: Q2–Q3 2025 (24 weeks, 12 sprints)

Q2 2025
  Sprint 01: Apr 07 – Apr 18   [Foundation: Execution Core]
  Sprint 02: Apr 21 – May 02   [Foundation: Prompt Registry]
  Sprint 03: May 05 – May 16   [RAG: Ingestion Pipeline]
  Sprint 04: May 19 – May 30   [RAG: Retrieval and Synthesis]
  Sprint 05: Jun 02 – Jun 13   [Agents: Tool Registry]
  Sprint 06: Jun 16 – Jun 27   [Agents: Multi-Step Execution Loop]

Q3 2025
  Sprint 07: Jul 07 – Jul 18   [Reliability: Output Validation]
  Sprint 08: Jul 21 – Aug 01   [Reliability: State and Recovery]
  Sprint 09: Aug 04 – Aug 15   [Evaluation: Offline Harness]
  Sprint 10: Aug 18 – Aug 29   [Evaluation: CI Integration]
  Sprint 11: Sep 01 – Sep 12   [Hardening: Observability]
  Sprint 12: Sep 15 – Sep 26   [Hardening: Cost and Routing]
```

---

## Q2 2025 — Sprints 01–06

---

### `CORE-1.1 — Workflow Graph Runtime`

**Sprint:** 01  
**Points:** 13 (two-sprint candidate; front-load design in S01, implementation completes S02)  
**Owner:** Rust backend

**User Story**
```
As the Ferrum execution engine,
I want to represent a workflow as a directed acyclic graph of typed execution nodes,
So that multi-step LLM pipelines can be composed, scheduled, and replayed without
encoding execution order in imperative control flow.
```

**Technical Description**

The workflow graph is the central data structure of the entire engine. Each node
represents a discrete pipeline stage — an LLM call, a retrieval query, a tool
invocation, a validation step — and carries a typed input/output contract. Edges
encode data dependencies, not temporal ordering; the scheduler derives execution
order from a topological sort.

The graph must support fan-out (one node's output feeding multiple downstream nodes),
fan-in (a synthesis node waiting on multiple upstream results), and conditional
branching (routing based on a typed discriminant in a node's output). This rules out
a simple linked-list execution model. The `petgraph` crate provides the graph
primitives; the Ferrum layer adds typed node descriptors, I/O schema enforcement at
edge boundaries, and a serialization format stable enough to reconstruct execution
state after a crash.

The scheduler runs as a tokio task pool. Ready nodes — those whose upstream
dependencies are all resolved — are submitted to the pool concurrently. This
means the graph runtime must handle concurrent state mutation safely: node
completion writes are serialized through a `DashMap`-guarded state store, not
through shared mutable references.

**Acceptance Criteria**
- [ ] A workflow graph with fan-out and fan-in topology executes correctly with
      concurrent node resolution
- [ ] Topological sort detects cycles and returns a typed error; malformed graphs
      are rejected at submission time, not at runtime
- [ ] The full graph state (node status, input/output payloads, edge assignments)
      serializes to a stable JSON schema and round-trips without loss
- [ ] A workflow interrupted mid-execution can be reconstructed from its serialized
      state and resumed from the last completed node
- [ ] Node I/O type mismatches at edge boundaries are caught at graph validation
      time, not at execution time
- [ ] Concurrent node execution does not produce data races under `cargo test` with
      `--cfg loom` or equivalent concurrency test tooling

**Implementation Tasks**
```
- [ ] Define NodeDescriptor, EdgeDescriptor, and WorkflowGraph types in ferrum-core
- [ ] Implement topological sort with cycle detection over petgraph::DiGraph
- [ ] Build the concurrent node scheduler using tokio task pool with DashMap state store
- [ ] Implement WorkflowState serialization/deserialization (serde_json, stable schema)
- [ ] Write graph validation pass: type-check edge I/O contracts before execution
- [ ] Implement resume-from-checkpoint: reconstruct scheduler state from serialized WorkflowState
- [ ] Write integration tests covering fan-out, fan-in, conditional branch, and mid-run interruption
```

**Open-Source Components**
```
petgraph        — DiGraph for DAG representation and topological sort
tokio           — async task pool for concurrent node execution
dashmap         — concurrent hashmap for node state store
serde_json      — stable serialization of WorkflowState
```

**Definition of Done**  
A workflow graph with at least three concurrent branches, one fan-in, and one
conditional branch executes correctly, survives a simulated mid-run crash, and
resumes from checkpoint in integration tests.

---

### `CORE-1.2 — Inference Backend Abstraction`

**Sprint:** 01  
**Points:** 8  
**Owner:** Rust backend + ML engineer

**User Story**
```
As the workflow execution engine,
I want all LLM inference calls to go through a typed backend trait,
So that the engine is not coupled to any single inference runtime and model
swaps do not require changes to orchestration logic.
```

**Technical Description**

The inference backend is the single integration point between the Ferrum engine and
any model runtime. It must be defined as a Rust async trait with a minimal surface:
accept a structured prompt (not a raw string — a resolved `PromptPayload` from the
template registry), return a typed `InferenceResponse` that carries the raw text,
token counts, and backend metadata, and propagate errors through a typed
`InferenceError` that allows the caller to distinguish retryable from fatal failures.

Two concrete backends ship in this sprint: a `LlamaCppBackend` that speaks to a
locally running `llama-cpp-python` HTTP server (the simplest path to GGUF model
support), and a `CandelBackend` that runs inference in-process via the `candle`
crate. The trait boundary ensures neither bleeds into the scheduler. The backend
is selected per-node in the workflow graph, not globally — this is what enables
model routing in Sprint 12.

Token counting must occur at the backend layer, not in the orchestrator. Each
backend implementation is responsible for reporting prompt tokens and completion
tokens accurately. The scheduler reads these from `InferenceResponse` and writes
them to the node's trace entry. This is the data source for Sprint 12's cost router.

**Acceptance Criteria**
- [ ] `InferenceBackend` trait is defined with `async fn complete` accepting
      `PromptPayload` and returning `Result<InferenceResponse, InferenceError>`
- [ ] `LlamaCppBackend` connects to a local llama-cpp HTTP endpoint and maps
      its response to `InferenceResponse`
- [ ] `CandelBackend` loads a GGUF-format model and runs completion in-process
- [ ] Token counts (prompt + completion) are populated in `InferenceResponse`
      for both backends
- [ ] `InferenceError` variants distinguish: model unavailable, timeout, context
      length exceeded, and malformed response
- [ ] Swapping backends in a workflow graph requires no change to scheduler code

**Implementation Tasks**
```
- [ ] Define InferenceBackend trait, PromptPayload, InferenceResponse, InferenceError types
- [ ] Implement LlamaCppBackend: HTTP client (reqwest), response mapping, error classification
- [ ] Implement CandelBackend: model loading, tokenization, sampling, response wrapping
- [ ] Write backend selection logic in NodeDescriptor (backend field, not global config)
- [ ] Write unit tests for each backend against a locally served Mistral-7B GGUF
- [ ] Write mock backend for use in scheduler integration tests (deterministic, no model required)
```

**Open-Source Components**
```
candle-core / candle-nn   — in-process tensor inference
reqwest                   — HTTP client for llama-cpp-python server
tokenizers                — tokenization for accurate prompt token counting
```

**Definition of Done**  
The same workflow graph executes correctly against both `LlamaCppBackend` and
`CandelBackend` with token counts populated, verified by integration test with a
locally served Mistral-7B-Instruct GGUF.

---

### `PROMPT-2.1 — Versioned Prompt Template Registry`

**Sprint:** 02  
**Points:** 8  
**Owner:** Rust backend

**User Story**
```
As a workflow author,
I want prompt templates stored as versioned, parameterized artifacts independent
of service code,
So that prompt changes can be tested, diffed, and rolled back without a service
deployment.
```

**Technical Description**

Prompts are not strings embedded in source code. They are versioned artifacts with
their own lifecycle. The Ferrum prompt registry stores templates as TOML files in a
`/prompts` directory, each with a name, semver version, parameter schema (a JSON
Schema fragment), and the template body using `{{param}}` interpolation syntax
(Handlebars-compatible, implemented without the full Handlebars dependency).

The registry is loaded at engine startup and held in memory as an immutable
`Arc<PromptRegistry>`. Templates are resolved by `(name, version)` tuple. If a
workflow requests a template that does not exist in the registry, the error is
caught at graph validation time, not at execution time.

Template resolution produces a `PromptPayload` — a fully rendered prompt with all
parameters injected — which is the input type for `InferenceBackend::complete`.
This means the scheduler never handles raw strings; it handles typed payloads with
full lineage (which template, which version, which parameters produced this payload).

Parameter injection must validate parameters against the template's declared schema
before rendering. A missing required parameter or a type mismatch must produce a
typed `PromptError`, not a runtime panic or a silently malformed string.

**Acceptance Criteria**
- [ ] Templates defined in TOML with name, version, parameter schema, and body
- [ ] Parameter injection validates against schema before rendering; type errors
      surface as `PromptError::SchemaMismatch`
- [ ] Two versions of the same template can coexist in the registry; workflows
      pin to a specific version
- [ ] `PromptPayload` carries full provenance: template name, version, resolved
      parameters, rendered body
- [ ] Registry loading is deterministic: same directory contents produce identical
      in-memory state
- [ ] A test suite compares rendered output between two versions of the same
      template against a fixed parameter set (the "prompt diff" use case)

**Implementation Tasks**
```
- [ ] Define PromptTemplate and PromptRegistry types; implement TOML deserialization
- [ ] Implement parameter injection with schema validation (jsonschema crate)
- [ ] Implement version resolution: (name, version) -> PromptTemplate
- [ ] Write PromptRegistry loader: scan directory, parse, validate, build index
- [ ] Implement render() -> PromptPayload with provenance fields
- [ ] Write snapshot tests: fixed parameter set, fixed template -> golden rendered output
- [ ] Write diff utility: given two template versions, emit a structured diff of rendered outputs
```

**Open-Source Components**
```
toml             — template definition format
jsonschema       — parameter schema validation
serde            — template deserialization
```

**Definition of Done**  
Two versions of a RAG synthesis prompt coexist in the registry; a test asserts
that changing the template version changes the rendered output in a detectable,
schema-valid way; a missing required parameter produces a typed error caught before
inference.

---

### `PROMPT-2.2 — Runtime Prompt Trace Log`

**Sprint:** 02  
**Points:** 5  
**Owner:** Rust backend

**User Story**
```
As the Ferrum execution engine,
I want every resolved PromptPayload logged with its full provenance before
it is sent to inference,
So that any model output can be traced back to the exact prompt that produced it.
```

**Technical Description**

Debugging an incorrect LLM output requires knowing exactly what the model received.
"The system prompt plus some retrieved documents" is not sufficient — the exact
rendered string, at the exact template version, with the exact parameter values, must
be recoverable from a run ID. This is distinct from general observability (Sprint 11);
this is prompt-specific lineage, and it must be in place before the RAG pipeline
is built in Sprint 03, because retrieval introduces prompt content that varies
per-query and is otherwise unrecoverable.

The trace log is an append-only SQLite table (`prompt_traces`) written via sqlx.
Each row stores: run ID, node ID, template name, template version, parameter values
(JSON), rendered body (text), and a SHA-256 hash of the rendered body. The hash
allows deduplication queries ("how many distinct prompts were sent to inference this
week?") and cache key generation (Sprint 12).

**Acceptance Criteria**
- [ ] Every `InferenceBackend::complete` call is preceded by a write to `prompt_traces`
- [ ] The trace record is written before inference begins; a failed inference still
      has a trace record
- [ ] SHA-256 of rendered body is stored and can be used as a cache key
- [ ] Trace records are queryable by run ID, node ID, template name, and date range
- [ ] Trace writes do not block the inference call; they are fire-and-forget on a
      tokio task, with failures logged but not propagated to the workflow

**Implementation Tasks**
```
- [ ] Define prompt_traces schema; write sqlx migration
- [ ] Implement PromptTracer: async write to prompt_traces, SHA-256 hash generation
- [ ] Wire PromptTracer into the node execution path before InferenceBackend::complete
- [ ] Write query helpers: by_run_id, by_template, by_date_range
- [ ] Write test: assert trace record exists after a workflow execution, including
      failed inference
```

**Open-Source Components**
```
sqlx    — async SQLite writes with compile-time query checking
sha2    — SHA-256 hashing of rendered prompt bodies
```

**Definition of Done**  
After any workflow execution, every node that called inference has a corresponding
`prompt_traces` record recoverable by run ID with correct provenance fields.

---

### `RAG-3.1 — Document Ingestion Pipeline`

**Sprint:** 03  
**Points:** 13  
**Owner:** ML engineer + Rust backend

**User Story**
```
As the Ferrum retrieval subsystem,
I want documents ingested through a pipeline that chunks, embeds, and indexes
them into a local vector store with metadata,
So that the RAG retrieval stage has a queryable knowledge base that can be
updated incrementally without full reindexing.
```

**Technical Description**

Ingestion is not loading a file into a vector database. It is a multi-stage
pipeline where each stage has correctness requirements. The stages are: parse
(extract text from source format), chunk (split text into semantically coherent
units with controlled overlap), embed (generate dense vectors per chunk using a
local embedding model), and index (write vectors with metadata to qdrant).

The chunking stage is the most consequential design decision in the RAG system.
Fixed-size character splitting produces poor retrieval because sentence boundaries
are severed. The Ferrum chunker uses a sentence-boundary-aware strategy: split on
sentence boundaries (using a simple FSM, not a full NLP library), then group
sentences into chunks that do not exceed a configurable token budget (not character
count — token budget, because the retrieval model and the synthesis model have
token-based context windows). Chunks overlap by a configurable number of sentences.

Embedding runs via the Python `sentence-transformers` service, exposed as a local
HTTP endpoint. The Rust pipeline calls this endpoint per batch. This keeps the
embedding model out of the Rust binary while maintaining the model-agnostic contract:
the embedding service can be swapped without changing the Rust ingestion pipeline.

Each qdrant point stores: the chunk text, a hash of the source document + chunk
index (used for incremental update detection), source metadata (document ID,
title, page or section if available), and the embedding vector. The pipeline
checks the hash before re-embedding: unchanged chunks are not re-sent to the
embedding service.

**Acceptance Criteria**
- [ ] Pipeline ingests plain text, Markdown, and PDF (via `pdf-extract`) source formats
- [ ] Chunker produces sentence-boundary-aligned chunks within a configurable
      token budget; no sentence is split across chunks
- [ ] Chunk overlap is configurable in number of sentences; default is 2
- [ ] Embedding batching: chunks are sent to the embedding service in configurable
      batch sizes; the pipeline handles partial batch failures with retry
- [ ] Incremental ingestion: re-running the pipeline on an updated document
      re-embeds only changed chunks, detected by hash comparison
- [ ] Each qdrant point carries: chunk text, source metadata, chunk hash, document ID
- [ ] Pipeline is instrumented with tracing spans; total ingestion time and
      embedding service latency are logged per document

**Implementation Tasks**
```
- [ ] Implement document parsers: plain text, Markdown (pulldown-cmark), PDF (pdf-extract)
- [ ] Implement sentence-boundary chunker with token budget enforcement
- [ ] Implement embedding client: HTTP to local sentence-transformers service, batched
- [ ] Implement qdrant ingestion: upsert with hash-based deduplication
- [ ] Write ingestion pipeline orchestrator: parse -> chunk -> embed -> index
- [ ] Write integration test: ingest a 50-page Markdown document, assert point count
      and metadata correctness in qdrant
- [ ] Write incremental update test: modify one section, re-ingest, assert only
      affected chunks were re-embedded
```

**Open-Source Components**
```
qdrant-client           — vector upsert and collection management
pulldown-cmark          — Markdown parsing
pdf-extract             — PDF text extraction
tokenizers              — token budget enforcement in chunker
reqwest                 — HTTP client for embedding service
tracing                 — pipeline stage instrumentation
```

**Definition of Done**  
A 50-page Markdown document ingests correctly; incremental re-ingestion of a
modified document re-embeds only changed chunks; all qdrant points carry complete
metadata verified by a post-ingestion query.

---

### `RAG-3.2 — Semantic Retrieval Stage`

**Sprint:** 03  
**Points:** 8  
**Owner:** ML engineer + Rust backend

**User Story**
```
As a RAG workflow node,
I want to retrieve the top-k most relevant chunks for a query using dense
vector search with optional keyword augmentation,
So that the synthesis stage receives grounded context rather than relying on
parametric model knowledge.
```

**Technical Description**

The retrieval stage is a workflow node type (`RetrievalNode`) that accepts a query
string, generates a query embedding, executes a vector search against qdrant, applies
an optional reranking pass, and returns a typed `RetrievalResult` carrying the
retrieved chunks with their scores and source metadata.

Query embedding is generated via the same embedding service used in ingestion — this
is not incidental, it is a correctness requirement: the query vector must live in the
same vector space as the document chunk vectors. Any mismatch in embedding model
between ingestion and retrieval produces silently degraded results with no error
signal.

The reranking pass is a cross-encoder scoring step (also via the Python embedding
service, using a `cross-encoder/ms-marco-MiniLM-L-6-v2` or equivalent open model).
Reranking is optional per-node but enabled by default: bi-encoder retrieval has
known precision limitations at the top of the ranking, and cross-encoder reranking
on the top-20 retrieved candidates before passing top-5 to synthesis is a
standard mitigation. The reranker is called as a separate HTTP endpoint on the
embedding service.

Keyword augmentation uses tantivy: the same query is executed as a BM25 full-text
search in parallel with the vector search, and the results are merged by reciprocal
rank fusion before reranking. This handles cases where the query contains rare
technical terms (product names, identifiers) for which dense retrieval underperforms.

**Acceptance Criteria**
- [ ] `RetrievalNode` accepts a query string and returns `RetrievalResult` with
      chunks, scores, and source metadata
- [ ] Query embedding is generated by the same model version used for ingestion;
      version is checked at startup against a stored embedding model hash
- [ ] Reranking is applied to the top-k bi-encoder results before returning to
      the synthesis node; top-k is configurable
- [ ] Keyword augmentation via tantivy BM25 runs in parallel with vector search;
      results merged via reciprocal rank fusion
- [ ] `RetrievalResult` carries: chunk text, source document ID, chunk index,
      bi-encoder score, reranker score (if applied)
- [ ] Retrieval latency (embedding + search + rerank) is recorded in the node trace

**Implementation Tasks**
```
- [ ] Implement RetrievalNode: query embedding -> qdrant search -> rerank -> merge
- [ ] Implement tantivy index mirror of qdrant corpus (indexed at ingestion time)
- [ ] Implement reciprocal rank fusion over dense + sparse results
- [ ] Implement reranker client: HTTP to embedding service cross-encoder endpoint
- [ ] Write embedding model version check at engine startup
- [ ] Write retrieval benchmarks against MS MARCO dev set (open dataset, HuggingFace)
- [ ] Write integration test: known query against known corpus, assert top-1 is correct chunk
```

**Open-Source Components**
```
qdrant-client    — vector search
tantivy          — BM25 full-text search
reqwest          — cross-encoder reranker HTTP client
```

**Definition of Done**  
Retrieval against a 10,000-chunk corpus returns correct top-1 for 10 hand-curated
query/answer pairs; reranking measurably improves MRR@5 over bi-encoder alone on
the MS MARCO dev subset, verified by the evaluation script.

---

### `RAG-4.1 — Context Assembly and Synthesis Node`

**Sprint:** 04  
**Points:** 8  
**Owner:** Rust backend + ML engineer

**User Story**
```
As a RAG workflow,
I want retrieved chunks assembled into a prompt context with source citations,
and passed to a synthesis LLM node that produces a grounded answer with
traceable source attribution,
So that answers can be verified against the retrieved evidence and hallucination
can be distinguished from retrieval failure.
```

**Technical Description**

The synthesis node receives a `RetrievalResult` and a query, assembles a
`PromptPayload` using the `rag_synthesis` template from the prompt registry,
calls inference, and returns a `SynthesisResult` carrying the answer text, the
source citations (chunk IDs and scores), and the full `PromptPayload` used.

Context assembly is where several correctness constraints converge. The assembled
context must fit within the synthesis model's context window. The assembly stage
must: (1) compute the token count of the system prompt and question, (2) fill the
remaining context budget with retrieved chunks in reranker-score order, (3) truncate
at a chunk boundary (not a token boundary — severing a chunk mid-sentence degrades
synthesis quality), and (4) record which chunks were included and which were
truncated due to budget. The truncation log is part of `SynthesisResult` and is
used by the evaluation harness in Sprint 09 to distinguish "retrieval found the
right chunk but it was truncated" from "retrieval did not find the right chunk."

Citation generation assigns a `[N]` reference marker to each chunk included in the
context. The synthesis prompt instructs the model to use these markers in its
answer. Citation extraction parses the model's response for `[N]` patterns and
maps them back to source chunk IDs. If the model produces a `[N]` reference that
does not correspond to a provided chunk, this is flagged as a citation hallucination
in the trace.

**Acceptance Criteria**
- [ ] Context assembly respects a configurable token budget; chunks are truncated
      at boundaries, not mid-sentence
- [ ] Truncation log in `SynthesisResult` records included and excluded chunks
      with reasons
- [ ] Citation markers are injected into context and extracted from model response;
      invalid citations are flagged
- [ ] `SynthesisResult` carries: answer text, citation map, included chunks,
      truncated chunks, token usage, `PromptPayload` used
- [ ] The full synthesis pipeline (retrieve -> assemble -> synthesize) executes
      as a single workflow subgraph composable with other nodes
- [ ] An answer produced with no retrieved context (retrieval returned zero results)
      is distinguishable in the trace from an answer produced with low-scoring results

**Implementation Tasks**
```
- [ ] Implement context assembler: token budget management, chunk ordering, truncation log
- [ ] Implement citation injector: assign [N] markers to chunks in context
- [ ] Implement citation extractor: parse model response for [N] patterns, validate against context
- [ ] Define rag_synthesis prompt template in registry (with context, question, citation instructions)
- [ ] Implement SynthesisNode: assembler -> prompt render -> inference -> citation extraction
- [ ] Write integration test: end-to-end RAG pipeline on a small corpus with known answers
- [ ] Write citation hallucination test: assert [N] references in response map to provided chunks
```

**Open-Source Components**
```
tokenizers    — token counting for context budget enforcement
```

**Definition of Done**  
An end-to-end RAG workflow (ingest -> retrieve -> synthesize) on a 100-document
corpus returns grounded answers with valid citations on 5 hand-curated questions;
citation hallucination test passes; truncation log is populated when context
budget is exceeded.

---

### `RAG-4.2 — Hybrid Retrieval Quality Benchmarks`

**Sprint:** 04  
**Points:** 5  
**Owner:** ML engineer

**User Story**
```
As the ML engineering team,
I want retrieval quality measured against an open benchmark corpus before
the RAG pipeline is declared production-ready,
So that retrieval regressions introduced by chunking or embedding changes
are detectable against a fixed baseline.
```

**Technical Description**

Before the RAG pipeline is used in anger, its retrieval quality must be measured
against a fixed, reproducible baseline. This sprint delivers the retrieval
evaluation script and the first set of baseline numbers — not the full evaluation
framework (that is Sprint 09), but enough to confirm the retrieval stack is
functioning correctly and to set the numbers that future changes must not regress.

The benchmark corpus is the Natural Questions (NQ) open subset from HuggingFace
(`datasets` library). The evaluation computes: Recall@5 (is the answer-bearing
chunk in the top 5 results?), MRR@10 (mean reciprocal rank), and NDCG@10. These
are computed against the NQ dev split (3,610 questions). The script runs fully
offline against the locally indexed NQ corpus.

The evaluation script is a standalone Python script (not part of the Rust engine)
that queries the Ferrum retrieval HTTP endpoint for each NQ question and computes
the metrics. The results are written to a JSON file that becomes the retrieval
baseline artifact committed to the repo.

**Acceptance Criteria**
- [ ] NQ dev corpus ingested into local qdrant and tantivy index
- [ ] Evaluation script computes Recall@5, MRR@10, NDCG@10 against NQ dev split
- [ ] Baseline numbers committed to `benchmarks/retrieval_baseline.json` in the repo
- [ ] Script is reproducible: same corpus + same engine version produces identical numbers
- [ ] Dense-only vs. hybrid (dense + BM25) retrieval comparison included in baseline

**Implementation Tasks**
```
- [ ] Write NQ corpus ingestion script (HuggingFace datasets -> Ferrum ingestion pipeline)
- [ ] Write retrieval evaluation script: query Ferrum HTTP endpoint, compute metrics
- [ ] Run baseline evaluation; commit results JSON
- [ ] Document reproduction steps in benchmarks/README.md
```

**Open-Source Components**
```
datasets              — NQ corpus loading
sentence-transformers — embedding model (same as engine)
ragas                 — evaluation metric utilities
```

**Definition of Done**  
`benchmarks/retrieval_baseline.json` committed with Recall@5, MRR@10, and NDCG@10
for both dense-only and hybrid retrieval; reproduction script runs in under 30
minutes on a machine with a local qdrant instance.

---

### `AGENT-5.1 — Typed Tool Registry`

**Sprint:** 05  
**Points:** 8  
**Owner:** Rust backend

**User Story**
```
As the Ferrum agent subsystem,
I want tools registered as typed function descriptors with explicit input/output
schemas and capability grants,
So that an LLM agent can discover, call, and chain tools within a workflow
without the engine executing arbitrary code paths.
```

**Technical Description**

Tool calling in LLM agent systems is frequently implemented as raw function dispatch:
the model names a function, the engine calls it. This is correct when the tool set
is fixed and trusted. Ferrum's tool registry takes a stricter position: every tool
is registered with a typed JSON Schema descriptor (name, description, input schema,
output schema, capability grants), and the engine validates both the model's argument
structure against the input schema before dispatch and the tool's return value
against the output schema before returning it to the workflow.

Capability grants are the key addition. A tool that makes HTTP requests carries
`capability: ["network"]`. A tool that reads from the filesystem carries
`capability: ["fs_read"]`. A workflow node that executes tools must declare which
capabilities it permits. If a model attempts to call a tool whose capabilities
exceed the node's grant, the call is rejected with a typed `ToolError::CapabilityDenied`
and the rejection is logged. This is not a security boundary — it is an audit
boundary, making tool capability usage visible in the trace.

The tool registry is loaded at engine startup from a `tools/` directory of TOML
descriptors, analogous to the prompt registry. Tool implementations are Rust async
functions registered against the registry by name. The decoupling of descriptor
(TOML) from implementation (Rust) allows tool schemas to be versioned and diffed
independently of implementation changes.

**Acceptance Criteria**
- [ ] Tools defined in TOML with name, description, input schema, output schema,
      and capability list
- [ ] Tool arguments validated against input schema before dispatch; invalid
      arguments return `ToolError::SchemaMismatch` without calling the implementation
- [ ] Tool return values validated against output schema; schema violations return
      `ToolError::OutputMismatch`
- [ ] Capability grant checking: tool call rejected if tool capabilities exceed
      node's declared grant; rejection logged to trace
- [ ] Tool registry loaded from `tools/` directory at startup; missing tool
      name at call time returns `ToolError::NotFound`
- [ ] Five reference tool implementations ship: HTTP GET, SQLite query, file read,
      current timestamp, structured search over tantivy index

**Implementation Tasks**
```
- [ ] Define ToolDescriptor, ToolRegistry, ToolCall, ToolResult types
- [ ] Implement TOML-based descriptor loading and validation
- [ ] Implement pre-dispatch argument validation (jsonschema)
- [ ] Implement post-dispatch output validation (jsonschema)
- [ ] Implement capability grant enforcement in ToolExecutor
- [ ] Write five reference tool implementations (HTTP GET, SQLite query, file read, timestamp, tantivy search)
- [ ] Write unit tests for each validation and grant-checking path
```

**Open-Source Components**
```
jsonschema    — input/output schema validation
toml          — tool descriptor format
reqwest       — HTTP GET reference tool
sqlx          — SQLite query reference tool
tantivy       — search reference tool
```

**Definition of Done**  
All five reference tools execute correctly within a workflow; capability grant
violation is caught and logged without executing the tool; invalid arguments are
rejected before dispatch in unit tests.

---

### `AGENT-5.2 — Function Calling Protocol`

**Sprint:** 05  
**Points:** 8  
**Owner:** Rust backend + ML engineer

**User Story**
```
As the agent execution loop,
I want the LLM to express tool invocations as structured JSON within its
response, and want the engine to parse, validate, and dispatch these
invocations without treating them as free text,
So that tool calls are auditable, retryable, and type-safe regardless of
which model is producing them.
```

**Technical Description**

Different inference backends implement function calling differently. Some produce a
dedicated `tool_calls` field in a structured response object. Others (particularly
local GGUF models via llama-cpp) produce tool calls as JSON embedded in the response
text using a grammar-constrained generation mode. Ferrum's function calling protocol
must handle both without the workflow author knowing which backend is in use.

The `FunctionCallParser` accepts an `InferenceResponse` and extracts a
`Vec<ToolCall>`. For backends that produce structured tool call objects, the parser
reads the dedicated field. For text-generating backends, the parser looks for a
JSON block matching the tool call schema in the response text (using constrained
generation or post-hoc extraction). The extraction is strict: if no valid tool call
is found in a response that was expected to contain one, this is a
`ParseError::NoToolCall`, not a fallback to treating the text as an answer.

Grammar-constrained generation via llama-cpp is the preferred path for text backends:
the engine passes a BNF grammar derived from the tool registry to the llama-cpp
server for the inference call, constraining output to valid tool call JSON. This
eliminates post-hoc JSON extraction entirely for llama-cpp backends. The grammar
is generated at call time from the registered tool descriptors.

**Acceptance Criteria**
- [ ] `FunctionCallParser` extracts `Vec<ToolCall>` from both structured and text
      inference responses
- [ ] Grammar-constrained generation grammar is generated from tool descriptors
      and passed to llama-cpp backend
- [ ] Invalid tool call JSON in a text response produces `ParseError::NoToolCall`,
      not a partial result
- [ ] Parallel tool calls (model requests two tools in a single turn) are parsed
      and dispatched concurrently
- [ ] Tool call parsing result is stored in the node trace alongside the raw
      inference response

**Implementation Tasks**
```
- [ ] Implement FunctionCallParser: structured response path and text extraction path
- [ ] Implement BNF grammar generator from ToolRegistry (for llama-cpp constrained generation)
- [ ] Wire grammar generation into LlamaCppBackend for tool-calling nodes
- [ ] Implement parallel tool call dispatch in ToolExecutor
- [ ] Write tests: structured response parsing, text extraction, grammar-constrained output,
      parallel dispatch
```

**Open-Source Components**
```
serde_json     — JSON extraction and validation
```

**Definition of Done**  
A llama-cpp backend node successfully calls two tools in parallel using
grammar-constrained generation; both calls are parsed, dispatched, and results
returned to the workflow with full trace records.

---

### `AGENT-6.1 — Bounded Agent Execution Loop`

**Sprint:** 06  
**Points:** 13  
**Owner:** Rust backend

**User Story**
```
As the Ferrum agent runtime,
I want a ReAct-style thought/action/observation loop that executes tool calls
iteratively until a terminal condition is reached or a hard iteration limit
is exceeded,
So that complex multi-step tasks can be automated without the risk of
unbounded execution or runaway token costs.
```

**Technical Description**

The agent loop is a specialized workflow subgraph type: `AgentNode`. It encapsulates
a ReAct-style iteration: reason (LLM generates a thought and a tool call or final
answer), act (execute the tool call), observe (inject the tool result into the next
iteration's context), repeat. The loop terminates on: (1) the model producing a
final answer token sequence without a tool call, (2) the iteration count reaching
the node's configured `max_iterations`, or (3) the cumulative token spend for the
node exceeding the node's configured `token_budget`.

The iteration context is the critical state management problem. Each iteration
appends the thought, tool call, and observation to a running conversation history.
This history is re-injected into the prompt on each iteration. Without careful
management, the history grows to fill and then exceed the context window. The
`AgentContextManager` handles this: it tracks the token count of the accumulated
history and applies a truncation policy (oldest turns first) when the budget is
approached. Truncation is logged in the node trace.

Iteration state (current history, iteration count, token spend, tool calls made)
is serialized after each completed iteration. A crash mid-loop can be recovered
to the last completed iteration checkpoint.

**Acceptance Criteria**
- [ ] Agent loop terminates on final answer, `max_iterations` exceeded, or
      `token_budget` exceeded; termination reason is recorded in trace
- [ ] `AgentContextManager` truncates conversation history to stay within context
      window; truncation events are logged
- [ ] Iteration state is serialized after each completed iteration; loop can
      resume from last checkpoint after crash
- [ ] A node that exceeds `max_iterations` without a final answer produces
      `AgentError::IterationLimitExceeded`, not a hung task
- [ ] Token spend is tracked per iteration and cumulatively; budget enforcement
      is checked before each inference call, not after
- [ ] The full iteration trace (thought, tool call, observation, per each iteration)
      is stored in the run's trace log

**Implementation Tasks**
```
- [ ] Implement AgentNode: iteration loop with termination conditions
- [ ] Implement AgentContextManager: token-counted history with truncation policy
- [ ] Implement iteration state serialization and crash recovery
- [ ] Implement pre-inference budget check in agent loop
- [ ] Write integration test: agent solving a multi-step task over a local SQLite
      database using the SQLite query tool (5+ iterations)
- [ ] Write termination tests: max_iterations exceeded, token_budget exceeded,
      final answer detection
- [ ] Write crash recovery test: interrupt loop at iteration 3, resume, assert
      correct completion
```

**Open-Source Components**
```
tokio     — async iteration; each inference call is an awaited future
sqlx      — iteration state persistence
tracing   — per-iteration span instrumentation
```

**Definition of Done**  
An agent node resolves a 5-step task against a local SQLite database (find the
record, follow a foreign key, compute an aggregate, format a response); crashes
at iteration 3 are recovered correctly; budget exhaustion terminates cleanly
with a typed error.

---

### `AGENT-6.2 — Agent Session Memory`

**Sprint:** 06  
**Points:** 8  
**Owner:** Rust backend

**User Story**
```
As a multi-turn agent workflow,
I want relevant facts from prior sessions retrievable at the start of a new
session without replaying the full prior conversation,
So that long-running agent interactions accumulate useful context without
unbounded context window growth.
```

**Technical Description**

The agent loop's within-session context management (Sprint 6.1) handles the context
window problem within a single run. Cross-session memory is a different problem: a
user returns the next day and the agent should remember that the user's name is
Alice, that she is working on project Helios, and that the last action was blocked
on a dependency. Replaying the full prior conversation is not viable beyond a few
sessions.

Ferrum's session memory is a structured extract, not a transcript. At the end of
each agent session, a `MemoryExtractionNode` runs a summarization prompt over the
session's iteration trace, extracting typed facts into a `MemoryRecord` (entity:
value pairs with confidence scores). These records are stored in SQLite and indexed
in the tantivy full-text index for retrieval.

At the start of a new session, the `MemoryRetrievalNode` queries the memory store
by entity type and relevance to the current task, and injects the retrieved facts
into the agent's initial context. The key design constraint: memory injection is
bounded. The memory retrieval node operates under the same token budget as the
retrieval node; it does not inject unbounded history.

**Acceptance Criteria**
- [ ] `MemoryExtractionNode` runs after agent session completion; produces
      `Vec<MemoryRecord>` stored in SQLite
- [ ] Memory records carry: entity type, entity value, confidence score, source
      session ID, extraction timestamp
- [ ] `MemoryRetrievalNode` retrieves relevant memory records within a token budget
- [ ] Memory injection into agent context is bounded; exceeding budget truncates
      lowest-confidence records first
- [ ] Memory store is queryable by entity type and by relevance to a query string
      (tantivy full-text)
- [ ] A five-session simulation test asserts that facts established in session 1
      are correctly retrieved and injected in session 5

**Implementation Tasks**
```
- [ ] Define MemoryRecord schema; write sqlx migration for memory_records table
- [ ] Implement MemoryExtractionNode: summarization prompt + structured fact extraction
- [ ] Implement MemoryRetrievalNode: tantivy query + token-budget-bounded injection
- [ ] Implement confidence-ordered truncation policy for memory injection
- [ ] Write five-session simulation test
```

**Open-Source Components**
```
sqlx       — memory record persistence
tantivy    — full-text index for memory retrieval
```

**Definition of Done**  
Five-session simulation test passes: facts from session 1 appear in the agent's
initial context in session 5 with correct confidence ordering; memory injection
does not exceed the configured token budget.

---

## Q3 2025 — Sprints 07–12

---

### `VALID-7.1 — Structured Output Validator`

**Sprint:** 07  
**Points:** 8  
**Owner:** Rust backend

**User Story**
```
As a workflow node that depends on a structured LLM output,
I want the output validated against a declared JSON Schema before it is
passed to downstream nodes,
So that schema violations are caught at the boundary, not silently propagated
as malformed data into downstream logic.
```

**Technical Description**

LLM outputs that are supposed to be JSON frequently are not: the model may wrap
the JSON in markdown code fences, produce trailing commas, omit required fields,
or hallucinate field names. The validation layer must handle all of these cases
without failing silently.

The `OutputValidator` runs after every inference call on nodes that declare an
output schema. It attempts: (1) direct JSON parse, (2) extraction from markdown
code fence if direct parse fails, (3) schema validation on the parsed value. A
validation failure triggers the repair pipeline (Sprint 7.2). The validator records
the validation result (pass, extracted, repaired, failed) in the node trace.

The output schema is declared in the `NodeDescriptor` and stored in the prompt
registry alongside the template — the template and its expected output schema
are versioned together. This coupling is intentional: a prompt change that alters
the output structure must also update the schema.

**Acceptance Criteria**
- [ ] JSON extraction attempts direct parse then markdown fence extraction before
      failing
- [ ] Schema validation runs on extracted JSON; missing required fields and type
      mismatches produce typed `ValidationError` variants
- [ ] Validation result (pass / extracted / repaired / failed) written to node trace
- [ ] Output schema versioned alongside prompt template in registry
- [ ] A node with no declared output schema passes through the validator without
      modification (validator is a no-op, not a default-fail)

**Implementation Tasks**
```
- [ ] Implement OutputValidator: parse -> extract -> validate pipeline
- [ ] Implement markdown code fence extractor (regex-based, no full markdown parse)
- [ ] Wire OutputValidator into node execution path after InferenceBackend::complete
- [ ] Write tests: valid JSON, fenced JSON, missing required field, wrong type,
      no schema declared
```

**Open-Source Components**
```
jsonschema    — schema validation
serde_json    — JSON parsing
```

**Definition of Done**  
All five test cases (valid JSON, fenced JSON, missing field, wrong type, no schema)
produce the correct validation result and trace entry.

---

### `VALID-7.2 — Output Repair Pipeline`

**Sprint:** 07  
**Points:** 8  
**Owner:** Rust backend + ML engineer

**User Story**
```
As the output validation layer,
I want a repair pipeline that attempts to recover a schema-invalid LLM output
by re-prompting with the validation error before declaring the node failed,
So that transient schema violations do not terminate a workflow that could
have been recovered with a targeted correction prompt.
```

**Technical Description**

Schema validation failure has two root causes: the model genuinely cannot produce
the required structure (a model capability issue, not recoverable by re-prompting
with the same prompt) or the model produced a near-miss that a targeted correction
prompt can fix (a prompt engineering issue, recoverable). The repair pipeline
addresses the second case.

The `OutputRepairNode` takes the original prompt, the invalid output, and the
schema validation error, and constructs a repair prompt: "Your previous response
did not conform to the required schema. The error was: [error]. The schema requires:
[schema]. Please produce a corrected response." This is a second inference call on
the same node. The repair prompt is a versioned template in the registry
(`output_repair_v1`). The repair attempt is limited to a configurable number of
retries (default: 2). If all repair attempts fail, the node produces
`NodeError::OutputRepairExhausted` with the full repair history in the trace.

The repair history (original output, validation error, repair prompt, repair output,
validation result) is stored in the node trace. This data is the primary input for
prompt quality improvement: it identifies which output schemas are frequently failing
to repair, indicating that the synthesis prompt needs revision.

**Acceptance Criteria**
- [ ] Repair pipeline triggers automatically on schema validation failure
- [ ] Repair prompt is a versioned template in the registry; not hardcoded
- [ ] Repair attempts are limited by configurable `max_repair_attempts`
- [ ] Full repair history (attempt N: prompt, output, error) stored in node trace
- [ ] `NodeError::OutputRepairExhausted` is a distinct error type from
      `NodeError::InferenceFailed` — distinguishable downstream
- [ ] A workflow with a repair-requiring node does not terminate before repair
      attempts are exhausted

**Implementation Tasks**
```
- [ ] Define OutputRepairNode and repair attempt state types
- [ ] Implement repair prompt construction from validation error + schema
- [ ] Register output_repair_v1 template in prompt registry
- [ ] Implement retry loop with configurable max_repair_attempts
- [ ] Wire repair into OutputValidator failure path
- [ ] Write tests: successful repair on attempt 1, successful repair on attempt 2,
      exhausted repair with full history in trace
```

**Open-Source Components**
```
jsonschema    — validation error serialization for repair prompt
```

**Definition of Done**  
A node configured with `max_repair_attempts: 2` successfully repairs a near-miss
schema violation on attempt 1; a genuinely unrecoverable output exhausts both
attempts and produces `NodeError::OutputRepairExhausted` with full repair history
in trace.

---

### `STATE-8.1 — Workflow State Persistence and Recovery`

**Sprint:** 08  
**Points:** 8  
**Owner:** Rust backend + infra

**User Story**
```
As the Ferrum execution engine,
I want all workflow state written to durable storage after every node
completion, with a recovery path that resumes execution from the last
completed node after a crash,
So that long-running workflows are not lost to infrastructure failures.
```

**Technical Description**

The graph runtime (Sprint 1.1) supports resume-from-checkpoint conceptually.
This sprint makes it durable. After every node completes, the scheduler writes
the updated `WorkflowState` to the `workflow_runs` SQLite table via sqlx. The write
is synchronous with respect to the node completion — the scheduler does not mark
a node complete in memory until the durable write succeeds.

On startup, the engine scans for `workflow_runs` rows in state `Running` or
`Pending`. These are in-flight runs that were interrupted. The engine offers a
recovery API (`POST /runs/{id}/recover`) that reconstructs the `WorkflowGraph` from
the stored `WorkflowState`, marks all completed nodes as resolved, and re-submits
the graph to the scheduler starting from incomplete nodes. Nodes that were `Running`
at crash time (i.e., their inference call was in flight) are reset to `Pending` and
re-executed; inference calls are not guaranteed idempotent across crashes.

The `workflow_runs` schema stores: run ID, workflow definition (JSON, the graph
descriptor), current state (JSON, the `WorkflowState`), submission time, last update
time, and terminal status. The workflow definition and state are separate columns
because the definition is immutable after submission and the state is append-only
growing.

**Acceptance Criteria**
- [ ] `WorkflowState` written to `workflow_runs` after every node completion;
      write failure propagates as `PersistenceError` and halts the run
- [ ] Recovery API reconstructs and resumes a run interrupted at any node boundary
- [ ] Nodes in `Running` state at crash time are reset to `Pending` on recovery
- [ ] Recovery is idempotent: calling recover on an already-recovered run is a no-op
- [ ] Workflow definition stored immutably at submission; not re-serialized on update
- [ ] SQLite WAL mode enabled; concurrent reads during active write do not block

**Implementation Tasks**
```
- [ ] Write workflow_runs SQLite schema; enable WAL mode; write sqlx migration
- [ ] Implement durable state write after node completion in scheduler
- [ ] Implement startup scan for interrupted runs
- [ ] Implement recovery API endpoint: reconstruct graph, re-submit to scheduler
- [ ] Write crash simulation test: kill engine at node N, restart, assert correct
      completion from node N+1
- [ ] Write idempotency test: recover an already-completed run, assert no re-execution
```

**Open-Source Components**
```
sqlx        — durable state persistence, WAL mode config
```

**Definition of Done**  
Crash simulation test passes: engine killed after node 3 of a 6-node workflow
restarts, recovers, and completes nodes 4–6 correctly without re-running nodes 1–3.

---

### `STATE-8.2 — Human Review Queue`

**Sprint:** 08  
**Points:** 8  
**Owner:** Rust backend + frontend (TS)

**User Story**
```
As a compliance-sensitive workflow,
I want low-confidence or policy-flagged outputs routed to a human review queue
before being passed to downstream nodes,
So that incorrect or policy-violating outputs are intercepted before they
produce downstream effects.
```

**Technical Description**

The `ReviewGateNode` is a workflow node type that suspends execution pending human
approval. It accepts: the output from an upstream node, a confidence score (either
from the model's logprobs or from a separately configured scoring prompt), and a
routing policy (confidence threshold + optional policy flags). If the confidence
score is below the threshold or a policy flag is set, the node writes a
`ReviewRequest` to the `review_queue` SQLite table and suspends the workflow run
(sets its status to `AwaitingReview`).

The reviewer interacts via a minimal React UI (the only TypeScript deliverable in
this sprint). The UI polls the `review_queue` endpoint, displays the pending output
and its provenance, and offers three actions: approve (pass the output to downstream
nodes as-is), reject with replacement (submit a corrected output that replaces
the model's output), and reject without replacement (terminate the workflow with
`WorkflowError::ReviewRejected`). Approval and rejection are written to the
`review_decisions` audit log table with reviewer ID (from HTTP basic auth or a
configured static token), decision, timestamp, and rationale (free text, optional).

**Acceptance Criteria**
- [ ] `ReviewGateNode` suspends workflow run on confidence-below-threshold;
      run status transitions to `AwaitingReview`
- [ ] Review request written to `review_queue` with: run ID, node ID, pending output,
      confidence score, policy flags, provenance
- [ ] Reviewer UI displays pending reviews; approve/reject actions call API
- [ ] Rejection with replacement accepts corrected output; downstream nodes receive
      replacement, not original
- [ ] All review decisions written to `review_decisions` audit log with
      reviewer ID, decision, timestamp
- [ ] Workflow resumes automatically after approval without manual engine restart

**Implementation Tasks**
```
- [ ] Define review_queue and review_decisions schemas; write sqlx migrations
- [ ] Implement ReviewGateNode: confidence check, policy flag evaluation, suspension
- [ ] Implement review API endpoints: list pending, approve, reject, reject-with-replacement
- [ ] Implement workflow resume-on-approval: approval callback wakes suspended run
- [ ] Build reviewer UI (React + TanStack Query): pending review list, detail view,
      approve/reject/replace actions
- [ ] Write end-to-end test: workflow with ReviewGateNode, simulate approval, assert
      downstream node receives correct output
```

**Open-Source Components**
```
sqlx              — review queue and audit log persistence
axum              — review API endpoints
vite + react      — reviewer UI
tanstack-query    — UI async state
zod               — API response validation in UI
```

**Definition of Done**  
End-to-end test passes: workflow suspends at ReviewGateNode, reviewer approves via
UI, workflow resumes and downstream node receives approved output; audit log contains
correct decision record.

---

### `EVAL-9.1 — Offline Evaluation Harness`

**Sprint:** 09  
**Points:** 13  
**Owner:** ML engineer + Rust backend

**User Story**
```
As the ML engineering team,
I want an offline evaluation harness that runs a workflow against a versioned
test corpus and produces structured benchmark reports,
So that workflow regressions introduced by prompt changes, model updates, or
retrieval modifications are detected before deployment.
```

**Technical Description**

The evaluation harness is the system that makes the rest of the engine testable.
Without it, the team has no way to know whether a prompt change improved or
degraded end-to-end quality. The harness runs a workflow in a special
`EvalMode` where: (1) inference calls use a pinned model snapshot (a specific
GGUF quantization committed to the eval fixture store), (2) the vector database
is a fresh, deterministic instance loaded from a fixed corpus snapshot, and (3)
all randomness is seeded deterministically.

The evaluation corpus is a `EvalDataset`: a set of `(input, expected_output)`
pairs stored as JSONL files in the `evals/` directory. Each dataset is versioned
(semver, committed to the repo). The harness runs the workflow once per item,
collects the `WorkflowState` trace for each run, and passes the actual output
and the expected output to a set of `EvalScorer` implementations.

Scorers implemented in this sprint: `ExactMatchScorer` (string equality),
`RougeScorer` (ROUGE-L F1, Python subprocess to `rouge_score`), `LLMJudgeScorer`
(uses a local judge model to score answer quality on a 1–5 scale, with a
versioned judge prompt in the registry). `LLMJudgeScorer` is the most important:
it approximates human evaluation and generalizes to tasks where exact match and
ROUGE are insufficient.

Benchmark results are written to `evals/results/{dataset_name}/{run_id}.json`.
The schema carries: dataset version, model version, workflow version (git commit),
scorer name, per-item scores, aggregate statistics, and run timestamp. This schema
is the contract between the harness and the CI integration (Sprint 10).

**Acceptance Criteria**
- [ ] Harness runs a workflow in `EvalMode` with deterministic inference and
      a fresh, fixed-corpus vector database
- [ ] `EvalDataset` loaded from versioned JSONL files in `evals/` directory
- [ ] Three scorers implemented: `ExactMatchScorer`, `RougeScorer`, `LLMJudgeScorer`
- [ ] `LLMJudgeScorer` uses a versioned judge prompt from the registry; judge
      model is a local GGUF
- [ ] Results written to `evals/results/` in a stable JSON schema
- [ ] Harness run is reproducible: same inputs produce identical results (modulo
      LLMJudgeScorer variance, which is measured and reported as scorer variance)
- [ ] A run against the RAG pipeline on the NQ eval subset completes and produces
      results within 2 hours on a single GPU workstation

**Implementation Tasks**
```
- [ ] Implement EvalMode workflow runner: pinned model, deterministic qdrant fixture
- [ ] Define EvalDataset JSONL format; implement loader with schema validation
- [ ] Implement ExactMatchScorer
- [ ] Implement RougeScorer (subprocess to Python rouge_score)
- [ ] Implement LLMJudgeScorer: judge prompt template, local model inference, 1–5 scale
- [ ] Define and implement eval result JSON schema
- [ ] Run end-to-end eval on NQ subset; commit first results as baseline
- [ ] Document harness invocation in evals/README.md
```

**Open-Source Components**
```
datasets            — eval corpus loading utilities
rouge_score         — ROUGE-L computation (Python subprocess)
serde_json          — result serialization
```

**Definition of Done**  
Harness runs the full RAG pipeline on a 200-item NQ eval subset; results JSON
committed at `evals/results/nq_dev/baseline.json`; `LLMJudgeScorer` variance
(stddev across 3 runs) is measured and included in the result schema.

---

### `EVAL-9.2 — Prompt Regression Test Suite`

**Sprint:** 09  
**Points:** 5  
**Owner:** ML engineer

**User Story**
```
As a workflow author making a prompt change,
I want a regression test suite that compares the new prompt version against
the prior version on a fixed eval dataset,
So that prompt changes that degrade quality are detected before they reach
the prompt registry's stable channel.
```

**Technical Description**

The prompt registry supports multiple template versions. This sprint adds a
comparison harness: given two template versions and a fixed eval dataset, run the
evaluation harness twice (once per version) and produce a structured diff of the
scorer results.

The comparison report flags: regressions (score decreased by more than a configurable
threshold), improvements (score increased), and neutral changes. It is designed to
be human-readable as a Markdown report and machine-readable as JSON.

The key implementation constraint: the comparison harness must use the same
inference fixture for both runs (same GGUF snapshot, same eval dataset version).
Any difference in results is attributable to the prompt change, not to model or
data variance.

**Acceptance Criteria**
- [ ] Comparison harness runs the eval harness twice with two template versions
- [ ] Same inference fixture used for both runs; variance attributable only to
      the prompt change
- [ ] Comparison report produced as both Markdown and JSON
- [ ] Regressions flagged with configurable threshold (default: -0.05 on primary scorer)
- [ ] Comparison report committed to `evals/comparisons/` with both versions named

**Implementation Tasks**
```
- [ ] Implement comparison harness: run eval twice, diff results
- [ ] Implement Markdown and JSON report generation
- [ ] Implement regression threshold check with configurable threshold
- [ ] Write test: introduce a deliberately worse prompt version, assert regression flagged
```

**Definition of Done**  
Deliberately degraded prompt version is flagged as a regression by the comparison
harness on the NQ eval subset; report committed to `evals/comparisons/`.

---

### `EVAL-10.1 — CI Evaluation Pipeline`

**Sprint:** 10  
**Points:** 8  
**Owner:** Infra + ML engineer

**User Story**
```
As the engineering team,
I want the evaluation harness to run automatically on pull requests that
modify prompt templates or workflow definitions,
So that regressions are caught before merging, not after deployment.
```

**Technical Description**

CI integration for LLM evaluation is non-trivial because evaluation is slow (a 200-item
NQ run takes tens of minutes) and requires model fixtures. The design trades completeness
for speed: the CI pipeline runs a fast eval subset (20 items from the NQ eval, hand-curated
to cover the most failure-prone query types) and uses a quantized 4-bit GGUF model to
reduce inference time. Full eval runs are triggered manually or on a nightly schedule.

The CI pipeline runs in a self-hosted GitHub Actions runner or a local Gitea CI instance
(no external SaaS CI dependency). The runner has the GGUF model fixture and a pre-built
qdrant instance with the eval corpus cached. The pipeline: (1) detects whether any
`prompts/` or `workflows/` files changed in the PR, (2) if so, runs the fast eval subset,
(3) compares results against the committed baseline using the comparison harness, (4) fails
the check if any scorer regresses beyond the threshold.

The eval baseline JSON is committed to the repo. Intentional baseline updates require
a deliberate `make update-eval-baseline` invocation, which re-runs the full eval and
commits the new results. This makes baseline drift visible in git history.

**Acceptance Criteria**
- [ ] CI pipeline detects changes to `prompts/` and `workflows/` in a PR diff
- [ ] Fast eval subset (20-item) runs against the 4-bit GGUF fixture in under 10 minutes
- [ ] Comparison against committed baseline runs; regressions fail the CI check
- [ ] Full eval run triggered manually via `make eval-full`; results committable as
      new baseline via `make update-eval-baseline`
- [ ] CI runs on self-hosted runner (GitHub Actions or Gitea); no external SaaS dependency

**Implementation Tasks**
```
- [ ] Write CI pipeline definition (GitHub Actions YAML or Gitea CI)
- [ ] Write fast eval subset selection script: 20-item curated NQ subset
- [ ] Configure self-hosted runner with GGUF model fixture and qdrant eval corpus
- [ ] Implement PR diff detection for prompts/ and workflows/
- [ ] Write Makefile targets: eval-fast, eval-full, update-eval-baseline
- [ ] Write documentation: how to update baseline, how to interpret CI eval output
```

**Open-Source Components**
```
Gitea / GitHub Actions    — self-hosted CI
```

**Definition of Done**  
A PR that introduces a deliberately regressed prompt version fails the CI eval check;
a PR with no prompt or workflow changes passes without running eval; full eval runs
in under 10 minutes on the 20-item fast subset.

---

### `EVAL-10.2 — Hallucination Classification`

**Sprint:** 10  
**Points:** 8  
**Owner:** ML engineer

**User Story**
```
As the evaluation harness,
I want a hallucination classifier that labels each synthesis output as grounded,
ungrounded, or citation-hallucinated,
So that hallucination rate is a tracked metric in the eval pipeline and
degradations are detectable.
```

**Technical Description**

Hallucination in RAG systems takes three forms: the model makes a claim not
supported by any retrieved chunk (ungrounded), the model attributes a claim to
a chunk that does not contain it (citation hallucination, already partially
detected by the citation extractor in Sprint 4.1), or the model refuses to answer
despite sufficient evidence being present (under-generation). The classifier in
this sprint addresses the first two.

Groundedness scoring uses an NLI (natural language inference) model: for each
factual claim in the answer (extracted by a sentence splitter), the classifier
checks whether any retrieved chunk entails that claim. An open NLI model (e.g.,
`cross-encoder/nli-deberta-v3-small` via HuggingFace) is used. Claims with no
entailing chunk are labeled `ungrounded`. Claims with a `[N]` citation where
chunk N does not entail the claim are labeled `citation_hallucinated`.

The hallucination classifier runs as part of the eval harness scorer chain (as
`GroundednessScorer`). It produces a per-item report: claim count, ungrounded count,
citation hallucination count, and a composite groundedness score (0–1). These are
added to the eval result JSON schema.

**Acceptance Criteria**
- [ ] `GroundednessScorer` implemented as an `EvalScorer` in the harness
- [ ] Sentence splitter extracts factual claims from synthesis output
- [ ] NLI model (local, open) checks each claim against retrieved chunks
- [ ] Per-item report: claim count, ungrounded count, citation hallucination count,
      groundedness score
- [ ] Groundedness score added to eval result JSON schema
- [ ] Hallucination rate included in CI eval check; regressions fail the check

**Implementation Tasks**
```
- [ ] Implement sentence splitter for claim extraction
- [ ] Integrate NLI model via HuggingFace transformers (Python subprocess from harness)
- [ ] Implement GroundednessScorer: per-claim NLI check, score computation
- [ ] Extend eval result JSON schema with groundedness fields
- [ ] Wire GroundednessScorer into CI eval check
- [ ] Write tests: known-grounded answer, known-ungrounded answer, known-citation-hallucinated
```

**Open-Source Components**
```
sentence-transformers    — NLI model inference
datasets                 — eval corpus access
```

**Definition of Done**  
GroundednessScorer correctly labels a known-ungrounded answer and a known-citation-
hallucinated answer in unit tests; groundedness score appears in the CI eval output.

---

### `OBS-11.1 — Distributed Workflow Tracing`

**Sprint:** 11  
**Points:** 8  
**Owner:** Rust backend + infra

**User Story**
```
As an engineer debugging a workflow failure,
I want every node execution, inference call, tool invocation, and state
transition to emit structured trace spans to a local OpenTelemetry collector,
So that the full execution timeline of any workflow run is reconstructable
from the trace backend without reading application logs.
```

**Technical Description**

The `tracing` crate has been in use since Sprint 01, but individual components have
been adding spans without a coordinated schema. This sprint retrofits the engine with
a consistent trace schema and wires all components to a single OpenTelemetry exporter.

The trace schema defines mandatory span attributes for each component type:

- **WorkflowRun span:** run_id, workflow_name, total nodes, status
- **NodeExecution span:** run_id, node_id, node_type, status, duration_ms
- **InferenceCall span:** node_id, backend_type, model_name, prompt_tokens,
  completion_tokens, duration_ms, status
- **ToolCall span:** node_id, tool_name, capabilities_used, duration_ms, status
- **RetrievalCall span:** node_id, query_tokens, results_returned, reranker_used, duration_ms
- **ValidationResult span:** node_id, schema_name, result (pass/extracted/repaired/failed),
  repair_attempts

All spans carry `run_id` and `node_id` as mandatory attributes — these are the join
keys for correlating spans across a workflow. A Jaeger instance runs locally; the
engine exports to it via the OTLP HTTP exporter.

**Acceptance Criteria**
- [ ] All engine components emit spans with the schema defined in this story
- [ ] `run_id` and `node_id` are present on all spans; queries by run_id in Jaeger
      return all spans for that run
- [ ] Inference token counts appear on `InferenceCall` spans; they are the data
      source for cost analysis (Sprint 12)
- [ ] OpenTelemetry exporter configured for local Jaeger OTLP endpoint
- [ ] A workflow with one RAG node and one tool-calling node produces a Jaeger trace
      with all expected span types present

**Implementation Tasks**
```
- [ ] Define mandatory span attribute schema for each component type
- [ ] Retrofit WorkflowGraph scheduler with WorkflowRun and NodeExecution spans
- [ ] Retrofit InferenceBackend implementations with InferenceCall spans
- [ ] Retrofit ToolExecutor with ToolCall spans
- [ ] Retrofit RetrievalNode with RetrievalCall spans
- [ ] Retrofit OutputValidator with ValidationResult spans
- [ ] Configure OpenTelemetry OTLP exporter targeting local Jaeger
- [ ] Write trace completeness test: execute a known workflow, assert all expected
      span types present in Jaeger via the Jaeger query API
```

**Open-Source Components**
```
tracing-opentelemetry    — OpenTelemetry bridge for tracing crate
opentelemetry-otlp       — OTLP exporter
opentelemetry            — span API
Jaeger (local Docker)    — trace backend
```

**Definition of Done**  
A complete RAG + tool-calling workflow produces a Jaeger trace with all expected
span types; `run_id` query in Jaeger returns all spans for the run; inference token
counts are visible in the `InferenceCall` spans.

---

### `OBS-11.2 — Failure Classification and Alerting`

**Sprint:** 11  
**Points:** 5  
**Owner:** Rust backend

**User Story**
```
As an engineer operating Ferrum in a production-like environment,
I want workflow failures classified by type and surfaced as structured events,
So that operational response to failures is guided by failure type rather
than by reading logs.
```

**Technical Description**

Workflow failures currently produce a `WorkflowError` in the engine and a log line.
This sprint adds a failure classification layer that maps `WorkflowError` variants
to structured `FailureEvent` records written to a `failure_events` table and emitted
as OpenTelemetry events on the parent span.

The taxonomy: `InferenceFailed` (model unavailable or timeout), `OutputRepairExhausted`
(persistent schema violation), `ToolCapabilityDenied` (policy enforcement), `IterationLimitExceeded`
(agent runaway), `ReviewRejected` (human rejection), `RetrievalEmpty` (no chunks
returned), `StateWriteFailed` (persistence layer error). Each maps to an operational
response: retry eligible, prompt engineering required, policy configuration review,
budget adjustment, acceptance, query reformulation, infrastructure investigation.

The operational response mapping is stored in a config file (`failure_policies.toml`),
not in the code. An operator can change the retry-eligible set without redeploying.

**Acceptance Criteria**
- [ ] All `WorkflowError` variants map to a `FailureEvent` with classification and
      operational response
- [ ] `FailureEvent` written to `failure_events` table on every workflow failure
- [ ] `FailureEvent` emitted as OpenTelemetry event on parent span; visible in Jaeger
- [ ] Operational response mapping configurable in `failure_policies.toml`
- [ ] `FailureEvent` schema: run_id, node_id, error_type, operational_response,
      timestamp, context (JSON, error-specific)

**Implementation Tasks**
```
- [ ] Define FailureEvent type and failure_events schema; write sqlx migration
- [ ] Implement failure classifier: WorkflowError -> FailureEvent
- [ ] Write failure_policies.toml with retry-eligibility and response guidance
- [ ] Wire failure classifier into workflow error path
- [ ] Emit FailureEvent as OpenTelemetry event on parent run span
- [ ] Write tests for each failure type classification
```

**Open-Source Components**
```
sqlx                     — failure_events persistence
opentelemetry            — event emission on span
toml                     — failure policy config
```

**Definition of Done**  
Each of the seven failure types produces the correct `FailureEvent` classification
in unit tests; events are visible on the parent span in Jaeger for a simulated
workflow failure.

---

### `COST-12.1 — Model Routing by Complexity`

**Sprint:** 12  
**Points:** 8  
**Owner:** Rust backend + ML engineer

**User Story**
```
As the Ferrum execution engine,
I want workflow nodes to be dynamically routed to inference backends based
on a query complexity classification,
So that simple tasks consume small models and complex reasoning tasks consume
larger models, without the workflow author managing routing manually.
```

**Technical Description**

Model routing is the operational lever for cost control. The routing decision
happens at node submission time in the scheduler: before dispatching a node to
an `InferenceBackend`, the scheduler passes the resolved `PromptPayload` to a
`ComplexityClassifier` that returns a `ComplexityBucket` (Simple, Medium, Complex).
The bucket maps to a backend via a `RoutingPolicy` loaded from config.

The classifier uses a lightweight local model — a fine-tuned DeBERTa classifier
is ideal but out of scope for this sprint; instead, a heuristic classifier is
shipped: prompt token count + presence of multi-hop reasoning indicators (heuristic
keyword set) produces a bucket. This is explicitly marked as a v1 heuristic with
a known accuracy limitation; the Sprint 12 open question is whether the heuristic
is sufficient or whether a learned classifier is needed.

The `RoutingPolicy` maps buckets to backend names: e.g., `Simple -> phi3-mini-4bit`,
`Medium -> mistral-7b-instruct-4bit`, `Complex -> mixtral-8x7b-instruct-4bit`. Backends
are registered by name in the engine config. The policy is hot-reloadable from config
without restart.

**Acceptance Criteria**
- [ ] `ComplexityClassifier` returns `ComplexityBucket` for any `PromptPayload`
- [ ] `RoutingPolicy` loaded from config; maps buckets to registered backend names
- [ ] Scheduler consults classifier and policy before dispatching each node
- [ ] Policy is hot-reloadable: config file change reflected in routing within 30 seconds
      without restart
- [ ] Routing decision recorded in node trace span (`ComplexityBucket`, `BackendSelected`)
- [ ] A workflow with mixed simple and complex nodes verifiably dispatches to
      different backends in integration test

**Implementation Tasks**
```
- [ ] Define ComplexityBucket enum and ComplexityClassifier trait
- [ ] Implement heuristic ComplexityClassifier (token count + keyword set)
- [ ] Implement RoutingPolicy: config loading, bucket-to-backend mapping
- [ ] Wire classifier and policy into node dispatch in scheduler
- [ ] Implement hot-reload: watch config file for changes, reload policy
- [ ] Record routing decision in NodeExecution trace span
- [ ] Write integration test: mixed-complexity workflow dispatches to correct backends
```

**Open-Source Components**
```
notify    — filesystem watcher for hot-reload
```

**Definition of Done**  
Integration test asserts that a simple classification node dispatches to the
small model and a multi-hop reasoning node dispatches to the large model; routing
decision appears in trace; config hot-reload confirmed within 30 seconds.

---

### `COST-12.2 — Semantic Response Cache`

**Sprint:** 12  
**Points:** 8  
**Owner:** Rust backend

**User Story**
```
As the inference dispatch layer,
I want semantically similar prompts to return cached responses rather than
triggering new inference calls,
So that token costs for repeated or near-identical queries are eliminated
without requiring exact string match on prompts.
```

**Technical Description**

Exact-match response caching (keyed on the SHA-256 hash from Sprint 2.2) handles
identical prompts but misses the common case: the same question asked with minor
phrasing variations. Semantic caching uses the prompt embedding to check whether
a submitted prompt is within a cosine similarity threshold of a previously answered
prompt.

The semantic cache is a separate qdrant collection (`response_cache`) populated on
every inference call: the rendered prompt body is embedded (via the embedding
service), the vector is upserted with the response, model name, and a TTL timestamp.
Before dispatching to inference, the scheduler queries the cache with the new prompt
embedding; if a result above the similarity threshold (default: 0.95) is returned
and the cached response is within TTL, the cached response is returned as
`InferenceResponse` with `cache_hit: true`.

The 0.95 threshold is conservative by default — semantic caching risks returning
a wrong cached response for a superficially similar but semantically different prompt.
The threshold is configurable per node type; tool-calling nodes default to 1.0
(exact match only) because tool call arguments are sensitive to exact prompt content.

Cache TTL is configurable globally and per-template. Templates that are expected to
return time-sensitive information (e.g., a template that asks about current state)
should set TTL to 0 (no caching). This is declared in the template TOML.

**Acceptance Criteria**
- [ ] `response_cache` qdrant collection populated on every non-cached inference call
- [ ] Cache lookup runs before inference dispatch; hits return cached response
      with `cache_hit: true`
- [ ] Similarity threshold configurable globally and per-node type; tool-calling
      nodes default to 1.0
- [ ] TTL enforced: expired cache entries are not returned; expiry checked at lookup
- [ ] TTL configurable per template in TOML; TTL=0 disables caching for that template
- [ ] Cache hit rate recorded in node trace span and in a `cache_stats` table

**Implementation Tasks**
```
- [ ] Create response_cache qdrant collection with TTL metadata field
- [ ] Implement cache lookup: embed prompt, query qdrant, check threshold and TTL
- [ ] Implement cache write: embed prompt, upsert with response and TTL timestamp
- [ ] Wire cache lookup and write into inference dispatch path
- [ ] Implement per-node-type threshold override in scheduler dispatch
- [ ] Add TTL field to PromptTemplate TOML schema
- [ ] Write cache_stats table schema and sqlx writes
- [ ] Write tests: cache hit within threshold, miss below threshold, TTL expiry,
      tool-calling node bypasses semantic cache
```

**Open-Source Components**
```
qdrant-client    — response_cache collection operations
```

**Definition of Done**  
Cache hit test: same question rephrased (within 0.95 cosine similarity) returns
cached response; TTL expiry test: cached response not returned after TTL;
tool-calling node test: always misses cache regardless of similarity.

---

## System Evolution Narrative

**Sprints 1–2 (Apr 7 – May 2): Execution substrate.**  
The engine can represent and execute a typed workflow graph with deterministic replay.
An engineer can define a multi-node workflow as a DAG, submit it to the local engine,
and receive a serialized trace. Inference calls route through the backend abstraction;
both the in-process Candle backend and the local llama-cpp server backend are
functional. Prompt templates are versioned and logged. Nothing retrieves documents
yet; nothing calls external tools yet. But the execution substrate is load-bearing:
every subsequent sprint builds on the scheduler, the backend trait, and the prompt
registry established here.

**Sprints 3–4 (May 5 – May 30): RAG pipeline.**  
The engine ingests documents, retrieves relevant chunks, assembles context, and
synthesizes grounded answers with source citations. The retrieval stack (qdrant +
tantivy, with cross-encoder reranking) is benchmarked against NQ dev and baseline
numbers are committed. The synthesis node enforces a token budget and logs truncation
decisions. The first end-to-end RAG workflow is runnable. Quality is measured but not
yet enforced by CI — the evaluation framework is not built until Sprint 9.

**Sprints 5–6 (Jun 2 – Jun 27): Agent capabilities.**  
The engine gains a typed tool registry, grammar-constrained function calling for local
models, a bounded ReAct execution loop with crash recovery, and cross-session memory.
An agent can now perform multi-step tasks against local data sources (SQLite, tantivy
search, HTTP) with auditable tool call history. The iteration limit and token budget
enforcements mean the agent cannot run unboundedly. Session memory means multi-session
agents accumulate structured context without growing the context window.

**Sprints 7–8 (Jul 7 – Aug 1): Reliability and human oversight.**  
The engine gains structured output validation and repair, durable workflow state
persistence with crash recovery at any node boundary, and a human review queue with
a minimal approval UI and audit log. These three sprints collectively address the
failure modes that would make the engine unsafe to use for consequential decisions:
schema violations are caught and repaired or surfaced; run state is never lost to
infrastructure failures; policy-sensitive outputs are interceptable before downstream
effects.

**Sprints 9–10 (Aug 4 – Aug 29): Evaluation infrastructure.**  
The offline evaluation harness, prompt regression comparison, CI integration, and
hallucination classifier are built and producing results. The baseline numbers are
committed. CI enforces that prompt changes do not silently degrade quality. The
groundedness scorer gives the team a leading indicator of hallucination rate as
the prompt corpus and model versions evolve. At the end of Sprint 10, the engine is
evaluable: any change to prompts, retrieval configuration, or model versions produces
a measurable quality signal before it reaches staging.

**Sprints 11–12 (Sep 1 – Sep 26): Observability and cost control.**  
The full execution trace is visible in Jaeger with a consistent span schema. Workflow
failures are classified and operationally actionable. Model routing dispatches simple
tasks to small models and complex reasoning to large models, with the routing decision
visible in the trace. The semantic cache eliminates redundant inference for
near-duplicate queries. At the close of Sprint 12, the engine is operable: an
engineer can respond to a failure by reading a trace, understand cost distribution
by query type, and tune routing policy without a code change.

---

## Expertise Signal Appendix

### What this half-year proves you can do

**Workflow execution and DAG scheduling**  
Stories: CORE-1.1, AGENT-6.1  
*What a non-expert gets wrong:* Encoding execution order in imperative control flow
instead of deriving it from data dependencies; missing the concurrent state mutation
problem when multiple nodes complete simultaneously.  
*The correct insight:* Workflow graphs are not scripts. The scheduler is a dependency
resolver operating over a typed state store; execution order is a derived property,
not an input.

**Inference backend abstraction**  
Stories: CORE-1.2, COST-12.1  
*What a non-expert gets wrong:* Coupling token counting, retry logic, and backend
selection to the inference call site; making model routing a global switch rather
than a per-node property.  
*The correct insight:* The backend trait surface should be minimal and typed. Token
counts are a backend-layer responsibility. Routing is a scheduler-layer decision made
on a resolved `PromptPayload`, not a config-time decision.

**Prompt engineering as a versioned artifact lifecycle**  
Stories: PROMPT-2.1, PROMPT-2.2, EVAL-9.2  
*What a non-expert gets wrong:* Treating prompts as strings in source code; no
mechanism to detect regressions when a prompt changes.  
*The correct insight:* Prompts have a semver lifecycle. The prompt registry, the
prompt trace log, and the comparison harness are three parts of the same system:
you cannot evaluate a prompt change without the other two.

**RAG system design**  
Stories: RAG-3.1, RAG-3.2, RAG-4.1, EVAL-10.2  
*What a non-expert gets wrong:* Using character-based chunking; not enforcing
embedding model version consistency between ingestion and retrieval; treating
citation extraction as optional.  
*The correct insight:* The chunker's unit of work is tokens, not characters.
Embedding model mismatch is a silent correctness failure. Citation extraction is
the primary signal for distinguishing hallucination from retrieval failure.

**Agent loop design**  
Stories: AGENT-6.1, AGENT-6.2  
*What a non-expert gets wrong:* Implementing the loop without iteration limits or
token budgets; using conversation history as an unbounded append-only string;
conflating within-session context management with cross-session memory.  
*The correct insight:* Every agent loop must have a hard termination condition.
The context window is a resource that must be budgeted, not a buffer that fills
until it overflows. Session memory is a structured extract, not a transcript.

**Evaluation methodology**  
Stories: EVAL-9.1, EVAL-9.2, EVAL-10.1, EVAL-10.2  
*What a non-expert gets wrong:* Using live model inference for CI evaluation (non-reproducible);
using only ROUGE or exact match as scorers; not measuring hallucination rate as a
first-class metric.  
*The correct insight:* CI evaluation requires a pinned model fixture and a deterministic
corpus. LLM-as-judge is the only scorer that generalizes to open-ended tasks.
Groundedness scoring is the metric that distinguishes "model is wrong" from
"retrieval failed to provide evidence."

**Observability for multi-step LLM systems**  
Stories: OBS-11.1, OBS-11.2  
*What a non-expert gets wrong:* Treating LLM application observability as logging;
missing `run_id` as a mandatory join key across all spans; not distinguishing failure
types operationally.  
*The correct insight:* An LLM workflow is a distributed system. The trace is the
primary debugging surface, not the log. Every span needs `run_id` and `node_id`
as mandatory attributes or the trace is unqueryable. Failure classification is an
operational contract, not a log message.

**Cost management**  
Stories: COST-12.1, COST-12.2  
*What a non-expert gets wrong:* Applying caching at the session level; using exact
string match for cache keys; setting a universal similarity threshold regardless of
node type.  
*The correct insight:* Token cost attribution must be per-node, not per-session.
Semantic caching threshold must be conservative and type-aware: tool-calling nodes
require exact match because argument precision matters; free-text synthesis nodes
tolerate semantic similarity.

---

*End of artifact. Version 0.1.0. Ready for sprint planning.*

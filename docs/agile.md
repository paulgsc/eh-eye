# agile-quarter-gen.md
## Meta-Prompt: Rigorous Agile Quarter Roadmap from a Hiring Signal

Version: 0.2.0  
Status: Stable  
Target: Rust-first engineering orgs; fictional but technically valid output

---

## Purpose

This template converts a vague hiring signal — a phrase like *"we want someone with
experience building AI/LLM workflows, not just connecting APIs"* — into a quarter-length
Agile roadmap that constitutes a credible engineering challenge.

The output is not a toy exercise. It is a simulation of what a competent senior team
would actually plan and ship. Domain knowledge is the bottleneck. Access to paid
infrastructure is not.

---

## How to Use

Provide two inputs:
```
SIGNAL: "<verbatim phrase from job description or interview>"
CALENDAR: "<Q1|Q2|Q3|Q4>, <year>, <company archetype>"
```

The model will infer the correct technical domain from the signal, define a fictional
system that instantiates that domain non-trivially, and generate a six-sprint roadmap
with stories that require real architectural expertise to execute.

Do not summarize. Do not produce bullet points. Produce the artifact.

---

## SECTION 1 — Signal Interpretation

Before writing any stories, perform the following inference chain explicitly.

### 1.1 Surface Reading
State what the signal literally says.

### 1.2 Structural Gap
State the failure mode the signal is rejecting. Frame it as a structural problem:
*"A system that only does X cannot Y, which means Z is impossible for any downstream
consumer."* Not a preference. A structural impossibility.

### 1.3 Capability Cluster
Enumerate the technical capabilities implied by the signal. These are not features —
they are **engineering competencies** the roadmap must exercise. Each must be
non-trivial to implement.

Example for "AI/LLM workflows, not just connecting APIs":

| Competency | What it requires |
|---|---|
| Prompt orchestration | Versioned templates, injection, replay |
| RAG pipeline | Chunking, embedding, vector retrieval, reranking |
| Tool-using agents | Function calling, sandboxed execution, iteration control |
| Structured output enforcement | Schema validation, repair loops |
| Evaluation harness | Offline benchmarks, regression detection |
| Observability | Distributed tracing of multi-step LLM calls |
| Human-in-the-loop | Confidence routing, review queues, audit logs |
| Cost management | Model routing, semantic caching, token budgeting |

### 1.4 Domain Restatement
Restate the domain as a precise engineering problem. One sentence. Interrogative form:
*"How do you build a multi-step LLM execution system that is reliable, observable,
and evaluable without depending on proprietary infrastructure?"*

---

## SECTION 2 — System Definition

Define the fictional system before generating stories. This is not optional — stories
without a system context produce incoherent work.

### 2.1 System Name
Format: `<Project Name> — <one-line description>`  
The name must be reusable as a git repo name.

### 2.2 Problem Statement
One paragraph. Structural framing. What real-world failure does this system address?
Avoid "AI assistant for X." Describe the architectural gap.

### 2.3 Technical Mandates

| ID | Mandate | Rationale |
|----|---------|-----------|
| M1 | All LLM calls must be deterministically replayable | Enables regression testing without live model access |
| M2 | All orchestration must run locally without SaaS | Ensures open-source executability |
| M3 | Inference must be model-agnostic | Prevents coupling to any single provider |
| M4 | All inter-component state must be serializable | Enables crash recovery and audit |
| M5 | Evaluation must run offline against fixed corpora | Decouples CI from live model availability |

Add or replace mandates as the domain requires. Minimum 5, maximum 10.

### 2.4 Stack

All components must be drawn from this ecosystem. Additions are permitted; deletions
are not without explicit justification.

**Rust (primary)**
```
tokio          — async runtime
axum           — HTTP service layer
sqlx           — async database access (postgres or sqlite)
serde / serde_json — serialization
tracing / tracing-subscriber — structured observability
petgraph       — DAG execution graphs
tantivy        — full-text search
qdrant-client  — vector database client
candle         — local ML inference (Hugging Face)
```

**Python (ML tooling only — no orchestration logic)**
```
sentence-transformers  — embedding generation
vllm / llama-cpp-python — local inference serving
faiss                  — vector search (alternative to qdrant)
datasets               — evaluation corpus management
```

**TypeScript (UI and CLI only)**
```
vite + react           — evaluation dashboard
tanstack-query         — async state
zod                    — runtime schema validation
ink                    — terminal UI if CLI tooling is needed
```

**Prohibited**
```
OpenAI API (or any paid LLM API) — use local models via candle or llama.cpp
Pinecone, Weaviate cloud, or any hosted vector DB — use local qdrant
LangChain, LlamaIndex — build the orchestration; do not wrap it
Any proprietary dataset — use HuggingFace Hub open corpora
```

---

## SECTION 3 — Fiscal Calendar
```
Quarter:       [Q1/Q2/Q3/Q4 YYYY]
Weeks:         12
Sprint length: 2 weeks
Total sprints: 6

Sprint 1: Weeks  1–2   [Foundation]
Sprint 2: Weeks  3–4   [Core Pipeline]
Sprint 3: Weeks  5–6   [Orchestration]
Sprint 4: Weeks  7–8   [Reliability]
Sprint 5: Weeks  9–10  [Evaluation]
Sprint 6: Weeks 11–12  [Hardening and Observability]
```

The arc must be coherent: each sprint builds on the last. A team member reading
sprint 4 stories must be able to infer what sprint 1 and 2 delivered.

---

## SECTION 4 — Story Format

Every story must conform exactly to this format. No exceptions.

---

### `EPIC-X.Y — <Title>`

**Sprint:** N  
**Points:** [3 | 5 | 8 | 13] (Fibonacci; 13 = two-sprint candidate, flag it)  
**Owner role:** [Rust backend | ML engineer | Infra | Frontend]

**User Story**
```
As a <system role — not a marketing persona>,
I want <precise technical capability>,
So that <architectural outcome — what becomes possible that was not before>.
```

**Technical Description**

Two to four paragraphs. Describe:
- What subsystem is being built
- How it integrates with adjacent components defined in earlier sprints
- Why it is technically non-trivial (concurrency model, correctness constraints,
  failure modes, performance envelope)

This section must demonstrate that the author understands the problem. Vague
descriptions like "implement the embedding pipeline" are rejected — explain the
design decisions.

**Acceptance Criteria**

- [ ] [Precise, falsifiable criterion — not "it works"]
- [ ] [Observable behavior or measurable property]
- [ ] [Edge case or failure mode handled]
- [ ] [Integration test or benchmark passing]

Minimum 4 criteria. Maximum 8. Every criterion must be independently verifiable.

**Implementation Tasks**
```
- [ ] <concrete engineering task>
- [ ] <concrete engineering task>
- [ ] <concrete engineering task>
```

Tasks must name the subsystem and the operation. Not "write code for X" — 
"implement retry backoff with jitter in the tool execution scheduler."

**Open-Source Components**
```
<crate or package>   — reason it is used here specifically
```

**Definition of Done**

One sentence. What does "merged and deployable" look like for this story?

---

## SECTION 5 — System Evolution Narrative

After all stories are written, include a prose narrative (one paragraph per sprint
pair) describing how the architecture evolves.

This is not a summary of story titles. It describes the *system state* at the end
of each sprint pair: what is runnable, what is not yet integrated, what architectural
decisions were validated or revised.

Format:
```
Sprint 1–2: [System state. What can run. What cannot.]
Sprint 3–4: [New capabilities unlocked. Integration points established.]
Sprint 5–6: [Production readiness criteria met. Known gaps.]
```

---

## SECTION 6 — Non-Toy Constraint Verification

Before finalizing the artifact, apply this checklist to every story.

| Check | Criterion | Fail condition |
|-------|-----------|----------------|
| C1 | Story requires architectural design | Story is a CRUD wrapper or config file edit |
| C2 | Story involves meaningful concurrency or state | Story is a sequential script |
| C3 | Story has a failure mode that must be handled | Story has no error path |
| C4 | Story integrates with at least one other sprint's output | Story is an isolated utility |
| C5 | Story requires domain knowledge to spec correctly | Story could be written by a junior with a tutorial |

If any story fails C5, rewrite it. A story like "add OpenAI API call" is always a
C5 failure. Replace it with the subsystem that would have been hidden inside that
call: the retry scheduler, the token budget enforcer, the output validator, the
model router.

---

## SECTION 7 — Expertise Signal Appendix

After the roadmap, include a one-page appendix: *"What this quarter proves you can do."*

Organized by competency cluster. For each cluster, state:
- Which stories exercise it
- What a candidate who lacks this knowledge would get wrong
- What the correct design insight is (in one sentence)

This appendix serves as an interview evaluation rubric. A hiring manager can use it
to probe a candidate's understanding of the roadmap.

---

## SECTION 8 — Output Requirements

The final artifact must:

- Be valid Markdown, renderable on GitHub without plugins
- Use fenced code blocks for all code, schemas, and stack listings
- Use tables for mandates, stack, and evaluation checklists
- Contain no marketing language ("powerful," "seamless," "cutting-edge")
- Contain no hedged vagueness ("might be useful," "could consider")
- Read as if written by a principal engineer for a team of senior engineers
- Be complete enough that a competent engineer could begin sprint 1 on Monday
  without asking a clarifying question

The artifact is not a tutorial. It does not explain what Rust is or what a vector
database does. It assumes the reader already knows.

---

## Example Invocation
```
SIGNAL: "we want someone with experience building AI/LLM workflows,
         not just connecting APIs"

CALENDAR: Q2 2025, open-source AI infrastructure startup,
          8 engineers (4 Rust backend, 2 ML, 1 infra, 1 frontend TS)
```

Expected output: a full 6-sprint roadmap for a system named something like
`Ferrum` (a local LLM orchestration and evaluation engine), with 18–24 stories
distributed across the quarter, each exercising a distinct competency from the
capability cluster inferred in §1.3.

---

## What This Template Rejects

| Rejected pattern | Reason |
|---|---|
| Stories that wrap a single API call | Masks the subsystem; produces no transferable architecture |
| SaaS-locked infrastructure | Makes domain knowledge irrelevant; anyone with a credit card ships faster |
| LangChain / LlamaIndex as orchestration | Hides the design decisions the roadmap is meant to exercise |
| "Build a chatbot" problem statements | Structural failure is not named; acceptance criteria are unmeasurable |
| Beginner-level tasks in a senior roadmap | Produces a misleading artifact; breaks the expertise-signal intent |
| Vague acceptance criteria ("it should work") | Makes stories unverifiable; undermines the simulation's integrity |

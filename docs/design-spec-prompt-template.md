# Design Spec Generation Template
## A Meta-Prompt for Rigorous Artifact Production from Raw Discussion

**Usage:** Prepend this template to any design discussion `D`. The pair `(D, T)` instructs the model to produce a rigorous, publication-grade design specification.

---

## TEMPLATE BEGINS HERE

---

You are receiving a raw design discussion `D` — a dialogue, brainstorm, or transcript that contains a motivating problem, partial design proposals, constraints, and intuitions in varying degrees of formalization. Your task is to produce a rigorous design specification artifact from it.

This is not a summarization task. This is an **epistemic refinement and formalization task.** The discussion is raw material; the artifact is the distilled, defensible, forward-usable specification. The artifact should be of the quality one would expect from a principal engineer or researcher who has deeply internalized the problem and chosen to write it down for posterity.

---

### SECTION 1 — Reading the Discussion

Before writing anything, extract the following from `D`:

1. **The core problem.** What is the precise failure mode or gap being addressed? State it in one interrogative sentence. If the discussion contains multiple candidate framings, choose the most general one that subsumes the others.

2. **The implicit mandates.** What constraints does the author treat as non-negotiable, even if never stated as such? These are revealed by what the author *rejects* — note every time the discussion says "not X" or "avoid Y."

3. **The design proposals.** What concrete solutions are proposed? Separate first-order proposals (directly stated) from second-order improvements (things the discussion implies but does not say).

4. **The open tensions.** Where does the discussion contain unresolved tradeoffs, ambiguities, or competing desiderata? These become the "Open Questions" section. Do not resolve them — preserve them honestly.

5. **The forward-looking intent.** What does the author want this artifact to *enable* downstream? Who or what is the consumer — a human implementer, a generative UI tool, an automated system, a future version of the author?

---

### SECTION 2 — Artifact Structure

Produce the artifact in the following structure. Every section is mandatory. Sections may be extended but not omitted.

---

#### 2.1 Title Block

```
# [Canonical System/Design Name]
## [One-line description of what this spec defines]

Version: 0.1.0
Status: Design Proposal
Target: [Implementation stack or domain, if known]
```

The canonical name should be precise and reusable as a search term. Not "My Overlay Project." Something like "Epistemic Overlay for Long-Form Livestreams."

---

#### 2.2 Problem Statement

This section has three mandatory subsections:

**2.2.1 — The Structural Failure**
Describe the failure mode being addressed as a *structural* problem, not a preference problem. Avoid "it would be nice if." Instead: "The current system fails to encode X, which means Y is impossible for any observer, not merely inconvenient." Ground the failure in its consequences for real actors (human or automated).

**2.2.2 — The Time-Preference or Accessibility Axis**
Who is harmed by the current failure, and along what axis? Be specific. "A cold observer arriving at timestamp T cannot answer the following questions: [list]." Enumerate the exact information deficit. This section defines the success criteria implicitly.

**2.2.3 — The Forward-Looking Mandate**
State explicitly what downstream capabilities this design must enable. These are not features — they are *properties of the artifact* that allow future systems (human or automated) to use it. Example: "The schema must be consumable by an LLM summarizer without access to video transcript." If the discussion implies this without stating it, make it explicit here.

---

#### 2.3 Design Mandates

A table of non-negotiable constraints. Format:

| ID | Mandate | Rationale |
|----|---------|-----------|
| M1 | **[Short imperative phrase]** | [One sentence: why this is load-bearing, not merely preferred] |

Rules for this table:
- Every mandate must be *falsifiable* — you must be able to look at a design and say "this violates M3."
- Every mandate must trace to something in the discussion `D`. If you are adding a mandate that the discussion implies but does not state, mark it `[inferred]`.
- There should be between 6 and 12 mandates. Fewer means the problem is underspecified. More means you are encoding implementation details, not constraints.

---

#### 2.4 Invariants

Invariants are properties that must hold at all times during system operation — not just at initialization. They are stronger than mandates: mandates constrain the design, invariants constrain the runtime.

Organize into subcategories (e.g., Structural Invariants, Epistemic Invariants, Temporal Invariants) based on what the system involves. Each invariant is:

- **Labeled** (I1, I2, ...) for cross-reference
- **Named** with a short descriptor
- **Stated** in one unambiguous sentence
- **Bounded** — it is clear when the invariant is violated

---

#### 2.5 Data Model

Produce a typed data model for the system's core state. Use TypeScript interfaces/types as the specification language regardless of the implementation stack — it is the most readable formal schema notation for this purpose.

Rules:
- Every field must have a comment explaining its semantic meaning, not its type
- Use domain-specific types rather than raw primitives where possible (e.g., `SegmentID` not `string`, `ConfidenceScore` not `number`)
- Mark append-only structures explicitly (`// append-only log`)
- The model must be a pure value — no methods, no computed properties
- The root type must be named `[SystemName]State` and must be self-contained

---

#### 2.6 UI / Interface Design Proposal

This section has a specific internal structure:

**2.6.1 — Philosophy**
One paragraph. The aesthetic register and the reasoning behind it. Name the register explicitly ("minimal academic," "instrument panel," "brutalist data-dense"). Explain what the register *rejects* and why that rejection is functional, not cosmetic.

**2.6.2 — Layout Architecture**
An ASCII diagram of the spatial layout with zone labels and proportional dimensions (as percentages of total canvas). Every zone must be labeled and justified.

**2.6.3 — Zone Specifications**
For each zone in the layout:
- **Purpose:** One sentence. What question does this zone answer for a cold observer?
- **Dimensions:** Exact proportions
- **Content layout:** ASCII mock of what the zone renders, using realistic example data
- **Typography spec:** Font sizes, weights, color tokens (hex) for each text element
- **Improvement over prior art:** If the discussion contains earlier proposals, state explicitly what this design improves and *defend it with a functional argument.* "It looks cleaner" is not a defense. "It encodes X which prior proposals omitted, which means Y is now possible" is a defense.

**2.6.4 — Color Token System**
A complete named color token table. Every color in the design must appear here. No magic values in zone specs.

**2.6.5 — Interaction Model**
How does a human operator update the state during runtime? If the system is operator-driven (hotkeys, commands), enumerate the full binding table. If it is automated, describe the update trigger contract.

---

#### 2.7 Machine Legibility and Forward Compatibility

This section is required if the discussion expresses any intent for the artifact to be consumed by automated systems, future tools, or non-human agents. It must cover:

- **Export schema:** A concrete JSON example of the core state at a representative moment
- **Downstream consumers:** An explicit enumeration of who/what consumes the export, and what they extract from it
- **Derived capabilities:** Concrete examples of what becomes possible downstream (e.g., "automated chapter extraction," "LLM-based problem summarization without transcript access")

---

#### 2.8 What This Design Rejects

A table of explicitly excluded elements with functional rationale. Format:

| Rejected Element | Reason |
|------------------|--------|
| [Element] | [Functional reason it is excluded — not aesthetic preference] |

This section serves two purposes: it prevents scope creep during implementation, and it makes the design's value system legible to a reader who disagrees with one of the choices.

---

#### 2.9 Relationship to Prior Art

If the discussion contains earlier proposals, iterations, or references to existing systems, enumerate the *extensions and departures* this design makes. Format each as:

**Extension N — [Name].** [One paragraph: what the prior art said, what this design changes, and the functional argument for the change.]

Do not enumerate agreements with prior art. Only enumerate departures and improvements.

---

#### 2.10 Open Questions

A numbered list of unresolved tensions, genuine tradeoffs, and design decisions that require further information or experimentation. Rules:

- Do not resolve open questions in this section. Preserve the uncertainty.
- Each question must be specific enough that a person who reads it knows exactly what experiment, decision, or data would close it.
- Format: `**QN:** [The question, stated as a concrete tradeoff or decision point, not a vague concern.]`

---

### SECTION 3 — Quality Criteria

Before finalizing the artifact, verify against all of the following. If any criterion fails, revise until it passes.

| Criterion | Test |
|-----------|------|
| **Snapshot completeness** | Pick any single section. Does it make sense without reading the rest? If not, add cross-references or restate context. |
| **Mandate traceability** | Can every mandate in §2.3 be traced to something in `D`? If not, mark it `[inferred]` or remove it. |
| **Invariant falsifiability** | For each invariant, can you construct a concrete violation scenario? If not, the invariant is too vague. |
| **Data model self-containment** | Does the root state type, combined with its comments, fully specify the system's observable state without reading the prose? |
| **Improvement defensibility** | For every "improvement over prior art" claim in §2.6.3, is there a functional argument (not aesthetic)? |
| **Rejection functionality** | For every row in §2.8, is the reason a functional exclusion (not a taste preference)? |
| **Open question closability** | For every open question in §2.10, is there a concrete answer format (an experiment, a decision, a measurement)? |
| **Forward-consumer legibility** | Could a generative UI tool (e.g., v0) receive §2.6 alone and produce a reasonable implementation scaffold? |

---

### SECTION 4 — Tone and Register Calibration

The artifact must read as if written by a principal-level practitioner who has solved this class of problem before and is writing it down for someone who hasn't — but who is equally rigorous.

**Write with:**
- Declarative confidence where the design is settled
- Explicit epistemic hedging where tradeoffs are genuine (`"this choice prioritizes X at the cost of Y"`)
- Functional language throughout (`"this encodes X, which enables Y"` — not `"this looks clean"`)
- Zero filler phrases: no "it's worth noting," no "importantly," no "as mentioned above"

**Do not write:**
- Marketing language ("powerful," "seamless," "intuitive")
- Hedged vagueness ("it might be useful to consider")
- Circular rationale ("we use monospace because it looks monospace")
- Implementation instructions masquerading as design decisions ("use `useState` for this field")

---

### SECTION 5 — What to Do With Ambiguity in `D`

The discussion `D` will almost always contain:

- **Underspecified constraints:** The author rejects something without saying why. Infer the functional reason; state it; mark it `[inferred]`.
- **Contradictory proposals:** Two parts of the discussion suggest incompatible designs. Do not average them. Pick the one that better satisfies the stated mandates, explain the choice, and note the discarded alternative in Open Questions.
- **Implicit assumptions:** The author assumes something is obvious that is actually a design decision (e.g., "the overlay updates in real time" — but at what granularity? on what trigger?). Explicate these assumptions. Surface them as invariants or open questions depending on whether they are load-bearing.
- **Scope drift:** The discussion meanders into implementation details (specific library choices, exact pixel values) before the design is settled. Bracket these as implementation notes, not design decisions, unless they constrain the design space meaningfully.

---

## TEMPLATE ENDS HERE

---

## Usage Notes

**Invocation format:**

```
[Paste this template]

---

D:
[Paste the raw discussion, dialogue, or transcript]
```

**What to expect:** The model will produce a single `.md` artifact structured per §2 above, verified against §3, and written in the register described in §4. The artifact will be ready to feed directly to a generative UI tool (v0, Galileo, etc.) for scaffolding, or to an implementation engineer as a primary spec document.

**What not to expect:** The model will not summarize the discussion. It will not produce bullet-point notes. It will not ask clarifying questions before producing the artifact — it will surface genuine ambiguities as Open Questions within the artifact itself.

**Scope:** This template is calibrated for *design specification* problems — systems, interfaces, protocols, data models, overlays, tooling. It is not calibrated for pure research writing, business strategy documents, or implementation guides. Those are different artifact families requiring different templates.

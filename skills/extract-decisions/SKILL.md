---
name: extract-decisions
description: Extract structured design decisions from a technical document. Reads the document, identifies architectural choice points, interviews the author to fill gaps, and writes complete decision frameworks as markdown and JSONL files. Use when you have a design guide, architecture spec, or technical document and want to capture its decision knowledge in a form AI systems can reason over.
---

# Design Decision Extraction

Extract the design decisions embedded in a technical document. Each decision captures the alternatives, the conditions under which you'd choose each, the trade-offs, the implications, and the evidence — structured so AI systems can reason over them.

## Overview

This skill runs a five-phase workflow:

1. **Inventory** — Read the document and identify all architectural choice points
2. **Filter** — Present candidates; user removes any that aren't worth capturing
3. **Gap Analysis** — Assess which of the 5 decision elements exist in the document vs. need to be filled in
4. **Elicitation** — Interview the user one question at a time to fill the gaps
5. **Synthesis & Output** — Write complete decision frameworks to disk as markdown + JSONL

## Starting the Workflow

Ask the user for the document path if they haven't provided one. Then begin Phase 1.

---

## Phase 1: Inventory

Read the document using the Read tool. If the document is very large (over 800 lines), read it in sections.

Then identify design decision candidates by thinking like a **Network Architecture and Design Analyst**:

A valid design decision:
- Involves choosing between 2 or more viable alternatives
- Has situations or conditions where you'd choose one over another
- Affects deployment architecture (not just operations or troubleshooting)
- Is discussed or implied in the document

Look for:
- Technology selection (X vs Y)
- Deployment topology choices (centralized vs distributed, active-active vs standby)
- Redundancy and failover strategies
- Control plane or data plane design choices
- Scaling strategies
- Integration approaches
- Component placement (on-prem vs cloud)
- Transport and connectivity choices

Skip purely operational content (how to run or maintain something after it's deployed).

For each candidate, note:
- A draft question framing the choice ("Should you use X or Y?" or "When to use X vs Y?")
- The alternatives you identified
- Where in the document it appears
- Your confidence (high / medium / low) and a brief reason

---

## Phase 2: Filter

Present the candidates as a numbered table:

```
#  | Decision                          | Alternatives          | Confidence
---|-----------------------------------|-----------------------|----------
1  | Should you use X or Y?            | X, Y                  | ● high
2  | When to use centralized vs dist.? | Centralized, Dist.    | ◐ medium
```

Legend: ● high  ◐ medium  ◑ low

Tell the user the total count and ask them to remove any they don't want to capture. They can specify:
- Single numbers: `3`
- Ranges: `1-5`
- Mixed: `1-3, 5, 7-9`

Confirm the final list before proceeding.

---

## Phase 3: Gap Analysis

For each selected candidate, assess it as a **Gap Analyst**:

The five required decision elements are:
1. **alternatives** — Are the 2+ alternatives each described with purpose, characteristics, and contrast?
2. **conditions** — Are the conditions where you'd choose each alternative explicitly explained and quantified?
3. **tradeoffs** — Are the trade-offs between alternatives (cost, complexity, flexibility, etc.) documented?
4. **implications** — Are the downstream consequences of each choice clear?
5. **evidence** — Is there evidence, examples, or validation for the guidance?

Classify each element:
- **SUFFICIENT** — source contains synthesis-ready detail
- **WEAK** — partial content but vague, unquantified, or incomplete
- **ABSENT** — nothing usable in the document

For every WEAK or ABSENT element, prepare 2–3 targeted elicitation questions.
For every SUFFICIENT element, prepare 1 expansion question asking the expert to confirm and elaborate in their own words.

Show the user a brief gap summary before starting elicitation:

```
Decision 1: Should you use X or Y?
  ✓ alternatives (sufficient)   ? conditions (weak)   ✗ tradeoffs (absent)
  ✗ implications (absent)       ✓ evidence (sufficient)
  → ~6 questions estimated
```

---

## Phase 4: Elicitation

Work through each decision one at a time. For each decision:

1. Show a one-line header: `Decision 2/5 — "Should you use X or Y?" — Question 3/7`
2. For WEAK elements, show the relevant excerpt from the document first, then ask the question so the user refines rather than starts from scratch
3. Ask one question and wait for the answer before asking the next

**User commands to accept at any prompt:**
- Any answer — record it and move to the next question
- `skip` — skip this question, move on
- `defer` — mark for later, move on
- `in the doc` — note that the answer is already in the source material
- `save` — checkpoint progress and continue
- `quit` — save progress and stop (can resume later)

After all questions for a decision are complete, show the user a summary of their answers and ask if they want to correct anything before synthesis.

**Key rule: one question at a time. Never present multiple questions together.**

---

## Phase 5: Synthesis and Output

For each completed decision, act as a **Decision Framework Synthesizer**:

Write a complete, self-contained decision framework. Every sentence must stand alone — no references to other sections, figures, or external documents.

**Writing rules:**
- Write in clear, direct prose — no marketing language
- Quantify conditions where possible ("more than 100 branches" not "large deployments")
- Name trade-offs with their conditions and impact
- Explain downstream consequences and which decisions they affect
- Include examples, validation results, or real-world outcomes for evidence
- Annotate every sentence with its origin: `[S]` source-derived, `[E]` elicited, `[I]` inferred from domain knowledge
- Minimum content: 1–2 sentences per alternative, 1–2 sentences per condition, 1–2 sentences per implication, 2+ sentences for evidence
- No placeholder dashes, no `[MISSING: ...]` markers — every section must have substantive content

**Decision framework format:**

```markdown
**Decision Question:** [The architectural choice point]

**Alternatives:**
- **[Alternative 1]:** [1-2 sentences: what it is, purpose, key characteristics] [S/E/I]
- **[Alternative 2]:** [1-2 sentences: what it is, purpose, key characteristics] [S/E/I]

**When to Use Each Alternative:**
- **[Alternative 1]:** [Conditions/situations, quantified where possible] [S/E/I]
- **[Alternative 2]:** [Conditions/situations, quantified where possible] [S/E/I]

**Trade-offs:**
- **[Alt 1 vs Alt 2]:** [Cost, complexity, flexibility, performance — with conditions] [S/E/I]

**Implications:**
- **[If you choose Alt 1]:** [Downstream decisions and consequences] [S/E/I]
- **[If you choose Alt 2]:** [Downstream decisions and consequences] [S/E/I]

**Evidence:** [Examples, validation results, real-world outcomes] [S/E/I]
```

After writing, re-read every sentence and flag any that reference something outside the framework itself.

**Write the output immediately after each decision is synthesized** — don't wait until all decisions are complete.

### Output files

Write to the directory the user specifies (default: `./output/`):

**Per decision — markdown file** (`output/<decision-id>.md`):
The full decision framework with `[S]`/`[E]`/`[I]` annotations.

**Aggregate JSONL file** (`output/rag/chunks.jsonl`):
One JSON object per line, one per section per decision. Append as each decision completes.

```json
{"decision_id": "decision_001", "decision_question": "Should you use X or Y?", "section": "alternatives", "text": "Alternative 1 is... Alternative 2 is...", "source_doc": "filename.pdf", "extracted_date": "2026-05-10", "has_inferred_content": false}
```

Sections: `anchor`, `alternatives`, `when_to_use_each`, `tradeoffs`, `implications`, `evidence`

Strip `[S]`, `[E]`, `[I]` annotations from JSONL text (keep them only in the markdown).
Set `has_inferred_content: true` if any sentence in that section has an `[I]` annotation.

---

## Resuming an Interrupted Session

If the user says they want to resume, ask for the output directory where the previous session's files are. Read any existing `.md` files there to understand which decisions are already complete, then continue with the remaining ones.

---

## Output Summary

When all decisions are complete, report:
- How many decision frameworks were written
- Where the files are
- Total questions asked vs. answered
- Any decisions that have inferred content (flag for expert review)

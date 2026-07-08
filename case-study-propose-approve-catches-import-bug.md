# Case Study: How Propose → Validate → Approve → Commit Caught a Silent Traceability Bug

**A real example from our own testing, not a hypothetical.**

## The setup

We stress-test TraceBoard's AI features against intentionally messy source documents — the kind of inconsistent, half-finished SRS drafts that show up constantly in real engineering teams. One test case was a Software Requirements Specification with a mix of primary requirements, conditional "SHOULD" variants, and response-time constraints, all using a hierarchical ID scheme (e.g., `REQ-002`, `REQ-002a`, `REQ-002c`).

We ran the same document through TraceBoard's AI Import and AI Audit Narrative features using two independent LLMs — Mistral Small and DeepSeek — expecting to compare output quality between a smaller and larger model.

## What we found

The Audit Narrative from both models cited a specific "high severity" finding: a logical contradiction between three requirements, `REQ-043`, `REQ-044`, and `REQ-045`, over database vendor strategy (one requirement mandated vendor independence, another hard-coded PostgreSQL, a third carved out an SQLite exception).

At first glance, this looked like a model hallucination — a plausible-sounding but fabricated citation, a known failure mode for LLMs asked to reference specific data points. That's a serious concern for a compliance tool: an audit finding that cites an artifact ID with no real referent is worse than an obviously wrong one, because it looks trustworthy on the surface.

But the same three IDs showed up identically across independent runs, on two different models. That consistency was the actual signal. Independent models don't hallucinate identical fabricated IDs by chance — they were both faithfully reporting on something that was really in the database. The bug wasn't in the audit layer at all. It was one step upstream, in the AI Import pipeline.

## The real bug

The import tool was flattening hierarchical requirement IDs into a sequential flat list. `REQ-002`, `REQ-002a`, and `REQ-002c` — a primary SHALL requirement, a conditional SHOULD variant, and an unrelated response-time constraint — were being silently renumbered into three new sequential IDs with no relationship to the source document. Repeated across ~30 sections, this produced exactly the kind of clean-looking, higher-numbered IDs (`REQ-043`, `REQ-044`, `REQ-045`) that the Audit Narrative later cited.

The consequences of this were quiet but serious:

- **Traceability was silently broken.** Anyone trying to verify an audit finding against the original SRS would find that `REQ-043` simply doesn't exist in the source document — a dead end, with no indication that anything had gone wrong.
- **Semantic meaning was lost.** A SHALL requirement, a SHOULD requirement, and a performance constraint — three different requirement types with different compliance weight under ISO 26262 — were collapsed into indistinguishable siblings.
- **The AI Audit layer inherited the error downstream**, faithfully and confidently reporting on data that no longer meant what it appeared to mean.

## Why the architecture caught it anyway

This is the scenario our AI architecture is built for. TraceBoard's core principle is that the deterministic engine is fully functional without AI, and the LLM layer only ever proposes — it never commits without explicit human review. Because of that:

1. The bug lived in the deterministic import stage, not the AI reasoning stage — exactly where it needed to be caught, since that's the layer meant to be reliable and inspectable.
2. Cross-checking two independent models against the same source data surfaced an inconsistency that neither model alone would have revealed.
3. Nothing was silently committed. The renumbered IDs were visible in generated output specifically because the system doesn't auto-approve AI or import output — a human was in the loop to notice the mismatch against the source document.

## The fix

We changed the AI Import feature to preserve original source IDs as immutable references rather than silently renumbering, and to flag ambiguous sub-requirement structures (like conditional SHOULD variants layered under a primary SHALL) for human review before import — rather than making a unilateral decision about how to split them.

We extended the same transparency principle to the Audit Narrative itself: every section now clearly marks which parts come from the deterministic engine (counts, statuses, links — verifiable, reproducible) and which are LLM-reasoned interpretation (risk narratives, recommendations). And AI-generated Tasks and Tests are now opt-in and clearly labeled as AI-generated, never silently created alongside real project data.

## The takeaway

We don't present this as a story about an AI model failing. It's the opposite — the audit models did their job correctly, faithfully reporting on the data they were given. The actual defect was upstream, in a deterministic transformation step, and it was invisible until we deliberately tested with independent models and treated their agreement as a signal worth investigating rather than dismissing as coincidence.

That's the case for propose-only AI in regulated engineering workflows: not that AI won't make mistakes, but that a system designed so nothing commits without a traceable path back to source — and a human checkpoint before it does — turns a silent data-integrity bug into a caught one.

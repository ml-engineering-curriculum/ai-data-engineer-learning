# Exercise 05 — Role Scope and the Deferral Contract

**Estimated effort:** 2 hours
**Chapter dependency:** [Chapter 06 — The AI Data Engineer on the Level Ladder](../06-role-scope-on-the-ladder.md) — and all preceding chapters.

## Objective

Draw the scope of the AI Data Engineer role on your (real or hypothetical) team, write the deferral contract that will live in the team README, and simulate three cross-team scenarios where the contract is tested. The deliverable is a document a new team member or a cross-team partner can read in 10 minutes and know what to ask you for and what not to.

## Prerequisites

- You have read Chapter 06 and the preceding five chapters.
- You know the roles in the ML Engineering, AI Governance, and AI Infrastructure families at the levels referenced in Chapter 06 — at minimum: `ml-engineer` (L20), `ai-governance-analyst` (L15), `fine-tuning-engineer` (L30), `rag-engineer` (L30), `model-evaluation-engineer` (L30), `nlp-engineer` (L30), `ai-eval-engineer` (L30, AI Engineering), `ai-risk-engineer` (L25, Governance), `ai-infra-ml-platform-engineer` (L30), `training-pipeline-engineer` (L35), `ai-evaluation-engineer` (L35, Governance), `staff-ml-engineer` (L40).
- You have a concrete use case in mind — this exercise is much easier against a real one.

## Problem statement

You are the newly hired AI Data Engineer on a team that trains and maintains an in-house 7B chat model for internal customer-support use. Your manager asks for a 1-page scoping document and a deferral contract to share with the four partner teams: ML (modelling), governance, legal, and infra. Alongside, you will work three incoming requests through the contract.

## Starter guidance

- Lead with the one-sentence scope from Chapter 06; then enumerate what you own, what you consume, and what you refuse.
- Name the roles by level and family, not just by title; "level 30 `fine-tuning-engineer`" is more useful than "the fine-tuning folks."
- The deferral contract works best as a two-column table: "ask me for X" and "ask Y for Z". Chapter 06 has a prose version; your version can be shorter and sharper.
- For each of the three incoming scenarios, say explicitly what you do, what you hand off, and what you refuse. "Refuse politely" is a valid response; the contract exists so refusal has backing.
- You do not need to invent role names outside the ladder in Chapter 06. Use the existing taxonomy.

## Deliverables

### Part A — Scope diagram

`scope.md` with:

- One-sentence scope.
- A table of **artifacts you emit** (dataset cards, source manifests, licence matrix, quality logs, bias audits, dedup reports, contamination firewall reports, PII reports, lineage events, signed snapshots, synthetic-data operators, labelling-ops output, monitoring contracts).
- A table of **inputs you consume** (AI system inventory IDs, impact assessments, lawful-basis / licence postures, legal opinions, platform orchestrator and feature-store services).
- A table of **adjacent-role handoffs** — for each adjacent role, one line each on *you hand them* and *they hand you* (same shape as Module 00 of this curriculum).

### Part B — Deferral contract

`deferral_contract.md` — a two-column table in the shape of Chapter 06's sample. Minimum 12 "ask me for" entries and 10 "ask someone else for" entries with the target role named.

### Part C — Scenario walk-throughs

`scenarios.md` — walk the three scenarios below. For each, write ≤200 words covering: *what you do, what you hand off and to whom, what you refuse and why.* Cite the deferral-contract row that backs your response.

**Scenario 1.** A product manager asks you to re-train the model with "better" examples because a customer complained about tone. No clearer signal than that.

**Scenario 2.** The governance analyst asks you to write the organisational AI inventory entry for the chat model because "you know the system best."

**Scenario 3.** The fine-tuning engineer asks you to run a DPO dataset mix experiment: three preference-pair sources at varying ratios to see which combination reduces a specific hallucination mode.

## Acceptance criteria

- Scope is expressed in artifacts and handoffs, not in verbs. "I curate" is weaker than "I emit the per-slice dedup report and the source manifest."
- The deferral contract covers at least the four roles below plus two more: `fine-tuning-engineer`, `ai-governance-analyst`, `training-pipeline-engineer`, `staff-ml-engineer`.
- Every handoff names the artifact that crosses the boundary, not just the topic.
- Scenario 1 response includes a request for a more specific signal — "tone" is not a dataset-shaped request. You either help the PM translate it (e.g. into gold-labelled examples of acceptable vs. unacceptable tone from `model-evaluation-engineer`) or you refuse until the signal exists. Either is fine if backed by the contract.
- Scenario 2 response explicitly declines the inventory-authoring ask and routes it back to governance, while offering to supply the data-slice evidence the inventory entry needs.
- Scenario 3 response accepts the dataset-mix *execution* but names the DPO *design* as `fine-tuning-engineer`'s call — you run what they design, you do not design the mix.

## Stretch goals

- Turn `deferral_contract.md` into a GitHub-style CODEOWNERS or routing file consumable by an on-call rotation.
- Add a "disputed" column to the deferral contract for responsibilities commonly disputed in the industry — e.g. "who owns dataset-side monitoring when it overlaps with model-output monitoring?" — with your proposed resolution and a one-line rationale.
- Draft the 1-page memo your manager would send to the three partner-team leads announcing the contract. Keep it under 300 words.
- Map each entry in the deferral contract back to the specific NIST AI RMF sub-category, ISO/IEC 42001 Annex A control, or EU AI Act Article that drives the responsibility. This turns the contract into a cross-reference for the earlier exercises.

# Chapter 02 — Briefing the AI Governance Council and the Data Council

## Why this chapter exists

Chapter 01 named the forums and the artifacts. This chapter is the authoring craft — how a monthly pre-read, a release one-pager, and a decision log are *actually written* so the council can decide on them in the time it has, and so the written record survives personnel and regulator turnover.

The underlying problem: the people on these councils do not read code. They read documents. If your document is three pages when it should be one, or thirty slides when it should be five, the decision does not happen — or worse, it happens on the fraction they actually read. Learning to write for a council is as load-bearing as the pipeline work it describes.

## Know the reader

Before writing anything, write a two-line reader profile at the top of the draft.

- **AI governance council** reader: legal/compliance/risk-oriented, non-technical by default. Cares about regulator exposure, residual risk, precedent, and sign-off defensibility. Will read the executive summary and the asks; may read the rest if a specific risk catches attention.
- **Data council** reader: data-platform-oriented, half-technical. Cares about inventory accuracy, classification compliance, lineage, retention. Reads the inventory and lineage tables carefully; skims the rest.
- **Model/product team** reader: technical but on a tight budget. Cares about what they can pin to, what changed, and what breaks. Reads the diff section and the acceptance criteria; skips the governance appendix.

A document written for the wrong reader fails for the right reader. One council paper per reader class.

## Authoring the monthly pre-read

### The shape (fixed every month)

```markdown
# AI governance council — data program pre-read — 2026-11

**Author:** alice@acme (ai-data-engineer)
**Audience:** AI governance council
**Pre-read window:** circulated 2026-10-28; meeting 2026-10-30
**Decisions requested:** 3 (see § Asks)

## Executive summary
One paragraph (≤ 120 words). What changed this month, what we are asking for.

## Slices shipped
| Slice | Version | SHA (prefix) | Shipped | Consumer | Risk-register impact |
|---|---|---|---|---|---|
| cc_en   | v3.2    | a1e2b3c4 | 2026-10-07 | foundation-model-v4 training | none |
| cc_multi| v1.0    | f9e8d7c6 | 2026-10-18 | foundation-model-v4 training | rr.cc_multi.lang_balance (closed) |
| sft_core| v6.1    | 3b4c5d6e | 2026-10-24 | finetune-chat-v2              | none |

## Decisions logged (bi-weekly admissions review)
| ID | Summary | Decision | Signatories |
|---|---|---|---|
| dec.2026-10-09.admit.cc_fr         | Admit Common Crawl FR subset | admit                      | alice, bob, carol |
| dec.2026-10-16.filter.min_doc_len  | Raise minimum document length from 128 → 192 tokens | approve                    | alice |
| dec.2026-10-23.retire.books_v2     | Retire books_v2 (superseded) | retire (grace: 2026-12-31) | alice, bob |

## Open risk items
| ID | Owner | Status | Blocking next release |
|---|---|---|---|
| rr.cc_multi.lang_balance    | alice | closed  | no  |
| rr.synth.ipr_residual       | risk-eng | open  | no  |
| rr.contamination.gsm8k_leak | eval-eng | open  | yes |

## Incidents (October 2026)
- **inc.2026-10-14.pii_regex** — residual-PII rate on `sft_core.v6.0` spiked to 4.1e-5 at refresh. Root cause: locale-specific phone regex missed `+XX-XXX-XXXX-XXXX` form. Fix: add pattern, re-release as `v6.1`. Postmortem attached. Severity: retrain_blocking (the cited pattern never reached production; retrain-blocking because v6.0 had already been pinned by a downstream PR).

## Regulatory calendar
- **2026-11-30** — EU AI Office Annex XI technical-documentation update for `foundation-model-v4` due (Chapter 07).
- **2027-02-02** — High-risk-system Annex IV refresh cycle for `product-decisioning-v1`.

## Asks (decisions requested at this meeting)
1. **Approve risk acceptance** on `rr.synth.ipr_residual` — accept residual IP risk on synthetic Q&A pairs generated from Wikipedia with documented mitigations (one-pager attached).
2. **Approve release** of `cc_multi.v1` for foundation-model-v4 training (one-pager attached).
3. **Direction** on retention policy for intermediate dedup indexes (currently 90 days; consumer requests 180).
```

### What good looks like

Two readability checks:

1. **One-screen rule.** The executive summary plus the asks fit on one screen. A council member who reads only those two sections has enough to vote on the asks.
2. **Table-first rule.** Every section that can be a table is a table. Prose is for context; tables are for state. A reader who scans the tables gets 80% of the value; the prose fills the rest.

A failure pattern to avoid: the pre-read as a narrative. Councils do not want a chronological story. They want *state, decisions, risks, asks*.

### Length discipline

Hard cap: **four pages**. Below that is better. If you cannot fit the month's activity in four pages:

- Move operational detail to the dataset cards it belongs on.
- Collapse low-impact decisions into a single row "n other admissions, see appendix".
- Separate the ask-bearing sections from the status-only sections; a council can ignore the status if the asks are self-contained.

## Authoring the release one-pager

### When a one-pager is required

Every release that requires sign-off beyond the self-approval class (Chapter 01 release-class table). A patch slice ships without a one-pager; a minor, major, or regulated-use-case release does not ship without one.

### The shape

```markdown
# Release one-pager — cc_multi.v1 — foundation-model-v4 training

**Author:** alice@acme   **Signatories requested:** alice, bob (governance), carol (legal), dan (model-team)
**Release class:** Major (new source class: non-English Common Crawl)
**Dataset card:** s3://training-data/cc_multi/v1/dataset_card.md  **SHA:** f9e8d7c6…

## What is releasing
`cc_multi.v1` — a 28-language Common Crawl-derived pretraining slice. 1.4T tokens after dedup. Sourced from CC-MAIN-2026-40 through CC-MAIN-2026-42. Signed snapshot at the SHA above.

## Why now
Foundation-model-v4 training needs multilingual pretraining data that complies with the EU DSM Article 4(3) TDM opt-out. The slice is the first non-English release of the `cc_*` family.

## What is new vs. the previous version
This is the first `cc_multi` release. Compared to `cc_en.v3`:
- Language coverage: 28 languages vs. English-only.
- Robots TDM-reservation enforcement: identical (mod-102 Chapter 04).
- Dedup granularity: identical (mod-105 near-duplicate + MinHash).
- Contamination firewall: extended to Flores-200 and XCOPA eval sets (mod-106).

## Evidence attached
- Dataset card (HF format + Google Data Card + EU public summary).
- Quality report (`per_slice_quality.yaml`).
- Dedup report (`dedup_report.yaml`).
- Contamination report (`contamination_report.yaml`).
- Privacy risk report (`privacy_risk_report.yaml`).
- Bias audit (`bias_audit.md`).
- Signed lineage (`openlineage_run.json`, Sigstore Rekor entry `rekor:<idx>`).

## Known residual risks
- Per-language quality variance: Flores-200-based classifiers tag Swahili at precision 0.81 (vs. 0.94 for English). The slice still ships because the staff-ml-engineer's data-mixture (L40) applies a per-language quality weight. Tracked as `rr.cc_multi.sw_quality`.
- Residual-PII rate at 7.2e-6 (threshold 1e-5). Within band but higher than `cc_en.v3`'s 3.1e-6. Tracked as `rr.cc_multi.pii_watch`.

## Sign-off requested
☐ ai-data-engineer (alice) — pipeline shape and evidence
☐ ai-governance-analyst (bob) — inventory entry, DPIA, Art 10 package
☐ legal (carol) — licence posture, DSM Art 4 compliance
☐ model-team (dan) — consumption readiness

## Follow-ups (not blocking)
- Add Swahili-specific quality filter (SWh-targeted) for `cc_multi.v2` (ETA 2027-Q1).
- Widen PII regex coverage for low-resource languages (ETA 2027-Q1).
```

### What a reviewer looks for

A seasoned governance reviewer will scan a one-pager in this order:

1. Release class and signatories — is the sign-off routing correct for this class?
2. Diff section — what *changed* and is the delta inside the admitted envelope?
3. Known residual risks — are they acknowledged and tracked, or hidden?
4. Follow-ups — are the deferred items captured, with owners and ETAs?

If any of those answers are "no", the one-pager returns for rework.

## Authoring the decision log

### The log as an artifact

The decision log is **not a meeting-minutes file**. It is a per-decision, machine-readable record stored under version control. Every decision has its own file (as in the YAML example in Chapter 01). The log is the collection of those files plus an index.

Three design principles:

1. **One decision per file.** Enables grep, enables auditor review, enables cross-referencing by id.
2. **Append-only.** If a decision is reversed, write a new decision (`dec.2027-02-14.rescind.…`) referencing the previous. Never edit history.
3. **Machine-readable key fields.** `kind`, `date`, `decision`, `signatories`, `conditions`, `next_review`, `links.*` — so the governance analyst can roll up by `kind` or `rmf_subcategory` without re-reading the prose.

### The index

A single file that lists every decision chronologically:

```yaml
# decisions/INDEX.yaml
decisions:
  - id: dec.2026-11-04.admit.cc_fr
    kind: source_admission
    date: 2026-11-04
    summary: Admit Common Crawl FR subset.
  - id: dec.2026-10-23.retire.books_v2
    kind: slice_retirement
    date: 2026-10-23
    summary: Retire books_v2 (superseded by books_v3).
# …
```

A pre-commit hook regenerates the index from the files so drift is not possible.

### Common decision kinds

Keep the `kind:` vocabulary stable so audits are tractable:

- `source_admission` — admit a new source.
- `source_retirement` — stop ingesting a source.
- `licence_change` — change the licence posture of an admitted source.
- `filter_change` — change a filter's parameters or add/remove a filter.
- `slice_release` — approve a slice release.
- `slice_retirement` — retire a slice version.
- `incident_resolution` — close out a dataset-side incident with root cause and mitigation.
- `risk_acceptance` — accept a residual risk on a release.
- `policy_change` — change a program policy (retention, classification, charter).
- `synth_generator_admission` — admit a new synthetic-data generator pipeline.

If you need a new kind, propose it at the quarterly retro and amend the charter. Ad-hoc kinds proliferate and audits suffer.

### Wiring the log to the pre-read

The monthly pre-read's "Decisions logged" table is **generated** from the decision log, filtered by date range. If you are hand-typing that table, the wiring is wrong. A ten-line script reading the YAML files produces the table and the pre-read's narrative points to the files for detail.

## Communicating decisions outward

A decision is only effective if the people affected know. Three outbound channels:

### To the model/product team

A Slack / internal-bulletin post with a stable template:

```
📦 Data program decision — dec.2026-11-04.admit.cc_fr
Decision: admit Common Crawl FR subset to the cc_multi.v2 pipeline.
Impact on model teams: cc_multi.v2 (ETA 2026-12-02) will include ~36B FR tokens after dedup.
Pin / contract: no change to the slice handoff contract.
Link: https://repo/decisions/2026-11-04.admit.cc_fr.yaml
```

Keep the template stable so the readers learn to scan for the pin / contract line.

### To the governance analyst (L15)

A direct notification only when the decision affects the inventory entry, the DPIA, or the Article 10 package. Wire this as a CI hook — the decision file's `links.inventory_entry:` (or `.rmf_subcategory:`, or `.ai_act_article:`) triggers an issue in the governance analyst's queue.

### To the data council (quarterly)

The quarterly data-council report aggregates all `source_admission`, `source_retirement`, and `policy_change` decisions since the last report, plus the inventory refresh (classifications, retention postures, lineage attestations). The report is a 1–2 page document plus the inventory diff.

## Antipatterns to avoid

Six patterns that look reasonable in the moment and corrode the program over months:

1. **Writing the pre-read as a meeting-minutes recap.** Councils want *state*, not *a story*. Reserve prose for the executive summary and the risk explanations.
2. **One mega-document for everything.** A fused "governance + inventory + release" doc guarantees nobody reads the whole thing. Split by reader class.
3. **"We discussed this in Slack" as a decision record.** Slack is not auditable. If it mattered enough to discuss, it mattered enough to log.
4. **A "temporary exception" with no expiry.** Every exception needs a `next_review:` date. Exceptions without expiry become permanent policy by neglect.
5. **Separating "the decision" from "the evidence that supports it".** Both live in the decision file (`inputs:` field). An auditor reading the file nine months later has the full basis.
6. **Letting the pre-read become a security blanket.** The purpose is council decisions, not reporting activity. If a month has no asks, write a four-sentence pre-read that says so, and skip the meeting.

## Summary

- Councils read documents, not code. Writing for the council is as load-bearing as the pipeline work it describes.
- The **monthly pre-read** is a four-page maximum document with a fixed shape — executive summary, slices, decisions, risks, incidents, regulatory calendar, asks. Tables beat prose.
- The **release one-pager** is a one-page document required for every release above the self-approval class. A reviewer scans it in a fixed order; design for that order.
- The **decision log** is an append-only, machine-readable, one-file-per-decision record with a stable `kind:` vocabulary and a generated index.
- Wire the artifacts to each other: the pre-read's decision table is generated from the log; the log's `links` fire CI hooks into the governance analyst's queue; the data council's quarterly report aggregates the log.
- Six antipatterns to avoid — prose-as-minutes, mega-documents, Slack as the record, undated exceptions, separating decisions from evidence, pre-read-as-blanket.

Chapter 03 moves from the operating rhythm to the **documents the pipeline itself ships** — starting with the dataset-card format landscape and the Datasheet that underlies all three forms.

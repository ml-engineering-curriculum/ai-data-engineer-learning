# Chapter 08 — The Model-Card Data Slice and the System-Card Dataset Slice

## Why this chapter exists

A model card and a system card are **not your documents**. They are authored by the model team (for the model card) and by the AI system or product team (for the system card). But each has a data-slice section into which you contribute, as the AI Data Engineer who produced the training corpora, labelling pipelines, and eval-set curation.

Getting the contribution right matters because the model card and the system card are the two documents most-often cited externally. A gap in your data slice becomes a gap the model team either has to improvise around or back-fill with a request chain. A tight contribution ships clean and survives review.

This chapter is the authoring craft for *contributing to someone else's document*. The format to contribute into, the content to supply, the SHA that keys your contribution to a specific evidence base, and the division of responsibility with the model team and the system team.

## The two documents and their lineage

### Model cards — Mitchell et al. 2019

The original model-card format was proposed in Mitchell et al., *"Model Cards for Model Reporting"*, FAccT 2019. The paper defined nine sections for reporting a trained ML model:

1. Model details
2. Intended use
3. Factors (relevant demographic/environmental)
4. Metrics
5. Evaluation data
6. Training data
7. Quantitative analyses
8. Ethical considerations
9. Caveats and recommendations

The **Hugging Face Model Card** format (used on the Hub) is a direct implementation of Mitchell et al. with YAML frontmatter and markdown body — structurally parallel to the HF Dataset Card (Chapter 04). Documentation: <https://huggingface.co/docs/hub/model-cards>. Template: <https://github.com/huggingface/huggingface_hub/blob/main/src/huggingface_hub/templates/modelcard_template.md>.

Your contribution flows into sections 5 (Evaluation data) and 6 (Training data).

### System cards — AI system / product-level documentation

System cards document the deployed AI system — the model plus the surrounding product, including guardrails, retrieval augmentation, routing logic, evaluation harnesses, and operational monitoring. The format is less standardised than the model card but has conventional sections documented in system-card releases from OpenAI (GPT-4 system card, GPT-4V system card, GPT-4o system card), Anthropic (Claude system cards), Google DeepMind (Gemini safety cards), and others. <!-- needs-research: cite the exact system-card URLs for current GPT-4-family, Claude-family, and Gemini-family systems at reading time; the system-card concept predates the EU AI Act's Article 53 regime but anticipates many of its concerns. -->

A system card typically covers:

1. System description — model(s), inference stack, retrieval, agentic/tool surfaces.
2. Intended and non-intended uses.
3. Safety evaluations — red-team results, bias/toxicity benchmarks, dangerous-capability evaluations.
4. Data — training data (cross-reference to the model card), fine-tuning data, RLHF data, evaluation data.
5. Deployment safeguards — guardrails, content policy, operational monitoring.
6. Known limitations and risks.
7. Change log — what changed since the previous system-card release.

Your contribution flows into section 4 (Data). The content overlaps with the model-card training-data section but adds detail on fine-tuning/RLHF data and application-specific eval-set data.

## The model-card data slice — what to contribute

### Section 5: Evaluation data

A short subsection describing the evaluation datasets used to measure the model. Two paragraphs typically:

- Which evaluation sets were used and at what versions.
- The decontamination posture of each against the training data (from mod-106).

Example contribution:

> Evaluation data: MMLU 5-shot (`cais/mmlu`, version `2023-01`), GSM8K (`gsm8k`, version `main`), HumanEval (`openai/humaneval`, version `main`), HellaSwag (`Rowan/hellaswag`, version `main`), TruthfulQA (`truthful_qa`, version `main`), Flores-200 (`facebook/flores200`, version `v1.1`). Contamination against the training mix measured at 13-gram overlap: MMLU 0.03%, GSM8K 0.01%, HumanEval 0%, HellaSwag 0.02%, TruthfulQA 0.00%. All eval sets admitted after review of Min-K% Prob and n-gram overlap against the firewall in `contamination_report.cc_multi.v1_plus_sft_v6.yaml`.

Why this shape: the model-team author of the model card wants to drop this paragraph in with minimal editing. Give them the exact version strings, the exact contamination figures, and the SHA of the report to link. One well-formed paragraph is better than three paragraphs of hand-wavy prose.

### Section 6: Training data

The longer contribution. Mitchell et al. explicitly ask for:

- Datasets used, and preprocessing applied.
- Where the data came from.
- Any filtering, including the removal of certain populations.
- Discussion of representativeness and known biases.

Your contribution for a GPAI pretraining + SFT + RLHF stack typically has three sub-sections:

**6.1 Pretraining data**. Source-class composition (parallel to the Meta Data Card § 3 from Chapter 06), cutoff date, per-source-class licence posture, decontamination posture. 3–5 short paragraphs or a composition table + a preprocessing summary + the snapshot SHA.

**6.2 Supervised fine-tuning data**. The SFT mix: number of conversations, source (human-authored, model-generated, vendor-provided), annotator pool composition (from mod-107), inter-annotator-agreement range, PII handling. 2–3 paragraphs plus a composition table.

**6.3 Preference / RLHF data**. The preference-pair dataset: number of pairs, scoring rubric, annotator pool, inter-annotator agreement, PII handling, red-team / adversarial subset composition. 2 paragraphs plus a composition table.

All three subsections point to the dataset cards (HF or Google form) for full detail. The model card is the *summary* with pointers; the dataset cards are the *detail*.

### Section 8: Ethical considerations (data-side)

The model card's ethical-considerations section asks about potential harms and demographic risks. The data-side contribution here is a short paragraph on:

- Representativeness limitations (language, geographic, demographic) and their downstream implications.
- Filter-induced biases (Dodge et al. 2021 pattern — blocklist filters disproportionately affecting marginalised communities).
- Synthetic-data provenance risks (if the training mix contains model-generated data, state the policy on derivative-data legal posture).

The model-team author will integrate this with model-behaviour considerations (toxicity, harmful completions, misuse scenarios). Your contribution is the *data-side* half of that integration.

## The system-card dataset slice — what to contribute

### Section 4: Data (the dataset slice)

A system card's data section asks for a tighter, deployment-oriented disclosure. Four sub-sections typical:

**4.1 Training data (cross-reference)**. One paragraph summarising the pretraining and fine-tuning data, with a pointer to the model card's § 6. Do not restate; cross-reference.

**4.2 Deployment-specific data**. Any data that was added for the specific deployment — retrieval-augmented-generation corpora, domain-specific fine-tuning slices, in-product evaluation data. This is where the AI Data Engineer's work on domain adaptation or RAG-side corpora lands.

**4.3 Operational data flows**. What happens to user inputs — are they retained for training, for evaluation, for operational monitoring? What privacy controls apply? What is the retention policy? This paragraph is co-authored with legal and product; the data-engineer contribution is the operational mechanics.

**4.4 Evaluation data** (for safety evaluations). The gold datasets used to measure safety behaviour of the deployed system, cross-referencing the dangerous-capability eval sets from mod-106 and mod-107, plus any application-specific eval datasets.

### Section 6: Known limitations and risks (data-side)

Shorter than the model-card ethical section. A system card calls out limitations specifically relevant to the deployment:

- Training-data cutoff and deployment-time staleness.
- Language coverage gaps for a multilingual deployment.
- Domain coverage gaps for a domain-specific deployment.
- Retention / privacy posture of user-data flows.

One to two paragraphs of data-side content is typical.

## The SHA-keyed contribution discipline

The model card and system card are *point-in-time documents*. They describe a specific model version and a specific system release. Your data contribution must be keyed to a specific evidence base so the document is auditable.

Three specific SHAs your contribution cites:

- **Training-mix SHA** — the `sha256` of the combined pretraining + SFT + RLHF data as assembled for the model release. Parallel to the Meta Data Card § 10 identifier.
- **Snapshot catalog pointer** — the DataHub / Unity Catalog URI or OpenLineage run id that the training-mix SHA was assembled from.
- **Evaluation-firewall report SHA** — the `sha256` of the contamination report used to admit the eval sets.

If any of these move after the model card is published, a corrigendum is issued and the card is versioned. The card is not edited in place; it is re-released. This discipline mirrors Chapter 04's rule for dataset cards.

## The hand-off pattern

How this works in practice across the model release cycle:

### T-6 weeks: the model team requests contributions

The model team (or model-card author, often the `staff-ml-engineer` at L40 or a designated model-card owner) emails or opens an issue:

> Model release `foundation-model-v4` is targeting 2026-12-02. Model card data slice contributions due 2026-11-18. Please provide:
> - Section 5 contribution (evaluation data, 2 paragraphs).
> - Section 6 contribution (training data, 3 sub-sections + composition tables).
> - Section 8 data-side contribution (3 paragraphs).
> - Training-mix SHA and the contamination firewall report SHA.

Your response is driven from the evidence base you've already maintained. The composition tables, the per-source-class figures, the decontamination tables are already in `dataset_evidence.yaml`. The authoring step is **selecting** and **phrasing** for the model card's audience, not re-deriving facts.

### T-4 weeks: the draft contribution

Deliver the contribution as a markdown fragment the model-card author can paste in:

```markdown
### 6.1 Pretraining data

The model was pretrained on approximately 15 trillion tokens of multilingual
text and code combined from five source classes:

| Source class | Approx. share | Licence posture | Cutoff |
|---|---|---|---|
| Common Crawl-derived web text | ~62% | Robots-TDM + DSM Art 4(3) honoured | 2026-10-23 |
| Licensed publisher data | ~14% | Bilateral licences | 2026-09-30 |
| Open-source datasets | ~11% | CC-BY, CC-BY-SA, MIT-equivalent | 2026-08-15 |
| Code repositories | ~8% | Permissive OSS | 2026-10-10 |
| Synthetic data | ~5% | Internal generation + validation | 2026-10-01 |

Pretraining data preprocessing included quality filtering (perplexity-based
+ heuristic), language detection (fastText lid-176 threshold 0.65),
deduplication (MinHash near-duplicate + exact), benchmark decontamination
(13-gram overlap + Min-K% Prob against the evaluation suite in § 5),
and PII scrubbing (Microsoft Presidio). Residual-PII rate: 7.2e-6
against the mod-110 evaluation set (threshold 1e-5). Snapshot identifier:
`cc_multi_v1_plus_licensed_v3_plus_code_v2_plus_synth_v1.sha256 = f9e8d7c6…`.

Full dataset cards per slice: `training-data.acme/cc_multi/v1/`,
`training-data.acme/licensed/v3/`, `training-data.acme/code/v2/`,
`training-data.acme/synth/v1/`. Contamination report:
`contamination_report.training_mix_v4.sha256 = 1a2b3c4d…`.
```

The model-card author drops this in with minimal editing.

### T-2 weeks: the review round

The draft circulates to legal, governance, risk, and the model team. Comments return. You update the evidence base (if a fact was wrong) and re-render the contribution. The second draft goes out.

### T-0: release

The model card publishes. The contributions you supplied are cited in the model card's acknowledgement block. The training-mix SHA is public and audit-traceable.

### T+N: changes

If a data-side correction or clarification is needed post-release, a corrigendum issues. The model card is versioned (v1.0.1). The underlying evidence base updates.

## Common contribution mistakes

Six patterns that cost authoring-team time:

1. **Supplying the full dataset card instead of a model-card-shaped excerpt.** The model-card author then has to extract. Supply the fragment in the shape the model card expects.
2. **Figures that don't match the dataset cards.** The dataset card for `cc_multi.v1` says ~62% of the pretraining mix; the contribution to the model card says 65%. Both ship to the public simultaneously. Reviewer catches the discrepancy. Verify figures against the evidence base.
3. **Missing the training-mix SHA.** The per-slice SHAs are in the dataset cards; the combined training-mix SHA is a new object the data engineer assembles at release time. Compute and publish it.
4. **Over-specifying for the model-card audience.** The model card is a *summary*. Full per-source-URL disclosure belongs in the Annex XI package (Chapter 07). The model card points to the Annex XI package, not into it.
5. **Under-specifying the decontamination posture.** The single most-reviewed detail on a public model card. A table with per-eval-set measurement and firewall actions always ships.
6. **Forgetting to update at corrigendum time.** The model card is versioned; your contribution is re-rendered from the updated evidence base. Author's instinct is often to edit the text directly; discipline is to update the base and re-render.

## The system-card-specific disciplines

Three disciplines specific to system-card contributions:

### 1. Scope the data section to the deployment

A system card is per-deployment. If the deployment is `acme-chat`, the data section covers what `acme-chat` specifically uses — the base model's training data by cross-reference, plus the application-specific RAG corpus, plus the application eval dataset, plus the operational data flow. Do not restate the model-card's training data in detail.

### 2. Describe operational data flows honestly

Section 4.3 (operational data flows) is where regulators and privacy reviewers focus. The honest paragraph:

- Are user inputs logged? For how long?
- Are user inputs used for future training? Under what consent and opt-out?
- Are user inputs used for safety evaluation? Under what access controls?
- What is the retention policy?
- How are rights-holders of user-contributed content handled (e.g. if a user pastes copyrighted text, what happens)?

This paragraph is co-authored; your contribution is the **operational mechanics** (logging pipeline, retention enforcement, access-control implementation).

### 3. Keep the change log current

A system-card § 7 Change Log reports what changed since the previous release. Data-side changes you contribute to the change log:

- Training-data refresh and the new training-mix SHA.
- New RAG corpus version or new application eval dataset version.
- Data-retention-policy change.
- Opt-out mechanism change.

A system card without a current change log is a signal that the system is not actively maintained.

## How the data slice connects to the rest of mod-112

Chapters 04–07 cover documents you own (your dataset cards and the Article 53 public summary). Chapter 08 covers contributions to documents other roles own (the model card and the system card). All five render from the **same evidence base**:

- `dataset_evidence.yaml` + `preparation_log.yaml` + `contamination_report.yaml` + `privacy_risk_report.yaml` + `bias_audit.md` + `openlineage_run.json` + `snapshot.sha256`.

Chapter 09 names the five roles that consume this evidence base alongside you — governance analyst, risk engineer, evaluation engineer, staff ML engineer — and how the handoffs work.

## Summary

- The **model card** (Mitchell et al. 2019 / HF format) is authored by the model team. Your contributions land in § 5 (evaluation data), § 6 (training data: pretraining, SFT, RLHF sub-sections), and § 8 (ethical considerations, data-side).
- The **system card** is authored by the system/product team. Your contributions land in § 4 (data: training cross-reference, deployment-specific, operational flows, evaluation) and § 6 (known limitations, data-side).
- Contributions are **keyed to three SHAs**: training-mix SHA, snapshot catalog pointer, evaluation-firewall report SHA. The documents are point-in-time and auditable against the evidence base.
- The hand-off pattern runs on a 6-week cycle: T-6 request, T-4 draft, T-2 review, T-0 release, T+N corrigenda. Contributions render from the same evidence base that drives your own dataset cards and the Article 53 summary.
- Six common mistakes: supplying full cards instead of excerpts, mismatched figures, missing training-mix SHA, over-specifying, under-specifying decontamination, failing to re-render at corrigendum time.
- System-card-specific disciplines: scope the data section to the deployment, describe operational data flows honestly, keep the change log current.

Chapter 09 moves to the role-boundary layer — the four peer and adjacent roles your contributions flow to, and how the handoff works for each.

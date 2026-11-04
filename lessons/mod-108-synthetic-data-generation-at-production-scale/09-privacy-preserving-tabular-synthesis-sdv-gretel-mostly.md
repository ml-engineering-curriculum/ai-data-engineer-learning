# Chapter 09 — Privacy-Preserving Tabular Synthesis: SDV, Gretel, MOSTLY AI, and Differential Privacy

## Why this chapter exists

Chapters 02–08 cover synthetic text. Not every synthetic-data task is text. A distinct and important slice is **tabular** / structured synthesis: produce rows that are shaped like a sensitive source table (customer transactions, clinical records, telemetry logs) but that do not enable membership inference against the real records.

The use case is specific. A team wants to:

- Share a representative dataset with a partner or vendor who cannot have access to the real data.
- Build and test downstream analytics or models without touching regulated records.
- Produce an evaluation set for a model that must behave like it was trained on the real distribution.
- Substitute regulated data in a development or training environment.

Textual synthesis tools (NeMo Curator, Data-Juicer, Self-Instruct, etc.) do not solve this. They do not learn joint distributions over structured columns; they do not provide formal privacy guarantees; they do not respect inter-column constraints.

Three tools (at a minimum) do solve it: **Synthetic Data Vault (SDV)**, **Gretel AI**, and **MOSTLY AI**. Each supports **differentially-private** training of the generator, which is the formal privacy guarantee required when the use case is substitution of regulated data.

The chapter covers the three tools and their DP variants, the shape of the utility-vs-privacy trade-off, and the hand-off to counsel. As with Chapter 05, you own the engineering and the ledger; counsel owns the legal opinion.

## What "privacy-preserving tabular synthesis" means

Pin the shape. A tabular synthetic-data task has four inputs:

1. **A real table** `T` with rows that are individuals (or individual-adjacent — accounts, devices).
2. **A privacy target** — a formal statement of what must be preserved against which adversary. Usually a differential-privacy guarantee `(ε, δ)`.
3. **A utility target** — what downstream task(s) the synthetic data must support. Usually at least marginal-distribution fidelity on named columns; sometimes a specific downstream model's accuracy.
4. **A hand-off** — who receives the synthetic table and under what terms.

A synthesis tool produces a generator `G` trained on `T` and a synthetic table `T_syn` sampled from `G`. The deliverable:

- `T_syn`, with provenance back to `T` and `G`.
- A **DP report**: `(ε, δ)` of the generator training, the privacy analysis, the composition across queries.
- A **utility report**: marginal-distribution fidelity, pairwise-correlation fidelity, downstream-task accuracy on `T_syn` vs `T`.
- A **counsel sign-off**: the risk-acceptance artifact for the hand-off.

## Differential privacy in one paragraph

**Differential privacy** (Dwork et al. 2006) is a mathematical guarantee that the presence or absence of any single record in the training set has a bounded effect on the output of the trained generator. Formally: for every pair of neighbouring tables `T` and `T'` (differing in one row), every event `S`, and a `(ε, δ)`-DP mechanism `M`:

> `Pr[M(T) ∈ S] ≤ exp(ε) · Pr[M(T') ∈ S] + δ`

What this means operationally:

- `ε` (epsilon) is the **privacy budget**. Smaller is stronger. `ε ≤ 1` is often cited as strong; `ε ∈ [1, 10]` as moderate; `ε > 10` as weak-to-meaningless in a formal sense. The exact interpretation depends on the composition of queries and the mechanism. <!-- needs-research: avoid citing a single ε threshold as "regulator-acceptable"; this varies by jurisdiction, use case, and regulator. -->
- `δ` (delta) is a small failure probability. Standard practice is `δ ≤ 1/|T|` so that `δ` does not permit leaking an entire record with reasonable probability.
- The guarantee applies to the *training procedure*. Once trained with DP, the generator's outputs inherit the guarantee under post-processing.
- Composition matters: running multiple DP mechanisms spends the budget. Pin your composition analysis.

The primary references:

- Dwork, McSherry, Nissim, Smith. *Calibrating Noise to Sensitivity in Private Data Analysis.* TCC 2006.
- Dwork, Roth. *The Algorithmic Foundations of Differential Privacy.* Foundations and Trends in TCS 2014. <https://www.cis.upenn.edu/~aaroth/privacybook.html>.
- Abadi, Chu, Goodfellow, McMahan, Mironov, Talwar, Zhang. *Deep Learning with Differential Privacy.* CCS 2016 (DP-SGD). <https://arxiv.org/abs/1607.00133>.

## The three tools

### SDV — Synthetic Data Vault

- **Project home:** <https://github.com/sdv-dev/SDV>. Docs: <https://docs.sdv.dev/>.
- **Origin:** MIT DAI Lab; Patki, Wedge, Veeramachaneni. *The Synthetic Data Vault.* DSAA 2016. <https://dai.lids.mit.edu/>.
- **What it does:** library-shaped, open-source. Models for tabular, time-series, and multi-table (relational) synthesis. Baseline models include CTGAN (Xu et al. 2019, *Modeling Tabular Data using Conditional GAN*), TVAE, PAR (time-series), HMA (hierarchical multi-table).
- **DP posture:** SDV ships a DP-GAN model variant and integrates with `smartnoise-synth` for additional DP mechanisms. DP-CTGAN exists as a research-paper variant and in community forks. <!-- needs-research: confirm which DP models are first-class in the current SDV release and which require a plugin. -->
- **Licence:** Business Source License (BSL 1.1) after a recent relicensing; verify the current terms of the specific release you use. <!-- needs-research: SDV's licence has changed; verify the terms for the version you install. -->
- **Scale:** library-scale. Fits datasets up to the order of millions of rows on a single machine; the larger relational cases require cluster execution.

### Gretel AI

- **Project home:** <https://gretel.ai/>. Docs: <https://docs.gretel.ai/>. Open-source: <https://github.com/gretelai/gretel-synthetics>.
- **What it does:** commercial platform plus an open-source `gretel-synthetics` library. Supports tabular (ACTGAN, DGAN variants), textual, time-series, and relational synthesis. Provides a managed hosted service and a self-hosted option.
- **DP posture:** DP training is a first-class option in both the open-source library and the platform. The platform exposes `(ε, δ)` as a configuration knob and reports the realised budget.
- **Licence:** `gretel-synthetics` is Apache 2.0; the hosted platform is commercial.
- **Scale:** platform-scale. Designed for enterprise datasets; manages cluster execution.

### MOSTLY AI

- **Project home:** <https://mostly.ai/>. Docs: <https://mostly.ai/docs/>. Open-source: <https://github.com/mostly-ai/mostlyai>.
- **What it does:** commercial platform plus an open-source `mostlyai` SDK. Focused on structured tabular synthesis with regulatory-compliance use cases (financial services, healthcare, insurance). Supports tabular, relational, and sequential synthesis.
- **DP posture:** DP training is a first-class option. The platform publishes benchmark comparisons against SDV and other open-source tools.
- **Licence:** open-source SDK is Apache 2.0; hosted platform is commercial.
- **Scale:** platform-scale. Similar operating target to Gretel.

### Picking between the three

Decision criteria:

| Criterion | SDV | Gretel | MOSTLY AI |
|---|---|---|---|
| **Open-source baseline** | yes | yes (`gretel-synthetics`) | yes (`mostlyai`) |
| **Hosted platform** | no (community-driven) | yes | yes |
| **DP first-class** | partial (via plugin / variant) | yes | yes |
| **Relational / multi-table** | yes (HMA) | yes | yes |
| **Time-series** | yes (PAR) | yes (DGAN) | yes |
| **Regulatory use cases** | research & dev | enterprise | enterprise, healthcare-oriented |

As a heuristic:

- For research, exploratory work, or non-regulated use cases: start with **SDV**.
- For enterprise use cases with a managed platform requirement: pick **Gretel** or **MOSTLY AI** based on your legal team's vendor preference and the region of operation.
- For deep DP requirements or cross-dataset composition: evaluate all three against your utility targets with the same `(ε, δ)` setting; the empirical utility can differ by model class.

The vendor question is partly engineering, partly procurement. The engineering half is the reproducibility and DP posture of the specific model you pick. Pin both.

## The utility-vs-privacy curve

Every DP generator has a trade-off: tighter `(ε, δ)` → more training noise → worse utility on downstream tasks. The curve is not linear and differs by:

- The size of the real table (more rows → more utility at the same `ε`).
- The complexity of the joint distribution (simple marginals → cheaper to preserve; complex multi-column dependencies → more utility loss per unit of noise).
- The downstream task (marginal-distribution fidelity is cheapest to preserve; downstream model accuracy is most expensive).

The operational pattern:

1. Pick a privacy target (e.g., `ε = 2`, `δ = 10^{-6}`) from the risk analysis.
2. Train the generator at that target.
3. Measure the utility report against `T` on your downstream task(s).
4. If utility is unacceptable, revisit the privacy target *with counsel* — this is a risk-acceptance change, not a configuration change.
5. If utility is acceptable, ship.

The step (4) escalation path is important. Loosening `ε` is not a unilateral engineering decision.

## The utility report

What to measure, in increasing cost and informativeness:

### Level 1 — marginal distribution fidelity

Per column, compare the distribution in `T_syn` to `T`. Metrics:

- Categorical columns: total variation distance, chi-squared test.
- Numeric columns: Kolmogorov-Smirnov statistic, Wasserstein distance.

Fast to compute; necessary but not sufficient.

### Level 2 — pairwise and low-order joint fidelity

Pairwise correlations (Pearson for numeric, Cramér's V for categorical, mutual information for mixed) in `T_syn` vs `T`. Low-order joints (three-way contingency tables for the most important column triples) where the use case is sensitive to interactions.

### Level 3 — downstream model accuracy

Train a reference downstream model on `T_syn`, test on a held-out real split. Compare to the same model trained on `T`. The ratio of accuracies is the TSTR (Train on Synthetic, Test on Real) utility score — the operational metric most product users care about.

### Level 4 — privacy audit

Membership-inference and attribute-inference attacks against the generator. Even with DP, run empirical privacy attacks — they are a sanity check that the DP implementation actually behaved as advertised. Shokri et al. 2017 (*Membership Inference Attacks Against Machine Learning Models*, arXiv 1610.05820) is the canonical reference for membership inference.

### Report shape

```yaml
# tabular/utility_report.md
privacy:
  mechanism: DP-GAN
  library: gretel-synthetics
  library_version: 0.21.0
  epsilon: 2.0
  delta: 1.0e-6
  composition: RDP
  training_iterations: 10000
  data_sensitivity: 1.0

utility:
  marginal:
    mean_tv_distance: 0.04
    worst_column_tv: 0.11       # <name>
  pairwise:
    mean_correlation_error: 0.07
    worst_pair_error: 0.22       # <pair>
  tstr:
    task: credit_default_classifier
    accuracy_real: 0.843
    accuracy_syn: 0.802
    accuracy_ratio: 0.951
  privacy_audit:
    membership_inference_auc: 0.52     # near 0.5 is good
    attribute_inference_accuracy: baseline+0.01
```

## The counsel hand-off

As in Chapter 05, legal opinion is out of scope. Capture the facts; counsel renders the call.

What the ledger captures:

```yaml
# tabular/counsel_signoff_ref.md
real_table:
  identity: finance_customer_records_2025_q3
  regulatory_class: [GDPR, GLBA]
  source_dataset_licence_ref: ...
  data_subject_count: 420318

generator:
  tool: gretel_platform
  tool_version: 2026.01
  model_class: ACTGAN
  training_epochs: 100
  epsilon: 2.0
  delta: 1.0e-6
  training_log_sha256: ...

intended_use:
  recipient: partner_bank_quant_team
  jurisdiction: EU
  purpose: downstream_modelling_of_default_risk
  retention_period: 180 days
  distribution_terms: restricted-use-and-redaction-contract

questions_for_counsel:
  - Does the real-table processing authorisation permit training a generator on this data?
  - Is DP training with ε=2, δ=1e-6 an acceptable risk profile for this use case under GDPR Art. 25 and the DPA's guidance?
  - Can the synthetic output be shared with the named recipient under the terms specified?

counsel_signoff:
  reviewer: TBD
  reviewer_email: TBD
  reviewed_at: TBD
  opinion_summary: TBD
  opinion_sha256: TBD
  gate_passed: TBD
```

This ledger plugs into the Chapter 05 licence-ledger pattern. The tabular case is a specialisation; the same hand-off shape applies.

## Where tabular synthesis breaks

### 1. Implicit non-DP baseline

Symptom: a tool's "synthetic data" output is actually produced without DP training; the privacy guarantee is marketing, not formal.

Catch: always ask for the composition analysis and the training-time privacy accountant output. A tool that cannot produce an `(ε, δ)` at a specific confidence is not actually training with DP.

Fix: use a model class and a library configuration that is *advertised and documented* as DP, not merely "privacy-preserving". DP-GAN, DP-CTGAN, DP-SGD-trained models are DP. Plain CTGAN, TVAE, Bayesian-network models are not DP (they produce synthetic data, but with no formal guarantee).

### 2. Column-level leakage under weak DP

Symptom: `ε = 50` at tight `δ` — the DP guarantee is effectively trivial; a column with a rare value category can leak directly.

Catch: the DP accountant output is a release artifact. Values of `ε` that are high enough to be effectively meaningless are a yellow flag; counsel review is a gate.

Fix: tighten `ε` and re-run utility measurement. If utility is unacceptable at a reasonable `ε`, the use case may not be DP-tabular-eligible — route back to the risk analysis.

### 3. Over-fitting to rare rows

Symptom: the generator has memorised specific rare records; sampling produces near-copies of them. A membership-inference attack succeeds.

Catch: Level 4 privacy audit catches this empirically. Train-set membership-inference AUC meaningfully above 0.5 is a fail.

Fix: tighten DP training (lower noise scale, more gradient clipping). If memorisation persists, the model class or the training data size is unsuitable.

### 4. Downstream utility collapse under DP

Symptom: TSTR accuracy drops by double-digit percent under DP compared to non-DP training. The downstream user complains that the synthetic data is unusable.

Catch: utility report quantifies the drop before the hand-off.

Fix: this is a true utility-privacy trade-off. Options: loosen `ε` with counsel approval, pick a different model class (DP-CTGAN often preserves utility differently than DP-GAN), widen the training data, or accept the utility cost. Not a bug; a design decision.

### 5. Relational / multi-table breakage

Symptom: the generator produces individual tables that look fine, but join keys do not align, foreign-key constraints fail, and rows in child tables orphan.

Catch: validate join keys and constraints on the generated output. HMA (SDV) and the enterprise tools handle this natively; naive per-table training does not.

Fix: use the relational / hierarchical model variants, not per-table independent generators.

## Scope, not legal opinion

Mark the boundary explicitly:

- You pick the tool, train the generator, measure utility, measure privacy, and author the ledger.
- You do **not** decide whether the DP guarantee satisfies a specific regulatory requirement (GDPR, HIPAA, CCPA, GLBA, the AI Act's data-governance requirements, etc.). That is counsel's call.
- You do **not** decide the risk acceptance for a given downstream hand-off. That is counsel's call, informed by `ai-risk-engineer`.
- You do **not** opine on whether DP-protected synthetic data is "anonymised" for regulatory purposes. This is a jurisdiction-specific question.

The hand-off is identical to Chapter 05: you write the ledger; counsel signs.

## Summary

- Privacy-preserving tabular synthesis is a distinct slice from text synthesis. Three tools cover it: **SDV**, **Gretel AI**, and **MOSTLY AI**. Each supports **differentially-private** training of the generator.
- **Differential privacy** `(ε, δ)` is the formal guarantee. Smaller `ε` is stronger; composition matters; `δ ≤ 1/|T|` is standard. DP applies to the *training procedure* and propagates under post-processing.
- The **utility report** has four levels: marginal, pairwise, TSTR downstream-model accuracy, and empirical privacy-attack audit. Deliver all four for a regulated use case.
- Pick between the tools on: open-source vs managed platform, DP first-class vs plugin, regulatory posture, and vendor preference.
- The utility-vs-privacy trade-off is non-linear and depends on the downstream task. Loosening `ε` is a risk-acceptance change with counsel, not an engineering tweak.
- The **counsel hand-off** is identical to Chapter 05. You author the ledger (regulatory class, generator identity, DP parameters, utility results, intended use); counsel signs.

Chapter 10 picks up with the contamination firewall on synthetic generation — the final defence that keeps the generator's benchmark history from laundering into your training corpus.

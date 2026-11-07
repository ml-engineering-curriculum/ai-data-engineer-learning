# Chapter 07 — Data Poisoning and Adversarial Perturbations: Nightshade, Glaze, ATLAS, OWASP

## Why this chapter exists

Dedup is a fingerprint pipeline. It already builds the index that catches "the same thing appearing many times". The adversarial-perturbation / data-poisoning problem is adjacent: *crafted content that is designed to pass a dedup filter but still influence downstream training*. The dedup pipeline is one of the natural places to also detect this content, because the same fingerprint / cluster infrastructure can be adapted to spot anomalous repetition patterns, known poisoning tool signatures, and suspicious near-duplicates that look crafted rather than organic.

This chapter:

- Introduces the public data-poisoning tools you must know by name (**Nightshade**, **Glaze**) and the published poisoning-attack techniques they represent.
- Maps the threat surface to **MITRE ATLAS** techniques and **OWASP ML Top 10 ML02 (Data Poisoning)** / **OWASP LLM Top 10 LLM03 / LLM04** so you can talk about the risk in the vocabulary your security partners use.
- Pins the division of labour: **`ai-risk-engineer` (peer, level 25) owns the detection methodology; you own the pipeline-side quarantine responsibility.** Both are needed. Neither is optional.
- Gives you detection heuristics you can run from the dedup pipeline *today*, and a handoff shape to escalate real finds.

## The attacker models you must know

Data poisoning is a broad field. Three attacker models are relevant at the pretraining-data stage:

### Clean-label poisoning (training-time)

- The attacker publishes content that looks legitimate but is engineered to shift the model's behaviour on a specific target.
- Classical literature: Shafahi et al. 2018 *Poison Frogs*; Turner et al. 2018 *Clean-label backdoor attacks*; Carlini et al. 2023 *Poisoning Web-Scale Training Datasets is Practical* (arXiv 2302.10149) — the key paper proving web-scale pretraining corpora *can* be poisoned by domain-expiry attacks and front-running attacks for pennies.

### Imperceptible adversarial perturbations

- Pixel-level (or token-level) perturbations that change the model's internal representation without visibly changing the content. The **Nightshade** and **Glaze** family are the public tools in this space.

### Content-level poisoning (opinion flooding)

- Mass-producing low-noise content to drown out legitimate sources or to inject a specific bias into a domain. Relevant to news / forum / Q&A corpora.

The first two are the ones dedup can directly contribute to detecting. The third is upstream — source-allowlist and credibility work from [`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md).

## Nightshade (Shan et al. 2024)

**Paper:** Shan, Ding, Passananti, Wu, Zheng, Zhao. *Nightshade: Prompt-Specific Poisoning Attacks on Text-to-Image Generative Models.* IEEE S&P 2024. <https://ieeexplore.ieee.org/document/10646862> / arXiv 2310.13828 <https://arxiv.org/abs/2310.13828>. Project page: <https://nightshade.cs.uchicago.edu/>.

**What it is:** a perturbation applied to an image before it is uploaded to the web. The perturbation is designed to associate the image's concept (e.g. "dog") with a **different, attacker-chosen target concept** (e.g. "cat") when that image is later scraped and used to train a text-to-image model. The perturbation is imperceptible or near-imperceptible to humans.

**Why it is relevant to a text / multimodal data engineer:**

- Multimodal corpora you build (LAION-shape caption / image pairs) will contain Nightshade-perturbed images if you crawl post-October-2023 web content without a quarantine step.
- Downstream multimodal model training will drift silently on affected concepts if the shade is not removed.

**Detection posture:**

- Nightshade's canonical implementation has distinctive distributional signatures in the image spectrum (the paper discusses this; detection research is active and evolving).
- Dedup pipelines contribute by surfacing *clusters of images with the same perturbation fingerprint applied across different base images* — those clusters are suspicious even without a direct Nightshade detector.
- Operational response: quarantine, not drop. You are not qualified to make the "is this Nightshade" call alone; `ai-risk-engineer` makes it. You produce the evidence.
- As of writing (Nightshade 1.1, April 2026), the SAND Lab publishes the perturbation tool but does not ship an official detector; the public defender-side work on detecting Nightshade / Glaze perturbations (e.g. the 2025 *LightShed* paper) comes from the research community rather than the SAND lab itself. Re-check the project page and recent literature at read time before citing a specific detector.

## Glaze (Shan et al. 2023)

**Paper:** Shan, Cryan, Wenger, Zheng, Hanocka, Zhao. *Glaze: Protecting Artists from Style Mimicry by Text-to-Image Models.* USENIX Security 2023. <https://www.usenix.org/conference/usenixsecurity23/presentation/shan> / arXiv 2302.04222. Project page: <https://glaze.cs.uchicago.edu/>.

**What it is:** a style-protective perturbation applied to an artwork image that hinders style mimicry by text-to-image models trained on the perturbed content. Glaze is *defender-sided* — it is applied by artists who want their work to be less effective as training data for style-mimicry models.

**Why a dedup pipeline cares:** Glaze is *not malicious*, but it changes the data distribution. A caption-image corpus containing many Glazed images shifts the model's representation of styles in artist-labelled content. Glaze also has a distinctive distributional signature in image spectrum. The operational question is: *for this corpus, are we licensed to train on content that has been Glazed?* That is a licensing / ethics question (see [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md)); the dedup pipeline's job is to **surface the content so the licensing decision can be made**.

## Why both tools are "dedup problems"

Nightshade and Glaze perturbations tend to cluster: an artist Glazes a batch of images; an attacker Nightshades a batch of images; the perturbation signature (frequency-domain footprint, pixel-level noise pattern) is recognisable across the batch. In a multimodal corpus, the dedup pipeline's fingerprint over images is a natural place to also compute a perturbation-detection signal — because:

- You already pay the cost of loading every image.
- You already compute a hash / fingerprint per image.
- Adding a perturbation-signature column to the fingerprint record is cheap.

What this chapter is *not*: the authoritative detection methodology. Detection is `ai-risk-engineer`'s responsibility. What this chapter *is*: the pipeline-side responsibility to collect the right signals and route them to that peer.

## MITRE ATLAS mapping

MITRE ATLAS is the adversarial-ML analogue of the ATT&CK framework. It is the vocabulary your security partners use. <https://atlas.mitre.org/>.

The techniques most directly relevant to this chapter:

- **AML.T0020 — Poison Training Data.** Covers both clean-label and backdoor poisoning. Your dedup-side quarantine of suspected poisoned clusters is the control for this technique.
- **AML.T0019 — Publish Poisoned Datasets.** Covers the attacker shape where the attacker publishes a dataset expecting it to be scraped.
- **AML.T0051.001 — LLM Prompt Injection via Training Data.** Relevant to LLM training on web text where instruction-like content has been injected.
- **AML.T0018 — Backdoor ML Model.** Downstream consequence of successful training-data poisoning.

Your dedup pipeline's quarantine-and-escalate behaviour is a **mitigation** for AML.T0020 and AML.T0019 and an **evidence source** for the detection tooling `ai-risk-engineer` owns. (ATLAS has continued to evolve: the 2026.07 release, for example, renamed AML.T0020 from "Poison Training Data" to "Training Data Poisoning" without changing the identifier. Re-check the current technique titles and coverage at <https://atlas.mitre.org/> before citing.)

## OWASP ML Top 10 and OWASP LLM Top 10

**OWASP ML Top 10 (2023):** <https://owasp.org/www-project-machine-learning-security-top-10/>.

- **ML02:2023 Data Poisoning Attack.** The canonical entry. Your controls: integrity-checked ingestion (Chapter 02's hashes), outlier detection in training data (near-dup cluster size anomalies), source-allowlist constraints ([`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md) carries the allowlist).

**OWASP LLM Top 10 (2023):** <https://owasp.org/www-project-top-10-for-large-language-model-applications/>.

- **LLM03:2023 Training Data Poisoning** in the 2023 edition; in the 2025 edition this is renumbered and broadened as **LLM04:2025 Data and Model Poisoning** (<https://owasp.github.io/www-project-top-10-for-large-language-model-applications/>). The item has moved across editions, so cite the specific edition and item number you are auditing against. Your dedup pipeline's duplicate-cluster-size distribution and per-source removal rate are direct evidence for or against poisoning in a slice.

**The division-of-responsibility language from `mod-101`:**

- `ai-risk-engineer` (peer, level 25) owns **detection methodology** — chooses and runs the detectors, tunes thresholds, interprets ambiguous results.
- You (`ai-data-engineer`) own **pipeline-side quarantine responsibility** — build the hooks, emit the signals, carry out the quarantine policy, produce auditable artifacts.

A real production split that works: `ai-risk-engineer` writes `detectors/nightshade.py`; you integrate it as a pipeline operator; you quarantine the flagged content into a separate bucket for review; both of you co-own the operational response.

## Pipeline-side detection heuristics you can run today

Even without a Nightshade-specific detector, a dedup pipeline can emit useful signals. Four worth adding to the dedup report:

### 1. Cluster-size anomaly

A cluster of 50,000 near-duplicates from the same TLD, published in a one-week window, is already suspicious. Emit cluster-size histograms per source per time bucket. An attacker publishing poisoned content at scale has a cluster-size signal whether they intended one or not.

### 2. Perturbation-signature column (multimodal)

For each image (or audio sample), emit:

- Standard perceptual hash (e.g. pHash, dHash).
- A frequency-domain summary (e.g. DCT coefficient statistics at a few bands).
- An image-noise statistic (e.g. ratio of high-frequency content to expected natural-image content).

Known perturbation tools tend to cluster in these statistics. The column is cheap to compute and gives `ai-risk-engineer` a feature set to build detectors on.

### 3. Source-level removal-rate spikes

A source whose removal rate jumps from 2% to 60% in one crawl is either (a) newly serving syndicated content or (b) adversarially crafted content designed to pass your filters but fail dedup. Alert on this.

### 4. Zero-day clusters

A cluster of near-duplicates where **every source is new** (never seen in prior slices) is suspicious. Legitimate near-dup clusters usually span sources seen before. Zero-day clusters are candidates for quarantine review.

None of these are a substitute for a real detector. They are *pipeline-emitted signals* that a real detector consumes.

## The quarantine shape

A quarantine-and-escalate pipeline, rather than drop-silently, has three properties:

1. **Flagged content is routed to a separate bucket** (e.g. `s3://.../my_slice/quarantine/<reason>/`), not deleted.
2. **The quarantine manifest is signed and dated**, with the reason-code, the detector / heuristic version, and the evidence SHAs.
3. **`ai-risk-engineer` has a well-defined review workflow** against the quarantine bucket — SLA, review template, and a release / confirm-delete decision gate.

The quarantine bucket is itself an audit artifact. It feeds the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) field *"known filtered-out content and why"* and the privacy-risk report ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)) field *"adversarial perturbation review verdicts"*.

## What *not* to do

Four anti-patterns that come up:

- **Do not silently drop suspected poisoned content without a quarantine record.** Silent drops leave no audit trail and cannot be reviewed by `ai-risk-engineer`.
- **Do not aggregate cluster-size histograms *only* across the whole slice.** Per-source and per-time-window disaggregation is where the signal lives.
- **Do not promise detection coverage for threats you have not implemented a detector for.** The dedup report should be explicit about which threats it covers (dedup-visible repetition patterns) and which it does not (zero-trace clean-label poisoning).
- **Do not run production detectors you built yourself without `ai-risk-engineer` review.** Detection methodology is not your role; building the pipeline that routes signals to the right detector is.

## What goes into the dedup report at this layer

Chapter 08 collects at minimum:

- `cluster_size_histogram` per `(source, time bucket)`.
- `source_removal_rate_delta_vs_prior_slice`.
- `zero_day_cluster_count`.
- `perturbation_signature_stats` (multimodal slices only).
- `quarantined_doc_count` and `quarantine_manifest_sha256`.
- `atlas_techniques_referenced` and `owasp_items_referenced` for the slice's risk register.
- `ai_risk_engineer_review_sign_off`: boolean + reviewer identifier + date.

A dedup report that cannot point at an `ai-risk-engineer` sign-off for the adversarial-perturbation review is an incomplete report.

## Summary

- **Dedup pipelines are a natural place to emit adversarial-perturbation signals** because they already fingerprint every document.
- **Nightshade** and **Glaze** are the two public perturbation tools you must know by name; detection research for both is active.
- **MITRE ATLAS** (AML.T0019, AML.T0020, AML.T0051.001, AML.T0018) and **OWASP ML Top 10 ML02** / **OWASP LLM Top 10** training-data-poisoning items are the vocabulary your security partners use. Cite them in the dedup report.
- **Division of labour:** `ai-risk-engineer` owns detection methodology; you own pipeline-side quarantine responsibility. Both deliverables are mandatory.
- **Quarantine, do not silently drop.** Emit a signed, dated, SHA-linked quarantine manifest.
- **Pipeline-side heuristics** (cluster-size anomaly, perturbation-signature columns, source-level removal-rate spikes, zero-day clusters) are evidence sources for the real detector — not a substitute for it.

Chapter 08 assembles the module's deliverable — the dedup report — including the adversarial-perturbation section this chapter feeds.

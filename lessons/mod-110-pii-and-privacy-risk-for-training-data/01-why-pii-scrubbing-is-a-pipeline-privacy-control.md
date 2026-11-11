# Chapter 01 — Why PII Scrubbing Is a Pipeline Privacy Control

## Why this chapter exists

Pipeline-side PII work is often introduced as a tidying step — "run a regex, scrub some emails, call it compliance". That framing is wrong by an order of magnitude. Peer-reviewed research shows that training-data pipelines are a decisive leverage point in determining whether downstream LLMs leak PII:

- A production-scale LM can be queried to **extract verbatim training-data records**, including names, email addresses, phone numbers, and GPS coordinates (Carlini et al. 2020, USENIX Security 2021).
- The probability a sequence is memorised scales **log-linearly with the number of times it appears in the training data**, with model parameter count, and with prompt length (Carlini et al. 2022, ICLR 2023).
- **Deduplication alone** — a pipeline-side control — reduces training-data-extraction-attack success rates by **roughly an order of magnitude** across tested models (Kandpal et al. 2022, ICML 2022).

In other words, the training-data pipeline is not just where PII enters the system — it is where the **memorisation-risk surface is set**, before a single forward pass is run. This chapter grounds the rest of the module in that evidence so that when you fight for CPU-hours and engineering headcount against a tight deadline, you can defend the work.

## The three claims you must be able to defend

Pin these three claims. They are the ones you will be asked to support in a design review and in a counsel briefing.

### 1. Training-data extraction is a measured phenomenon, not a theoretical worry

**Paper:** Carlini, Tramèr, Wallace, Jagielski, Herbert-Voss, Lee, Roberts, Brown, Song, Erlingsson, Oprea, Raffel. *Extracting Training Data from Large Language Models.* USENIX Security 2021 (arXiv 2012.07805). <https://arxiv.org/abs/2012.07805>.

Key findings, in the shape you should cite:

- The authors demonstrate a **black-box** training-data extraction attack against GPT-2. Using only API-style queries, they recover verbatim training sequences including PII (names, email addresses, phone numbers), code snippets, UUIDs, and IRC conversations.
- The attack works because large LMs assign unusually high likelihood to sequences they memorised verbatim. Prompt + sampling + membership-inference-style scoring surfaces these sequences.
- The paper demonstrates the attack on GPT-2, but the mechanism (high likelihood → recoverable) is a property of maximum-likelihood training, not of GPT-2 specifically. The follow-up work by Nasr et al. 2023 (*Scalable Extraction of Training Data from (Production) Language Models*) demonstrates the same mechanism at production-model scale.
- Quantitatively (Carlini et al. 2021, §6): the authors confirmed **604 unique memorised training examples** out of 1,800 candidate generations, an aggregate true-positive rate of **33.5%**. Among the 604, their category breakdown includes **46 "named individuals"** and **32 "contact information"** examples (addresses, emails, phone numbers, social handles) — i.e., on the order of tens of verbatim PII strings recovered from the public GPT-2 model via black-box queries.

**Why it matters for your pipeline:** every PII string that enters the corpus is a candidate extraction target for someone who later queries the trained model. The pipeline is the only point at which you can prevent that string from becoming one.

### 2. Memorisation is a smooth function of duplicate count

**Paper:** Carlini, Ippolito, Jagielski, Lee, Tramèr, Zhang. *Quantifying Memorization Across Neural Language Models.* ICLR 2023 (arXiv 2202.07646). <https://arxiv.org/abs/2202.07646>.

Key findings:

- Memorisation scales **log-linearly with duplicate count** — a sequence seen 10× is dramatically more memorisable than one seen once; a sequence seen 100× more again.
- Memorisation scales **with model parameter count** — larger models memorise more aggressively at fixed data.
- Memorisation scales **with prompt length** — longer prompts extract longer verbatim completions.
- The effect is **not threshold-shaped**; there is no "safe" duplicate count below which memorisation is zero. The curve is smooth, which means pipeline interventions compound.

**Why it matters for your pipeline:** the pipeline controls exactly one of those three levers — duplicate count. A single PII string that leaks through the detector is a lower risk than the same PII string duplicated across fifty mirrored sites. The dedup step ([`mod-105`](../mod-105-deduplication-at-scale/README.md)) and the detector step (this module) are complementary, not redundant.

### 3. Deduplication is itself a privacy control

**Paper:** Kandpal, Wallace, Raffel. *Deduplicating Training Data Mitigates Privacy Risks in Language Models.* ICML 2022 (arXiv 2202.06539). <https://arxiv.org/abs/2202.06539>.

Key findings:

- The **frequency of a sequence in training data is the strongest predictor** of its susceptibility to membership-inference and training-data-extraction attacks.
- Measured on GPT-Neo / GPT-2 style models: after dedup, **extraction-attack success rates drop by roughly an order of magnitude** across models tested in the paper.
- Dedup produces this gain **independently of any PII-specific scrubbing**.

**Why it matters for your pipeline:** dedup is already in your pipeline for quality reasons ([`mod-105`](../mod-105-deduplication-at-scale/README.md), Lee et al. 2021). The Kandpal result is that it is *also* carrying load on privacy. If you were planning to skip dedup on a "small" slice because it felt wasteful, Kandpal is the reason not to. If a stakeholder asks whether we "really need both scrubbing and dedup", the answer is yes: they attack the same memorisation surface from different angles (identification vs. frequency), and the gains compound.

## The pipeline's three jobs

Given those three claims, the pipeline has exactly three jobs. This chapter is the scope definition for the rest of the module; every subsequent chapter maps to one of them.

1. **Prevent PII from entering the corpus where we can detect it** — Chapters 02–05.
2. **Minimise the memorisation surface that remains** — pipeline-side controls are scrub + dedup + quarantine high-risk documents. Chapter 06 wires them together; Chapter 07 is the honest accounting of what the pipeline can and cannot stop.
3. **Emit an auditable privacy-risk report** — the deliverable that counsel, `fine-tuning-engineer`, `ai-risk-engineer`, and the dataset card all consume. Chapter 09 is the schema; Chapter 08 is the lawful-basis evidence that the report packages.

## Scope boundary — what this module owns and what it hands off

Pipeline-side privacy work ends at the signed report. Model-side privacy work begins at training. The boundary is sharp and well-studied — do not blur it.

| Concern | Owned by | Why |
|---|---|---|
| PII detection in corpus text | **This module** | It is a data-pipeline concern. Detector tooling runs on the raw corpus. |
| Redaction primitive choice per entity type | **This module** | The utility implications are data-shape implications. |
| Dedup as a privacy control | **[`mod-105`](../mod-105-deduplication-at-scale/README.md), coordinated here** | The pipeline step lives in `mod-105`; this module cites it. |
| High-risk-document quarantine | **This module** | We produce the quarantine manifest; counsel decides whether to release the slice. |
| Lawful-basis engineering evidence | **This module** | We package the evidence; counsel forms the opinion. |
| Legal opinion on lawful basis | **`privacy counsel`** | We do not practise law. Chapter 08 is explicit about this. |
| DP-SGD training-side noise injection | **`fine-tuning-engineer` (peer, L30)** | It is a training-loop concern, not a corpus concern. |
| Output-filter / canary-based leak detection | **`fine-tuning-engineer`** | Post-training evaluation. |
| Attack-simulation tooling (MIA / extraction) | **`ai-risk-engineer` (peer, L25)** | Adversary-mindset craft. We provide the signals; they build the detectors. |
| Red-team exercises against the trained model | **`ai-risk-engineer`** | Post-training. |

The handoff mnemonic: *we scrub and dedup the corpus and document what we did; fine-tuning adds noise during training and documents what they did; risk simulates attacks on the result and documents what it found*. All three outputs land in the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)).

## The ISO framing

This framing fits cleanly into the ISO vocabulary [`mod-101`](../mod-101-ai-data-governance-and-position/README.md) pinned for you:

| Standard / clause | How this module contributes |
|---|---|
| **ISO/IEC 25012 — confidentiality** | The quality characteristic most directly operationalised by PII scrubbing. The residual-PII count in the privacy-risk report is the measure. |
| **ISO/IEC 27701 — PIMS** | PII categories and purposes are tracked per-slice; the privacy-risk report is a PIMS record. |
| **ISO/IEC 20889** | The de-identification technique vocabulary used throughout Chapter 05. |
| **ISO/IEC 5259-5** | The Part-5 measure family for privacy / PII coverage in training data. |
| **ISO/IEC 42001 — Annex A.7** | The audit hook. Sensitive-data handling is an auditable control; `privacy_risk_report.yaml` is the evidence. |

## What a "credible" pipeline looks like

A pipeline that cannot answer these six questions is not training-ready:

1. Which detectors ran, with which thresholds, on which stage? → Chapter 02–04, 06.
2. What is the detector's precision and recall on a slice-representative eval set? → Chapter 04.
3. Which redaction primitive was applied per entity type, and what is the training-utility cost? → Chapter 05.
4. What is the residual PII count, disaggregated by source? → Chapter 09 §3.
5. What is the memorisation-risk delta attributable to dedup? → Chapter 06 § and Chapter 09 §4.
6. What lawful-basis evidence has been packaged for counsel? → Chapter 08, Exercise 03.

Every exercise in this module contributes an answer to one of those questions. Exercise 05 assembles them into the signed report.

## The costs you are offsetting

A sincere cost-benefit case for your lead should include the honest costs on the other side of the ledger:

- **Pipeline complexity.** Presidio's analyser is a non-trivial stage with its own tuning surface (Chapter 04). Running a transformer NER at corpus scale is a GPU-hours bill.
- **False positives.** A detector that flags too broadly destroys legitimate training signal (Chapter 05 — if you redact "the president of the United States is …" because you flagged PERSON, you have broken the sentence).
- **False negatives.** A detector that misses too often provides false assurance; "we scrubbed it" without recall evidence is worse than "we did not scrub it" because it mis-sets downstream confidence.
- **Utility loss from redaction.** Over-aggressive hashing / masking destroys the distributional signal the model would otherwise learn. Chapter 05's primitive-choice table is the mitigation.
- **Vendor and jurisdiction coupling.** Cloud DLP tools (Google DLP, AWS Comprehend PII) are strong detectors but create a data-residency and vendor-dependency question. Chapter 03 treats this.

The empirical answer from Carlini 2020 / 2022 and Kandpal 2022 is that the benefits outweigh these costs at the corpus scales you will operate on. You should be able to recite that trade-off without flinching.

## Summary

- PII scrubbing in the training-data pipeline is a **load-bearing privacy control**, not operational hygiene. Three findings pin this: **Carlini 2020** (extraction is measured), **Carlini 2022** (memorisation is a smooth function of duplicate count / model size / prompt length), **Kandpal 2022** (dedup alone cuts attack success by an order of magnitude).
- The pipeline has three jobs — **detect**, **minimise memorisation surface**, **report**. Chapters 02–05 cover detection and redaction; Chapters 06–07 cover minimisation; Chapters 08–09 cover reporting.
- The boundary is sharp: **DP-SGD belongs to `fine-tuning-engineer`**; **attack simulation belongs to `ai-risk-engineer`**; **legal opinion belongs to counsel**. We own corpus-side scrubbing, dedup coordination, quarantine, and the audit evidence.
- The deliverable is `privacy_risk_report.yaml` — Sigstore-signed, consumed by the dataset card, the lineage record, counsel, and the two peer roles.

Chapter 02 picks up with the main detection toolkit — Microsoft Presidio — and the Analyzer / recogniser / context model that the rest of the toolchain chapters build on.

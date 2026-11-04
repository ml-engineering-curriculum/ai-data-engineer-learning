# Chapter 07 — Memorisation Attacks and Pipeline-Side Defences

## Why this chapter exists

Chapters 02–06 built the pipeline. This chapter is the honest accounting of **what the pipeline can and cannot stop**, and the scope boundary with `fine-tuning-engineer` and `ai-risk-engineer`.

Three claims from Chapter 01 are revisited here in attack-shape:

- **Carlini et al. 2020** demonstrates a working black-box extraction attack against GPT-2. Pipeline defences reduce the extractable surface but do not eliminate it.
- **Carlini et al. 2022** quantifies memorisation as a smooth function of three inputs: duplicate count, model size, prompt length. The pipeline controls only the first.
- **Kandpal et al. 2022** shows that deduplication alone cuts attack success by roughly an order of magnitude. The *compound* benefit of scrub + dedup is larger.

Internalising these three claims lets you have the design-review conversation without over-promising. "We ran Presidio" does not close the privacy threat model. "We ran Presidio, dedup'd per `mod-105`, and quarantined high-density documents; residual memorisation risk is in the privacy-risk report and the attack-simulation evaluation is `ai-risk-engineer`'s next step" does.

## The Carlini 2020 extraction attack — in one page

**Paper:** Carlini, Tramèr, Wallace, Jagielski, Herbert-Voss, Lee, et al. *Extracting Training Data from Large Language Models.* USENIX Security 2021 (arXiv 2012.07805). <https://arxiv.org/abs/2012.07805>.

### The attack mechanism

1. **Generate** — prompt the target model with many short prefixes (prompts can be anything: the empty string, random text, common phrases) and collect long completions.
2. **Rank** — score each completion by a membership-inference signal. The simplest is **perplexity ratio**: the completion's perplexity under the target model divided by its perplexity under a reference model (a smaller LM, zlib compressibility, lowercase perplexity, etc.). Low ratios suggest the completion is memorised.
3. **Verify** — manually (or automatically) check whether the top-ranked completions are present in the training data.

The paper demonstrates recovery of PII (names, emails, phone numbers), code snippets, URLs, UUIDs, and IRC conversations from GPT-2. The attack is a **black-box** attack: no gradient access, no training-data access, just an API that returns token probabilities.

### What the pipeline can do

- **Remove easy targets**: a PII string that is not in the training data cannot be extracted from the model. Detection + redaction (Chapters 02–05) does exactly this.
- **Reduce frequency**: duplicate frequency is the strongest predictor of extraction success. Dedup (Chapter 06, `mod-105`) collapses frequency.
- **Quarantine high-risk documents**: documents that are dense with PII are both hard to scrub and high-memorisation-risk. Quarantine them (Chapter 06).

### What the pipeline cannot do

- **Stop memorisation of long-tail sequences**: even a sequence that appears exactly once can be memorised by a large enough model (Carlini 2022). Pipeline-side scrub is a probabilistic reduction, not a guarantee.
- **Stop the attack itself**: the mechanism (likelihood ranking) is intrinsic to maximum-likelihood training.
- **Prove absence of memorisation**: running Presidio is not a proof. The residual PII count is a ceiling on the amount detection + redaction removed; the amount memorised is a separate measurement.

## The Carlini 2022 memorisation-scaling result

**Paper:** Carlini, Ippolito, Jagielski, Lee, Tramèr, Zhang. *Quantifying Memorization Across Neural Language Models.* ICLR 2023 (arXiv 2202.07646). <https://arxiv.org/abs/2202.07646>.

Three empirical laws, measured on GPT-Neo:

- **Memorisation is log-linear in duplicate count.** A sequence seen 10× is roughly an order of magnitude more extractable than one seen once; 100× is two orders.
- **Memorisation is log-linear in model size.** Doubling parameters roughly doubles the memorisation rate at fixed data.
- **Memorisation grows with prompt length.** Longer prompts produce longer verbatim completions.

What this means for the pipeline:

- The pipeline controls duplicate count directly. Each duplicate removed is a corresponding reduction in memorisation probability.
- The pipeline does **not** control model size — that is `fine-tuning-engineer` / model-team territory. If the model is scaled up and the training mixture is unchanged, memorisation goes up.
- The pipeline does **not** control prompt length at inference — but it does control sequence length during training. If your tokenisation step produces 2048-token sequences, you have set an upper bound on single-context memorisation.

The log-linear scaling has an implication for budgeting: the compound reduction (scrub + dedup) is multiplicative. If scrub removes 90% of PII spans and dedup reduces the surviving memorisation probability by 10×, the net reduction is roughly 100×. The two controls are **not redundant**.

## The Kandpal 2022 result — pipeline side

**Paper:** Kandpal, Wallace, Raffel. *Deduplicating Training Data Mitigates Privacy Risks in Language Models.* ICML 2022 (arXiv 2202.06539). <https://arxiv.org/abs/2202.06539>.

The specific claim: on tested GPT-Neo / GPT-2 configurations, extraction-attack success rates **drop by roughly an order of magnitude** after deduplication of the training corpus. The pipeline implication is covered in Chapter 06 — scrub and dedup, in that order, cite the dedup report by SHA, report both pre-dedup and post-dedup memorisation-risk numbers.

## The Nasr 2023 follow-up — attacks at production scale

**Paper:** Nasr, Carlini, Hayase, Jagielski, Cooper, Ippolito, Choquette-Choo, Wallace, Tramèr, Lee. *Scalable Extraction of Training Data from (Production) Language Models.* arXiv 2311.17035, 2023. <https://arxiv.org/abs/2311.17035>.

This paper demonstrates that the Carlini 2020 attack mechanism **scales** to production-class models. It is not a research curiosity against GPT-2; it is a production threat model.

The pipeline-side implications are the same as Carlini 2020. The paper is listed here because when a stakeholder asks whether the Carlini 2020 result still matters for a frontier-scale model, the answer is Nasr 2023.

## The pipeline-side defence stack, summarised

| Defence | What it does | Covered in |
|---|---|---|
| **Detection + redaction** | Removes PII spans before they enter training | Chapters 02–05 |
| **Dedup** | Collapses duplicate frequency | `mod-105` + Chapter 06 |
| **High-risk document quarantine** | Drops documents too dense with PII to redact safely | Chapter 05, Chapter 06 |
| **Reproducible, audited pipeline** | Lets you trace what happened to a given document | Chapter 06, Chapter 09 |
| **Per-run detector-quality manifest** | Measures detector precision / recall against an eval set | Chapter 04, Chapter 06 |
| **Signed privacy-risk report** | Carries the evidence to counsel and to the training team | Chapter 09 |

This is **the full set of pipeline-side defences**. There is no sixth control hiding in a vendor SKU. Anything claiming to be one is almost certainly a model-side control dressed up as a data-side one.

## The handoff boundary — what we do not own

### DP-SGD and other training-side noise mechanisms → `fine-tuning-engineer` (peer, L30)

**Differentially-private stochastic gradient descent (DP-SGD)** (Abadi et al. 2016, *Deep Learning with Differential Privacy*, CCS 2016, <https://arxiv.org/abs/1607.00133>) is the leading training-side privacy mechanism. It clips per-example gradients and injects calibrated noise during training, producing a model with a formal (ε, δ)-DP guarantee.

Why it is not ours:

- The implementation lives in the training loop (Opacus, TensorFlow Privacy, JAX-privacy). It modifies gradients, not data.
- The hyperparameters (clip norm, noise multiplier, privacy budget ε) are a training-side trade-off against accuracy — not a corpus-side one.
- The guarantee it produces is about the trained model, not about the corpus.

What we hand over:

- The scrubbed corpus and the privacy-risk report. `fine-tuning-engineer` uses them to decide the DP budget and the training recipe.
- The memorisation-risk delta. If dedup already cut extraction risk by 10×, the marginal privacy budget needed from DP-SGD is smaller.

The division of labour is **complementary, not duplicative**. A corpus with no scrubbing and tight DP-SGD is still worse than a corpus with scrubbing and tight DP-SGD, because DP-SGD only bounds the risk; scrubbing removes targets.

### Output filtering, canary insertion, extraction-attack simulation → `ai-risk-engineer` (peer, L25)

**Output filters** (post-hoc PII detectors on model generations), **canary insertion** (planted sequences to measure memorisation), and **extraction-attack harnesses** (automated Carlini 2020-style probes) are post-training evaluations and mitigations.

Why they are not ours:

- They operate on a trained model, not on a corpus.
- They require adversary-mindset design and attack-model iteration — `ai-risk-engineer` craft.

What we hand over:

- The detection-policy manifest. `ai-risk-engineer` uses it to seed canaries that match the policy's known blind spots.
- The eval-set itself. The same eval set used to measure detector recall is a useful seed for extraction-attack probes.
- The high-risk quarantine manifest. Documents we quarantined are documents the attack harness should probe hard against.

### Legal opinion on lawful basis → `privacy counsel`

Chapter 08 is the regulatory literacy. We do not provide legal opinions. We provide the engineering evidence that counsel needs. That boundary is non-negotiable; crossing it is practising law without a licence.

## A probe you can run before training — the self-check

Before releasing the slice, run a small-scale memorisation probe yourself. This is **not** an extraction-attack simulation (that is `ai-risk-engineer`), but a cheap sanity check that the pipeline did what it claimed:

1. **Plant canaries.** Insert 100–1,000 unique synthetic PII-shaped sequences (e.g., random valid-looking card numbers or distinctive strings like "The quick brown kiwi jumps over the lazy aardvark #{UUID}") into the corpus at known positions.
2. **Run a small LM (125M–1B params) on the scrubbed slice.** A tiny model, a short training run.
3. **Prompt the trained model** with the prefixes of the planted canaries.
4. **Measure** how many canaries are regurgitated verbatim. Report the fraction in the privacy-risk report.

This probe is cheap and gives a concrete number for the handoff. It is **not a substitute** for a full extraction-attack evaluation on the production model — the small model's memorisation behaviour is a lower bound on the production model's, not an upper bound. Document that caveat in the report.

Exercise 04 of this module is exactly this probe. The acceptance criterion is a reproducible per-canary-recovery-rate number.

## The attack-literacy reading list

For the design-review conversation, you should have read (at minimum):

- Carlini et al. 2020 (the attack).
- Carlini et al. 2022 (the scaling).
- Kandpal et al. 2022 (the dedup result).
- Nasr et al. 2023 (production-scale follow-up).

The optional-but-helpful reading list is in [`resources.md`](resources.md). It includes membership-inference foundations (Shokri 2017), MIA from first principles (Carlini 2022b), and Lukas et al. 2023 on PII leakage specifically.

## Summary

- The pipeline-side defences are **detection + redaction + dedup + quarantine + reproducible audit trail**. Each is necessary; none is sufficient alone.
- The compound reduction is **multiplicative**. Scrub and dedup attack the same surface from different angles and compound.
- The pipeline **cannot** eliminate memorisation of long-tail sequences, scale with model size, or provide a formal DP guarantee. Those are model-side concerns.
- **DP-SGD belongs to `fine-tuning-engineer`.** We hand over the scrubbed corpus and the privacy-risk report; they set the training recipe.
- **Attack-simulation tooling belongs to `ai-risk-engineer`.** We provide the detection manifest, the eval set, and the quarantine manifest.
- **Legal opinion belongs to counsel.** We provide the engineering evidence (Chapter 08); they form the opinion.
- Run a **canary probe** before release as a cheap pipeline-side self-check. Exercise 04 of this module is that probe.

Chapter 08 picks up with lawful basis — the GDPR / CCPA / HIPAA literacy and the engineering-evidence package that counsel needs from this module.

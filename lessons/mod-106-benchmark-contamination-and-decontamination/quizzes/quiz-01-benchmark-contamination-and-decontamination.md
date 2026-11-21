# Quiz 01 — Benchmark Contamination Detection and Decontamination Engineering

**Module:** [mod-106 — Benchmark Contamination Detection and Decontamination Engineering](../README.md)
**Format:** 12 multiple-choice questions, one correct answer per question.
**Suggested time:** 25 minutes.
**Coverage:** Chapters 01–08.

These questions test whether you can *use* the module's operators — defend the firewall as a non-negotiable credibility control, pin an eval manifest that stays aligned with the scoring harness, pick an n-gram width against a labelled sample, decide when the LLM-substring detector is the right tool, allocate the firewall budget across the four pipeline surfaces, pick a public recipe (and defend the pick), anticipate Min-K% Prob and the Oren exchangeability test, and emit a decontamination log a reviewer can read in five minutes — not whether you can recite chapter wording. Answer key with references is at the bottom.

---

## Q1. Why the contamination firewall is non-negotiable

A programme lead asks why a week of pipeline work on a contamination firewall is worth the compute budget. Chapter 01 says there are *three* public findings you must be able to cite, in a specific shape. Which option matches?

- **A.** It is a compliance requirement under ISO/IEC 42001 Annex A.7; no empirical justification is needed.
- **B.** **GPT-3 Appendix C (Brown et al. 2020)** — the leading pretraining lab shipped a load-bearing contamination appendix in 2020 using a 13-gram overlap convention, and every subsequent open LLM release is read against that bar; **Shi et al. 2023 Min-K% Prob + Oren et al. 2023 exchangeability test** — two canonical post-hoc detectors that prove contamination of a black-box model with only API access, for which there is **no technical defence** once the model is released; and **EleutherAI LM Evaluation Harness + AllenAI Dolma** — two public, open-source decontamination recipes that make the practice standardised and reproducible. Together these pin the firewall as a credibility control, not operational hygiene.
- **C.** Dolma published a 3T-token corpus; that single reference is enough on its own.
- **D.** Min-K% Prob is cheap to defend against post-hoc, so the firewall is only needed for publicity reasons.

---

## Q2. The three operational definitions of "contamination"

A stakeholder asks whether a firewall that catches byte-equal copies of eval prompts is "enough". Chapter 01 says the word is slippery and distinguishes **three** operational definitions whose intersection is empty. Which option matches?

- **A.** "Exact-prompt" and "n-gram overlap" are synonyms; a byte-equal firewall is sufficient.
- **B.** The three definitions are **exact-prompt** (byte-equal after normalisation; easiest to detect and most legally damning), **n-gram overlap** (a sliding-window span from the eval item appears in training — the GPT-3 Appendix C definition — catches lightly-edited and templated copies), and **semantic** (the content is paraphrased, translated, or retold; not fully solvable at pipeline scale and relies on the LLM-based substring detector). An n-gram firewall at width 13 does not remove paraphrases; the Min-K% Prob audit will catch whichever definition you did not implement. State explicitly which definitions the firewall implements.
- **C.** Semantic contamination can be caught at trillion-token scale by tuning the n-gram width down to 3.
- **D.** The three categories are "pretrain", "SFT", and "preference-pair" — surfaces, not definitions.

---

## Q3. Variant drift at the manifest-signoff gate

Your firewall defended the `mmlu` test split. On the eve of release, the model-evaluation-engineer peer submits headline scores against `MMLU-Pro`. Chapter 02 is explicit about how this situation is prevented and what the release-time check is. Which option matches?

- **A.** `MMLU-Pro` is a superset of `MMLU`; the firewall's pass on the original is automatically valid.
- **B.** The two are statistically interchangeable; the release can proceed.
- **C.** The `eval_manifest.yaml` has a **mandatory `variant` field** for benchmarks with variants (MMLU → original / Pro / Redux; HumanEval → original / plus / MultiPL-E). The **model-evaluation-engineer sign-off block in the manifest is a release gate** — if the sign-off cites `mmlu_pro` and the firewall defended `mmlu_original`, the release is **blocked**, not reconciled after the fact. Chapter 08 §1 (identity section) then requires both the manifest version and the specific benchmark variants cited in the log.
- **D.** Pin the HF `revision` hash and the problem is solved regardless of variant.

---

## Q4. The normalisation contract between firewall and scoring harness

A new engineer ships a firewall that lowercases MMLU prompts and strips `(A)` / `A.` / `A)` renderings to a single canonical letter — but the scoring harness renders `A.` verbatim and is case-sensitive. Chapter 02 (and Chapter 06) pin the operational rule. Which option matches?

- **A.** The firewall should always be stricter than the harness; mismatch is harmless.
- **B.** **The detector's normalisation must match the scoring harness's normalisation.** Mismatch is the **dominant source of silent false negatives** — the firewall sees `(A)` and the harness scores against `A.`, so the firewall misses a leak the harness would score correctly. The contract is implemented via the `normalisation_rules_sha256` field (the same SHA applies to both sides of the detector), and the cheapest insurance is the Chapter 06 scoring-alignment round-trip: render 20 items as the harness renders them, inject them into synthetic training documents, and expect **20 / 20** flags from the firewall.
- **C.** Normalisation does not matter because `xxh128` collisions dominate at trillion-token scale.
- **D.** The firewall should normalise both sides more aggressively than the harness to improve recall.

---

## Q5. Picking the n-gram width

A reviewer asks why your firewall uses `n = 13`. Chapter 03 pins the failure modes at each end of the curve and the operational way to pick. Which option matches?

- **A.** `n = 13` is universal because GPT-3 Appendix C used it; no calibration is needed.
- **B.** Picking `n` too *small* produces **false positives** — common phrases (`"the answer is"`, `"import numpy as np"`) appear in both eval items and legitimate training content. Picking `n` too *large* produces **false negatives** — a lightly-edited copy of an MMLU item no longer contains any length-`n` window identical to the original. The operational drill is a **~200-pair labelled sample** with three-way labels (`leak` / `incidental-overlap` / `unrelated`), a precision / recall sweep over `n ∈ {8, 10, 13, 20, 50}`, and the choice is the smallest `n` whose precision ≥ 0.98. Different public recipes (EleutherAI, Dolma) pick different operating points on the same curve; both are defensible.
- **C.** Lower `n` is always better because it catches paraphrases too.
- **D.** Higher `n` is always better because it eliminates false positives.

---

## Q6. Suffix array vs streaming scan — picking the data structure

Chapter 03 names two canonical data-structure shapes for the n-gram detector. A reviewer asks which to use for a trillion-token pretrain corpus that will be extended monthly by continued pretraining batches. Which option matches?

- **A.** Always use a Bloom filter; it is the cheapest option regardless of use case.
- **B.** Always use the streaming set-scan; suffix arrays are impractical past 1 GB.
- **C.** **Suffix array** (Lee et al. 2021's `ExactSubstr`, Dolma-style) is correct when you need **per-document attribution** and **incremental updates** — a sidecar `offset_to_doc` mapping carries which training document contains each hit, and the SA can be built sharded and extended without a full rebuild. **Streaming set / Bloom-filter scan** (EleutherAI-harness-style) is correct when the **eval side is small** (fits in memory as ~10⁶ 128-bit hashes at `n = 13`) and the corpus is huge and streamable — it is embarrassingly parallel, one worker per shard. Pick by the question "do I need attribution and incremental updates?". For a trillion-token corpus with monthly continued-pretraining extensions, the SA-backed shape wins.
- **D.** The two shapes produce different removal counts at scale; pick whichever is lower.

---

## Q7. The LLM-substring detector — scope and when to use it

An engineer proposes running the LLM-substring detector corpus-wide across a 1T-token pretrain corpus "to catch every paraphrase". Chapter 04 pushes back. Which reply matches?

- **A.** Corpus-wide is fine — at Haiku 4.5 prices it is cheap.
- **B.** The detector is a **filter after prefiltering**, **not** a corpus-wide scanner at pretrain scale — invoking an LLM per span across 10¹² tokens is cost-prohibitive. The prefilter is the n-gram and exact-prompt detectors, which produce candidate spans (any ≥ 6-token overlap with any eval item, or a span from a **high-prior-leak source** such as a GitHub HumanEval repo, a LeetCode / Codeforces discussion, an AoPS page, a Wikipedia article about a specific MMLU item). On the **smaller surfaces** (SFT, preference-pair, synthetic generation), volume is low enough that the detector is promoted to **corpus-wide** — Chapter 05 pins the per-surface scope. The lever against cost is **prefilter tightness**, not model choice: a 10× tighter prefilter saves 10× on LLM spend.
- **C.** The detector should run only on preference-pair surfaces; pretrain is fine with n-gram only.
- **D.** The LLM detector always returns "copy" on commentary; disable it on prefiltered spans.

---

## Q8. Why SFT gets a disproportionate share of the firewall budget

A programme lead proposes spending 95% of the firewall engineering budget on pretrain and 5% on SFT, "because pretrain is 10⁵× larger". Chapter 05 is explicit about why that is the wrong allocation. Which reply matches?

- **A.** Correct — budget by volume.
- **B.** **One uncaught MMLU item in SFT is worth a thousand uncaught copies in pretrain.** Pretrain memorisation requires frequency (Kandpal 2022-style scaling); SFT memorisation requires only a single example at the right position in the training mix, seen multiple epochs at full learning rate, often with explicit answer conditioning. Per-example impact in SFT is orders of magnitude higher than per-document impact in pretrain. The firewall budget should reflect the *concentration*, not the volume — SFT is weekly cadence, corpus-wide LLM substring scanning (not candidate-only), and every SFT dataset version bump requires a decontamination-log row before promotion to training.
- **C.** Pretrain is the only surface Min-K% can detect; budget accordingly.
- **D.** SFT is handled downstream by `model-evaluation-engineer`; the firewall budget stays on pretrain.

---

## Q9. The synthetic-generator contract — cases A and B

Your team is about to fold synthetic SFT data from an open-weights generator into training. The generator's publisher has not published a decontamination log for LiveBench or LiveCodeBench. Chapter 05 pins the operational decision. Which option matches?

- **A.** Treat the generator as untrusted for every benchmark and run the full detector pipeline corpus-wide on all its output.
- **B.** Treat the generator as certified-decontaminated for every benchmark it was plausibly trained before; skip the firewall.
- **C.** The contract is **per-benchmark, not generator-wide**. For benchmarks covered by the publisher's decontamination log (case **A — certified**), the generator is **decontaminated-by-construction**; still run a **≥ 1% or ≥ 10,000-item spot-check** on its output because the certification is a strong prior, not a release-eligible guarantee. For benchmarks the publisher did **not** cover — including LiveBench / LiveCodeBench for any pre-2026 generator — treat the generator as **case B — untrusted** and run the n-gram, exact-prompt, and **LLM substring (corpus-wide)** detectors on its output before it enters any training surface. The `synthetic_generator_contract.yaml` is a release gate; without a signed contract the synthetic data does not enter training.
- **D.** Open-weights generators are always case A; certification follows from the open-weights licence.

---

## Q10. Picking between the EleutherAI harness and Dolma recipes

Your lab trains an open release on a ~3T-token pretrain corpus and scores with the EleutherAI LM Evaluation Harness. Chapter 06 pins three selection heuristics. Which option matches?

- **A.** `data-juicer` — it supports more operators than either of the two public recipes.
- **B.** EleutherAI only — Dolma is strictly inferior at every scale.
- **C.** Most production labs **hybridise**: **Dolma for pretrain** — the Mapper-side parallelism and the incremental-shard story are the load-bearing properties at trillion-token scale, and Dolma's public decontamination log is a direct template for Chapter 08's log — and **EleutherAI harness recipe for SFT / preference-pair** — alignment with scoring is the load-bearing property (the harness that produced the registry is the one that scores the final model, so normalisation drift risk is minimised). The release-gate check is that **both recipes consume the same `eval_manifest.yaml` SHA**. Chapter 08 §1 `toolchain.recipe_choice: hybrid` is where this is recorded; the harness and Dolma git SHAs must both be pinned ("latest" or "main" is a fail).
- **D.** Pick whichever recipe reports fewer removals; low removal count means the corpus is already clean.

---

## Q11. Min-K% Prob + Oren signal combinations, and why pre-registration matters

An internal audit returns a strong Min-K% Prob signal on GSM8K but a null Oren exchangeability p-value. A newer engineer proposes raising the Min-K% threshold until both signals are "clean". Chapter 07 is explicit about both the diagnostic reading of the signal combination and the integrity rule on the threshold. Which option matches?

- **A.** Both signals null means clean, both signal means contamination; any other combination is noise.
- **B.** The Min-K%-signal-with-Oren-null pattern suggests **item-level contamination without preserved canonical order** — typical of **SFT, preference-pair, or synthetic-data leaks**, not ordered pretrain ingestion. The audit-response protocol is to trace the flagged items to a surface, not retune. Thresholds are **pre-registered** — set before the audit runs and recorded with a SHA in `decontamination_log.yaml` §6 `pre_registered_sha256` *before* the detectors execute; **post-hoc threshold-tuning is not auditable**, and the log's pre-registration SHA is the audit's protection against it.
- **C.** The Min-K% signal on arithmetic-style benchmarks like GSM8K is always a false positive; raise the threshold.
- **D.** Raising thresholds after the audit is fine as long as the log reports the new value.

---

## Q12. The decontamination log is emitted, not written

A new engineer offers to hand-maintain `decontamination_log.yaml` after each release because "YAML is easy to edit and the pipeline output is messy", and proposes leaving §6 `pre_release_audit` and the `ai_risk_engineer_signoff` block empty for a quick pretrain-only release. Chapter 08 rejects all three moves. Which reply matches?

- **A.** Hand-maintenance is acceptable for the first few releases; empty sections can be filled in later.
- **B.** Hand-maintenance is fine but the pre-release audit block must be filled regardless.
- **C.** **All three moves are wrong.** The log is **emitted by the firewall**, not hand-authored — every number comes from a specific detector run's output; hand-authored summaries drift from reality within one re-run. **Per-benchmark × per-surface × per-detector disaggregation** is mandatory (aggregate-only "we removed 14,000 documents" hides which benchmark the firewall is weakest on). The `pre_release_audit` block requires a **pre-registration SHA** recorded *before* the detectors run; a missing SHA makes threshold-tuning indistinguishable from pre-registration. **Three sign-offs** — firewall engineer, `model-evaluation-engineer`, `ai-risk-engineer` — are a release-gate check, and the log is **signed** (Sigstore or equivalent). A log missing any of them is not release-eligible.
- **D.** The log is informal; sign-offs are not required for an internal release.

---

## Answer key

**Q1 — B.** Chapter 01 §"The three claims you must be able to defend" pins exactly the three citations and the shape in which to cite them: GPT-3 Appendix C (the 2020 precedent with the 13-gram, "`gram` = lowercase whitespace-delimited word with no punctuation", and the > 10-training-document boilerplate exclusion), Shi 2023 Min-K% Prob + Oren 2023 exchangeability (the two canonical black-box post-hoc audits, both ICLR 2024), and EleutherAI + Dolma (the two public open-source recipes). §"The firewall framing in ISO terms" ties them to ISO/IEC 25012 *credibility* and *traceability* and the ISO/IEC 5259-2 test-set-contamination measure family. A, C, and D each miss the three-citation framing the role is accountable for.

**Q2 — B.** Chapter 01 §"What 'contamination' means — and why it is slippery" lists the three operational definitions verbatim: exact-prompt (byte-equal after normalisation), n-gram overlap (the GPT-3 Appendix C definition), semantic (paraphrase / translation / retelling). The chapter is explicit that "the three definitions are not interchangeable" and that "an n-gram firewall at width 13 does not remove paraphrases". A conflates two definitions; C contradicts Chapter 03 §"false negative from paraphrase" ("the n-gram detector cannot solve this; do not try to tune `n` down to catch it"); D confuses contamination *definitions* with the Chapter 05 contamination *surfaces*.

**Q3 — C.** Chapter 02 §"Variants, versions, and the 'which split?' trap" pins both mitigations verbatim: the `variant` field is **mandatory** for benchmarks with variants (MMLU → original / Pro / Redux; HumanEval → original / plus / MultiPL-E), and the "model-evaluation-engineer sign-off block in the manifest is a release gate" — a mismatch between the signed-off variant and the firewall's defended variant is a release block, enforced at Chapter 08 §1 (identity section). A and B invent statistical interchangeability the chapter does not support; D names a necessary but insufficient control (revision pinning alone does not catch variant switches).

**Q4 — B.** Chapter 02 §"The normalisation contract" and Chapter 06 §"Alignment with the scoring harness" both pin this rule. Chapter 02 states the dominant-silent-false-negative risk explicitly and names the `normalisation_rules_sha256` field that applies to **both sides** of the detector (eval item and training span). Chapter 06 names the round-trip discipline — 20 items rendered as the harness renders them, injected into synthetic documents, expect 20 / 20 flags — as "the cheapest insurance in the firewall engineering budget". A, C, D each miss that mismatch (not strictness) is the risk.

**Q5 — B.** Chapter 03 §"Choosing the width: the two failure modes" names both ends of the curve verbatim: small-`n` false positives from common phrases (`"the answer is"`, `"import numpy as np"`) and large-`n` false negatives from light paraphrase. §"Calibration drill" gives the exact 200-pair, three-label, five-width sweep procedure and the "smallest `n` whose precision ≥ 0.98" picking rule. §"The GPT-3 Appendix C convention" notes that subsequent recipes (LLaMA-3 at 8-gram, GPT-4 at a 50-character window) pick different operating points on the same curve. A denies the calibration; C and D pick one end of the curve and ignore the opposite failure mode.

**Q6 — C.** Chapter 03 §"The two canonical data structures" + §"Picking a structure" pin both the shapes and the decision rule. Suffix array (Lee et al. 2021's `ExactSubstr`) is "correct when you need *attribution* — which training document contains the leak — and you need it repeatedly (new eval items added to the manifest; the firewall re-queried without rebuilding). Dolma's recipe uses this shape." Streaming scan is "correct when the eval manifest is small and the corpus is huge and can be read once … the shape of the EleutherAI harness decontamination script." §"Scaling: the data-engineering shape" calls out the sharded / incremental SA pattern for continuous pretraining. A, B, D each pick one tool without the decision axes or invent a difference the chapter does not claim.

**Q7 — B.** Chapter 04 §"When it is the wrong tool" is explicit: "Corpus-wide scanning. At trillion-token scale, invoking an LLM per span is cost-prohibitive. The detector is a **filter after prefiltering** …". §"When it is the right tool" names the high-prior-leak sources (GitHub HumanEval repos, LeetCode / Codeforces discussion, AoPS, Wikipedia pages about specific items) and the Oren-follow-up usage. Chapter 05 §"Surface 2 — SFT corpus" promotes the detector to corpus-wide on the smaller surfaces because "the small volume makes it affordable". Chapter 04 §"Cost model" names prefilter tightness as the dominant cost lever ("a 10× tighter prefilter saves 10× on LLM spend"). A ignores the cost shape; C and D misstate the per-surface scope and the quotation rule.

**Q8 — B.** Chapter 05 §"Surface 2 — SFT / instruction-tuning corpus" says verbatim: "One uncaught MMLU item in SFT is worth a thousand uncaught copies in pretrain." §"The intuition" pins the reason: pretrain memorisation requires frequency (Kandpal 2022), SFT memorisation requires only a single example at the right position in the training mix, seen multiple epochs at full learning rate with explicit answer conditioning. §"The firewall budget allocation should reflect this" is the budget-rule sentence. §"SFT firewall under-funded" is the catch — every SFT dataset version bump requires a decontamination-log row before promotion to training. A inverts the allocation; C misreads the Min-K% surface sensitivity (Chapter 07 §"Low-volume SFT contamination" says Min-K% is especially sensitive to SFT-style leaks); D mis-assigns the owner.

**Q9 — C.** Chapter 05 §"The firewall contract with the generation-side owner" pins the four-rule contract verbatim, including the per-benchmark split between cases A (certified-decontaminated) and B (untrusted), the ≥ 1% or ≥ 10,000-item spot-check on case A, the LLM-substring-corpus-wide rule on case B, and the "benchmarks_untrusted: [livebench, livecodebench]" example for any pre-2026 generator. §"Open-weight generator special case" is explicit: "When the generator is an open-weight model and the publisher's decontamination log is not available for all benchmarks in your manifest, treat the generator as untrusted *for the missing benchmarks only*." The contract is a release gate at Chapter 08 §4 `synthetic`. A over-generalises to every benchmark; B skips the firewall entirely; D is false — open-weights licence says nothing about contamination coverage.

**Q10 — C.** Chapter 06 §"Picking a recipe" lists three operational cases (A: open-source release, EleutherAI-harness-scored; B: large pretrain corpus, Dolma-like pipeline; C: lab-specific hybrid) and names C as the common choice: "Many labs run Dolma for pretrain + EleutherAI for SFT / preference". §"Case C" pins the release-gate check: "the eval manifest Dolma used and the manifest the EleutherAI harness used are the **same manifest** (same SHA, same benchmark list, same versions)". §"Public-log omissions" names the version-pinning rule — "'we used the harness' is not enough; pin the harness git SHA and the recipe config SHA". A invents an alternative not treated in the chapter; B denies the hybrid pattern; D inverts the low-removal-count signal (which can also mean the detector is blind — Chapter 03 §"pretrain-side removal rate").

**Q11 — B.** Chapter 07 §"Putting the two detectors together" names the four combinations verbatim, including the Min-K%-signal-Oren-null pattern: "Suggests item-level contamination without preserved order. Typical of SFT, preference-pair, or synthetic-data leaks." §"The audit-response protocol" is the four-step response (receive, reproduce, trace, respond) grounded in the Chapter 08 log, not in threshold-tuning. §"Post-hoc threshold-tuning" names the failure mode explicitly: "the audit ran, the signal was ambiguous, the team raised the threshold until the signal was 'clean'" — and the catch is "pre-register the threshold in the decontamination log before the audit runs. The log's version SHA is the pre-registration." Chapter 08 §6 `pre_registered_sha256` is where that SHA lives. A over-simplifies; C invents a universal false-positive on arithmetic that the chapter does not endorse; D contradicts the pre-registration rule.

**Q12 — C.** Chapter 08 §"The log is emitted, not written" is explicit: "Every number in it comes from a specific detector run's output; every SHA is a hash of a specific artifact; every sign-off is a programmatic field, not prose. Hand-authored summaries drift from reality within one re-run; emitted summaries cannot." §"Failure modes in the log" names all four patterns that apply here: aggregate-only numbers (per-benchmark × per-surface × per-detector disaggregation is "the most-often-missing structure; it is also the most load-bearing"), unversioned eval manifest reference, missing pre-registration (`pre_release_audit.pre_registered_sha256` must be set before the audit runs), missing sign-offs. §"Signing the log" prescribes Sigstore or an equivalent keyless signer. A, B, D each undercut the emitter-not-author rule, the disaggregation requirement, or the three-sign-off release gate.

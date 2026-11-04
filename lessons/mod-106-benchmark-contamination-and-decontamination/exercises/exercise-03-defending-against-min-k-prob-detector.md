# Exercise 03 — Defending Against the Min-K% Prob Detector (and the Oren Exchangeability Test)

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 07 — Defending against Min-K% Prob and Oren exchangeability](../07-defending-against-min-k-prob-and-oren-exchangeability.md), [Chapter 01 — Why the firewall is non-negotiable](../01-why-the-contamination-firewall-is-non-negotiable.md)

## Objective

Run **both** canonical post-hoc contamination detectors — Min-K% Prob (Shi et al. 2023, ICLR 2024) and the Oren exchangeability test (Oren et al. 2023, ICLR 2024) — against an open small model, using an eval-manifest benchmark as the "member" side and a release-date-indexed benchmark (LiveBench / LiveCodeBench items from after the target model's training cutoff) as the "non-member" reference. Pre-register the thresholds. Record the results in the decontamination log's §6 `pre_release_audit`. Produce an audit response plan for a hypothetical "signal detected" scenario grounded in the Chapter 07 §"audit-response protocol".

By the end you will have:

- A Min-K% Prob implementation (`min_k_prob.py`) that computes per-sequence scores from token log-probabilities under a target model.
- An Oren exchangeability test implementation (`oren_exchangeability.py`) that produces a permutation-test p-value for a benchmark under a target model.
- Pre-registered thresholds (`pre_registration.yaml`) with a SHA-256, written **before** the detectors run.
- Per-benchmark Min-K% distributions and Oren p-values for a target model (an open HF model whose training cutoff is known) against three benchmarks (two "members" from before the cutoff, one "non-member" from after).
- `decontamination_log.yaml` §6 `pre_release_audit` populated.
- A `audit_response.md` that walks through the Chapter 07 four-step protocol against one simulated "signal detected" result.

## Prerequisites

- Python 3.11+ with `transformers`, `torch`, `datasets`, `numpy`, `scipy`, `matplotlib`, `pyyaml`.
- A GPU is helpful for the token-log-probabilities step on larger models, but not required — this exercise is explicitly sized to run on a small (≤ 3B parameter) open model on CPU or a modest GPU.
- A target open model with a **known training cutoff**. Suggested options: a `pythia-1b` (<https://huggingface.co/EleutherAI/pythia-1b>) or `GPT-Neo` variant from EleutherAI, whose training cutoff is public. Any similar small open model works as long as the cutoff date is published.
- `eval_manifest.v1.yaml` and `decontamination_log.yaml` from Exercise 01 / Exercise 02.
- Read Chapter 07.

**Note on API vs local inference.** The exercise uses local inference on a small HF model to keep the exercise self-contained and reproducible. In production, the same detectors run against a hosted model's `logprobs` API (OpenAI, Anthropic). The core logic is identical.

## Problem statement

Pre-register the audit, run both detectors, interpret the result, and simulate a response.

1. **Pre-register thresholds.** Before running either detector, write `pre_registration.yaml` with:
   - The Min-K% separation above which a benchmark is flagged `contamination_detected` (Chapter 07 §"ambiguous signal" suggests ≥ 0.08 as a starting point; pick one and defend it).
   - The Oren p-value below which a benchmark is flagged `contamination_detected` (standard start: 0.01).
   - Hash the file to `pre_registration.sha256` and record both the path and the SHA in `decontamination_log.yaml` §6 before the audit runs.

2. **Implement Min-K% Prob.** For each sequence `x`:
   - Compute per-token log-probability with the target model.
   - Select the bottom-K% tokens (K ∈ {10, 20, 40} — run all three).
   - Average the log-probabilities of the selected tokens. That is the Min-K% Prob score.
   - Aggregate per benchmark: mean Min-K% score over all items.

3. **Implement the Oren exchangeability test.** For a benchmark `D` in canonical order:
   - Compute `L(identity)` = sum of per-sequence log-likelihoods under the target model.
   - For `N_perm = 1,000` random permutations σ, compute `L(σ)`.
   - Produce the permutation p-value: fraction of `σ` with `L(σ) ≥ L(identity)`.
   - Record per-benchmark.

4. **Pick targets.**
   - Member candidates (benchmarks the model plausibly saw): MMLU `test`, GSM8K `test`. Both are public and predate most open small-model training cutoffs. Verify each against your target model's cutoff before using it as a member.
   - Non-member reference: LiveBench or LiveCodeBench items released **after** the target model's training cutoff. Use these as the Chapter 07 §"reference-distribution contamination" clean reference.

5. **Run both detectors.** Save raw per-item Min-K% scores and raw per-permutation Oren likelihoods to Parquet for traceability.

6. **Compare against thresholds.** For each benchmark, emit `verdict ∈ {clean, ambiguous, contamination_detected}` based strictly on the pre-registered thresholds. Do not retune.

7. **Populate the log.** Fill in `decontamination_log.yaml` §6 `pre_release_audit` with the per-benchmark results, the reference distribution used, the pre-registration SHA, and the overall verdict.

8. **Simulate a response.** Pick one benchmark whose observed signal is strongest (even if below threshold) and write `audit_response.md` walking through the Chapter 07 four-step protocol *as if* that signal exceeded threshold. This is practice for the real audit response; the step is mandatory.

## Starter guidance

1. **Token log-probabilities.** Use `transformers` with `AutoModelForCausalLM` and `AutoTokenizer`. For each input sequence, run a forward pass, take the log-softmax of the logits at each position, and index the target token's log-prob. Return per-token log-probs as a NumPy array.

2. **Min-K% implementation.**

```python
import numpy as np

def min_k_prob(token_logprobs: np.ndarray, k_percent: float) -> float:
    n_bottom = max(1, int(np.ceil(len(token_logprobs) * k_percent / 100)))
    bottom = np.partition(token_logprobs, n_bottom - 1)[:n_bottom]
    return float(bottom.mean())
```

   Note: the detector works on the *input* side — the eval item's prompt + canonical answer, scored under the target model, not a generation.

3. **Oren implementation.**
   - Score each item independently, then sum log-likelihoods under a permutation by indexing into the cached per-item log-likelihoods. This avoids re-running the model per permutation.
   - Wait — the Oren test is subtler: it depends on *conditional* likelihood of each item *given its predecessors* under the permutation. The naive "sum of independent per-item likelihoods" is not the Oren test; it would be invariant to permutation and the p-value would be uninformative.
   - For a faithful implementation, concatenate the permuted sequence (item σ(1) ▸ item σ(2) ▸ ... ▸ item σ(n)) with a separator and score the concatenation. For modest benchmark sizes, this is feasible on a small model in a loop. The `N_perm = 1,000` is the dominant cost.
   - If the concatenation exceeds the model's context window, batch into sliding windows and sum — but document the workaround and the Chapter 07 §"ambiguous signal" implications.

4. **Reference distribution hygiene.** For each LiveBench / LiveCodeBench item used as a non-member, verify its release date is strictly after the target model's training cutoff. Filter out any item whose release date is before. The resulting non-member set may be small; use `N ≥ 100` items.

5. **Threshold interpretation.** For each benchmark, the Min-K% verdict is:
   - `clean`: observed separation ≤ threshold / 2
   - `ambiguous`: threshold / 2 < observed separation < threshold
   - `contamination_detected`: observed separation ≥ threshold
   
   Record the raw values; let the log reader see the gap.

6. **Simulate the response.** Pretend the strongest-signal benchmark exceeded threshold. In `audit_response.md`:
   - Paragraph 1 (the signal): what the (simulated) audit reports, with the numbers.
   - Paragraph 2 (the reproduction): what the decontamination log §4 says about that benchmark's removals, and whether the signalled items were in the removal manifest.
   - Paragraph 3 (the remediation): the detector-configuration change you would propose, or the escalation path if the signal is upstream.

## Deliverables

```
exercise-03/
  code/
    min_k_prob.py
    oren_exchangeability.py
    run_audit.py                   # orchestrates both detectors
    emit_log_update.py             # writes §6 of the decontamination log
  configs/
    target_model.yaml              # HF model id, revision, cutoff date
    pre_registration.yaml
    pre_registration.sha256
  data/
    member_manifests/
      mmlu_test.jsonl
      gsm8k_test.jsonl
    nonmember_manifest/
      livebench_post_cutoff.jsonl
  results/
    per_item_min_k_scores.parquet
    oren_permutation_loglik.parquet
    per_benchmark_verdicts.yaml
    decontamination_log.yaml       # extended with §6
    plots/
      min_k_distribution_per_benchmark.png
      oren_null_distribution_per_benchmark.png
  audit_response.md
  README.md
```

## Acceptance criteria

- `pre_registration.yaml` is written and SHA'd **before** the audit runs. The SHA is recorded in `decontamination_log.yaml` §6 `pre_registered_sha256`. A run with post-hoc threshold adjustment fails the criterion.
- The target model's **training cutoff is explicitly recorded** in `configs/target_model.yaml` and cited in the log. "Unknown cutoff" fails the criterion.
- The non-member reference set contains only items whose release date post-dates the training cutoff. The filter is implemented in code, not by hand; a reviewer can re-run the filter and get the same set.
- Both detectors run on **at least three benchmarks**: two member candidates and one non-member reference.
- `decontamination_log.yaml` §6 is fully populated: per-benchmark Min-K% stats, Oren p-values, verdict, overall verdict, pre-registration SHA, verdict thresholds.
- `audit_response.md` follows the Chapter 07 §"audit-response protocol" structure: three paragraphs (signal / reproduction / remediation).
- **Determinism (Oren).** Running the exchangeability test twice with the same seed produces the same p-value. Record the seed in `configs/pre_registration.yaml`.
- Any factual claim about the detector behaviour that cannot be verified against Shi et al. 2023 or Oren et al. 2023 at authoring time is marked `<!-- needs-research: ... -->`.

## Stretch goals

- **Compare across model sizes.** Run the same audit on two scales of the target model family (e.g., `pythia-410m` and `pythia-1b`). Report how the Min-K% separation changes with scale. Larger models memorise more, so the detector should be more sensitive on the larger model — confirm empirically.
- **Paraphrase sensitivity.** Paraphrase a sample of 50 MMLU items (one word per sentence changed). Re-run Min-K% on the paraphrases against the target model. Report how much the separation shrinks relative to canonical phrasing. This is the Chapter 04 §"when it is the right tool" motivation for the LLM substring detector.
- **Multiple non-member references.** Try three different non-member reference sets: LiveBench post-cutoff, synthetic text in the same style, paraphrased MMLU. Show how the separation depends on the reference distribution; defend one choice in the log.
- **Order-sensitivity ablation.** For one benchmark, run the Oren test with shuffled vs canonical order of the `test.jsonl` file. Confirm that shuffled-order ingestion produces a null Oren p-value even when the content is present. This is the Chapter 07 §"Oren test's structural insight" demonstration.

## What this exercise is not

- Not an attempt to break the target model. Both detectors are audit *diagnostics*, not attacks. The exercise is about producing a credible internal audit log.
- Not a benchmark-construction exercise. The benchmarks are consumed verbatim from the manifest.
- Not the response to a *real* external audit. The simulated response is practice; the real response (Chapter 07) involves `model-evaluation-engineer` and `ai-risk-engineer` co-signing.

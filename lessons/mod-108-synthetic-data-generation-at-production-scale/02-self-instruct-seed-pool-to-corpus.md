# Chapter 02 — Self-Instruct: Seed Pool to Corpus

## Why this chapter exists

Self-Instruct (Wang et al. 2022) is the baseline recipe for bootstrapping an instruction-tuning corpus from a tiny seed of hand-written tasks. Every later recipe in this module — Evol-Instruct, UltraChat, teacher-student distillation, preference-pair synthesis — reuses its three primitives: a **seed pool**, a **generation loop with a growing pool**, and a **dedup + heuristic filter** that stops the pool from collapsing to the generator's biased centre.

If you internalise one recipe end-to-end, make it this one. The failure modes you will hit on Self-Instruct — pool collapse, filter under-tuning, generator refusal patterns — are the same failure modes you hit on the fancier recipes, just with more moving parts.

The original paper: Wang, Kordi, Mishra, Liu, Smith, Khashabi, Hajishirzi. *Self-Instruct: Aligning Language Models with Self-Generated Instructions.* arXiv 2212.10560 (ACL 2023). <https://arxiv.org/abs/2212.10560>. The authors' reference implementation and seed-pool are public: <https://github.com/yizhongw/self-instruct>.

## The recipe at a glance

Four loops, run to a budget:

```
seed_pool  ─►  (sample 8)  ─►  generator  ─►  new_instruction_candidate
                                                      │
                                                      ▼
                                         rouge_l vs growing_pool
                                                      │
                                     ┌────────────────┴────────────────┐
                                     ▼                                  ▼
                          rouge_l > 0.7 → drop            rouge_l ≤ 0.7 → heuristic_filter
                                                                        │
                                                                        ▼
                                                                  ┌─────┴─────┐
                                                                  ▼           ▼
                                                                drop        keep
                                                                             │
                                                                             ▼
                                                                  generate_response
                                                                             │
                                                                             ▼
                                                                  add_to_pool
```

The original paper's operating choices (pin these; later recipes modify them):

- **Seed pool size:** 175 hand-written seed tasks.
- **Sampling per generation step:** 6 seed tasks + 2 previously-generated tasks (prevents the pool from drifting far from the seed style in early rounds).
- **Dedup threshold:** ROUGE-L > 0.7 against **any** existing pool item drops the candidate. ROUGE-L specifically — not BLEU, not exact match — because ROUGE-L captures longest-common-subsequence which generalises better over short imperative sentences.
- **Classification-task branching:** the generator first emits an instruction. If the instruction is classified as a classification task, the generator emits the classes first, then an instance; otherwise it emits an input and an output. The branching is an artifact of the GPT-3-era generator's weakness at joint classification-label-and-example emission; modern generators tolerate the non-branched variant.
- **Heuristic filters:** too-short (<3 words), too-long (>150 words), contains keywords the generator cannot handle in a text-only context (`image`, `graph`, `plot`, `file`, `figure`, `picture`, `files`, `images`, `graphs`, `plots`, `charts`, `pictures`), begins with a punctuation character, begins with a non-English character, duplicates with case-normalisation.

## The seed pool

The seed pool is the single most under-discussed design choice in Self-Instruct. The pool sets the style, the task distribution, and the ceiling of the generated corpus. A pool that is 90% summarisation tasks produces a corpus that is 90% summarisation tasks.

What a good seed pool contains:

1. **Task-type coverage.** Aim to include at least one seed per task family you want the student model to learn. The authors' seed pool deliberately covers classification, generation, extraction, reasoning, format conversion, and coding tasks.
2. **Difficulty coverage.** Easy and hard seeds in each family. The generator anchors to the seed difficulty; a pool of easy seeds produces an easy corpus.
3. **Length coverage.** Short imperatives ("Summarise this paragraph in one sentence") and longer, structured instructions ("Given the following table, group rows by the `region` column and emit the mean of `revenue`"). The length distribution of the pool strongly predicts the length distribution of the generated corpus.
4. **Format coverage.** Free-form, structured-output (JSON / YAML), step-by-step, and code. If the pool contains no JSON-output seeds, the corpus contains no JSON-output examples.

A seed pool of 100–200 items is typical. More seeds help in proportion to their *diversity*, not their count — a pool of 1,000 near-duplicate seeds is worse than a pool of 100 distinct ones.

### Seed-pool hygiene checklist

- Hash the pool (`seed_pool.sha256`) and pin the hash in the generation manifest. A pool that drifts silently between runs is a reproducibility bug.
- Run the dedup threshold (ROUGE-L > 0.7) *against the pool itself* before any generation. A pool that already contains internal near-duplicates has an elevated collapse risk.
- Classify every seed into task families manually. The task-family counts are a release-eligibility check: no family represented in the pool should be more than 2× the mean family count.
- Carry the seed licence through. If any seed is drawn from a public dataset (e.g. a subset of FLAN or Natural Instructions), the seed's licence travels with the corpus. [[teacher-licence-ledger]]

## The generation loop

The canonical generation prompt is a few-shot prompt containing the sampled seed tasks, followed by a next-task instruction. From the paper:

> Come up with a series of tasks:
> Task 1: {seed_task_1}
> Task 2: {seed_task_2}
> ...
> Task 8: {seed_task_8}
> Task 9:

The generator completes `Task 9` with a new task. The response is parsed on the `Task N:` delimiter. A single API call typically returns multiple candidate tasks — the paper's implementation requests up to 10 or 20 per call and keeps them all through the filter, at a cost-per-candidate much lower than per-call.

Operating notes for a modern generator:

- **Temperature.** The paper used 0.7. Modern generators (Anthropic Claude, OpenAI GPT-4-class, open-weight Llama-class) also work at 0.7–1.0 for this task. Below 0.5 the pool collapses; above 1.1 the heuristic-filter reject rate spikes.
- **Top-p.** Keep at the generator's default (typically 1.0 or 0.95). Self-Instruct is a diversity task; aggressive top-p truncation defeats the purpose.
- **Stop sequences.** Set `stop=["Task 25:", "\n\n\n"]` or the modern equivalent. Without a stop sequence the generator can run away for hundreds of tokens generating low-quality continuations.
- **Prompt budget.** The few-shot prompt is 8 seed tasks × ~50 tokens = ~400 tokens. The response is ~10 candidates × ~50 tokens = ~500 tokens. Budget accordingly for throughput and cost.
- **API refusal rate.** Modern hosted models refuse a measurable fraction of Self-Instruct prompts (typically on safety categories). The refusal rate is a hyper-parameter — track it, and if it exceeds a few percent, review the seed pool for topic drift.

### The sampling ratio

The paper's 6-seed + 2-generated split is a diversity-vs-stability trade-off. Pin it as a tunable:

- **6:2 (paper default).** Early-round generations stay close to the seed style; useful when the seed pool is high-quality and the generator is weak.
- **4:4.** Generated items play a larger role; useful for a longer generation run that should drift into the generator's own strengths.
- **0:8 (self-only).** A collapse recipe. Do not use after a few rounds; the pool collapses onto the generator's bias. This is the Curse-of-Recursion vector in miniature (Chapter 07).

Track the sampling ratio across rounds. The ratio is usually constant, but some teams anneal from 6:2 to 4:4 over the course of a run. The anneal schedule is a hyper-parameter; log it in the manifest.

## The ROUGE-L dedup filter

ROUGE-L is the Longest Common Subsequence measure between the candidate and the reference, normalised to `[0, 1]`. Its advantages for this specific application:

- **Order-sensitive but gap-tolerant.** It penalises re-orderings less harshly than BLEU-N but still rewards shared structure.
- **Short-sequence-friendly.** Instructions are typically 10–30 tokens. ROUGE-L is stable at this length; n-gram overlap metrics (BLEU-4) can be degenerate.
- **Cheap.** `O(|a| · |b|)` per pair; vectorised across a pool.

The 0.7 threshold is the paper's. The empirical shape:

- Below 0.5: lets through many near-duplicates. Pool size grows fast, diversity does not.
- 0.5 – 0.7: tightens the filter. Most teams operate here.
- 0.7 – 0.85: aggressive. Rejects slight-paraphrase candidates that are often fine for training.
- Above 0.85: effectively lexical-match only. Lets near-paraphrases through.

### Pool-size scaling problem

Naive implementation: compute ROUGE-L between the candidate and every item in the growing pool. At pool size `N`, this is `O(N)` per candidate. For a 50k-item corpus, the final rounds take `O(50k)` ROUGE-L computations per candidate — by far the dominant pipeline cost.

Three mitigations:

1. **Blocking by first-token prefix.** Group the pool by the first token of the instruction. Candidates are compared only against pool items sharing the first token. This is a false-negative gamble: candidates that share structure but differ in their first token escape.
2. **Embedding prefilter.** Compute a sentence embedding for every pool item. For each candidate, retrieve top-K nearest neighbours by cosine similarity (K = 50 is a common choice), then ROUGE-L only against those K. This is the pattern Chapter 03's Evol-Instruct uses for its elimination step, and the pattern NeMo Curator's Self-Instruct operator defaults to.
3. **MinHash LSH.** Reuse the MinHash LSH infrastructure from [`mod-105`](../mod-105-deduplication-at-scale/README.md). MinHash estimates Jaccard, not ROUGE-L, but the correlation is strong enough on short imperative sentences that the LSH buckets make a good prefilter. Verify with exact ROUGE-L on the LSH candidate set.

Pick one. Document which you picked in the generation manifest.

## Heuristic filters

The paper's heuristic filters are not aesthetic. They are *what makes Self-Instruct work* on a GPT-3-class generator. They matter less on modern generators that refuse to emit "describe this image" in a text-only context, but they still catch a non-trivial failure tail. Keep them.

- **Length.** Drop instructions of fewer than `MIN_WORDS` or more than `MAX_WORDS` words. Paper defaults: 3 and 150. Tune for your task shape.
- **Keyword blacklist.** Drop instructions containing modality keywords the generator cannot execute (`image`, `graph`, `plot`, `file`, `figure`, `picture`, and plurals). On a multimodal generator this filter is relaxed.
- **Language.** Drop non-English instructions (or non-target-language). FastText lid.176 or HF `facebook/fasttext-language-identification` is cheap and works at scale.
- **Punctuation start.** Drop instructions that begin with a non-alphabetic character. These are typically generator continuation artifacts.
- **Refusal patterns.** Drop instructions that begin with "I cannot", "I'm sorry", "As an AI language model", "I apologise", etc. Modern generators refuse more often than GPT-3 did; maintain the list.
- **Duplicate-of-seed.** Case-normalise and strip punctuation, then exact-match against the seed pool. The generator sometimes echoes a seed verbatim.

### Order of operations

Apply filters in the order **cheapest-rejecting-the-most** → **most-expensive-rejecting-the-fewest**:

1. Length (cheap, rejects a lot).
2. Punctuation start / refusal pattern (cheap, rejects some).
3. Language (moderate cost, rejects some).
4. Keyword blacklist (cheap, rejects some).
5. Duplicate-of-seed (moderate cost, rejects few).
6. ROUGE-L against pool (expensive, rejects variable).

Log the rejection count per filter per round. The rejection-count distribution is a diagnostic: a sudden spike in refusal-pattern rejects indicates the seed pool has drifted toward a topic the generator refuses; a sudden spike in ROUGE-L rejects indicates the generator has entered a mode-collapse phase.

## The response-generation step

Once an instruction candidate survives the filters, the pipeline generates a response. This is where modern recipes deviate from the paper most sharply:

- **Paper version:** two-pass generation. Pass 1 generates the instruction; Pass 2 generates the response (and optionally input) for that instruction. The two-pass structure keeps the diversity filters on instructions, not responses.
- **Modern simplification:** single-pass. One call asks for `(instruction, response)` jointly. Cheaper, but makes it harder to dedup on the instruction space alone. Recommended only when the seed pool is strong and the ROUGE-L filter is placed on *both* instruction and response.

Response-side filters:

- **Length.** Cap the response length at a level commensurate with the eventual training context. 512–2048 tokens is typical.
- **Refusal.** Drop responses that are refusals. Logging these separately is useful — the refusal rate is a generator-health signal.
- **Judge score.** Chapter 06's judge-model pattern applies here too — score each `(instruction, response)` pair, drop below a threshold. This is optional on Self-Instruct; it is mandatory on preference-pair synthesis.

## The generation manifest

Every Self-Instruct run produces a manifest row:

```yaml
# recipes/self_instruct_config.yaml
generator:
  model_id: claude-3.5-sonnet-2024-10-22    # or openai:gpt-4o-2024-08-06, or vllm:local-id
  temperature: 0.7
  top_p: 1.0
  max_tokens: 2048
  stop: ["Task 25:", "\n\n\n"]

seed_pool:
  path: seeds/seed_pool.jsonl
  sha256: f5d0...
  item_count: 175

sampling:
  seeds_per_prompt: 6
  prior_generations_per_prompt: 2
  anneal_schedule: constant

filters:
  rouge_l_threshold: 0.7
  rouge_l_backend: embedding_prefilter_top_50
  length_range: [3, 150]
  language: en
  keyword_blacklist: [image, graph, plot, file, figure, picture]
  refusal_patterns_path: filters/refusal_patterns.txt

targets:
  instruction_count: 50000
  per_round_cap: 2000
  max_rounds: 50

licence:
  teacher_licence_ref: generator/licence_ledger.yaml
```

The manifest is pinned in the program package (Chapter 01 §"The program package"). A Self-Instruct run without a signed manifest is a run you cannot reproduce or defend.

## Where Self-Instruct breaks

### 1. Pool collapse

Symptom: round-over-round, the ROUGE-L rejection rate grows, the rate of new accepted instructions falls, and the surviving instructions look increasingly similar in structure.

Catch: track pool-wide diversity metrics — unique first-tokens, mean pairwise ROUGE-L of a sample, embedding spread. Set a kill-switch: if diversity falls by more than X% round-over-round, stop and re-seed.

Fix: tighten the ROUGE-L threshold (0.7 → 0.6), reduce the ratio of generated-to-seed sampling (4:4 → 6:2), or expand the seed pool.

### 2. Keyword-filter leakage

Symptom: training examples with `image`, `plot`, or `file` tokens appear in the corpus despite the filter.

Catch: the filter must match on sub-tokens and plurals. `"describe this plot"` is a match on `plot`; `"describe these plots"` is a match on `plots` — maintain both. Run a unit test that scans the output corpus for every keyword and asserts zero matches.

Fix: expand the keyword list. Run the test as part of the exercise acceptance.

### 3. Seed-licence propagation drop

Symptom: the generated corpus is shipped without a licence reference, and a later audit finds that a quarter of the seeds were drawn from a non-commercial dataset.

Catch: the seed pool's licence is tracked per-seed. The corpus's licence is the intersection of all seed licences and the generator licence. If any seed is non-commercial, the corpus is non-commercial. Chapter 05 is the ledger.

Fix: filter the seed pool to a known licence before generation. Document the filter in the manifest.

### 4. Generator refusal drift

Symptom: on a generator upgrade (e.g., the provider releases a new model), the refusal rate spikes from a few percent to double-digit percent, and specific topics disappear from the corpus.

Catch: track refusal rate per model version. A new model version is an explicit generation-run boundary; do not silently continue a run across a model upgrade.

Fix: pin the generator version in the manifest (`claude-3.5-sonnet-2024-10-22`, not `claude-3.5-sonnet`). An upgrade is a new run.

## Summary

- Self-Instruct bootstraps an instruction corpus from a tiny seed using three primitives: a seed pool, a generation loop with a growing pool, and a filter stack (ROUGE-L dedup + heuristic filters).
- The seed pool is the single most under-discussed design choice. Pool quality bounds corpus quality; pool diversity bounds corpus diversity.
- The ROUGE-L > 0.7 threshold and the heuristic filter list are not aesthetic — they are what makes the recipe work. Removing them produces a near-duplicate corpus.
- At production scale the ROUGE-L step is the dominant cost. Prefilter with embeddings or MinHash LSH; verify with exact ROUGE-L on the candidate set.
- Pin the generator version, the sampling ratio, the dedup threshold, the keyword list, and the seed-pool SHA in the manifest. A run without a signed manifest is a run you cannot reproduce.

Chapter 03 picks up with Evol-Instruct — the evolution operators that turn a Self-Instruct pool into a difficulty-stratified one.

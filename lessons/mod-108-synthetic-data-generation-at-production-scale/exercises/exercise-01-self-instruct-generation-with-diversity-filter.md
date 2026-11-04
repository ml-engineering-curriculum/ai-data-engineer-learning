# Exercise 01 — Self-Instruct Generation with Diversity Filter

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 02 — Self-Instruct: seed pool to corpus](../02-self-instruct-seed-pool-to-corpus.md)

## Objective

Stand up a working Self-Instruct loop against a modern hosted or local generator on a seed pool you author yourself. Produce 2,000–5,000 filtered instruction-response pairs, with the full filter stack wired up (ROUGE-L dedup, heuristic filters, refusal detector). Produce the first blocks of the program package: `seeds/seed_pool.jsonl`, `recipes/self_instruct_config.yaml`, `outputs/sft_corpus.parquet`, `filters/rouge_l_report.parquet`, and a short `calibration.md` reporting the filter pass rates.

By the end you will have:

- A 100–200 item seed pool you authored, hashed into the manifest, and categorised by task family.
- A running Self-Instruct loop that can be stopped and resumed, writes a manifest row per run, and emits a parquet corpus with lineage columns.
- A ROUGE-L dedup filter with a documented operating threshold, and a report on how many candidates were dropped at each filter stage.
- A clear picture of your generator's refusal distribution and its behavioural shape on Self-Instruct prompts.

## Prerequisites

- Chapter 02 read.
- Python 3.11+ with `openai` or `anthropic` SDK (or `vllm` for a local generator), `rouge-score`, `sentence-transformers` or `openai`-embeddings for the prefilter, `pyarrow`, `pandas`, `pyyaml`, `fasttext` (for language ID).
- A generator. Any of:
  - A hosted API (Anthropic Claude, OpenAI GPT-4-class) with a budget of a few dollars of generation cost.
  - A locally hosted open-weight model served via `vllm` or `ollama` (a 7B-class model works; a 70B-class model gives better results).
- A reference implementation to consult (not copy): <https://github.com/yizhongw/self-instruct>.

## Problem statement

Pick **one** target domain for the seed pool. Options (pick one that matches a slice you already have or care about):

- General instruction following (175 seed-task style, the paper's default).
- Code generation (seed pool of 100–200 short coding tasks).
- Data manipulation (seed pool of SQL / pandas / jq-shaped tasks).
- Structured output (seed pool of JSON / YAML / markdown output tasks).

Build a Self-Instruct loop that produces an SFT-shaped corpus in your chosen domain. The corpus must pass every filter in Chapter 02 §"Heuristic filters" and must be deduped at ROUGE-L > threshold.

## Starter guidance

### 1. Author the seed pool

- Write 100–200 seed tasks in your chosen domain. Hand-authoring is slow — budget 30–60 minutes.
- Cover task families explicitly. For general instruction: classification, generation, extraction, reasoning, format conversion, coding. Aim for no family being more than 2× the mean.
- Tag each seed with `task_family`, `difficulty ∈ {easy, medium, hard}`, `expected_format ∈ {free_form, structured}`.
- Save as `seeds/seed_pool.jsonl`. Hash with SHA-256 and pin the hash in the manifest. Run the pool against itself at ROUGE-L > 0.7 to ensure no internal near-duplicates.

### 2. Author the generation prompt

- Follow the Chapter 02 §"The generation loop" shape. Include 6 sampled seed tasks + 2 previously-generated tasks in the prompt.
- Use the generator's instruct-chat format where one is native; use the few-shot base-model format otherwise.
- Set temperature 0.7, top-p default, max_tokens 2048, stop sequences at `Task 25:` and `\n\n\n`.
- Request 10–20 candidates per call (the generator will emit them as `Task 9:`, `Task 10:`, etc.).

### 3. Wire the filter stack

Implement the filters in the order from Chapter 02 §"Order of operations":

1. Length (3–150 words).
2. Punctuation start / refusal pattern.
3. Language (English only, via `fasttext.lid.176`).
4. Keyword blacklist (`image`, `graph`, `plot`, `file`, `figure`, `picture`, plurals).
5. Duplicate-of-seed (case-normalised exact match).
6. ROUGE-L against the growing pool.

For the ROUGE-L filter:

- Start naive: for each candidate, compute ROUGE-L against every pool item. Measure wall-clock at pool sizes 100, 500, 2000.
- Switch to the embedding-prefilter (top-50 nearest by cosine similarity, then ROUGE-L on those 50) once naive exceeds 2 seconds per candidate.

Log a per-filter rejection count per generation round. The log is a parquet file with columns `round`, `filter`, `rejected_count`, `accepted_count`.

### 4. Generate responses

- For each surviving instruction, generate a response in a second API call. Temperature 0.3–0.5 (lower than instruction generation; correctness, not diversity, is the target).
- Length cap: 1024 tokens.
- Drop responses that are refusals (use the same refusal patterns as step 3).
- Drop `(instruction, response)` pairs where the response is empty, truncated, or a parse-fail.

### 5. Build the manifest

Write `recipes/self_instruct_config.yaml` matching the shape in Chapter 02 §"The generation manifest". Include:

- Generator identity and version (`claude-3-5-sonnet-20241022`, not `claude-3-5-sonnet`).
- Seed pool path and SHA.
- Sampling ratio (6 seed : 2 generated).
- Filter thresholds and keyword list.
- Target count (2,000–5,000).

### 6. Write `calibration.md`

One page, including:

- How many total generation calls, how many candidate instructions, how many surviving instructions.
- Per-filter rejection rate as a table.
- Pool-wide diversity metric: unique first-tokens, mean pairwise ROUGE-L of a 500-item sample.
- Observed refusal rate.
- Wall-clock and cost breakdown per round.
- At least two example instructions that were accepted, two that were rejected at each filter stage, with the rejection reason visible.

## Deliverables

```
exercise-01/
  seeds/
    seed_pool.jsonl
    seed_pool.jsonl.sha256
  recipes/
    self_instruct_config.yaml
  code/
    build_seed_pool.py          # optional helper to seed-pool with task-family tagging
    generate.py                 # main loop
    filters.py                  # the filter stack
    rouge_l.py                  # ROUGE-L + embedding prefilter
    manifest.py                 # emit the YAML
  outputs/
    sft_corpus.parquet          # final (instruction, response) pairs with lineage
    sft_corpus.parquet.sha256
  filters/
    rouge_l_report.parquet      # per-round rejection counts
    rejected_samples.jsonl      # 50 sampled rejections across filter stages
  calibration.md
  README.md
```

## Acceptance criteria

- `seeds/seed_pool.jsonl` has 100–200 entries, each with `task_family`, `difficulty`, `expected_format`. Running the pool against itself at ROUGE-L > 0.7 produces zero internal hits.
- `recipes/self_instruct_config.yaml` pins the generator with a dated version string (e.g. `claude-3-5-sonnet-20241022`). An un-dated version (e.g. `claude-3-5-sonnet`) fails the criterion.
- `outputs/sft_corpus.parquet` has at least 2,000 rows. Each row has columns `instruction_id`, `instruction`, `response`, `seed_source_ids` (list, the seed ids sampled into the prompt), `generation_round`, `generator_model_id`, `generation_timestamp`.
- The parquet has **no near-duplicates**: no two rows exceed ROUGE-L > 0.75 on the instruction column.
- The parquet has **no banned-keyword instructions**: a regression test that scans the instruction column for `image|graph|plot|file|figure|picture` (and plurals) returns zero matches. Document any deliberate exemptions.
- `filters/rouge_l_report.parquet` has one row per `(round, filter)` tuple with `rejected_count` and `accepted_count` columns.
- `calibration.md` reports the per-filter rejection rate as a table, the observed refusal rate, and at least 4 concrete example instructions (2 accepted, 2 rejected-with-reason).
- **Determinism** (modulo generator non-determinism). Running `generate.py --resume` after a stop produces a continuing run; the parquet schema is stable across runs.
- Any factual claim in `calibration.md` or `README.md` that cannot be verified at authoring time is marked `<!-- needs-research: ... -->`.

## Stretch goals

- **Switch to 4:4 sampling.** Rerun the loop with a 4:4 ratio (4 seeds + 4 generated per prompt) after the 1,000th generated item. Report the change in pool diversity.
- **Multi-generator run.** Generate with a second, independent generator (different family). Report filter-pass rates per generator and look for systematic differences.
- **NeMo Curator or Data-Juicer port.** Re-implement the loop using NeMo Curator's `SelfInstructGenerator` or Data-Juicer's self-instruct mapper. Compare wall-clock and output shape against your hand-rolled loop.
- **Response-side judge.** Add a judge-model scoring step on the `(instruction, response)` pair. Report how many additional rejections it produces.
- **Seed-license tracking.** If any of your seeds were drawn from a public dataset, attach a `seed_license` column per seed and verify the corpus licence is the intersection of all seed licences and the generator licence. This previews Exercise 03.

## What this exercise is not

- Not an Evol-Instruct exercise (Exercise 02 picks up the depth-and-breadth evolution step).
- Not a preference-pair exercise (Exercise 04).
- Not a production-scale run — a few thousand examples at an hour of wall-clock is sufficient. Scale is Chapter 08's domain and is practised in Exercises 02 and 04.
- Not a licence ledger (Exercise 03).

It is the "I can run Self-Instruct end-to-end with real filters" muscle that every later exercise assumes.

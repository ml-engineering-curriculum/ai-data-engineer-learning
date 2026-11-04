# Chapter 03 — Evol-Instruct and WizardLM: Depth and Breadth Evolution

## Why this chapter exists

Self-Instruct (Chapter 02) produces a *flat* instruction pool — a broad distribution over task types at the generator's natural difficulty. Fine-tuned on a flat pool, the student model is fluent at the median task and brittle at the hard tail: compositional reasoning, multi-constraint prompts, long-form structured output.

Evol-Instruct (Xu et al. 2023, *WizardLM*) is the operator set that fixes this. Rather than generating more instructions from scratch, it **evolves** an existing pool along two axes:

- **In-Depth** evolution — make an instruction harder along a specific dimension (constraints, reasoning, concretisation).
- **In-Breadth** evolution — generate a new but topically related instruction, expanding the surface the pool covers.

The result is a corpus where each evolution lineage carries a difficulty gradient. `fine-tuning-engineer` uses the lineage to stratify a training curriculum; `model-evaluation-engineer` uses the hard-tail items for difficulty-controlled evaluation.

The original paper: Xu, Sun, Zheng, Geng, Zhao, Song, Zhang, Jiang. *WizardLM: Empowering Large Language Models to Follow Complex Instructions.* arXiv 2304.12244 (ICLR 2024). <https://arxiv.org/abs/2304.12244>. Related: Luo et al. 2023 (*WizardCoder*, arXiv 2306.08568) and Luo et al. 2023 (*WizardMath*, arXiv 2308.09583) apply the same operators to code and math domains.

## The recipe at a glance

```
seed_instruction  ─►  operator_router
                           │
             ┌─────────────┴─────────────┐
             ▼                             ▼
      in_depth_operator           in_breadth_operator
             │                             │
             ▼                             ▼
    evolved_instruction          evolved_instruction
             │                             │
             └──────────────┬──────────────┘
                            ▼
                    elimination_filter
                            │
                     ┌──────┴──────┐
                     ▼              ▼
                   drop           keep
                                    │
                                    ▼
                           generate_response
                                    │
                                    ▼
                           add_to_pool_with_lineage
```

Each round applies one operator to every surviving instruction, in parallel. The pool grows by (surviving fraction) × (pool size) per round. A typical run performs 3–5 rounds.

## The In-Depth evolving operators

Five operators. Each is a short prompt asking the generator to rewrite the instruction such that it is harder in a specific way, *without changing the input or the expected output format beyond the requested complexity*.

### 1. Add constraints

Introduce one additional constraint to the instruction.

Prompt shape (from the paper's Appendix; summarised here):

> I want you to act as a Prompt Rewriter. Your objective is to rewrite the given prompt into a more complex version by adding one more constraints/requirements. The rewritten prompt must be reasonable, must be understood and responded by humans, and cannot omit the non-text parts.

Example:

- Seed: `Write a function that returns the Fibonacci number at position n.`
- Add-constraint: `Write a function that returns the Fibonacci number at position n. The function must run in O(log n) time.`

### 2. Deepening

Make an existing requirement more specific.

Example:

- Seed: `Summarise the following passage.`
- Deepening: `Summarise the following passage in exactly three sentences, preserving the chronological order of events.`

### 3. Concretising

Replace a general concept with a specific instance.

Example:

- Seed: `Explain how a database query is optimised.`
- Concretising: `Explain how a query using `LEFT JOIN` across two 100M-row tables is optimised in PostgreSQL 15.`

### 4. Increased reasoning steps

Reshape the instruction so a correct answer requires more sequential reasoning.

Example:

- Seed: `What is the area of a rectangle with side 5 and side 3?`
- Reasoning: `A rectangle has a diagonal of length 7 and a side of length 3. What is its area? Show the steps.`

### 5. Complicate input

Thicken the input data without changing the output shape.

Example:

- Seed: `Parse the date from: "Jan 3 2024".`
- Complicate-input: `Parse the date from: "The meeting was scheduled for the third of January, Anno Domini 2024, in the afternoon."`

### Operator selection within a round

The paper samples one of the five In-Depth operators uniformly at random for each seed. This is intentional: the pool ends up with coverage along all five dimensions, and no single dimension dominates. If you weight one operator heavily, the pool skews toward that dimension's style — useful for a targeted fine-tune, but a release-eligibility check on the paper-faithful recipe should keep the operators roughly balanced.

## The In-Breadth evolving operator

A single operator. Generate a new instruction that is topically adjacent to the seed but is not simply a rewrite of it.

Prompt shape:

> I want you to act as a Prompt Creator. Your goal is to draw inspiration from the given prompt to create a brand new prompt. This new prompt should belong to the same domain as the given prompt but be even more rare. The LENGTH and difficulty level of the created prompt should be similar to that of the given prompt.

Example:

- Seed: `Write a Python function that returns the n-th Fibonacci number.`
- In-Breadth: `Write a Python function that returns the n-th Pell number.`

In-Breadth is the operator that keeps the pool from narrowing. Without it, depth evolution from a seed pool with limited topic coverage produces a very hard corpus on a very narrow surface — exactly the opposite of what you want.

### Depth : breadth ratio

The paper mixes the operators in roughly equal proportions. Pin the ratio as a hyper-parameter in your manifest:

- **High depth (e.g., 80:20).** Useful when the seed pool already covers the target topic surface and you want to stress the student on hard variants.
- **Paper default (roughly 50:50).** The baseline. Produces a pool that is both difficulty-stratified and topic-broad.
- **High breadth (e.g., 20:80).** Useful when the seed pool is narrow and the student needs more topic coverage before difficulty becomes the bottleneck.

Track the ratio per round; the manifest records it.

## The Elimination Evolving filter

The filter is as load-bearing as the operators. The paper names four specific failure modes to drop:

1. **The evolved instruction is nearly identical to the seed.** Measure with a similarity metric (embedding cosine or ROUGE-L); drop above a per-operator threshold.
2. **The generator cannot answer the evolved instruction.** The paper uses the generator itself as a judge: it asks the generator whether it can produce a correct response, and drops instructions where the generator admits it cannot.
3. **The response is information-poor.** Patterns like "Sure, I can help with that." or the generator repeating the operator prompt instead of the instruction. Drop with a short regex list and a length-of-response floor.
4. **The evolved instruction is the operator prompt leaked back.** When the generator reproduces its instruction verbatim (e.g., `"Please rewrite the following prompt..."`), drop. Exact-match against the operator-prompt library suffices.

Modern-recipe additions:

- **Judge-model elimination.** Use a second, independent model to score `(seed_instruction, evolved_instruction)` for *harder but correct*. This moves the filter from "the generator said it could answer" to a stricter check and reduces the depth-illusion failure mode (see §"Where Evol-Instruct breaks").
- **Round-level dedup.** After a round, ROUGE-L-dedup the surviving evolved instructions against each other, not just against the seed. Round-to-round dedup prevents the pool from accumulating near-duplicate evolutions from different seeds.
- **Length sanity.** An evolved instruction that is 10× the length of the seed is often a run-on; drop above a length-ratio ceiling.

## Lineage tracking

Every evolved instruction carries a lineage pointer back to its seed and (recursively) through its evolution path:

```json
{
  "instruction_id": "ev-00123",
  "lineage": [
    {"id": "seed-042", "operator": null},
    {"id": "ev-00067", "operator": "deepening"},
    {"id": "ev-00123", "operator": "add_constraints"}
  ],
  "round": 3,
  "depth": 2,
  "breadth": 0
}
```

The lineage is a first-class artifact. `fine-tuning-engineer` uses it to build a curriculum (train on round 0 first, then round 1, then round 2, etc.). `model-evaluation-engineer` uses it to construct difficulty-controlled evaluation splits. [[preference-pair-synthesis]]

### Why depth and breadth counts matter

Separate the depth count (number of In-Depth operators in the lineage) from the breadth count (number of In-Breadth operators). A high-depth / low-breadth lineage is a hard-but-narrow instruction. A balanced lineage is a hard-and-varied instruction. The counts let you filter for either shape.

## The WizardMath and WizardCoder variants

Two closely related recipes apply the same operator framework to math and code:

- **WizardMath (Luo et al. 2023, arXiv 2308.09583)** adds a "Reinforced Evol-Instruct" (RLEIF) step: the evolved instructions are scored by a math-correctness reward model, and the pool is filtered down by reward.
- **WizardCoder (Luo et al. 2023, arXiv 2306.08568)** restricts the operator set to those that preserve code-problem well-formedness: adding edge cases, requiring higher algorithmic complexity, mixing in tool-calling.

Treat these as domain adaptations of the same recipe. The operator contract — a prompt that rewrites, plus an elimination filter — is unchanged.

## Operating on NeMo Curator or Data-Juicer

Both NeMo Curator and Data-Juicer ship an Evol-Instruct-shaped operator in their synthetic-data module (Chapter 08). Prefer the library implementation over hand-rolled — the libraries manage parallelism, retry, dedup, and lineage. Pin the library version and the operator version in the manifest.

Rough mapping:

- **NeMo Curator** — `nemo_curator.synthetic.NemotronGenerator` (and friends) offers the generation-plus-elimination pattern with pluggable operators; the Evol-Instruct operator set is implemented against it. <!-- needs-research: confirm exact class / module path for the current NeMo Curator release, since the synthetic-data submodule has reorganised across versions. -->
- **Data-Juicer** — ships Self-Instruct and Evol-Instruct as data-recipe mappers under `data_juicer.ops.mapper`. <!-- needs-research: confirm exact op names for the current Data-Juicer release, since the mapper naming has evolved. -->

## The evolution manifest

```yaml
# recipes/evol_instruct_config.yaml
seed_corpus:
  path: outputs/sft_corpus.parquet   # e.g., a Self-Instruct output
  sha256: 9a21...
  item_count: 50000

generator:
  model_id: claude-3.5-sonnet-2024-10-22
  temperature: 0.7
  max_tokens: 2048

operators:
  in_depth:
    weight: 0.5
    operators:
      - add_constraints
      - deepening
      - concretising
      - increased_reasoning_steps
      - complicate_input
  in_breadth:
    weight: 0.5
    operators:
      - in_breadth

rounds: 4

elimination:
  seed_similarity_threshold: 0.9      # drop evolved ≈ seed
  cannot_answer_check: true           # generator self-judge
  information_poor_patterns_path: filters/info_poor_patterns.txt
  operator_prompt_leak_check: true
  judge_model: claude-3.5-sonnet-2024-10-22    # second-pass judge
  judge_threshold: 0.6
  round_dedup_rouge_l: 0.8
  max_length_ratio: 5.0

lineage:
  track_depth: true
  track_breadth: true

licence:
  teacher_licence_ref: generator/licence_ledger.yaml
```

## Where Evol-Instruct breaks

### 1. Depth illusion

Symptom: the evolved instructions *look* harder but the generator's responses are the same length and the same structure as the seed-response. The student model does not pick up any difficulty gradient from the corpus.

Catch: compare response-length distribution and response-token-diversity across rounds. If the evolved-round responses are indistinguishable from the seed responses, the depth signal is not making it to the response side.

Fix: add a response-side judge-model score. Promote the judge score to a filter (drop responses that do not reflect the increased difficulty of the instruction). This is the RLEIF idea from WizardMath applied as an elimination step.

### 2. Topic narrowing

Symptom: the pool gets harder but also narrower round-over-round. Topic-family counts shrink on topics that were weakly represented in the seed.

Catch: track task-family counts per round. A family that loses more than X% of its mass per round is on a collapse trajectory.

Fix: raise the breadth ratio; add synthetic breadth seeds mid-run.

### 3. Operator-prompt leakage

Symptom: the corpus contains instructions that are literally "Please rewrite the following prompt into a more complex version..." — the operator prompt itself, not an evolved instruction.

Catch: exact-match the operator-prompt library against the output pool. Any non-zero count is a filter miss.

Fix: strengthen the operator-prompt-leak check in the elimination filter. Run a regression test as part of the exercise acceptance.

### 4. Lineage loss under reshuffle

Symptom: the corpus is shuffled before export and the lineage pointers no longer line up — `fine-tuning-engineer` cannot build the curriculum.

Catch: a lineage graph that references IDs not in the corpus fails a referential-integrity check.

Fix: include `lineage` as a parquet column, not a sidecar file. The lineage travels with the row.

## Summary

- Evol-Instruct turns a flat Self-Instruct pool into a difficulty-stratified one through five In-Depth operators (add constraints, deepen, concretise, increase reasoning, complicate input) and one In-Breadth operator.
- The **Elimination Evolving** filter — seed-similarity, cannot-answer, information-poor, operator-prompt-leak — is as load-bearing as the operators themselves.
- Pin the depth:breadth ratio, the operator weights, the elimination thresholds, and the judge-model identity in the manifest.
- **Lineage is a first-class artifact.** `fine-tuning-engineer` builds the curriculum from it; `model-evaluation-engineer` builds difficulty-controlled evaluation splits from it.
- Prefer the NeMo Curator or Data-Juicer operator implementation over hand-rolled. Pin the library version.

Chapter 04 picks up with UltraChat — the recipe that produces multi-turn conversations rather than single-turn instructions.

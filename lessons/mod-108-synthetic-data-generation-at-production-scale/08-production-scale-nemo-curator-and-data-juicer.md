# Chapter 08 — Production Scale: NVIDIA NeMo Curator and Alibaba Data-Juicer

## Why this chapter exists

Chapters 02–06 pin the per-recipe operations. Running one of them on 10k examples is a weekend exercise. Running a composed pipeline — Self-Instruct seed → Evol-Instruct evolution → judge filter → preference-pair synthesis — on 10^6 examples under a cost ceiling is an engineering problem the recipe papers do not solve.

Two open-source operator libraries do: **NVIDIA NeMo Curator** and **Alibaba Data-Juicer**. Both expose the recipes as composable operators over pyarrow / parquet datasets, with managed parallelism, retry, dedup, and lineage. Both ship pretrained-generator integrations (OpenAI API, Anthropic API, vLLM / NIM for self-hosted, in-process Hugging Face `transformers`).

This chapter is not a line-by-line tutorial on either library. It is the shape of the operator-library decision — when the libraries pay off, how to pick between them, how to compose the recipes from Chapters 02–06 into a pipeline the libraries can execute, and where their respective limits are.

**Hard rule for this module: do not hand-roll an operator library.** The Self-Instruct ROUGE-L prefilter scaling problem (Chapter 02 §"Pool-size scaling problem"), the Evol-Instruct lineage tracking (Chapter 03 §"Lineage tracking"), the DPO judge throughput and retry logic (Chapter 06 §"Judge-model scoring"), and the cross-stage dedup are already solved in both libraries. Reimplementing them in Python-plus-asyncio is a predictable 2–8 week project that produces a worse answer than the library you did not use.

## The two libraries

### NVIDIA NeMo Curator

- **Project home:** <https://github.com/NVIDIA/NeMo-Curator>. Docs: <https://docs.nvidia.com/nemo/curator/>.
- **Focus:** production-scale data curation and synthetic data generation for LLM training. NVIDIA's own Nemotron recipes are the reference workloads.
- **Backing runtime:** Dask + RAPIDS for CPU/GPU pipeline work; integrates with NIM / Triton / vLLM for the generator-call side.
- **Synthetic-data operators (as of the current major release):** `NemotronGenerator` (and adapter classes) implement Nemotron-style instruction, conversation, and preference-pair generation; dedicated helpers for Self-Instruct, open-ended Q&A, and knowledge-distillation workloads are exposed under the synthetic-data module. <!-- needs-research: confirm the exact class / module path for the current NeMo Curator release; the synthetic-data submodule has reorganised across versions. -->
- **Licence:** Apache 2.0.

### Alibaba Data-Juicer

- **Project home:** <https://github.com/modelscope/data-juicer>. Paper: Chen, Yang, Li, Zheng, Dai, Ma, Shen, Liu, Chen, Yuan, Deng, Zhou, Zhang. *Data-Juicer: A One-Stop Data Processing System for Large Language Models.* arXiv 2309.02033 (SIGMOD 2024). <https://arxiv.org/abs/2309.02033>.
- **Focus:** one-stop pre-training and alignment data processing. Synthetic-data generation is one capability among many; dedup, format normalisation, PII, and language filtering are first-class.
- **Backing runtime:** Ray for distributed execution; local multiprocessing for smaller runs.
- **Synthetic-data operators (as of the current release):** mappers in `data_juicer.ops.mapper` implement Self-Instruct, Evol-Instruct, and distillation operators; filter operators in `data_juicer.ops.filter` cover length, language, quality judge, and perplexity. <!-- needs-research: confirm exact op names for the current Data-Juicer release; the mapper naming has evolved. -->
- **Licence:** Apache 2.0.

## When the libraries pay off

Pay off when you have:

- **A composed pipeline** — more than one recipe in series (e.g., Self-Instruct → Evol-Instruct → preference-pair).
- **A target count above ~10^5 examples.** Below that the library overhead is not worth the setup; above that the parallelism, retry, and lineage management are load-bearing.
- **A mix of local and API-hosted generators.** The libraries handle API rate limits, retries, and local-vLLM scheduling uniformly.
- **A requirement for lineage and reproducibility.** The libraries record per-operator inputs and outputs as part of their native data model.

Do not pay off for:

- **Ad-hoc exploration or seed-pool authoring.** Direct Python + `openai` / `anthropic` SDK is faster to iterate on.
- **Research experiments** on a specific recipe's failure modes (Exercise 05's Curse-of-Recursion ablation, for example). The library abstractions hide the experimental knob.
- **Non-LLM synthetic data** (tabular, image, audio). Both libraries are LLM-centric; see Chapter 09 for tabular tooling.

## How to pick between the two

The libraries overlap substantially. Decision criteria:

| Criterion | NeMo Curator | Data-Juicer |
|---|---|---|
| **Compute backend** | Dask + RAPIDS; strong GPU story | Ray; strong CPU-scale and mixed-cluster story |
| **Native generator-host integration** | NIM / Triton / vLLM / API | vLLM / API / local HF |
| **Dedup primitive** | MinHash / fuzzy at RAPIDS scale | MinHash / exact dedup via dedicated ops |
| **Config format** | Python API + YAML | YAML recipe + Python API |
| **Opinionation** | Nemotron-shaped; opinionated | More general-purpose; more composable |
| **Public reference pipelines** | Nemotron papers; `nemotron-cc` | WizardLM-replicate, OpenDataArena recipes, several HF Spaces |

As a heuristic:

- If you run on NVIDIA GPUs and expect to serve generators with NIM / vLLM on the same cluster, start with **NeMo Curator**.
- If you run on heterogeneous or CPU-dominant clusters and want the dedup / PII / language-filter story beside synthetic generation, start with **Data-Juicer**.
- If the choice is close, pick the library the rest of your data-pipeline is already in. The operator-library choice is sticky.

Both libraries evolve on a monthly cadence; version-pin the library in the manifest. An upgrade is a pipeline-boundary event.

## Composing the Chapters 02–06 recipes

A typical production pipeline:

```
┌─────────────────────────┐
│  seed_pool.jsonl        │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Self-Instruct operator  │  (Chapter 02)
│  • ROUGE-L dedup        │
│  • heuristic filters    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Evol-Instruct operator  │  (Chapter 03)
│  • 4 rounds, depth+breadth
│  • elimination filter   │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Response generation     │
│  (teacher distillation) │  (Chapter 05)
│  • self-consistency     │
│  • verifier / judge     │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Preference-pair synthesis│  (Chapter 06)
│  • N-candidate generation
│  • judge scoring        │
│  • pair construction    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Contamination firewall  │  (Chapter 10)
│  • n-gram / exact / LLM │
└────────────┬────────────┘
             │
             ▼
     preference_pairs.parquet
```

Three composition rules:

1. **One schema across stages.** Keep the artifact in parquet with a stable schema from Chapter 02 onward. Each operator reads and writes rows; lineage columns (`seed_id`, `evolution_lineage`, `candidate_source`, `judge_score`, `pair_role`) accumulate.
2. **Dedup at every stage boundary.** Self-Instruct dedups against seeds; Evol-Instruct round-dedups within and across rounds; preference-pair synthesis dedups candidates; firewall dedups final output against eval sets.
3. **Manifest at every stage boundary.** Each operator's config is a YAML file; the pipeline-level manifest is a composition of them, SHA-pinned.

Both libraries natively handle (1) and (3). (2) is an explicit design choice — the libraries have dedup operators; wire them in at every boundary, not just the end.

## The scaling knobs

### Parallelism

- **Generator-call parallelism.** API rate limits are the dominant constraint for hosted generators; self-hosted vLLM is throughput-limited by GPU batching. Both libraries manage this via a configurable concurrency cap.
- **Per-stage parallelism.** Dedup, filter, and format-normalisation stages parallelise across dataset partitions. Both libraries do this transparently with Dask / Ray.
- **Cross-stage pipelining.** Stage B can start on partition 0 while stage A is still running on partition 1. Both libraries support this for compatible stage pairs.

### Memory

- **ROUGE-L / embedding dedup** holds the entire reference set in memory or on fast storage. For large pools (>1M items), use the LSH backend; for very large pools, use RAPIDS / cuDF (NeMo Curator) or Ray-shuffled MinHash (Data-Juicer).
- **Embedding caching.** Compute embeddings once per item and reuse across stages. Both libraries cache embeddings as a parquet column.

### Cost

- **Prompt caching.** Both the OpenAI and Anthropic APIs support prompt caching; shared prefixes across many calls (e.g., the Evol-Instruct operator's instruction template) are a 50–90% cost saver when caching is enabled. The library's generator client exposes the caching parameter.
- **Judge-call batching.** A single judge-model call can score multiple candidates if prompted in batch. Both libraries support batched judge calls.
- **Cost projection.** Before running, estimate: (target_count × average_prompt_tokens × calls_per_item) × per-model price. Both libraries emit this estimate; pin the estimate in the manifest and compare to actual at run end.

## The pipeline manifest

```yaml
# recipes/pipeline.yaml
library:
  name: nemo_curator           # or: data_juicer
  version: 0.6.0
  commit_sha: abc123...

stages:
  - name: self_instruct
    operator: SelfInstructGenerator
    config_path: recipes/self_instruct_config.yaml
    input: seeds/seed_pool.jsonl
    output: outputs/sft_pool_v0.parquet

  - name: evol_instruct
    operator: EvolInstructEvolver
    config_path: recipes/evol_instruct_config.yaml
    input: outputs/sft_pool_v0.parquet
    output: outputs/sft_pool_v1.parquet

  - name: distillation
    operator: DistillationGenerator
    config_path: recipes/distillation_config.yaml
    input: outputs/sft_pool_v1.parquet
    output: outputs/distillation_corpus.parquet

  - name: preference_pairs
    operator: PreferencePairSynthesizer
    config_path: recipes/preference_pair_config.yaml
    input: outputs/distillation_corpus.parquet
    output: outputs/preference_pairs_raw.parquet

  - name: contamination_firewall
    operator: ContaminationFirewall
    config_path: recipes/firewall_config.yaml
    input: outputs/preference_pairs_raw.parquet
    output: outputs/preference_pairs.parquet

runtime:
  parallelism: 64
  dask_or_ray: ray
  retry_policy:
    max_retries: 3
    backoff: exponential
  cache:
    prompt_cache: true
    embedding_cache: true

cost_projection:
  estimated_usd: 1812.40
  budget_usd: 2000.00

licence:
  teacher_licence_ref: generator/licence_ledger.yaml
```

The pipeline-level manifest is pinned in the program package. Pipeline reruns against the same inputs must produce the same outputs (modulo generator non-determinism); the libraries' native caching supports this.

## Operator contribution to the program package

Each stage writes to the program package (Chapter 01 §"The program package"):

| Stage | Artifact |
|---|---|
| Self-Instruct | `outputs/sft_pool_v0.parquet`, `filters/rouge_l_report.parquet`, `recipes/self_instruct_config.yaml` |
| Evol-Instruct | `outputs/sft_pool_v1.parquet`, `filters/embedding_dedup_report.parquet`, `recipes/evol_instruct_config.yaml` |
| Distillation | `outputs/distillation_corpus.parquet`, `recipes/distillation_config.yaml` |
| Preference pairs | `outputs/preference_pairs_raw.parquet`, `filters/judge_scores.parquet`, `recipes/preference_pair_config.yaml` |
| Firewall | `firewall/removal_manifest.parquet`, `firewall/contamination_report.md`, `recipes/firewall_config.yaml`, final `outputs/preference_pairs.parquet` |

The library emits the artifacts; your job is to wire the manifest, pin the SHAs, and carry the licence and recursion-defence logs alongside.

## Where the libraries break

### 1. Operator-version silent drift

Symptom: NeMo Curator or Data-Juicer upgrades; an operator's filter default changes; your pipeline silently produces a differently-filtered corpus.

Catch: pin the library version and the operator version in the manifest. Treat an upgrade as a pipeline-boundary event (same policy as model-version drift in Chapter 05).

Fix: run a differential test before adopting an upgrade. Compare outputs on a held-out seed sample.

### 2. API rate-limit cascade

Symptom: the generator API returns 429 on every call for a stretch of hours; the library's retry policy saturates; the pipeline runs for days but accomplishes little.

Catch: monitor the per-stage throughput (items / minute). A sudden drop is a signal.

Fix: cap retries; return a failure report for the stage; the operator supports resume from the last successful item.

### 3. Cross-stage schema drift

Symptom: Stage B reads a column that Stage A wrote in an earlier run but not the current run. The operator throws a KeyError half-way through a run.

Catch: the pipeline manifest should pin the schema per stage. Both libraries support schema validation at operator boundaries.

Fix: validate schema at the stage boundary; fail fast on mismatch.

### 4. Library-specific lineage format lock-in

Symptom: a stage produces lineage in the library's native format (e.g., Data-Juicer's operator-call history). A downstream mod-109 lineage consumer expects a different shape.

Catch: the mod-109 hand-off is explicit — map the library's lineage into the pipeline's lineage format at each stage boundary.

Fix: an adapter layer between the library and mod-109's lineage schema. Keep it thin.

## Summary

- Running synthetic-data recipes at production scale requires an **operator library** — NVIDIA NeMo Curator or Alibaba Data-Juicer. Do not hand-roll.
- The two libraries overlap substantially; pick on compute backend (NVIDIA-GPU → NeMo Curator; CPU-dominant / Ray → Data-Juicer) and on what the rest of your pipeline uses.
- Compose the Chapters 02–06 recipes as **stages** in a pipeline: one schema across stages, dedup at every stage boundary, manifest at every stage boundary.
- Pin the library version, the operator version, and the SHAs of all inputs in the pipeline manifest. An upgrade is a pipeline-boundary event.
- Prompt caching, embedding caching, and judge-call batching are the cost-dominant optimisations. Both libraries expose them.
- The library writes to the program package; you own the licence ledger, the recursion-defence log, and the mod-109 lineage adapter.

Chapter 09 picks up with the one slice the LLM-centric libraries do not cover — tabular / structured synthetic data with differential privacy.

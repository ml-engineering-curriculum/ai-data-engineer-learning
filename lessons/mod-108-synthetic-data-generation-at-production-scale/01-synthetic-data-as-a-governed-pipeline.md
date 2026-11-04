# Chapter 01 — Synthetic Data as a Governed Pipeline

## Why this chapter exists

Synthetic generation sits in a strange place in the pipeline. From a cost-accounting view it looks like free labour: a generator model produces an unbounded supply of training-shaped text with no humans in the inner loop. From a quality-accounting view it looks like a lever that can lift the whole corpus — the generator is usually a stronger model than the one being trained, so its outputs are *higher* average quality than what the student model could produce on its own.

Both views are true. Both views are also incomplete in exactly the same way: they ignore the four things synthetic generation inherits silently from the generator.

1. **The generator's licence posture**, which may forbid using its outputs to train a competing model (Chapter 05).
2. **The generator's benchmark contamination**, which propagates into every synthetic example that touches a benchmark-adjacent topic (Chapter 10).
3. **The generator's own distribution** — which, if you iterate on it, collapses onto its own mode and loses the tails of the real distribution (Chapter 07).
4. **The generator's biases, privacy exposures, and failure modes** — which the student model is now being trained to imitate.

A synthetic-data program that does not own all four of these is a program that will produce a corpus which looks fine at build time and produces a model that fails one or more of: licence review, contamination audit, long-tail eval, PII audit. This chapter sets the engineering frame that owns all four.

## The deliverable — the synthetic-data generation package

The deliverable at the end of this module is a **synthetic-data generation package** for a real slice. The package has seven artifacts, each created by a later chapter:

1. **Generation program manifest** (Chapters 02, 03, 04, 08) — seeds, prompts, generator identity, generation settings, iteration budget, run SHAs.
2. **Diversity + quality filter pipeline** (Chapters 02, 03, 08) — dedup against seeds, ROUGE-L / embedding similarity thresholds, judge-model scores, length and language filters.
3. **Preference-pair artifact** (Chapter 06) — paired generations, judge verdicts, pair-balance report, difficulty-stratification bucket counts.
4. **Teacher-licence propagation ledger** (Chapter 05) — teacher model identity, teacher-side licence text SHA, downstream licence decision, counsel sign-off reference.
5. **Curse-of-Recursion defence log** (Chapter 07) — iteration counter, human-anchor ratio, round-over-round drift metrics, kill-switch threshold.
6. **Contamination firewall report** (Chapter 10) — mod-106 firewall re-run on the synthetic corpus, per-benchmark removal counts, generator-certification lookup.
7. **Tabular-synthesis package** (Chapter 09) — generator model, differential-privacy `(ε, δ)`, utility report, legal-counsel sign-off reference. Only when the program produces tabular / structured data.

Every chapter adds one block to this package. Chapter 01 is the defence of why the package exists.

## The four claims you must be able to defend

Pin these four. They are the ones you will cite when someone argues the synthetic-data step is prompt engineering, not engineering.

### 1. The Self-Instruct recipe is the baseline, and it only works because of its filters (Wang et al. 2022)

**Paper:** Wang, Kordi, Mishra, Liu, Smith, Khashabi, Hajishirzi. *Self-Instruct: Aligning Language Models with Self-Generated Instructions.* arXiv 2212.10560 (ACL 2023). <https://arxiv.org/abs/2212.10560>.

Key findings, in the shape you should be able to cite:

- Starting from **175 seed tasks** (hand-written by the authors), a generator (GPT-3 `text-davinci-001` in the paper) can be bootstrapped to produce a large instruction corpus.
- The generation loop is not single-pass. Every new instruction candidate is filtered against the growing pool by **ROUGE-L similarity**; candidates with ROUGE-L > 0.7 against any existing instruction are dropped as near-duplicates.
- Heuristic filters remove instructions that are too short, too long, that contain keywords the model cannot reliably handle (e.g. "image", "graph"), and instructions where the generator refused.
- **Without the filters**, the pool collapses to near-duplicates and surface patterns the generator is biased toward. The filters are not quality-of-life polish — they are what makes the recipe work.

**Why it matters for your pipeline:** every later recipe inherits this shape. Evol-Instruct adds *evolution* on top of filtered seeds. UltraChat adds *multi-turn structure* on top of filtered topics. The filters are the load-bearing piece. If you take the filters off, the corpus collapses, no matter which recipe you run.

### 2. Evol-Instruct turns a flat pool into a difficulty-stratified one (Xu et al. 2023)

**Paper:** Xu, Sun, Zheng, Geng, Zhao, Song, Zhang, Jiang. *WizardLM: Empowering Large Language Models to Follow Complex Instructions.* arXiv 2304.12244 (ICLR 2024). <https://arxiv.org/abs/2304.12244>.

Key findings:

- The **In-Depth Evolving** operators — add constraints, deepen, concretize, increase reasoning steps, complicate input — mutate an instruction toward the hard tail.
- The **In-Breadth Evolving** operator mutates toward a new but related topic, expanding the surface coverage of the pool.
- The **Elimination Evolving** filter drops evolved instructions that are too similar to the seed, that the generator cannot answer, that are information-poor ("I understand your question"), or that are copies of the operator prompt.
- The resulting pool is **difficulty-stratified**: each evolution round shifts the distribution of difficulty rightward, and you keep the evolved items alongside their seeds rather than replacing them.

**Why it matters for your pipeline:** downstream fine-tune evals are usually bottlenecked on the hard tail of the training distribution. A flat Self-Instruct pool trains a model that is fluent at the median task and brittle at the hard tail. Evol-Instruct is the operator set that fixes this without manual authoring.

### 3. Recursive training collapses the distribution (Shumailov et al. 2024)

**Paper:** Shumailov, Shumaylov, Zhao, Papernot, Anderson, Gal. *AI models collapse when trained on recursively generated data.* Nature 631 (2024). <https://www.nature.com/articles/s41586-024-07566-y>. Earlier arXiv: *The Curse of Recursion: Training on Generated Data Makes Models Forget.* arXiv 2305.17493.

Key findings:

- Train model generation 1 on real data. Train generation 2 on the outputs of generation 1. Train generation 3 on the outputs of generation 2. Each generation loses mass in the tails of the real distribution, and the loss is cumulative and (under the paper's conditions) irreversible.
- The collapse is visible on simple diagnostics — distribution KL divergence, perplexity on a held-out real-data tail, qualitative loss of rare categories — long before it is visible on average task metrics.
- The defence is **not** more generation. More generation on the collapsing distribution accelerates the collapse. The defence is bounded iteration, anchoring to real data, and quality filtering of synthetic outputs against the real distribution.

**Why it matters for your pipeline:** "synthetic data is cheap, let's regenerate every week against the latest checkpoint" is a Curse-of-Recursion failure mode with a release schedule. Chapter 07 sets the operating constraints: **iteration budget**, **human-anchor ratio**, and **round-over-round drift kill-switch**. Without these, you have a schedule for model collapse, not a pipeline.

### 4. Preference-pair synthesis is where the DPO / ORPO / KTO signal is made (Rafailov et al. 2023)

**Paper:** Rafailov, Sharma, Mitchell, Ermon, Manning, Finn. *Direct Preference Optimization: Your Language Model is Secretly a Reward Model.* arXiv 2305.18290 (NeurIPS 2023). <https://arxiv.org/abs/2305.18290>.

Key findings:

- DPO treats the policy as a reward model and trains directly on paired (chosen, rejected) completions — no reward-model training loop, no PPO.
- The loss is directly sensitive to **pair quality**. A noisy or balanced-but-wrong pair contributes a wrong gradient that cannot be compensated for by more pairs.
- The pair-synthesis operation — generate N candidates, score them, pair them — is the single operation where the preference signal is set. ORPO (Hong et al. 2024, arXiv 2403.07691) and KTO (Ethayarajh et al. 2024, arXiv 2402.01306) are different *losses* on the same pair-synthesis output; none of them rescue a bad pair-synthesis step.

**Why it matters for your pipeline:** if `fine-tuning-engineer` reports that a DPO run is not moving the policy, the most common root cause is the pair-synthesis step upstream, not the training recipe. Chapter 06 is the operational chapter; the hand-off contract pins what `fine-tuning-engineer` receives from you (pairs + judge metadata + pair-balance + difficulty stratification), and what they do with it (the training run).

## The synthetic-generation framing in ISO terms

As in every other module in this track, the framing lands on the ISO/IEC 25012 vocabulary you pinned in [`mod-101`](../mod-101-ai-data-governance-and-position/README.md) and [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md):

| ISO 25012 characteristic | How synthetic generation affects it |
|---|---|
| **Accuracy** | Judge-model filters (Chapter 06) are the accuracy proxy on synthetic examples. The judge's own accuracy is a hyper-parameter of the system. |
| **Completeness** | Evol-Instruct breadth evolution (Chapter 03) and UltraChat's topic tree (Chapter 04) are the two mechanisms for ensuring the generated corpus covers the input distribution, not just its centre. |
| **Consistency** | Synthetic examples share the generator's stylistic and structural biases — this is usually a *higher-than-human* consistency. On tabular data (Chapter 09) it is also where the generator reproduces inter-column constraints. |
| **Credibility** | Synthetic data's credibility is bounded above by the generator's. The licence ledger (Chapter 05) and firewall report (Chapter 10) are the credibility artifacts. |
| **Confidentiality** | Tabular synthesis (Chapter 09) with DP is the mechanism for emitting training data that is formally bounded against membership-inference. Textual synthesis has weaker guarantees — a generator trained on PII can regurgitate it. |
| **Traceability** | Every synthetic example traces to a seed (or an evolution lineage), a generator, a generation timestamp, a filter verdict, a firewall verdict. This is a [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) lineage event. |

**ISO/IEC 5259-3** (*AI — Data quality for analytics and machine learning — Part 3: Data quality management requirements and guidelines*) scopes **synthetic-data generation** as a managed data-quality activity within the training-data pipeline. The deliverable at the end of this module is the artifact that implements that scope for your slice. <!-- needs-research: confirm the 5259-3 clause numbers that pin synthetic-data management; the published edition has reorganised across revisions. -->

## What synthetic generation is — and what it isn't

Pin these definitions. Different recipes produce different *shapes* of corpus, and the operations generalise but the artifacts do not.

- **Instruction tuple** — a `(prompt, response)` pair, usually one-turn, used for SFT. Self-Instruct and Evol-Instruct produce these.
- **Multi-turn conversation** — a `(system, user_1, assistant_1, user_2, assistant_2, ...)` trace. UltraChat produces these.
- **Demonstration trace** — a `(prompt, reasoning_chain, response)` triple used for chain-of-thought SFT. Teacher-student distillation routinely produces these (Chapter 05).
- **Preference pair** — a `(prompt, chosen, rejected)` triple used for DPO; or a `(prompt, response, label ∈ {good, bad})` tuple used for KTO; or a `(prompt, response, reference_response)` triple used for ORPO (Chapter 06).
- **Tabular record** — a row or row-set drawn from a learned joint distribution over columns (Chapter 09). Shape-identical to the real training table.
- **Code trajectory** — a `(problem, candidate_program, test_outcome)` triple. Specialised; treated as a special case of demonstration traces for this module.

Each shape has its own filter pipeline, its own hand-off format, and its own threat model. The operations chapters (Chapters 02–06) pin the per-shape details; the operator-library chapter (Chapter 08) pins the pipeline-level composition.

## What the synthetic-data program is **not**

Mark the boundary explicitly — it is often where organisations misfile work:

- It is **not** training-time curation of a synthetic corpus. `fine-tuning-engineer` (peer, L30) owns preference-pair selection, filtering, and format conversion for a specific training run. You own the program that produces the pairs and their metadata (Chapter 06).
- It is **not** benchmark construction. `model-evaluation-engineer` (peer, L30) owns benchmark design; you own the firewall on the generator (Chapter 10) that keeps those benchmarks out of synthetic outputs.
- It is **not** legal opinion. Chapter 05 is the licence *ledger* — the artifact that captures teacher-model identity, teacher-side licence text SHA, downstream licence decision, and counsel sign-off reference. The legal opinion itself is counsel's work. You structure the facts; they render the call.
- It is **not** the PII / privacy-risk programme. [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md) and `ai-risk-engineer` own the privacy assessment. Chapter 09's DP guarantees feed into their assessment; they are not a substitute for it.
- It is **not** a content-filter tuning programme. Harmful-content filters are an `ai-risk-engineer` concern; you apply them, you do not design them.

## The economic shape — why synthetic generation is a tooling problem

A production synthetic-generation run routinely emits 10^5–10^7 examples. Per-example API-model cost depends on model, modality, and output length; the point is that **the generator compute cost is comparable in magnitude to the student model's SFT compute cost, and much smaller than its pretraining compute cost**. <!-- needs-research: avoid citing a specific per-token dollar figure; treat the shape qualitatively and point a reader to each vendor's current pricing page at authoring time. -->

Two operational consequences:

1. **Operator libraries matter.** Hand-rolling a generation + filter + dedup loop in Python is a 30× slowdown next to NeMo Curator or Data-Juicer on the same hardware (Chapter 08). The generator calls can be throughput-bound even on the cheap tier, which means the surrounding pipeline needs to be throughput-matched.
2. **Iteration is cheap enough to get you in trouble.** Because the generator is cheap, teams are tempted to iterate against the latest student checkpoint. This is exactly the Curse-of-Recursion failure mode. Chapter 07 sets the budget constraints that bound the risk.

## The program package — what this module produces

Here is the artifact tree you will accumulate over Chapters 02–10 and the six exercises:

```
synthetic_program/
  manifest.yaml                      # chapter 01 shape, populated through chapters 02-04
  seeds/
    seed_pool.jsonl                  # chapter 02
    seed_pool.sha256
  generator/
    generator_identity.yaml          # chapter 05 — model id, provenance, licence refs
    licence_ledger.yaml              # chapter 05
  recipes/
    self_instruct_config.yaml        # chapter 02
    evol_instruct_config.yaml        # chapter 03
    ultrachat_config.yaml            # chapter 04
    distillation_config.yaml         # chapter 05
    preference_pair_config.yaml      # chapter 06
  outputs/
    sft_corpus.parquet               # chapters 02, 03, 04
    distillation_corpus.parquet      # chapter 05
    preference_pairs.parquet         # chapter 06
    tabular_output.parquet           # chapter 09 (if applicable)
  filters/
    rouge_l_report.parquet           # chapter 02
    embedding_dedup_report.parquet   # chapter 03
    judge_scores.parquet             # chapter 06
    heuristic_filter_report.parquet  # chapters 02-04
  recursion_defence/
    iteration_log.yaml               # chapter 07
    anchor_ratio_report.md
    drift_kill_switch_log.md
  firewall/                          # chapter 10
    synthetic_generator_contract.yaml  # inherits mod-106 shape
    removal_manifest.parquet
    contamination_report.md
  tabular/                           # chapter 09 (if applicable)
    dp_report.yaml
    utility_report.md
    counsel_signoff_ref.md
  program_report.yaml                # the single document that references all of the above
```

Every item in the tree has a chapter and (usually) an exercise that produces it. By the end of the module the tree is a real, signed, auditable deliverable for a real slice.

## Summary

- Synthetic generation is a **governed engineering operation**, not a prompt-engineering exercise. Four findings pin the framing: Wang 2022 (filters are load-bearing), Xu 2023 (evolution produces the hard tail), Shumailov 2024 (recursive training collapses), Rafailov 2023 (pair-synthesis is where the preference signal is made).
- It implements ISO/IEC 25012 *accuracy*, *completeness*, *consistency*, *credibility*, *confidentiality*, and *traceability* characteristics simultaneously and falls under ISO/IEC 5259-3's synthetic-data management scope.
- The deliverable at the end of the module is a **program package** — manifest, seeds, generator identity, filter reports, Curse-of-Recursion log, firewall report, licence ledger, tabular DP report — not a JSONL file with generations.
- Synthetic generation is cheap enough that iteration is tempting and dangerous. The three load-bearing budget levers are **iteration budget**, **human-anchor ratio**, and **round-over-round drift kill-switch**.
- Boundaries: `fine-tuning-engineer` owns training-time curation; `model-evaluation-engineer` owns benchmarks; `ai-risk-engineer` owns risk; legal counsel owns licence opinions. You own the operation that produces the synthetic examples they consume.

Chapter 02 is the practical starting point — Self-Instruct as the baseline recipe, with its seed pool, its generation loop, its ROUGE-L dedup, and its heuristic filters pinned concretely.

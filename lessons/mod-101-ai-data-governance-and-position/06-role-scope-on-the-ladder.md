# Chapter 06 — The AI Data Engineer on the Level Ladder

## Why this chapter exists

Every mod-101 ends with the question: *what, exactly, is my part of this work?* The frameworks in Chapters 01–05 can all be answered by multiple roles — a governance analyst can write a dataset card, an ML engineer can run a dedup job, a staff engineer can decide data-mixture strategy. Without a crisp scope, you end up either doing other people's work (slow, resented) or leaving holes (unsafe, uncertified).

This chapter draws the scope of the **AI Data Engineer (level 30, ML Engineering family)** by contrast with the roles above, below, and alongside. The level references are from the ML Engineering Curriculum's level ladder.

## The one-sentence scope

> The AI Data Engineer **owns the engineering craft of the training and evaluation data pipeline end-to-end** — sourcing, licensing, acquisition, curation, deduplication, decontamination, labelling operations, synthetic generation, versioning, PII scrubbing, quality monitoring — and emits the per-slice artifacts that let model trainers, governance, legal, and regulators reason about what was trained on.

Every other role either hands you input you consume, consumes output you produce, or operates on a different slice of the same stack.

## What `ml-engineer` (level 20) already owns

Level 20 is the general ML engineer. Below L30 in scope, above L10 in craft. At L20 you are already expected to know:

- Classical data-engineering primitives — pandas, SQL, Parquet, basic DVC usage.
- Classical ML development — scikit-learn, PyTorch, training loops, MLflow experiment tracking.
- Feature-store consumer tier — pulling features for a training job, not designing the store.
- Basic data-quality intuition — missingness, class imbalance, train/test leakage as a concept.

What you (L30 AI Data Engineer) do **not** re-teach: all of the above. You assume the student can build a Parquet pipeline, write a pandera schema, and run a PyTorch training loop before stepping into mod-101.

What you do that L20 does not:
- Governance-framework fluency (Chapters 01–05).
- Web-scale acquisition pipelines (Common Crawl, trillion-token dedup).
- Labelling operations at vendor scale.
- Benchmark-contamination firewalls.
- Signed, lineage-tracked, regulator-facing packaging of training data.

## What `ai-governance-analyst` (level 15) hands you

L15 is a specialist in the AI Governance family. Despite the lower number, this role is a *peer* in the sense that neither can do the other's job. The governance analyst produces the operational-legwork artifacts you consume:

- **AI system inventory entry** — the ID that your slice ties to. When the governance council asks "which slice fed model M?" the answer walks through the inventory.
- **Impact assessments / DPIAs** — written by the analyst. You produce the per-slice evidence that goes into the DPIA (quality report, bias audit, dedup report, PII report).
- **Lawful-basis and licence posture per source** — the analyst, with legal, decides the posture. You implement the admit function that enforces the posture.
- **Risk-register entries** — the analyst writes the register. You own the mitigation records for data-side risks (Chapter 02 → ISO/IEC 23894 Annex B risk sources).

In words: *the governance analyst writes what the organisation says it does. You run the pipeline that makes it true.*

## What peers at level 30 do on top of your output

Four peer roles in the ML Engineering family, plus one in the AI Engineering family, consume your pipeline output:

- **`fine-tuning-engineer` (L30)** — SFT dataset design, preference-pair curation, DPO/ORPO/KTO design, SFT-side dataset-mix cards. Consumes your sourced + curated + deduplicated + decontaminated + PII-scrubbed corpus. You do not design the SFT mix; you supply the substrate and the metadata.
- **`rag-engineer` (L30)** — retrieval-side chunking, embedding selection, vector-DB design, hybrid retrieval. Consumes your clean corpus (and, for RAG, your continually-refreshed corpus with the same quality controls as training data). You do not design the chunking strategy or the embedding model; you supply the clean input.
- **`model-evaluation-engineer` (L30)** — benchmark construction, gold-eval-set design, judge calibration, canary sets. Consumes your labelling-operations output (annotated gold items) and the contamination firewall's output (holdout integrity). You do not design the benchmark; you run the labelling program and the firewall.
- **`nlp-engineer` (L30)** — tokeniser training, tokeniser evaluation. Consumes your pre-tokenisation corpus (text, metadata). You do not train tokenisers; you deliver the input corpus in a tokeniser-trainable layout.
- **`ai-eval-engineer` (L30, AI Engineering family)** — application-trace evaluation, judge design, online evaluation on production traces. Consumes your labelling-operations output for application-specific eval-item curation. You do not own trace analysis; you own the labelling pipeline.

The pattern across all five: *they decide the modelling or evaluation strategy; you supply the data substrate and the per-slice evidence that the substrate is trustworthy.*

## What `training-pipeline-engineer` (level 35) inherits

L35 is the next step up on the ML Engineering ladder. The handoff between L30 AI Data Engineer and L35 Training Pipeline Engineer is about *where the data pipeline stops and the training pipeline starts*.

Your boundary:
- **You** design the pipeline shape, author the operators, produce the slice, emit the artifacts.
- **L35** takes the slice and makes it trainable at cluster scale — WebDataset tar-shard layout, MosaicML StreamingDataset packing, Hugging Face Datasets Parquet/Arrow format, Ray Data streaming, tokenisation throughput, I/O pipelines, dataloader contention at thousand-GPU scale.

The deliverable at the boundary is a **slice handoff contract** — a documented contract (part of the dataset card) that specifies:
- Slice SHA and signing identity.
- Format and partitioning.
- Expected cardinality and byte size.
- The quality-measure log and known-gaps declaration.
- Any parameters the training pipeline needs (e.g. recommended shuffle buffer size, token-count per shard).

L35 does not re-do your dedup or your contamination firewall; the contract treats those as given. If L35 discovers an issue at cluster scale (e.g. shard-level duplicates across your dedup granularity), it feeds back to you as a defect, not a fix.

## What `staff-ml-engineer` (level 40) inherits

L40 is staff-level ML engineering with foundation-model data-mixture strategy at org scope. The handoff at this level is less of a handoff and more of a *consumption*:

- **You** produce per-source and per-slice datasets with full documentation.
- **L40** decides how slices are *mixed* into a training run — ratios, scheduled curriculum, upsampling weights, diversity constraints, token-budget allocation across sources.

L40 does not re-sourcing or re-curating; L40 chooses *which* of your slices and in what proportions. L40 relies on your evidence to make the mixture call — the bias audit tells them whether to upsample underrepresented sources, the contamination report tells them which slices are safe for the eval mixture, the dedup report tells them what the effective token count is after cross-slice dedup.

## What you defer to peers outside the ML Engineering family

Two roles in the AI Governance family you must know:

- **`ai-risk-engineer` (L25, Governance family)** — harm modelling, adversarial-ML methodology, guardrail design. You defer here on the methodology of data-poisoning defence (Nightshade/Glaze detection in mod-105 you implement; the methodology lineage is with `ai-risk-engineer`).
- **`ai-evaluation-engineer` (L35, Governance family)** — release-assurance, audit-trail methodology, regulator-facing package authoring. You defer here on what the final regulator-facing dossier looks like; you provide the data slice.

One role in the AI Infrastructure family:

- **`ai-infra-ml-platform-engineer` (L30)** / **`ai-infra-mlops` (L25)** — the ML platform you run on. You defer here on orchestrator (Airflow / Prefect / Argo / Kubeflow / Metaflow) design, feature-store internals, experiment-tracking infrastructure. Your pipeline is a workload on their platform.

## Out of scope, full stop

- **Legal counsel** — legal opinion on copyright, licensing, jurisdictional data law, M&A data treatment. You package the evidence; counsel opines.
- **SOC / DFIR** — incident response for a data breach. If a secret leaks from a training corpus or a dataset goes missing, the security operations function runs the incident; you supply the forensic data.
- **Product management** — which use case the training supports, which customers it is sold to, what the SLA is. The model/product team owns product decisions; you supply the data-side feasibility read.

## Deferral contract — how to communicate scope

Scope is clearest when written. A one-page *deferral contract* that lives in your team's README answers, for anyone reading, "what to ask this role for and what not to ask":

```markdown
# AI Data Engineer — deferral contract

I own the engineering craft of the training and evaluation data pipeline end-to-end. Ask me for:
- Per-slice datasets with full per-source manifests, licence tags, and lawful-basis records
- Dataset cards (HF / Google / Meta format) and training-data summaries (EU AI Act Art 53(1)(d) template)
- Quality reports (ISO/IEC 25012 + ISO/IEC 5259-2) and bias audits (Dodge-style)
- Deduplication reports (MinHash + LSH at document / paragraph / sentence)
- Benchmark contamination firewall reports (MMLU / GSM8K / HumanEval / HELM / BigBench / LiveBench)
- Labelling operations — in-house (Label Studio, Argilla, Prodigy, doccano) and vendor
- Preference-labelling programs for RLHF / DPO / ORPO
- Synthetic-data generation (Self-Instruct / Evol-Instruct / UltraChat / teacher-student)
- Dataset versioning (DVC / lakeFS / Pachyderm) + lineage (OpenLineage / Marquez / DataHub)
- PII detection + redaction (Presidio / scrubadub / Cloud DLP / Comprehend)
- Dataset-side monitoring + regression detection (whylogs / Evidently / Soda Cloud)

Do not ask me for:
- Legal opinion on copyright, licensing, or jurisdictional data law → counsel
- The AI system inventory, DPIAs, or the organisational risk register → `ai-governance-analyst`
- SFT recipe design or DPO/ORPO objective design → `fine-tuning-engineer`
- Benchmark design or gold-eval-set methodology → `model-evaluation-engineer`
- Tokeniser training or tokeniser evaluation → `nlp-engineer`
- RAG chunking / embedding / vector-DB selection → `rag-engineer`
- Cluster-side training-data I/O (WebDataset / Mosaic Streaming / Ray Data) → `training-pipeline-engineer`
- Foundation-model data-mixture strategy at org scope → `staff-ml-engineer`
- The platform (orchestrator, feature store, experiment tracking) → `ai-infra-ml-platform` / `ai-infra-mlops`
- Incident response for data breach → SOC / DFIR
```

Post this. Link it in your team's wiki. When someone asks for something off the list, route them — don't do it quietly.

## Summary

- The AI Data Engineer (L30) owns the engineering craft of the training-data pipeline end-to-end.
- `ml-engineer` (L20) provides your prerequisite craft; you build on it. `ai-governance-analyst` (L15) provides the inventory and impact-assessment legwork you consume.
- Peer L30 roles (`fine-tuning-engineer`, `rag-engineer`, `model-evaluation-engineer`, `nlp-engineer`, `ai-eval-engineer`) consume your pipeline output and apply modelling or evaluation strategies on top.
- `training-pipeline-engineer` (L35) inherits cluster-side ingestion; `staff-ml-engineer` (L40) inherits data-mixture strategy. Hand off via a documented slice contract.
- `ai-risk-engineer` (L25) owns harm-model methodology; `ai-evaluation-engineer` (L35, Governance family) owns regulator-facing package authoring — you provide the data slice of each.
- Write a deferral contract. Post it. Route cleanly.

Exercise 05 asks you to draw this diagram and the deferral contract for a concrete use case. The exercises in this module close the loop between the five chapters and the artifact-emitting pipeline you will spend the rest of the curriculum building.

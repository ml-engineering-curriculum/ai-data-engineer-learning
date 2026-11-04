# Exercise 04 — Data-Juicer Operator-Chain Authoring

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 05 — Operator libraries](../05-operator-libraries.md), [Chapter 02 — CCNet-style pipeline](../02-ccnet-pipeline-end-to-end.md)

## Objective

Author, run, and instrument a complete CCNet-style curation pipeline using Alibaba Data-Juicer's operator library — no custom operators, only configuration of library-supplied ones. The goal is to internalise what Data-Juicer makes free, what still requires custom code, and how to add project-standard instrumentation around library operators.

By the end you will have:
- A YAML operator chain running at least ten Data-Juicer operators against your Exercise-01 slice.
- A uniform instrumentation layer that logs config, drop fraction, and dropped-sample examples per operator.
- A "custom-code gap" report explaining which pipeline responsibilities Data-Juicer did not cover and where you wrote code to cover them.

## Prerequisites

- You have read Chapter 05 and skimmed the Data-Juicer paper (Chen et al. 2023, <https://arxiv.org/abs/2309.02033>) and the operator catalog in the repo README.
- You have `data-juicer` installed (`pip install py-data-juicer` plus the `ray` extra if you want the Ray backend — verify the current install incantation at <https://github.com/modelscope/data-juicer>).
- You have completed Exercise 01 and have a slice of ~100,000 English documents in JSONL on disk or in S3.
- You have downloaded any model files Data-Juicer operators require (FastText `lid.176.bin` for `language_id_score_filter`, a KenLM model if you use `perplexity_filter`).

## Problem statement

Your organisation has standardised on Data-Juicer as the operator library for web-text curation. You need to produce a curated version of your Exercise-01 slice that is production-grade: language-filtered, length-bounded, repetition-filtered, perplexity-tiered, exactly-deduplicated and near-duplicated, boilerplate-removed, and PII-scrubbed to a basic level — using only Data-Juicer operators.

You also need to give the on-call engineer who operates this pipeline a uniform way to answer "what happened at operator X?" without reading each operator's source.

## Starter guidance

1. **Design the operator chain.** On paper first, without touching YAML. At minimum include:
   - URL / host filter (if your JSONL carries URLs).
   - HTML cleaner (if your JSONL carries raw HTML; skip if you already extracted in Exercise 01).
   - Text length filter.
   - Language ID score filter (English, threshold 0.5).
   - Special-character ratio filter.
   - Word-repetition filter.
   - Line-level deduplication.
   - Perplexity filter.
   - Document-level MinHash deduplicator.
   - A basic PII operator (regex-based email / phone redaction) — Data-Juicer has `clean_email_mapper` and `clean_ip_mapper`; others may be added by the time you read this.
   Verify each operator name against the current Data-Juicer operator catalog in the repo; names evolve.

2. **Author `recipe.yaml`**. Use Data-Juicer's `tools/process_data.py` entrypoint. Set `executor_type` to `default` for a workstation or `ray` for a cluster.

3. **Add your instrumentation.** Data-Juicer logs per-operator counts; your layer standardises the output. Options:
   - Fork or wrap Data-Juicer's `TracerProvider` to emit JSONL lines with your fields.
   - Run a lightweight post-processor that reads Data-Juicer's trace output and emits your standard file.
   - Hook into Data-Juicer's `Analyzer` to emit per-operator aggregates.
   Whichever you pick, your output per operator must include: `name`, `version`, `config_sha256`, `docs_in`, `docs_out`, `drop_fraction`, `wall_clock_seconds`, `sample_dropped` (10 records with reasons).

4. **Run the pipeline** against your Exercise-01 slice. Measure end-to-end wall-clock and compare to your custom Exercise-01 pipeline if you have one.

5. **Emit the slice manifest** in the Chapter 07 schema, populating the `operators` list from your instrumentation output.

6. **Write the custom-code gap report.** List every pipeline responsibility you expected to cover with Data-Juicer and which ones you had to write code for. Expected gaps include: the acquire-stage opt-out check (Chapter 06), project-specific URL deny-lists beyond the public UT1 lists, benchmark decontamination against your specific internal benchmarks (Data-Juicer's decontam operator — if present at read time — may not cover your target benchmark set), and the slice manifest emission itself.

## Deliverables

- `recipe.yaml` — the Data-Juicer operator-chain config.
- `code/instrumentation.py` — the standardised per-operator logger.
- `code/manifest_emitter.py` — reads instrumentation output and emits the Chapter-07 slice manifest.
- `outputs/` — the curated shards.
- `instrumentation/` — one JSONL per operator with the standardised fields.
- `manifest/slice_manifest.json` — the final manifest.
- `custom_code_gap_report.md` — the gap report.
- `README.md` — reproduction instructions + pinned Data-Juicer version.

## Acceptance criteria

- At least 10 Data-Juicer operators run in the chain, each with documented config.
- Operator catalog entries are verified against the Data-Juicer repo at your pinned version. Operators that have been renamed or removed since the module was authored are handled with `<!-- needs-research: ... -->` plus the current equivalent.
- Instrumentation produces the same schema for every operator; `docs_in`, `docs_out`, `drop_fraction`, `wall_clock_seconds`, `sample_dropped` (with reasons) are present per operator.
- The slice manifest validates against the Chapter-07 schema shape; the `operators` list is populated from the instrumentation.
- The custom-code gap report names at least three pipeline responsibilities that Data-Juicer did not cover, with explanations.
- The README's reproduction instructions actually re-run the pipeline end-to-end on a fresh checkout.

## Stretch goals

- **Run the same chain with NeMo Curator operators** on the same input and compare operator counts, config complexity, and wall-clock. Report which library you'd pick for this specific slice and why.
- **Run the same chain in the Dolma toolkit** (tagging + mixing configs) on the same input and do the same comparison. The three-way comparison across Data-Juicer / NeMo Curator / Dolma is the strongest "which library do I pick" exercise.
- **Scale the Data-Juicer run to a Ray cluster** and measure the scaling curve. If you have a single multi-node cluster available, 2-node and 4-node runs tell you where Data-Juicer's Ray backend is CPU-bound vs network-bound.
- **Contribute an operator upstream.** If your custom-code gap report names a widely useful missing operator, consider filing an issue or a PR against Data-Juicer. Document it in the deliverables.

## What this exercise is not

- It is not an argument that Data-Juicer is the right library for every pipeline. Chapter 05 and this exercise's stretch goals cover the comparison; the exercise's point is to make one library's operator catalog muscle-memoried.
- It is not a benchmark of operator *quality*. Operator A might drop more documents than operator B at the same config; whether that's "better" depends on downstream evaluation. That's mod-104 territory.
- It is not a complete production pipeline. You still have to layer the Chapter-06 acquire-stage check, the mod-105 cross-source dedup if applicable, the mod-106 decontamination against your internal benchmarks, the mod-110 PII pass with context-aware recognisers, and the mod-109 signing + attestation. Data-Juicer handles the operators in the middle; the pipeline is the whole thing.

Running this exercise and then writing the gap report is the single best way to understand the *custom-code surface* this module keeps pointing at: what is your code vs. what is the library's.

# Exercise 02 — Redaction Primitive Choice Per Use Case

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 05 — Redaction primitives and training utility](../05-redaction-primitives-and-training-utility.md), and Exercise 01 as upstream.

## Objective

Choose a redaction primitive (mask / tokenise / surrogate / FPE / hash) per entity type with written justification; implement them with Presidio's `AnonymizerEngine`; validate utility and privacy properties on a small held-out slice; populate §4 (redaction policy) and §5 (residual PII accounting) of the Chapter 09 privacy-risk report.

By the end you will have:

- A per-entity-type redaction policy table with written rationale per choice.
- A working Anonymizer pipeline that applies each primitive consistently across the slice.
- A **self-consistency check**: the production detector re-run over the scrubbed slice (residual PII count).
- A small-slice **utility probe**: perplexity delta from a tiny LM trained on the scrubbed vs. unscrubbed slice.
- A partial `privacy_risk_report.yaml` with §1–§5 populated.

## Prerequisites

- Exercise 01 complete; `findings.parquet` and the §1–§3-populated report are inputs here.
- Python with: `presidio-anonymizer`, `faker`, `cryptography` (for the FPE / hashing comparison), plus a small LM runtime for the utility probe — any of `transformers` + a 125M-parameter model (e.g. `gpt2`), or `nanoGPT`-style scaffolding.
- Chapter 05 read; you should be able to recite the five primitives, their reversibility profile, and the "mask-by-default for pretraining" rule.

## Problem statement

Design and implement a redaction stage that:

1. Authors a per-entity-type primitive-choice table with rationale.
2. Implements each primitive using Presidio's Anonymizer (or a direct implementation for primitives Presidio does not ship).
3. Produces a scrubbed slice.
4. Re-runs the detector over the scrubbed slice (self-consistency) and records residual PII counts per entity type and per source.
5. Runs a small-slice utility probe (perplexity delta).
6. Emits §4–§5 of the privacy-risk report.

## Starter guidance

1. **Author the policy table.** In `redaction_policy.md`:
   - For each entity type produced by Exercise 01, record: primitive choice, operator parameters, rationale. Follow the Chapter 05 §"The per-entity-type policy table" default for pretraining; deviate only with written rationale.
   - For at least one entity type, explicitly choose a **surrogate** primitive (Faker-driven consistent-within-document). This forces the seed-management and consistency discussion.
   - For at least one entity type, explicitly choose **pass-through** with rationale.
   - Hash the file; `redaction_policy.md.sha256` is §4 `redaction.policy_sha`.

2. **Implement the primitives.**
   - **Mask**: `OperatorConfig("replace", {"new_value": "<...>"})` for most.
   - **Surrogate**: subclass `Operator` or use `OperatorConfig("replace", ...)` driven by a `faker.Faker(seed=...)` call. Ensure consistency within a document (same input → same surrogate within the same `doc_id`).
   - **Hash / HMAC**: a keyed-hash operator for comparison (even if you choose not to use it in production). Demonstrate the HMAC-with-secret approach.
   - **FPE**: an FFX / FF1 implementation (via `pyffx` or `fpe` libraries) for comparison. Demonstrate it on a credit-card-shaped entity type.
   - **Tokenisation**: a vault-based tokeniser (SQLite-backed is fine for the exercise). Record the vault file path in the policy table and discuss the vault threat model in your README.

3. **Produce the scrubbed slice.** Apply the policy to every document. Emit `scrubbed_slice.parquet` with the same schema as the input plus a `scrubbed_text` column. Hash the file.

4. **Self-consistency check.** Re-run the Exercise 01 detector over `scrubbed_text`. Count residual findings per entity type and per source. Populate §5 of the report.

5. **Utility probe.** Train a tiny LM (e.g., fine-tune `gpt2` with ~1000 steps) on two slices: scrubbed and unscrubbed-but-dedup'd. Measure perplexity on a held-out validation split. Record the delta.
   - If GPU budget is zero, use `distilgpt2` or a smaller model. The point is the methodology, not the scale.
   - Document the training hyperparameters so the probe is reproducible.
   - This delta number is a **crude** signal — document the limitations (small model, short run, specific slice).

6. **Emit the partial report.** `privacy_risk_report.yaml` with §1–§5 populated; §6–§12 placeholders.

## Deliverables

```
exercise-02/
  code/
    anonymizer_ops.py        # custom operators (surrogate, HMAC, FPE)
    build_vault.py           # tokenisation vault builder
    run_anonymize.py         # batched map-style anonymiser
    self_consistency.py      # re-run detector over scrubbed slice
    utility_probe/
      train_tiny_lm.py       # small-LM training
      eval_ppl.py            # perplexity on held-out split
      results.yaml
  data/
    scrubbed_slice.parquet
    scrubbed_slice.parquet.sha256
    operator_result_log.jsonl
    vault.sqlite             # if tokenisation primitive is used
    residual_findings.parquet
  redaction_policy.md
  redaction_policy.md.sha256
  privacy_risk_report.yaml   # §1-§5 populated
  README.md                  # includes vault threat-model discussion and probe caveats
```

## Acceptance criteria

- `redaction_policy.md` contains an entry per entity type with primitive choice, parameters, and written rationale (at least two sentences per entry).
- At least four distinct primitives are represented across the entity-type table (mask, surrogate, hash, and either FPE or tokenisation).
- `scrubbed_slice.parquet` is produced; the operator-result log is preserved.
- Self-consistency re-run numbers populate §5. Every entity type with a non-zero §3 count has a §5 count.
- Per-source residual PII is reported for at least three sources.
- The utility probe runs end-to-end and produces a perplexity delta with documented hyperparameters. The caveats (small model, specific slice) are called out in the README.
- The README discusses the **vault threat model** if tokenisation is used, and the **FPE small-domain caveat** if FPE is used (the Durak-Vaudenay 2017 result, see [`resources.md`](../resources.md)).
- The surrogate primitive is **consistent within document** (same input → same output within the same `doc_id`). This is tested.
- Running the anonymizer twice on the same inputs with the same seed produces byte-identical output (determinism check).

## Stretch goals

- **Compare three PERSON-redaction policies** (mask, surrogate-random-per-occurrence, surrogate-consistent-within-document) with three independent utility probes. Discuss which the model team should prefer and why.
- **Surrogate-generator locale swap.** Test the surrogate generator on a non-English slice (Japanese names via `Faker('ja-JP')`). Note the generator's limitations and the eval-set implications.
- **Vault access audit**. Author a mini-access-log for the tokenisation vault; demonstrate who-queried-what logging.
- **FPE small-domain demonstration**. Apply FF3 to a 4-digit-PIN-shaped entity type and discuss why the output is cryptographically weak — grounds for choosing masking instead.

## What this exercise is not

- It is not the detection exercise (Exercise 01).
- It is not the regulatory exercise (Exercise 03).
- It is not the attack-probe exercise (Exercise 04).
- It is not the full-report-assembly exercise (Exercise 05).

It is the primitive-choice exercise. Every entity type you redacted comes with a defensible answer to "why this primitive and not another?".

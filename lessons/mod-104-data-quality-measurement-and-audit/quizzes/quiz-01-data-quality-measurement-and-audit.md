# Quiz 01 — Data Quality Measurement, Representativeness, and Corpus Audit

**Module:** [mod-104 — Data Quality Measurement, Representativeness, and Corpus Audit](../README.md)
**Format:** 12 multiple-choice questions, one correct answer per question.
**Suggested time:** 25 minutes.
**Coverage:** Chapters 01–08.

These questions test whether you can *use* the module's measurement discipline — reason about what a measure *is*, pick the right tool for a stage of the pipeline, read a Dodge or Kreutzer audit's shape, pick a divergence measure for a dimension, run a data-centric ablation that is actually attributable, and emit a per-slice quality report that downstream consumers can trust — not whether you can recite the chapter wording. Answer key with references is at the bottom.

---

## Q1. What "a measure" is in this module

A reviewer points at a line in your quality report that reads `consistency: 0.994` and asks whether that one number is sufficient evidence of consistency for the slice. According to Chapter 01, which answer matches?

- **A.** Yes — a single scalar per 25012 characteristic is the whole point of the standard.
- **B.** No — a measure is a tuple of `(25012 characteristic, 5259-2 measure name, proxy, implementation, threshold, evidence SHA, cadence)`; the scalar is one field of that tuple and is useless without the other six. The report's measure record must carry all seven fields.
- **C.** No — the scalar should be reported as a percentile against an industry baseline instead.
- **D.** Yes, as long as the scalar is signed with Sigstore.

---

## Q2. The seven load-bearing characteristics

Your governance partner asks which ISO/IEC 25012 characteristics this module's objectives specifically call out as the load-bearing ones for training data. According to Chapter 01, which option lists them correctly?

- **A.** Accuracy, completeness, consistency, availability, portability, recoverability, understandability.
- **B.** Accuracy, completeness, consistency, credibility, currentness, confidentiality, traceability — and the chapter explicitly notes that the remaining eight 25012 characteristics are addressed in adjacent modules (confidentiality overlap in mod-110, traceability in mod-109, compliance in mod-101/102).
- **C.** The fifteen characteristics are all equally load-bearing; the chapter does not single out a subset.
- **D.** Only representativeness and label-noise, since those are the 5259-2 additions.

---

## Q3. When to pick Great Expectations vs Soda Core (and when to use both)

You are standing up quality checks for a new slice. The pipeline author wants rich, Python-expressed row-level checks with a web-renderable HTML report; the CI reviewer wants a short YAML file they can read in five minutes that runs as a one-liner. According to Chapter 02, which assignment matches?

- **A.** Use Great Expectations for both roles — it has an HTML report and a Python API.
- **B.** Use Soda Core for both roles — SodaCL is YAML and any Python can be done in a user-defined check.
- **C.** Use **Great Expectations** as the authoring suite the engineer maintains (rich row-level checks, custom Python expectations, HTML "data docs"); use **Soda Core** in CI and in the scheduled monitor as the YAML-only check file any reviewer can read and that runs in seconds. They are complements, not alternatives.
- **D.** Pick whichever has the better dashboard for your organisation; the two tools are otherwise interchangeable.

---

## Q4. The sidecar mapping discipline

Chapter 02 is explicit about where the mapping from (25012 characteristic, 5259-2 measure) to the implementing expectation / check lives. Which answer matches?

- **A.** As comments in the suite source — the expectation names are self-documenting.
- **B.** In a sidecar YAML file (`measure_to_expectation.yaml` for GE, `measure_to_check.yaml` for Soda) kept alongside the suite. The expectations themselves are implementation; the sidecar is the engineering contract that downstream auditors read.
- **C.** In the dataset card only — repeating the mapping per tool is duplication.
- **D.** Inferred at report-emission time by naming convention; no file is required.

---

## Q5. Why Deequ exists next to GE and Soda

You are validating a 10 TB Spark-native slice with a suite of 40 constraints. Chapter 03 names the specific Deequ property that makes it the right primitive for this workload. Which option captures it?

- **A.** Deequ's YAML syntax is terser than SodaCL's for Spark backends.
- **B.** Deequ constraints compile to a **single Spark aggregation per `VerificationSuite.run()`** — one pass over the DataFrame independent of the number of constraints — and the metric repository persists results keyed by `(dataset, tag, timestamp)`, which is the substrate for Chapter 08's quality-report time-series and for mod-111's drift detection.
- **C.** Deequ's Python API is more approachable for compliance reviewers than SodaCL.
- **D.** Deequ replaces GE and Soda entirely once the slice is Spark-native.

---

## Q6. What pandera is, and what pandera is not

A colleague suggests switching the pipeline's quality reporting from Great Expectations to pandera because "pandera does the same checks and is more Pythonic." Chapter 03 pushes back. Which reply matches the chapter?

- **A.** Correct — pandera is a drop-in replacement for GE and emits equivalent JSON reports.
- **B.** Wrong — pandera is a **dataframe contract library**, not a batch-validation-and-reporting library: it belongs at **every function boundary within your Python pipeline** as a runtime-asserted type-system-like guard that raises `SchemaError` on violation. GE is a batch-validated report; pandera is a runtime contract. You want both — pandera on every function boundary, GE / Soda / Deequ on the materialised slice — and the chapter also warns that pandera-on-pyspark is less mature than Deequ for 10 TB-class slices.
- **C.** Wrong — pandera is only for pandas and cannot interoperate with the rest of the stack.
- **D.** Correct in principle, but pandera's reports are too verbose for auditors.

---

## Q7. The three axes of a Dodge-style corpus audit

Chapter 04 is explicit that a Dodge-style audit answers three questions no row-level check can. Which option names those three axes correctly, with the C4 finding Chapter 04 highlights?

- **A.** Schema conformance, null rate, and duplicate rate. The chapter highlights the C4 schema-drift bug.
- **B.** Fetch success, robots compliance, licence tagging. The chapter highlights the C4 licensing audit.
- **C.** **What the filters strip**, **who the stripped content represents**, and **whether the kept set contaminates public benchmarks**. The chapter highlights Dodge et al.'s finding that C4's "bad words" blocklist — sourced from the LDNOOBW list — disproportionately strips documents written in African-American English (AAE), as measured with the Blodgett et al. 2016 probabilistic dialect classifier, along with documents discussing sexuality/anatomy non-sexually and documents from minority communities.
- **D.** URL distribution, text-length distribution, language distribution. The chapter highlights C4's TLD skew.

---

## Q8. The wrong control set for Axis 2

Chapter 04 §"Pitfalls" calls out a very specific mistake when profiling who the filters strip. An engineer proposes measuring the dropped set's dialect distribution and comparing it directly against the final (kept) slice's distribution to decide whether the filter is fair. What does the chapter say?

- **A.** This is the correct comparison — the kept set is the slice that ships, so it is the right reference.
- **B.** Wrong — the right comparison for Axis 2 is **dropped vs. raw population** or **dropped vs. kept**, with the raw population typically as the primary baseline. The **kept set is the one your filters already shaped**; treating it as the baseline hides the effect you are trying to measure.
- **C.** Wrong — the comparison should always be against a published external reference like Common Crawl's own TLD distribution.
- **D.** Correct, provided the comparison is done per language.

---

## Q9. Why automated LID is not the Kreutzer audit

A project manager on a multilingual slice proposes replacing the Kreutzer-style human rating step with a second-pass FastText `lid.176` or GlotLID classifier, arguing it is "the same check, just automated." According to Chapter 05, which reply matches?

- **A.** Acceptable — Kreutzer et al.'s central contribution is the rating taxonomy, not the human raters.
- **B.** Acceptable for languages above a given resource threshold where LID is known to be reliable.
- **C.** Wrong — using automated LID as the audit is **self-referential**: you would be auditing a classifier's output with the same class of classifier. Kreutzer et al.'s central finding is that **automated LID is systematically over-confident on low-resource languages** and that the only known correction is a **human spot-check** by an L1 speaker who can tell apart transliteration, code-switching, machine-translation detritus, pure boilerplate, and fossil content — none of which the LID classifier can see. Humans are the primitive.
- **D.** Wrong, but only because GlotLID and `lid.176` disagree frequently and the disagreement rate is what you should report.

---

## Q10. Picking a divergence per dimension

Chapter 06 is explicit that the divergence measure must match the dimension type and must be picked *before* you compute, with a threshold. You are building a representativeness report across language (categorical, bounded cells), source TLD (categorical, long tail), and document length (ordinal, long tail). According to Chapter 06, what is the right per-dimension picking?

- **A.** Use Jensen–Shannon divergence for all three dimensions — it is symmetric and bounded, so it travels across dimension types.
- **B.** Use Total Variation Distance for language, Jensen–Shannon for TLD, Kolmogorov–Smirnov for document length — pick the measure that matches the dimension type (TVD / JSD / chi-square for categorical; KS / Wasserstein for ordinal/continuous); record which you used, its formula source, and the threshold you committed to *before* computing. Post-hoc metric shopping is bias.
- **C.** Use KL divergence for every dimension — it is the most information-theoretically principled.
- **D.** Compute all four (TVD, KL, JSD, KS) for every dimension and report the smallest value.

---

## Q11. What a DataComp-LM ablation actually attributes

You are running a leave-one-out ablation to prove that your perplexity filter earns its place. Your teammate suggests also swapping the model to one with a better eval score on MMLU "so the numbers look more publishable." According to Chapter 07, which reply matches?

- **A.** Acceptable — the model change and the data change can be reported as two separate ablations in the same table.
- **B.** Wrong — the **core discipline is: fix the model, vary the data, measure the eval delta**. Everything that is not the data is pinned (model code and git SHA, architecture, tokenizer, optimiser, LR schedule, seeds, training tokens, eval suite revision, compute environment). Change the model and the attribution breaks — the eval delta no longer reads "this filter is worth +Xpp"; it reads "something changed." Also: the slice must be **decontaminated against the eval suite before the ablation runs**, or every eval delta is noise around data leakage.
- **C.** Wrong, but only because MMLU has known contamination issues that make the comparison meaningless.
- **D.** Acceptable if you run both variants under one random seed to save compute.

---

## Q12. The per-slice quality report is *emitted*, not written

Chapter 08 is explicit about how `per_slice_quality.yaml` is produced. A new engineer offers to hand-maintain the file after each slice build "because YAML is easy to edit and the pipeline output is messy." Which reply matches the chapter?

- **A.** Acceptable — the YAML is a summary, so hand-editing is fine as long as the summary matches the evidence.
- **B.** Acceptable for the first few releases while the pipeline matures; automate later.
- **C.** Wrong — the report is **emitted, not written**: a single script discovers all evidence blobs the pipeline produced (GE / Soda / Deequ / pandera outputs, Dodge audit, Kreutzer audit, `distribution.parquet`, ablation JSON), verifies SHA-256s, composes the YAML from the schema using `measure_to_*.yaml` sidecars, renders the Markdown summary, and signs the bundle with **Sigstore keyless signing** against a short-lived OIDC identity (the signed bundle is witnessed in the Rekor transparency log). Any field computed by hand is a bug; any sign-off recorded without a corresponding passing gate is a governance breach. Canonical YAML (sorted keys, fixed precision, timestamps in one well-defined field) is what makes the file diffable for the mod-111 monitor.
- **D.** Wrong — the file must be rewritten by the engineer per slice to carry their accountability.

---

## Answer key

**Q1 — B.** Chapter 01 §"A measure is a tuple, not a number" is explicit: a measure is `(characteristic, measure name, proxy, implementation, threshold, evidence SHA, cadence)`. The chapter walks through the seven fields in the example YAML record and says "Without this shape, 'we measured quality' means nothing to the next engineer." A reports the scalar; C, D are irrelevant to the definition.

**Q2 — B.** Chapter 01 §"ISO/IEC 25012 — the fifteen characteristics" names exactly the seven **accuracy, completeness, consistency, credibility, currentness, confidentiality, traceability** as "load-bearing … for training data" and says the others "matter but are addressed in adjacent modules (confidentiality in mod-110, traceability in mod-109, compliance in mod-101 and mod-102)." A lists system-dependent characteristics; C contradicts the chapter's deliberate narrowing; D confuses 25012 characteristics with 5259-2 measures.

**Q3 — C.** Chapter 02 §"When to pick which (and when to use both)" says exactly: "During pipeline authoring, use GE as the detailed suite the engineer maintains. Custom expectations are where you encode your project-specific proxies. In CI and in the monitor, use Soda as the YAML-only check file that any reviewer can read and that runs in seconds. Chapter 08 and mod-111 have Soda in front." The comparison table in the same section pins the ergonomics.

**Q4 — B.** Chapter 02 §"Mapping GE expectations to the characteristic spec" says exactly: "Record this mapping as a file in the slice (`measure_to_expectation.yaml`), not as a comment in the suite. Downstream auditors read the file; nobody reads your suite source." The same pattern is repeated for Soda in §"Mapping SodaCL checks to the characteristic spec". A is what the chapter rejects; C and D undercut auditability.

**Q5 — B.** Chapter 03 §"What Deequ gives you that GE / Soda don't" names "Spark-native aggregation. Constraints compile to a single `agg(...)` over the DataFrame — one pass per `VerificationSuite.run()`, independent of the number of constraints" and "the metric repository. Historical metric values are persisted and keyed by tags (`slice_id`, `shard`, `date`). This is the substrate for Chapter 08's quality-report time-series and for mod-111's drift detection." A, C are false claims about syntax/ergonomics; D is the anti-pattern — the chapter is explicit that the four tools co-exist, with pandera at code boundaries, Deequ at the Spark boundary, GE as the authored row-level suite, and Soda as the CI/monitor gate.

**Q6 — B.** Chapter 03 §"pandera is a contract library, not a reporting library" and §"When to use which — and the handoff pattern" say it plainly: pandera belongs between pipeline stages as a runtime-asserted contract; GE is the batch-validated report. The section also notes "pandera on pyspark exists but is less mature than Deequ. A 10 TB slice should go through Deequ constraints, not a pandera `validate`." The chapter's recommended shape runs all four tools together. A, C, D misread pandera's role.

**Q7 — C.** Chapter 04 §"Why this chapter exists" and §"The audit's three axes" name the three axes verbatim: *what filters strip*, *who the stripped content represents*, *does the kept set contain benchmark test inputs*. §"Axis 1" and §"Axis 2" together call out Dodge et al.'s finding on C4's LDNOOBW-based "bad words" blocklist: it removes "content well beyond its stated intent — documents discussing sexuality or anatomy non-sexually, documents from minority communities, documents containing medical or legal terminology" and disproportionately drops AAE. §"Dialect analysis" names Blodgett, Green, O'Connor (EMNLP 2016) as the classifier reference.

**Q8 — B.** Chapter 04 §"Pitfalls" says exactly: "Using the kept set as the 'control'. The right comparison for Axis 2 is *dropped vs. raw population* or *dropped vs. kept*. The *kept* set is the one your filters already shaped; treating it as the baseline hides the effect." The chapter's preferred framing is comparing against the raw population; dropped-vs-kept is acceptable as a secondary comparison. A, C, D each miss this.

**Q9 — C.** Chapter 05 §"Pitfalls" lists "Using automated LID as the audit. Self-referential: you are auditing a classifier's output with the same classifier. Humans are the primitive." §"What the audit catches that nothing else does" enumerates the failure modes only a human speaker can detect — transliteration, code-switching, machine-translation detritus, technical/non-prose, fossil content — and explicitly says "automated LID is systematically over-confident on low-resource languages and … the only known correction is a human spot-check." A and B misread the paper; D is a diagnostic but not the audit.

**Q10 — B.** Chapter 06 §"The two distributions you care about" and §"Pitfalls" together: pick the divergence that matches the dimension type — "TVD, KL, JSD, chi-square" for categorical; "KS, Earth-Mover's Distance (Wasserstein)" for ordinal/continuous — and "pick the metric before you compute, with a threshold. Post-hoc metric shopping is bias." Exercise 04's starter guidance uses the exact assignment in option B (TVD for language, JSD for TLD, KS for length). A overgeneralises; C breaks on the ordinal dimension; D is the post-hoc shopping the chapter forbids.

**Q11 — B.** Chapter 07 §"The DataComp(-LM) methodology in one picture" and §"What to hold fixed" list the pinned variables explicitly — model code + git SHA, architecture, tokenizer, optimiser (AdamW with fixed β's and ε), LR schedule, weight decay, dropout, precision, training tokens or steps, all four seeds (master, tokenization, sampling, init), eval suite SHA, compute environment. §"The eval suite" says "Decontaminate first. Every slice that flows into training must have been decontaminated against every eval in the suite. Otherwise the eval delta is noise around data leakage." §"Pitfalls" also names "Running eval on a contaminated slice" as invalidating the whole ablation. A, C, D each miss the pinning discipline.

**Q12 — C.** Chapter 08 §"Automation discipline" says exactly: the report is "emitted, not written" by a single script that discovers evidence blobs, verifies SHA-256s, composes the YAML, renders the Markdown, and signs with Sigstore. "Any field in the YAML computed by hand is a bug. Any sign-off recorded without a corresponding passing gate is a governance breach." §"Signing and chain of custody" names Sigstore keyless signing against a short-lived OIDC identity and Rekor transparency-log witnessing. §"Pitfalls" calls out "Making the report un-diff-able" and prescribes canonical YAML for the mod-111 monitor's diff to stay signal. A, B, D undercut automation, signing, or sign-off integrity.

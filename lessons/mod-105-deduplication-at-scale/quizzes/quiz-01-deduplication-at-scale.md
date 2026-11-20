# Quiz 01 — Deduplication at Trillion-Token Scale

**Module:** [mod-105 — Deduplication at Trillion-Token Scale](../README.md)
**Format:** 12 multiple-choice questions, one correct answer per question.
**Suggested time:** 25 minutes.
**Coverage:** Chapters 01–08.

These questions test whether you can *use* the module's operators — defend the dedup step as a core quality control, pick the right hash and set-data-structure for a scale, reason about the MinHash + LSH S-curve, pick a granularity for a threat model, pick a production toolchain for a deployment, keep the pipeline-side / detection split with `ai-risk-engineer` clean, and emit a dedup report that downstream consumers can rely on — not whether you can recite chapter wording. Answer key with references is at the bottom.

---

## Q1. Why dedup is not "operational hygiene"

A programme lead asks why a week of pipeline work on deduplication is worth the compute budget instead of shipping the slice and moving on. Chapter 01 says there are *three* public findings you must be able to cite, in a specific shape. Which option matches?

- **A.** Dedup is a compliance requirement under ISO/IEC 42001 Annex A.7; without it the slice cannot be released.
- **B.** **Lee et al. 2021** (perplexity and sample efficiency improve at matched compute), **Kandpal et al. 2022** (memorisation and extraction-attack success rates drop by roughly an order of magnitude after dedup), and the **SlimPajama 2023** report (RedPajama 1.21T → SlimPajama 627B tokens — roughly half the corpus was duplicate-shaped). Together these pin dedup as a quality control on the ISO/IEC 25012 *consistency* axis *and* a privacy control on the *confidentiality* axis simultaneously.
- **C.** Dedup saves training FLOPs; the SlimPajama report is the only citation needed.
- **D.** It is required by the OWASP LLM Top 10 ML02 item; no empirical justification is needed.

---

## Q2. xxh128 vs SHA-256 — where each belongs

Your pipeline needs a 128-bit document fingerprint for a 10⁹-document slice and a slice-level hash that will be referenced in a signed audit manifest. Chapter 02 pins the operational rule for which hash plays which role. Which option matches?

- **A.** SHA-256 per document for correctness; xxh128 as the slice-level audit hash for speed.
- **B.** xxh64 per document and SHA-256 per slice — xxh64 is fast and its collision probability is "effectively zero" at 10⁹.
- **C.** **xxh128 (or xxh3_128)** as the bulk per-document fingerprint — ~10–30 GB/s per core and collision probability `~2 × 10⁻²¹` at `N = 10⁹`, effectively zero; **SHA-256** as the slice-level audit hash so the slice artifact is signable and audit-grade. xxh64's birthday-bound collision probability at `N = 10⁹` is ~3% — not acceptable as a primary key.
- **D.** BLAKE3 for both roles because it is faster than SHA-256 with equivalent collision properties; the chapter recommends switching away from xxh128.

---

## Q3. What "exact" actually means

A new engineer runs raw-byte SHA-256 over documents and reports a 0.3% exact-duplicate rate, claiming that is "the" exact-dedup number. Chapter 02 says that number is almost certainly *under*-counted. Which option matches the chapter?

- **A.** The number is correct; raw-byte SHA-256 is the definition of "exact duplicate".
- **B.** The number is wrong because xxh128 would find more duplicates than SHA-256.
- **C.** "Exact" is a **policy**, not a given: common web pipelines produce many logically-identical-but-byte-different pairs (CDN-injected headers, Unicode NFC-vs-decomposed text, trailing whitespace, HTML-entity encoding differences). A fingerprint over a **documented canonical form** (Unicode normalisation form, whitespace rule, case rule, encoding) is what makes the exact-duplicate rate reproducible. Chapter 02 also notes the fingerprint is only "exact" relative to that policy — change the policy and you must re-fingerprint the full corpus.
- **D.** SHA-256 should be replaced with BLAKE3 for exact-dedup accuracy.

---

## Q4. Picking the set data structure by N

You have a 10¹⁰-document corpus and are deciding how to hold the "already seen fingerprints" set during exact-dedup. Two candidate paths are a single Python `set` in RAM on a very large box, and a Bloom filter at `ε = 10⁻⁴`. Chapter 02's scaling guide is explicit. Which option matches?

- **A.** A Python `set` is the correct default — CPython's `set` scales to 10¹⁰ on a 1 TB box.
- **B.** Neither: use **sharded hash partitions** (Spark / Ray / explicit workers) or a **Bloom filter** at `ε = 10⁻⁴` (~24 GB for 10¹⁰ elements). The Bloom filter is the right choice **only when a bounded false-positive rate is acceptable** — for most pretraining corpora, losing ~1 in 10,000 documents is a vanishingly small quality loss; for privacy / audit work it is not acceptable because a Bloom false-positive drops a unique PII-bearing document as a phantom duplicate.
- **C.** A Bloom filter is the single correct answer regardless of downstream; it is cheapest.
- **D.** An on-disk RocksDB is the only option — in-memory structures do not scale past 10⁹.

---

## Q5. MinHash signature length `K` and what it buys

A reviewer challenges your choice of `num_perm = 128` on a near-dup pipeline — "why not 32, it would be 4× cheaper?" Chapter 03 pins the bias / variance trade-off. Which reply matches?

- **A.** `K = 32` is fine; variance is an academic concern that does not affect production thresholds.
- **B.** MinHash estimates Jaccard with variance roughly `J(1 - J) / K`. At `K = 128` and `J = 0.8`, the standard deviation is `√(0.16 / 128) ≈ 0.035` — small enough that a 0.8 Jaccard threshold is stable. `K = 32` gives std dev `~0.071`, which blurs your threshold: documents with true Jaccard 0.8 end up with estimated Jaccard anywhere in roughly `0.65–0.95`, and the pipeline's recall / precision collapses. `K = 128` is the field default to internalise; `K = 256` is the tighter-threshold multilingual choice.
- **C.** Higher `K` lowers Jaccard estimates systematically; `K = 32` is actually the unbiased choice.
- **D.** `K` controls Hamming distance, not Jaccard, and is not relevant to a MinHash pipeline.

---

## Q6. When SimHash is the right call (and when it is not)

You are building a Jaccard-threshold document-level dedup for an English pretraining slice. A teammate argues SimHash is better because its 64-bit fingerprint is smaller than a 128-permutation MinHash signature. Chapter 03 pushes back. Which reply matches?

- **A.** Correct — SimHash is strictly better; the field has moved on from MinHash.
- **B.** Wrong — **MinHash + LSH at Jaccard threshold 0.7–0.9** is the field default for text pretraining dedup (Lee 2021, SlimPajama, FineWeb, Dolma, data-juicer, NeMo Curator all use it). SimHash estimates feature-weighted cosine similarity in Hamming distance — if your downstream threshold is expressed in **Jaccard** (because the field convention is Jaccard), SimHash in that regime is a correctness / portability liability. Pick SimHash when the corpus is web pages with TF-IDF-like semantics, fingerprint storage is a hard bottleneck, or you are inter-operating with a classical web-crawl stack that emits SimHash already — otherwise default to MinHash.
- **C.** Correct — Charikar 2002 supersedes Broder 1997 for all dedup workloads.
- **D.** Wrong because SimHash's 64-bit fingerprint has higher collision probability than a MinHash signature.

---

## Q7. The LSH S-curve and the two knobs

Chapter 04 gives the S-curve `P(flagged | J) = 1 − (1 − J^r)^b` with soft threshold `s* ≈ (1/b)^(1/r)`, subject to `K = b × r`. A reviewer asks what happens to recall and precision when you increase `b` (bands) while holding `K` constant. Which option matches?

- **A.** Increasing `b` raises the soft threshold `s*`, lowering recall and raising precision.
- **B.** **Increasing `b` lowers the soft threshold `s*`**, which **raises recall** (more candidate pairs flagged below your target threshold) at the cost of **more candidate pairs to verify** (more false positives to reject). Decreasing `b` (equivalently, increasing `r`) does the opposite. The product constraint `K = b × r` means you tune two independent knobs, and the right choice is set against a *labelled sample* calibration before shipping.
- **C.** Changing `b` has no effect on the soft threshold — only `K` matters.
- **D.** Increasing `b` lowers `s*` and makes verification cheaper because the signature is smaller.

---

## Q8. The mandatory verification step

A pipeline emits all pairs that collide in at least one LSH band as duplicates and writes them straight into the clusters file. Chapter 04 §"Common misconfigurations" calls this out specifically. Which option matches the chapter?

- **A.** This is correct — the S-curve already handles false positives; no further step is needed.
- **B.** Wrong — LSH band collision produces **candidate pairs**, which still include S-curve noise (e.g. ~6% false-positive probability at `J = 0.5` for a `(b = 16, r = 8)` configuration calibrated near `J = 0.7`). Every candidate pair must be **verified with the MinHash Jaccard estimator** (not the raw shingle intersection — that is what the threshold was calibrated against) at the operational threshold before being promoted to the clusters file. Verification is cheap; skipping it is one of the three most common MinHash-LSH misconfigurations.
- **C.** Wrong — pairs should be verified by exact shingle-set Jaccard, which is cheaper than the MinHash estimator.
- **D.** Correct in principle, but only if the pipeline is running at document granularity.

---

## Q9. Granularity for a Common Crawl slice

An engineer runs document-level MinHash + LSH at Jaccard 0.8 on a 1T-token Common Crawl slice and asks whether that is "enough". Chapter 05 is explicit about what document-only dedup leaves on the table for web corpora. Which option matches?

- **A.** Yes — SlimPajama demonstrated that document-level at Jaccard 0.8 is sufficient for every web corpus.
- **B.** No — document-only dedup on a web corpus **leaves per-site boilerplate memorised** (cookie banners, nav crumbs, author bios, standard disclaimers, repeated introductions). The correct shape for web / Common Crawl slices is document-level + **paragraph-level** on top, with a min-length gate (e.g. ≥ 100 chars), at a tighter Jaccard threshold (e.g. 0.85) because paragraphs are shorter and the Jaccard estimator is noisier. `dolma` includes paragraph-level deduplication as its mid-tier recipe for exactly this reason.
- **C.** No — sentence-level **blanket** dedup is required for every web slice.
- **D.** Yes, as long as the toolchain is NeMo Curator; the GPU acceleration changes the trade-off.

---

## Q10. Picking a production toolchain

You are choosing between `dolma`, `NeMo Curator`, and `data-juicer` for a 10¹⁰-document English-plus-Chinese pretraining slice. The team has GPU cluster capacity (A100-class), and you want fuzzy dedup and exact dedup together with first-class integration with PII and benchmark-contamination operators. Chapter 06's three selection heuristics say which?

- **A.** `dolma` — it is the reference implementation behind the Dolma corpus, so provenance is strongest.
- **B.** **`NeMo Curator`** — above ~10⁹ documents the GPU-accelerated MinHash signature path dominates economics (an order of magnitude faster per-document than CPU `datasketch`); it ships fuzzy + exact dedup from the same operator library with first-class PII and benchmark-decontamination integration; and it fits into the NeMo training stack if the team is already using it. `dolma` would be the CPU-cluster choice with the strongest public-reference-implementation story; `data-juicer` would be the choice for multimodal dedup or a broad operator catalogue including SimHash.
- **C.** `data-juicer` — Ray + 100+ operators is always the right choice at 10¹⁰.
- **D.** Spark + `datasketch` — rolling your own is always cheapest.

---

## Q11. The pipeline-side vs detection-methodology split

A security partner asks whether your dedup pipeline will "detect Nightshade-poisoned images" on a multimodal slice. Chapter 07 is explicit about the division of labour between `ai-data-engineer` and `ai-risk-engineer`. Which reply matches?

- **A.** Yes — the dedup pipeline owns the full Nightshade detection methodology; it is a pipeline concern.
- **B.** No — `ai-risk-engineer` (peer, level 25) owns **detection methodology** (chooses and runs the detectors, tunes thresholds, interprets ambiguous results). `ai-data-engineer` owns **pipeline-side quarantine responsibility**: emit the signals (cluster-size anomaly, per-source removal-rate spikes, zero-day clusters, and — on multimodal slices — a perturbation-signature column of pHash / DCT / high-frequency-noise statistics as a *feature set*, not a detection verdict), carry out the quarantine policy, produce the audit artifacts. Both deliverables are mandatory. The pipeline's job is to route evidence to the right detector, not to make the "is this Nightshade" call alone.
- **C.** Yes — SAND Lab ships an official Nightshade detector; the pipeline just has to call it.
- **D.** No — Nightshade detection is a GRC concern and does not belong to either role.

---

## Q12. The dedup report is *emitted*, not written

A new engineer offers to hand-maintain `dedup_report.yaml` after each slice build because "YAML is easy to edit and the pipeline output is messy," and proposes marking the §5 (adversarial-perturbation review) section "not applicable" on a text-only slice to avoid chasing down the `ai-risk-engineer` sign-off. Chapter 08 rejects both moves. Which reply matches?

- **A.** Hand-maintenance is acceptable for the first few releases; the "not applicable" verdict is fine for any text slice.
- **B.** Hand-maintenance is fine but the §5 block must be filled by `ai-risk-engineer` regardless.
- **C.** **Both moves are wrong.** The report is **emitted by the pipeline**, not hand-authored: every number comes from a specific pipeline step's output; hand-authored summaries drift from reality within one re-run. The six load-bearing sections (identity / provenance, fingerprint policy, removal statistics with **per-source disaggregation**, memorisation-risk delta, adversarial-perturbation review, evidence SHAs) each plug into a specific downstream handoff (`mod-109` lineage, `mod-110` privacy-risk, `mod-112` dataset card, `staff-ml-engineer` mixture). The `ai_risk_engineer_review` block is **mandatory**: "not applicable" is a valid *verdict* only if explicitly signed off by the peer; it is a hole if silently omitted. The report itself is signed (Sigstore or equivalent).
- **D.** Correct on both — the dedup report is an informal summary and the §5 block is optional for small slices.

---

## Answer key

**Q1 — B.** Chapter 01 §"The three claims you must be able to defend" pins exactly the three citations and the shape in which to cite them: Lee et al. 2021 (perplexity / sample efficiency at matched compute, with the C4 / RealNews / LM1B / Wiki-40B NearDup and ExactSubstr percentages), Kandpal et al. 2022 (memorisation and extraction-attack success rates drop by roughly an order of magnitude; memorisation scales log-linearly with duplicate count), and the SlimPajama 2023 report (1.21T → 627B tokens — roughly half duplicate-shaped). §"The quality-control framing in ISO terms" ties them to the ISO/IEC 25012 *consistency* and *confidentiality* characteristics simultaneously. A, C, and D each miss the three-citation framing the role is accountable for.

**Q2 — C.** Chapter 02 §"Which hash — and why they are not interchangeable" states the operational rule verbatim: "**xxh128 (or xxh3_128)** — bulk exact-dedup fingerprint column. Fast, 128-bit collision-safe for current corpus sizes. **SHA-256** — one field at the slice artifact level, so that the slice is signable and audit-grade. Not applied per-document." The chapter also pins xxh64's ~3% collision probability at `n = 10⁹` as unacceptable as a primary key, and notes BLAKE3 is a faster cryptographic alternative but has not become the fingerprint default in the main pretraining pipelines. A inverts the roles; B picks the wrong flavour of xxhash; D misreads the BLAKE3 note.

**Q3 — C.** Chapter 02 §"What 'exact' actually means — the normalisation decision" lists the normalisation layers (Unicode form, whitespace policy, case policy, encoding) and the common web pathologies that make byte-level hashing miss "logically identical but byte-different" pairs. §"Exact dedup is not idempotent across normalisation changes" is explicit: "the fingerprint column is a different column" if you change normalisation, and "there is no shortcut" for re-fingerprinting. A denies the policy framing; B compares hashes but misses the normalisation point; D is unrelated to the question.

**Q4 — B.** Chapter 02 §"Scaling guide summary" assigns `10⁸ – 10¹⁰` to hash-partitioned shards (Spark / Ray / explicit workers) or RocksDB, and `≥ 10¹⁰` to hash-partitioned shards in distributed memory, with Bloom filter as an option *when ε is acceptable*. §"Bloom filter" also calls out the privacy-work caveat: a Bloom false positive effectively drops a unique PII-bearing document as a phantom duplicate, which is not acceptable for audit precision. A, C, D each pick one tool without the policy framing.

**Q5 — B.** Chapter 03 §"Signature length `K` — the bias / variance trade-off" gives the formula verbatim: variance roughly `J(1 − J) / K`; `K = 128` standard pretraining choice with std dev `√(0.16 / 128) ≈ 0.035` at `J = 0.8`; `K = 256` for multilingual tighter thresholds; `K = 64` for quick audits; `K = 32 and below — do not`. A denies the trade-off; C inverts the direction of the bias; D confuses MinHash's Jaccard output with SimHash's Hamming output.

**Q6 — B.** Chapter 03 §"When to pick which" pins the default: "every public reference (Lee 2021, SlimPajama, FineWeb, Dolma, data-juicer, NeMo Curator) uses this" family — MinHash + LSH at Jaccard threshold 0.7–0.9. §"SimHash — Charikar 2002" and §"What SimHash does well and badly" note that SimHash is less flexible when the downstream threshold is expressed in Jaccard. The chapter lists the three cases where SimHash is the right call (feature-weighted web similarity, tight fingerprint-storage budget, interoperating with a classical web-crawl stack). A, C, D each miss the field default or misread the trade-off.

**Q7 — B.** Chapter 04 §"The three knobs, concretely" names the directions in a table: increasing `b` lowers the soft threshold `s*` (more candidate pairs, higher recall, more FPs to verify); increasing `r` raises `s*` (fewer pairs, lower recall, fewer FPs). §"Calibrating against a labelled sample" is the operational discipline — never ship a threshold you have not plotted against a labelled sample. A inverts the direction; C denies the bands knob entirely; D confuses verification cost with signature cost.

**Q8 — B.** Chapter 04 §"Common misconfigurations" names skipping the verification step as one of the three most common mistakes. §"The LSH-with-banding recipe" is explicit: "Verify candidate pairs with the full signature to reject band-collision accidents"; §"The `datasketch` API, explicit" states "Verify candidate pairs with the MinHash `jaccard` method, not the raw shingle intersection. The MinHash estimator is what the threshold was calibrated against." The S-curve-noise example at `(b = 16, r = 8)` shows the ~6% candidate-pair probability at `J = 0.5` the verification step rejects. A denies the misconfig; C substitutes the wrong verifier; D attempts to limit the rule to one granularity, which the chapter does not.

**Q9 — B.** Chapter 05 §"Document granularity" says document-only dedup "fails on: per-site boilerplate (standard disclaimers, cookie banners, author bios, nav crumbs), repeated introductions, templated product pages". §"When paragraph dedup pays off" names "web corpora with heavy per-site templating (most Common Crawl slices, product-page-heavy corpora, Q&A forums ...); `dolma` includes paragraph-level deduplication as its mid-tier recipe." The granularity decision tree in the same chapter adds paragraph-level for any web / CC-derived slice. §"Scenario A" in §"Examples of granularity shaping the result" treats exactly this question. A overgeneralises SlimPajama; C is the blanket-sentence anti-pattern; D misreads the toolchain decision.

**Q10 — B.** Chapter 06 §"Selecting a toolchain — three heuristics" says verbatim: "If you have GPUs and your corpus is 10⁹+, start with NeMo Curator. The economics change decisively above ~10⁹ documents." §"When to pick NeMo Curator" names fuzzy dedup + exact dedup from the same operator library, first-class PII and contamination integration, and the NeMo stack fit. §"When to pick `dolma`" is the CPU-cluster / public-reference-implementation heuristic; §"When to pick data-juicer" is the multimodal-or-broad-catalogue heuristic. A, C, D each pick one tool without the decision axes.

**Q11 — B.** Chapter 07 §"The division of labour" is explicit: "`ai-risk-engineer` (peer, level 25) owns detection methodology — chooses and runs the detectors, tunes thresholds, interprets ambiguous results. You (`ai-data-engineer`) own pipeline-side quarantine responsibility — build the hooks, emit the signals, carry out the quarantine policy, produce auditable artifacts." §"Pipeline-side detection heuristics you can run today" lists the four heuristics (cluster-size anomaly, perturbation-signature column, source-level removal-rate spikes, zero-day clusters) as *evidence sources* for the real detector, not a substitute. §"Nightshade (Shan et al. 2024)" also notes SAND Lab publishes the perturbation tool but not an official detector. A, C, D each mis-assign ownership.

**Q12 — C.** Chapter 08 §"The report is emitted, not written" is explicit: "The report should be emitted by the pipeline, not hand-authored. Every number in it should come from a specific pipeline step's output. Hand-authored summaries drift from reality within one re-run; emitted summaries cannot." §"Failure modes in the report" calls out "Missing `ai-risk-engineer` sign-off" with the exact clarification: "The report declares the adversarial-review section 'not applicable' because no detector was run. 'Not applicable' is a valid verdict *if* explicitly signed off by the peer; it is a hole *if* silently omitted." §"Signing the report" prescribes Sigstore or an equivalent keyless signer. A, B, D each undercut automation, disaggregation, or sign-off integrity.

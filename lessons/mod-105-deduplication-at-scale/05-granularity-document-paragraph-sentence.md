# Chapter 05 — Granularity: Document vs Paragraph vs Sentence Deduplication

## Why this chapter exists

Chapters 02–04 pinned the *algorithm* (exact + MinHash + LSH). This chapter pins the *unit* that algorithm runs over. The same pipeline at document granularity, paragraph granularity, and sentence granularity produces three *different quality controls* — not three settings on the same knob. Picking wrong is a surprisingly common and surprisingly costly mistake:

- **Document-only dedup** on a web corpus leaves per-site boilerplate memorised, which Kandpal 2022 shows is exactly the shape that creates extraction-attack risk.
- **Sentence-level dedup** on prose destroys common legal / idiomatic phrases, which hurts downstream model fluency on precisely those registers.
- **Paragraph-level dedup** is the right balance for *some* corpora and wasted compute for others.

The right answer per slice depends on what the duplicates in that slice *look like* and what downstream harm you are trying to prevent. This chapter gives you the vocabulary and the rules of thumb.

## Three granularities, three threat models

### Document granularity

- **Unit:** one hash / signature per document.
- **Catches:** full-page duplicates, mirrored sites, syndicated content (newswire re-posts), AI-generated copies of the same web page.
- **Fails on:** per-site boilerplate (standard disclaimers, cookie banners, author bios, nav crumbs), repeated introductions, templated product pages whose unique content is small.
- **Cost:** 1× — baseline. One signature per document.
- **Memorisation risk left on the table:** the boilerplate-memorisation surface. Modern LLMs are known to verbatim-emit cookie banners from popular sites — Kandpal 2022 and Carlini et al. 2022 cover the mechanism.

This is what Lee et al. 2021 `NearDup`, SlimPajama, FineWeb, and `dolma` run **by default**. It is the floor. If a slice gets any dedup at all, it gets document-level.

### Paragraph granularity

- **Unit:** one hash / signature per paragraph after splitting on `\n\n` / HTML `<p>` boundaries.
- **Catches:** per-site boilerplate that appears in many otherwise-distinct documents, re-posted paragraphs in forums, standard cookie banners, repeated "about this article" boxes.
- **Fails on:** very short paragraphs become noisy (same problem as very short documents in Chapter 03). Short-paragraph floors are common pre-filters.
- **Cost:** `~5–15×` document-level, depending on paragraphs-per-document ratio. If a corpus averages 10 paragraphs/doc, you build 10× the signatures.
- **What survives:** the parent document, with duplicate paragraphs excised OR the whole document kept and the duplicated paragraph annotated. Chapter 08 §"Removal vs annotation" treats this policy.

**When paragraph dedup pays off:** web corpora with heavy per-site templating (most Common Crawl slices, product-page-heavy corpora, Q&A forums with repeated "standard reply" patterns). `dolma` includes paragraph-level deduplication as its mid-tier recipe.

### Sentence granularity

- **Unit:** one hash per sentence after splitting with a sentence tokeniser (e.g. `spacy`, `pysbd`, `nltk.punkt`). Normalisation (Chapter 02) is applied per-sentence before hashing.
- **Catches:** benchmark-contamination-style overlaps ("The quick brown fox jumps over the lazy dog" from a specific benchmark), repeated proper-noun-heavy sentences (Wikipedia-style opening sentences replicated across mirrors), near-verbatim legal clauses that appear in thousands of TOS pages.
- **Fails on:** short / high-frequency idioms. "Thanks for your time" at sentence granularity would be removed from every polite email. Common legal boilerplate ("THIS AGREEMENT IS GOVERNED BY…") becomes a single survivor with thousands of near-copies deleted.
- **Cost:** `20–100×` document-level depending on sentences-per-document.
- **Threat model:** benchmark contamination and verbatim memorisation of specific identifier-bearing sentences (e.g. "User john.doe@example.com reported…"). Chapter 07 picks up on sentence-level fingerprinting in the poisoning context.

**When sentence dedup pays off:** benchmark-decontamination work (expanded in [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md)); any slice where the privacy-risk story depends on a per-sentence PII appearance count ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)).

**When sentence dedup is actively wrong:** fluency-sensitive registers where the *frequency* of common phrases is signal (polite closings, boilerplate legal text, common conversational patterns). Blanket sentence dedup on conversation data destroys the "common pleasantries" distribution.

## The granularity decision tree

```
                    ┌───────────────────────────────┐
                    │ Is this a pretraining slice?  │
                    └─────────┬───────────────────┘
                              │ Yes → everyone gets document-level.
                              │       That is non-negotiable.
                              ▼
                    ┌───────────────────────────────┐
                    │ Is this a web/CC-derived slice?│
                    └─────────┬───────────────────┘
                              │ Yes → add paragraph-level too, with a
                              │       min-length filter (e.g. ≥ 100 chars).
                              ▼
                    ┌───────────────────────────────────────────┐
                    │ Is the slice earmarked for a model that    │
                    │ will be eval'd on canonical benchmarks     │
                    │ or needs a defensible memorisation story? │
                    └─────────┬────────────────────────────────┘
                              │ Yes → add sentence-level dedup
                              │       targeted at benchmark n-grams
                              │       (see mod-106). Full sentence
                              │       dedup is rarely needed.
                              ▼
                    ┌───────────────────────────────────────────┐
                    │ Is PII risk the dominant concern?          │
                    │ (healthcare, financial, chat transcripts)  │
                    └─────────┬────────────────────────────────┘
                              │ Yes → add sentence-level dedup
                              │       gated on PII-detector hits
                              │       (coordinate with mod-110).
                              ▼
                       Stop. Record the choices in the slice artifact.
```

Rule of thumb: **document-level is default**, **paragraph-level is default for web**, **sentence-level is targeted, not blanket**. Every granularity costs more; each must earn its place.

## Interactions with the near-dup threshold

Jaccard thresholds behave differently at different granularities, because the shingle-set size behaves differently.

| Granularity | Typical shingle width | Typical threshold |
|---|---|---|
| Document | `k = 5` word shingles | `0.7–0.9` |
| Paragraph | `k = 5` word shingles with a min-length gate | `0.8–0.9` |
| Sentence | `k = 3` word shingles, or exact string match | `0.9–1.0`, or exact |

At sentence granularity, Jaccard is often a bad fit — the shingle set per sentence is too small to be a stable estimator. Many pipelines degrade sentence dedup to **exact matching** (Chapter 02's primitive, run over sentences) rather than MinHash — and it is deliberate.

## What goes into the dedup report at each granularity

Chapter 08 collects these for the final report. Per granularity you emit at minimum:

- `granularity: document | paragraph | sentence`
- `shingle_width: k` (if MinHash) or `exact: true`
- `threshold: J` (if MinHash)
- `pre_count` and `post_count` of units
- `clusters_formed`
- `removal_rate_overall` and `removal_rate_by_source`
- Pre/post token count for the slice (the headline SlimPajama-style number)
- Memorisation-risk note: Kandpal-style frequency-to-memorisation curve applied at this granularity (Chapter 08 §"Memorisation-risk delta")

If a slice runs dedup at two granularities, the artifact has one block per granularity. Do not merge them.

## Removal versus annotation

Two policy choices for *what* to do once a duplicate is identified:

1. **Removal.** Drop the duplicate. The downstream slice has fewer tokens; the one surviving document carries the content forward.
2. **Annotation.** Keep the duplicate, mark it with a `duplicate_cluster_id`, and let the training sampler decide (e.g. down-weight documents with large cluster sizes rather than deleting them).

Removal is cheaper and is what every public pipeline listed in this module does. Annotation allows a training-time mixture strategy to use duplicate-count as a signal, which is sometimes valuable for low-resource domains where even imperfect duplicates are rare high-quality examples.

**In this module we default to removal.** Annotation is a mixture-strategy concern beyond this role.

## A granularity-aware pipeline sketch

```
1. normalise + fingerprint docs             (Chapter 02)
2. exact-dedup docs                          (Chapter 02)
3. minhash + LSH at document granularity     (Chapter 04, J=0.8, K=128, b=16, r=8)
4. for web slices:
     split surviving docs into paragraphs
     filter paragraphs with length < threshold
     exact-dedup paragraphs                  (Chapter 02)
     minhash + LSH at paragraph granularity  (Chapter 04, J=0.85)
     re-assemble docs with duplicate paragraphs removed
5. for benchmark/PII-sensitive slices:
     targeted sentence-level exact-dedup for
     benchmark n-grams and PII-flagged sentences
6. emit the dedup report                     (Chapter 08)
```

Each step's output feeds the next step's input. The ordering matters: running paragraph-level before document-level is wasteful (you paragraphise then discard many parent documents). Follow the order.

## Examples of granularity shaping the result

Three concrete scenarios to internalise:

**Scenario A — "I ran document-level dedup on Common Crawl; is that enough?"**
No. Common Crawl is drowning in boilerplate — per-site navs, cookie banners, author bios. Document-level removes mirrored and syndicated pages but leaves the boilerplate. Add paragraph-level. SlimPajama got document-level only; its downstream users frequently *also* paragraph-dedup.

**Scenario B — "I ran sentence-level dedup and my downstream model lost its politeness register."**
Likely. Blanket sentence dedup removes high-frequency common phrases. If the downstream task is fluent conversation, do not run sentence dedup. If the downstream task is QA on specific documents, consider targeted sentence dedup against the benchmark n-grams only.

**Scenario C — "I have a PII-dense corpus (healthcare transcripts) and want a credible memorisation story."**
Document-level is table-stakes. Paragraph-level helps. The high-value add is **sentence-level exact-dedup gated on PII-detector hits**: for every sentence that a PII detector ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)) flags, hash and dedup. This forces each unique PII-bearing sentence to appear at most once, which is what Kandpal 2022's memorisation-vs-frequency curve says you want.

## Summary

- **Document, paragraph, and sentence** are three different quality controls with three different cost profiles and three different threat models.
- **Default document-level** everywhere; **default paragraph-level for web corpora**; **target sentence-level** at benchmark n-grams and PII-flagged sentences, not blanket.
- **Jaccard thresholds tighten as granularity shrinks.** Sentence dedup is often exact-match rather than MinHash.
- **Dedup order matters:** fingerprint → exact → near-dup at document → near-dup at paragraph (if needed) → targeted sentence (if needed) → emit report.
- **Record the granularity choice per slice** in the dedup report. "We de-duplicated" is not an answer; "we dedup'd at document and paragraph with J=0.8 and 0.85 respectively" is.

Chapter 06 picks up with production toolchains — `data-juicer`, `NeMo Curator`, `dolma dedup` — the layer at which these choices become throughput-engineering concerns rather than algorithm-tuning concerns.

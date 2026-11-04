# Chapter 05 — Kreutzer "Quality at a Glance": Multilingual Slice Auditing

## Why this chapter exists

Dodge-style audits (Chapter 04) assume you can read the text. For an English slice this is trivial. For a multilingual slice — even one dominated by high-resource languages — it is not. The dominant failure mode of a multilingual web corpus is not a missing field or a bad TLD; it is **mislabelled language**, **non-text-pretending-to-be-text**, and **systematic sentence-level corruption** in low-resource languages that the automated pipeline cannot see. Those failures do not show up in a FastText `lid.176` score and they do not show up in Chapter 02's row-level checks.

The reference is **Kreutzer, Caswell, Wang, Wahab, van Esch, Ulzii-Orshikh, Tapo, Subramani, Sokolov, Sikasote, Setyawan, Sarin, Samb, Sagot, Rivera, Rios, Papadimitriou, Osei, Suárez, Orife, Ogueji, Rubungo, Nguyen, Müller, Müller, Muhammad, Muhammad, Mnyakeni, Mirzakhalov, Matangira, Leong, Lawson, Kudugunta, Jernite, Jenny, Firat, Dossou, Dlamini, de Silva, Çabuk Ballı, Biderman, Battisti, Baruwa, Bapna, Baljekar, Azime, Awokoya, Ataman, Ahia, Ahia, Agrawal, Adeyemi, *Quality at a Glance: An Audit of Web-Crawled Multilingual Datasets*, TACL 2022** (<https://aclanthology.org/2022.tacl-1.4/>). The paper audits five widely-used multilingual web corpora (CCAligned, WikiMatrix, ParaCrawl, mC4, OSCAR) by sampling documents and having a human speaker of the labelled language rate them. The findings are alarming: for many low-resource languages, the correctly-labelled fraction is below 50% and the usably-correct text fraction is lower still.

The methodology — human spot-check with a structured rating sheet — is what this chapter operationalises. The deliverable is a **Kreutzer-style audit worksheet** you run on every multilingual slice before it leaves your team.

## The Kreutzer audit categories

The paper defines a short rating taxonomy a native or fluent speaker applies to each sampled document. The categories (paraphrased to fit a slice audit):

- **`C` — Correct**: the document is in the labelled language and is coherent, well-formed text.
- **`CS` — Correct short**: in the labelled language but very short (fewer than ~5 words), which may or may not count as useful.
- **`CB` — Correct boilerplate**: in the labelled language but entirely boilerplate (menu, footer, template).
- **`WL` — Wrong language**: in a different language than labelled. Record the actually observed language.
- **`NL` — Non-linguistic content**: product codes, phone numbers, DNA, source code, numeric tables.
- **`NS` — Not sensible**: a jumble of tokens that are not any language — machine-translation detritus, OCR errors, truncated text.
- **`P` — Pornographic / offensive**: adult content. Not a quality judgement per se but a flag for downstream filters.

For each language slice, the audit computes:

- **Percent correct (`%C + %CS + %CB`)** — the share of documents that are in the labelled language at all.
- **Percent sensible-text-in-label (`%C`)** — the share that is both in the right language and useful text.
- **Error taxonomy distribution** — the breakdown of the non-correct share across `WL / NL / NS / P`.

For a multi-language slice, you compute these per language and aggregate.

## Operationalising the audit on your slice

### Step 1 — stratify the sample

Random sampling of a multilingual corpus is dominated by the most-frequent languages; a 500-document random sample from a slice that is 92% English will contain almost no Swahili. Sample instead **per language stratum**:

- Group the slice by declared language (or detected language, per your pipeline).
- Per language, draw a minimum sample size. Kreutzer et al. use 100 documents per language-corpus pair where available. 50 is a workable floor for languages that are rare in the slice; fewer than 20 is not auditable.
- Record the sample size per language and the confidence interval on the proportion estimate (binomial CI with the chosen sample size).

### Step 2 — recruit raters

The audit requires **human speakers of each language** — not fluent L2 learners, not raters guessing from cognates, not machine-translation help. This is the hardest logistical part.

Practical paths:

- For languages your team covers, rate in-house.
- For languages you don't, partner with academic groups working on that language's NLP. Credit them in the dataset card.
- Platforms like Appen, Toloka, and Amazon Mechanical Turk have L1-speaker filters for some languages but coverage degrades for low-resource languages. Verify L1-speaker status via a short screener question in the native script — ask the rater to describe an image or respond to a prompt in the language. <!-- needs-research: platform availability and pricing for low-resource-language annotation shifts frequently; verify at read time. -->
- For languages with no accessible rater pool, **record this as a gap** in the audit. Do not proxy with machine translation.

Raters are paid at or above local prevailing wage for annotation work ([`mod-107`](../mod-107-labeling-operations-and-annotation-programs/README.md) covers the labour-ops discipline). A Kreutzer-style audit is a small enough job that you can budget an hour per rater per language; the sample size is 50–100 documents and each rating is a short categorical judgement.

### Step 3 — the rating sheet

A single sheet per rater, in the rater's working language. One row per document. Columns: `doc_id`, `label` (`C`/`CS`/`CB`/`WL`/`NL`/`NS`/`P`), `if_WL_observed_language`, `optional_notes`. Store the raw sheet; the audit result is an aggregation.

For reproducibility, record alongside the sheet:

- The sampling strategy (predicate + stratum definition + seed).
- The rater's language proficiency attestation (self-reported L1 / L2 + years).
- Inter-rater agreement on a shared double-rated subset (at least 20% of the sample; Cohen's κ is the usual summary). [`mod-107`](../mod-107-labeling-operations-and-annotation-programs/README.md) chapter on inter-annotator agreement is the reference.

### Step 4 — compute the per-language scores

Per language slice:

```python
# pseudocode
labels = pd.read_parquet("audit/kreutzer/ratings_sw.parquet")
n = len(labels)
pct_correct = (labels["label"].isin(["C","CS","CB"])).mean()
pct_sensible_in_lang = (labels["label"] == "C").mean()
error_mix = labels.loc[~labels["label"].isin(["C","CS","CB"]), "label"].value_counts(normalize=True)
```

Emit per-language:

```yaml
- language: sw
  sample_size: 100
  pct_correct: 0.68          # in the labelled language at all
  pct_sensible_in_lang: 0.41 # in the language AND useful text
  pct_wrong_language: 0.22
  pct_non_linguistic: 0.06
  pct_not_sensible: 0.04
  pct_porn_or_offensive: 0.00
  observed_wrong_languages: {pt: 0.12, "unknown-mix": 0.10}
  rater_pool:
    size: 3
    agreement_kappa: 0.71
```

### Step 5 — attach to the slice

The Kreutzer audit result is a section of the per-slice quality report (Chapter 08):

```yaml
kreutzer_audit:
  slice_id: multi-web-v2
  audit_date: 2026-11-04
  method_citation: "Kreutzer et al. 2022, TACL"
  languages:
    - { language: en, sample_size: 100, pct_correct: 0.97, pct_sensible_in_lang: 0.89, ... }
    - { language: zh, sample_size: 80,  pct_correct: 0.92, pct_sensible_in_lang: 0.78, ... }
    - { language: sw, sample_size: 100, pct_correct: 0.68, pct_sensible_in_lang: 0.41, ... }
  gaps:
    - { language: om, reason: "no accessible L1-speaker rater pool", recorded_at: 2026-11-04 }
```

## What the audit catches that nothing else does

The row-level tools in Chapters 02–03 cannot distinguish "`language='sw'`" from "`text` is actually Swahili". FastText `lid.176` scores a document on its content but the document may be:

- **Transliterated** — Arabic-script Hausa stored as `ha` when the code expects Latin-script Hausa.
- **Code-switched** — mixed-language documents where the dominant language is not the labelled one.
- **Machine-translated junk** — fluent in form but senseless; passes the LID classifier because it has the right n-grams.
- **Technical / non-prose** — pure boilerplate, product listings, code that happens to have the right language tag.
- **Fossil content** — the page was once in language X but the HTML now contains only menu items in a different language.

All of these are visible only to a human speaker. Kreutzer et al.'s central finding is that automated LID is systematically over-confident on low-resource languages and that the only known correction is a human spot-check. Your pipeline inherits the same risk.

## When to run which audit

| Slice shape | Dodge (Chapter 04) | Kreutzer (this chapter) |
|---|---|---|
| English-only web text | Required | Not applicable |
| Multilingual high-resource (en/es/fr/de/zh) | Required | Required — audit each language |
| Multilingual with low-resource languages | Required | Required — mandatory per language; gaps logged |
| Synthetic data | Required (for filter attribution) | Applicable if synthetic multilingual; see [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md) |
| Labelled task data | Partial (filter audit) | Applicable per language of the labelled task |

The two audits are complementary: Dodge catches filter-attribution and representativeness-of-exclusion failures; Kreutzer catches the *labelling-at-source* failure that precedes everything the pipeline does.

## Practical pitfalls

- **Using automated LID as the audit.** Self-referential: you are auditing a classifier's output with the same classifier. Humans are the primitive.
- **Under-sampling the low-resource tail.** Random sampling gives you almost no low-resource documents. Stratify.
- **Over-trusting a single rater.** For high-stakes slices, double-rate at least 20% of the sample and report κ. Single-rater results are a draft, not a release artifact.
- **Discarding "sensitive" documents before auditing.** The pornographic / offensive category is informative; filtering it out before the audit hides the base rate. Rate everything, then decide downstream.
- **Treating the audit as a floor, not a ceiling.** `pct_sensible_in_lang = 0.41` is an estimate from a small sample. The CI is wide; the correct read is "the slice may be ~41% usable in this language", not "the slice is exactly 41% usable".
- **Making the audit a release blocker without a remediation path.** If Swahili scores 0.41, you have options: downsample the language, add a filter trained on its features, re-source the language from a higher-quality corpus, exclude it from the slice. Log the choice in the dataset card.

## Tooling

- **GlotLID** (Kargaran et al. 2024, <https://aclanthology.org/2024.findings-acl.47/>, repo <https://github.com/cisnlp/GlotLID>) — modern multilingual language-ID model covering 1600+ languages. Use it alongside `lid.176` and a human audit, not instead of either.
- **No-code audit interfaces**. Label Studio (<https://labelstud.io/>), Prodigy (<https://prodi.gy/>), Argilla (<https://argilla.io/>), and Toloka work for the rating step. Prefer one that supports a right-to-left UI if any of your languages need it.
- **`datasets` for sampling**. Hugging Face `datasets` with `shuffle(seed=...).select(range(n))` after a per-language filter is the usual sampling primitive; record the seed.

## Summary

- The Kreutzer "Quality at a Glance" audit is the discipline for multilingual slices: human L1 speakers rate sampled documents on a short taxonomy (`C/CS/CB/WL/NL/NS/P`).
- Stratify sampling per language; aim for 100 per language, floor of 20, log gaps for languages without accessible raters.
- Record per-language `pct_correct` and `pct_sensible_in_lang`, the error taxonomy, the rater pool size, and Cohen's κ on a double-rated subset.
- The audit catches failures no automated tool can — transliteration, code-switching, machine-translation detritus, fossil content.
- Kreutzer and Dodge are complementary audits. Run both on multilingual slices; the Kreutzer results are the input to any "we release this slice in these languages" claim in the dataset card.
- The result section is a block in the per-slice quality report (Chapter 08).

Chapter 06 turns to the distribution-and-coverage half of the mod-104 spec: disaggregated counters, representativeness vs. a target population, and the measurement infrastructure that supports both Dodge and Kreutzer audits.

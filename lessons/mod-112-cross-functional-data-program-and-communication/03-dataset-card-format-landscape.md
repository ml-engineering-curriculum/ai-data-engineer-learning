# Chapter 03 — The Dataset-Card Format Landscape

## Why this chapter exists

"Dataset card" is not one thing. The term is used for at least four distinct documents in the field — the academic *datasheet* that started the practice, the **Hugging Face Dataset Card** format baked into the Hub, Google's **Data Cards Playbook** form, and Meta's **Data Cards** format as used in the Llama and later Meta model releases. The four cross-reference each other but target different audiences, carry different levels of detail, and are maintained by different communities.

Picking the format is the step before authoring. Pick the wrong format and you spend the next release cycle re-authoring content that was already correct, or you ship a document a regulator cannot act on. This chapter maps the landscape so that in Chapters 04–06 you can author each format with a clear picture of why the fields are what they are.

## The academic anchor: Datasheets for Datasets (Gebru et al.)

In 2018 Gebru et al. published "Datasheets for Datasets", proposing a questionnaire-shaped document attached to every dataset covering motivation, composition, collection process, preprocessing, uses, distribution, and maintenance. The 2021 CACM publication sharpened the question list into roughly 50 numbered questions organised into seven sections.

Datasheets for Datasets is the **schema that every production format in this chapter inherits from**. The HF Dataset Card's sections, Google's Data Cards Playbook form, and Meta's Data Cards all map onto the Gebru et al. question list in substantially the same order. When a regulator or auditor asks "where is X documented?", the Datasheet question number is a durable reference even if the specific format changes.

The seven sections:

1. **Motivation** — why the dataset was created, who created it, who funded it.
2. **Composition** — what instances represent, how many, what features, labels, relationships.
3. **Collection Process** — how the data was acquired, by whom, when, with what ethical review.
4. **Preprocessing / Cleaning / Labelling** — what was done to the raw data; is the raw data preserved; is the software available.
5. **Uses** — what the dataset has been used for; what uses it should and should not be used for.
6. **Distribution** — how the dataset is distributed, under what licence, with what access restrictions.
7. **Maintenance** — who maintains, how to contact, how updates are communicated, how long will it be supported.

Nothing about these seven sections is specific to AI-safety or to foundation models. The Datasheet is a general dataset-documentation artifact.

## Three production formats stacked on the datasheet

Three production formats are in wide use in 2026. Each extends the Datasheet differently, targets a different audience, and has a different authoring surface.

### Hugging Face Dataset Card

The **Hugging Face Dataset Card** is a markdown document (`README.md`) at the root of every dataset repository on the Hugging Face Hub, with YAML frontmatter that encodes machine-readable metadata (task, language, licence, size categories, source datasets, tags). Hugging Face maintains an authoring template at `huggingface/datasets-tagging` / `huggingface/dataset-metadata-schema` and surfaces a web-based editor on the Hub.

The card is both human-readable (rendered on the Hub) and machine-readable (consumed by `datasets.load_dataset()`'s metadata functions, the Hub's search facets, and model-card authoring tools that cross-reference the dataset).

Primary audience: **the ML practitioner who is about to `load_dataset("…")` and wants to know whether to use it**. Tone is engineering-oriented; licence and loading snippets are first-class.

Reference: <https://huggingface.co/docs/hub/datasets-cards> for the format, <https://github.com/huggingface/datasets/blob/main/templates/README.md> for the template.

### Google Data Cards Playbook

Google's **Data Cards Playbook** (Pushkarna et al., published by Google Research in 2022) is a design-oriented form and methodology for creating *transparency reports* on datasets. The playbook distinguishes it from a pure specification: the playbook describes a *process* — stakeholder mapping, authoring cycles, review — in addition to a card template.

The card itself is a structured, visually formatted document (originally designed for prose+graphics layout, not raw markdown) with sections like *Dataset Overview*, *Example of Data Points*, *Motivations & Intentions*, *Provenance*, *Human and Other Sensitive Attributes*, *Extended Use*, *Transformations*, *Annotations and Labeling*, *Validation Types*, *Sampling Methods*, *Known Applications and Benchmarks*, *Terms of Art*, *Reflections on the Data*, *Maintenance*.

Primary audience: **a non-engineering stakeholder — policy, product management, external audit** — who needs to reason about the dataset without reading the loading code. Tone is explanatory; limitations and reflections are first-class.

Reference: Pushkarna et al. "Data Cards: Purposeful and Transparent Dataset Documentation for Responsible AI", FAccT 2022; the Data Cards Playbook site at <https://sites.research.google/datacardsplaybook/>.

### Meta Data Cards / Llama-style disclosure

Meta's **Data Cards** are the format Meta publishes alongside its model releases (Llama 2, Llama 3, and successors). The Meta Data Card accompanies a **Model Card** and a **Responsible Use Guide** as part of the standard release package. Its sections typically include *Dataset Overview*, *Data Collection Process*, *Representation*, *Language*, *Preprocessing*, *Training Data Characterisation* (token count, mix by source class, cutoff dates), *Privacy*, *Known Biases*, *Intended Use*.

Primary audience: **the external community of consumers, researchers, and regulators** reading a Meta model release. Tone is concise and policy-aware; it rarely cites a specific URL for a dataset shard but discloses *source-class* composition, token counts, cutoff dates, and the aggregate decontamination/PII posture. This is the shape foundation-model providers have converged on for public model releases where full source-URL disclosure is either commercially infeasible or contractually restricted.

Primary reference example (not the only one): the Llama 3 release, with the Model Card published in the `meta-llama/llama3` GitHub repository (<https://github.com/meta-llama/llama3/blob/main/MODEL_CARD.md>) and the companion blog/launch page at <https://ai.meta.com/blog/meta-llama-3/>. Meta's release pages and the current Llama family index at <https://www.llama.com/> are updated over time.

## A field-by-field cross-reference

Below is a cross-reference table so you can see which section of each format maps to which Datasheet question group. Use this as the authoring aid when you have to ship more than one format from the same evidence base.

| Datasheet section (Gebru et al.)       | HF Dataset Card fields                                                        | Google Data Cards section                                  | Meta Data Card section                               |
|---------------------------------------|-------------------------------------------------------------------------------|-----------------------------------------------------------|-----------------------------------------------------|
| Motivation                            | *Dataset Description → Dataset Summary*, YAML `pretty_name`, `tags`           | *Motivations & Intentions*                                | *Dataset Overview*                                   |
| Composition                           | *Dataset Structure*, YAML `size_categories`, `task_categories`, `language`     | *Dataset Overview*, *Example of Data Points*              | *Representation*, *Language*, *Training Data Characterisation* |
| Collection Process                    | *Data Collection → Source Data, Initial Data Collection and Normalisation*     | *Provenance*, *Sampling Methods*                          | *Data Collection Process*                            |
| Preprocessing / Cleaning / Labelling  | *Dataset Creation → Annotations*, *Who are the annotators?*                   | *Transformations*, *Annotations and Labeling*             | *Preprocessing*                                      |
| Uses                                  | *Considerations for Using the Data → Social Impact of Dataset, Discussion of Biases, Other Known Limitations* | *Extended Use*, *Reflections on the Data*                 | *Intended Use*, *Known Biases*                       |
| Distribution                          | YAML `license`, *Licensing Information*                                       | *Terms of Art*                                            | (release-page footer — Meta publishes the licence next to the model, not in the Data Card itself) |
| Maintenance                           | *Dataset Card Authors, Dataset Card Contact Information, Discussion*          | *Maintenance*                                             | (handled at the model-release level, not per-dataset-card) |
| **AI-era additions** (not in original Datasheet) | | | |
| Contamination against eval sets        | *Considerations → contamination* (free-form; see Chapter 04)                  | *Known Applications and Benchmarks*                       | *Training Data Characterisation → contamination posture* |
| Residual-PII rate                     | *Considerations → personal information*                                       | *Human and Other Sensitive Attributes*                    | *Privacy*                                            |
| Lineage-traced snapshot hash          | YAML `source_datasets`, free-form                                             | *Provenance → version pin*                                | (release footer; Meta cites model checkpoint SHA, not per-dataset SHA) |
| Retrainability status flag            | *Discussion*                                                                   | *Reflections on the Data*                                 | (not standard; usually implicit in cutoff date)     |

Two takeaways:

- **Every production format is a projection of the Datasheet.** The content you write once can be laid out three different ways. If your repository stores the evidence base as Datasheet questions, the formats become downstream renderings, not separate documents.
- **The AI-era fields are where the formats diverge.** Contamination, residual-PII, lineage hash, retrainability — these were not in Gebru et al. 2018. Each format has chosen a different placement, and in some cases (Meta's) a different granularity. Chapters 04–06 are explicit about where each lands.

## How to pick the format for a given release

Four questions resolve the format choice.

1. **Who is the primary audience?**
   - Internal ML practitioner → HF format (even if not published externally, the HF layout is familiar to the engineering reader).
   - Executive, product, non-technical reviewer → Google Data Cards Playbook form.
   - External public release of a model → Meta / Llama-style Data Card.
2. **How is the dataset distributed?**
   - As a Hub artifact (public or private HF repo, or an org-internal HF-compatible store) → HF format is the native card.
   - As a design artifact without a code-level distribution surface → Google Data Cards Playbook.
   - As a *characterisation* of the aggregated training mix, with no per-source distribution → Meta/Llama-style.
3. **What is the regulator / audit requirement?**
   - EU AI Act Article 10 for a high-risk system → the Google Data Cards Playbook form matches the detail level Article 10 wants (bias audit, representativeness, mitigation records), and cross-references to the Datasheet number set survive legal review.
   - EU AI Act Article 53 for a GPAI provider → the Meta/Llama-style card is the public-facing companion to the Article 53(1)(d) **public training-data summary** (Chapter 07). Do not try to make the public summary and the Data Card the same document; they target different obligations.
4. **How much does source disclosure matter?**
   - Maximum disclosure (academic release, open-source dataset) → HF format with explicit source listing.
   - Moderate (consortium-shared training mix) → Google Data Cards Playbook with per-source-class detail.
   - Minimum compatible with Article 53 + Code of Practice (closed-weight foundation model) → Meta/Llama-style.

A single release can ship **more than one** format. A GPAI-provider release typically ships all three: an HF card on the Hub (if the slice is exposed), a Google-form card internally for the governance package, and a Meta-style card as the public companion to the model card.

## The authoring strategy: single source, multiple renderings

The practical pattern used by teams that ship more than one format is to **author one evidence base and render into each format**. The evidence base is a structured YAML keyed by Datasheet question number; each format's template consumes the YAML and produces the card. A sketch of the YAML shape:

```yaml
# dataset_evidence.yaml
datasheet_fields:
  1.1_purpose:                 # Datasheet Q1.1: for what purpose was the dataset created?
    answer: >
      Pretraining corpus for a multilingual general-purpose language model intended for
      research and commercial use with enterprise-grade governance.
    evidence_refs:
      - ../charter.md
  1.2_creator:
    answer: Acme AI, data-engineering team (ai-data-engineer@acme).
  3.1_acquisition_process:
    answer: >
      Common Crawl WARC files downloaded from the official S3 mirror on 2026-09-28,
      filtered for TLDs listed in config/tlds.yaml, parsed with trafilatura,
      language-detected with fastText lid-176.
    evidence_refs:
      - ../acquisition/run_20260928.log
      - ../openlineage/run_20260928.json

renderings:
  hf:
    output: cards/hf/README.md
    template: templates/hf.md.j2
  google:
    output: cards/google/data_card.md
    template: templates/google.md.j2
  meta:
    output: cards/meta/data_card.md
    template: templates/meta.md.j2
  annex_xi_public_summary:
    output: cards/annex_xi/public_summary.md
    template: templates/annex_xi.md.j2        # Chapter 07
```

Three benefits over writing each format separately:

1. **Consistency**. A fact changed once in the evidence base changes everywhere.
2. **Auditability**. The evidence base is the ground truth; each format is a derived artifact with a known template version.
3. **Format evolution**. When Hugging Face changes its template (periodically), or the EU AI Office issues a new public-summary template, only the template file changes; the content survives.

This pattern is the default for the exercise in `exercises/exercise-02-dataset-card-across-three-formats.md`.

## What this chapter is not

- Not the HF card authoring guide — Chapter 04.
- Not the Google Data Cards authoring guide — Chapter 05.
- Not the Meta/Llama-style authoring guide — Chapter 06.
- Not the EU AI Act public summary guide — Chapter 07. (A public summary is **not** a Data Card; it is an Article 53(1)(d) obligation with its own template.)

It is the map of the field so that the authoring chapters have a frame to slot into.

## Summary

- "Dataset card" names at least four distinct documents: Gebru et al.'s academic **Datasheet** (the schema that anchors the others), the **Hugging Face Dataset Card** (engineering audience, markdown + YAML frontmatter), the **Google Data Cards Playbook** form (non-technical stakeholder audience, design-oriented process + form), and the **Meta / Llama-style Data Card** (external community + regulator audience, part of a public model-release package).
- Every production format is a projection of the Datasheet question list. AI-era fields — contamination, residual-PII, lineage hash, retrainability — are where the formats diverge.
- Pick the format by (a) audience, (b) distribution surface, (c) regulator / audit requirement, (d) source-disclosure constraints. A single release can ship more than one.
- The scalable authoring pattern is **single evidence base, multiple renderings** — a YAML keyed by Datasheet question number plus one template per format.
- Chapters 04–06 are the three authoring chapters; Chapter 07 takes the Article 53 public summary as a separate obligation with its own template.

Chapter 04 picks up with the Hugging Face Dataset Card — the format most engineers encounter first and the one most likely to be your organisation's internal default.

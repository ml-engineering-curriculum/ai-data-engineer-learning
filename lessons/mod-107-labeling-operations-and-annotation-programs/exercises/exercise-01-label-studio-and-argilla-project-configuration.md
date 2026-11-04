# Exercise 01 — Label Studio and Argilla Project Configuration

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 02 — Open-source annotation tool landscape](../02-open-source-annotation-tool-landscape.md), [Chapter 03 — Project configuration, review queues, consensus, and adjudication](../03-project-configuration-review-consensus-adjudication.md)

## Objective

Stand up two production-shaped labeling projects — one in **Label Studio**, one in **Argilla** — on the same slice, with review queues, consensus, and adjudication configured. Produce the first two artifacts of the program package: the `guidelines.md` contract and the `tool_config/` project files. This is the practical floor for every later exercise in this module.

By the end you will have:

- A signed `guidelines.md` with scope, taxonomy, edge cases, UI mechanics, and version header.
- A `tool_config/label_studio_project.xml` and matching `tool_config/argilla_dataset.py` for the same task.
- A running Label Studio instance and Argilla instance (local Docker is fine) with the projects deployed.
- A 100-item calibration subset gold-labeled by you, loaded into both tools.
- A short `comparison.md` noting where each tool was better / worse for your task.

## Prerequisites

- Chapters 02 and 03 read.
- Docker (or Podman) installed.
- Python 3.11+ with `label-studio-sdk`, `argilla`, `pandas`, `pyarrow`.
- A slice to label. Pick one:
  - A text classification slice — 1k–5k items, 3–5 classes. Reuse any output of a prior `mod-104` exercise or any public dataset (e.g. a curated subset of HF `ag_news` with explicit class imbalance injected).
  - A preference-pair slice — 300–500 (prompt, response_a, response_b) triples. If needed, generate candidates by running a small LLM twice on the same prompt with different temperatures.

## Problem statement

Choose **one** modality (classification *or* preference-pair). Configure the same task in **both** tools.

The task is a production configuration task. Deliverables are binary: either the project runs with review queues, consensus, and adjudication wired up, or it doesn't.

## Starter guidance

### 1. Write `guidelines.md` first

Before touching any tool. The document must have the eight sections named in Chapter 03 §"What a production labeling-guidelines document contains". Minimum:

- Scope, downstream use (name a plausible downstream consumer and benchmark).
- Label taxonomy with 3–5 positive and 3–5 negative examples per label.
- Edge-case catalogue with at least 5 entries drawn from your calibration set.
- Annotator safety / PII handling.
- UI mechanics (fill in after step 3 once you have screenshots).
- Quality expectations — target IAA with a one-sentence justification.
- Version header (`v0.1.0`) and SHA of this file.

### 2. Build the 100-item calibration set

Pick 100 items from the slice. **You** gold-label them. Weight them per Chapter 03 §"The calibration set" — 40% canonical, 40% edge cases, 20% initially-contested (and resolved by you). Save as `calibration_set/items.parquet` and `calibration_set/gold_labels.parquet` (same keys).

### 3. Stand up Label Studio

- `docker run -p 8080:8080 -v $(pwd)/ls_data:/label-studio/data heartexlabs/label-studio:latest`
- Create an account at `http://localhost:8080`, log in, grab the API token.
- Author `tool_config/label_studio_project.xml`. For classification use `<Text>` + `<Choices>`; for preference-pair use the `<Pairwise>` pattern from Chapter 02. Include a `<TextArea>` for the annotator rationale.
- Use the Python SDK to create the project, set `maximum_annotations=3`, `overlap_cohort_percentage=100`, `review_settings={"sample_rate": 0.15}`.
- Import the slice and the calibration set as tasks. Tag calibration items with `calibration=true` in the metadata so you can filter later.
- Screenshot the annotator view and the review queue. Drop into `screenshots/`.

### 4. Stand up Argilla

- `docker run -p 6900:6900 argilla/argilla-quickstart:latest`
- Wait for the quickstart to settle, log in with `argilla` / `12345678`, confirm UI at `http://localhost:6900`.
- Author `tool_config/argilla_dataset.py`:

  ```python
  import argilla as rg

  client = rg.Argilla(api_url="http://localhost:6900", api_key="argilla.apikey")

  # choose fields and questions to match your Label Studio project exactly
  settings = rg.Settings(
      fields=[...],
      questions=[...],
      metadata=[rg.TermsMetadataProperty(name="calibration")],
      distribution=rg.TaskDistribution(min_submitted=3),
  )
  dataset = rg.Dataset(name="mod-107-ex01", settings=settings, client=client)
  dataset.create()

  records = [...]  # load from your slice + calibration set
  dataset.records.log(records)
  ```

- Screenshot the annotator view. Drop into `screenshots/`.

### 5. Run a short end-to-end session

- In **one** of the two tools, label 30–50 items yourself. Deliberately introduce 2–3 items that you expect would be adjudicated.
- Walk the review queue from annotation → review → accepted. In a tool that supports it, mark one item as rejected and send it back.
- Export the labeled output as parquet / JSON for both tools.

### 6. Write `comparison.md`

One page, including:
- Which tool took longer to configure, and why.
- Which annotator UI felt faster per item (you are the annotator; be honest).
- Which tool's output format is closer to what `fine-tuning-engineer` would consume for the chosen task.
- Which tool you would pick for this task in production, with one sentence of reasoning.

## Deliverables

```
exercise-01/
  guidelines.md
  guidelines.md.sha256
  calibration_set/
    items.parquet
    gold_labels.parquet
  tool_config/
    label_studio_project.xml
    argilla_dataset.py
  screenshots/
    label_studio_annotate.png
    label_studio_review.png
    argilla_annotate.png
  exports/
    label_studio_output.json
    argilla_output.json
  comparison.md
  README.md
```

## Acceptance criteria

- `guidelines.md` is dated before any tool_config file (first in the git log for the exercise directory, or explicit version-header date).
- Both tools have a project deployed with the same label set and consensus rule (`min_submitted=3` / `maximum_annotations=3`).
- The Label Studio project has `review_settings.sample_rate` explicitly set (not default).
- Calibration items are tagged in both tools and can be filtered.
- The export from each tool contains a `guideline_version` field matching `guidelines.md`'s version header.
- `comparison.md` names a production pick and a one-sentence reason.
- Any factual claim you could not verify at authoring time is marked `<!-- needs-research: ... -->` instead of guessed.

## Stretch goals

- **Add Doccano** as a third configuration for the same task. Note what you had to give up (consensus is not first-class in Doccano) — this is the lesson.
- **Add Prodigy** (if you have access to the licence) and implement a `textcat.teach` recipe for a model-in-the-loop version. Compare per-item throughput against the Label Studio / Argilla baselines.
- **Wire SSO** against a local Keycloak or Authentik test instance on either tool. Document the configuration.
- **Run with three annotators from colleagues / peers**, not just yourself, and compute Krippendorff's alpha on the 30-item overlap. This previews Exercise 03.

## What this exercise is not

- Not an IAA computation exercise (Exercise 03).
- Not a vendor-contract exercise (Exercise 02).
- Not a labeling-production run — you label only enough to exercise the UI and the review queue.

It is the "I can stand this up in two hours" muscle that every later exercise in this module assumes.

# Chapter 02 — Open-Source Annotation Tool Landscape

## Why this chapter exists

Four open-source annotation tools dominate production ML labeling in 2024–2025: **Label Studio**, **Argilla**, **Prodigy**, and **Doccano**. Every one of them can "label text". None of them is the right default for every task. Choosing the wrong one costs you a week of configuration, a cohort of frustrated annotators, and — most expensively — a labeling output whose structure does not survive the hand-off to `fine-tuning-engineer`.

This chapter maps **tools to modalities** so you can pick with intent. It covers the four tools' native modality support, the UI primitive they are built around, the review-queue model (deep dive in Chapter 03), and the Python / API surface you will script against.

By the end you should be able to answer, in a one-paragraph note, which tool you would use for:
1. A 100k-item topic-classification task on English news.
2. A medical-record NER task with ten entity types and contested boundaries.
3. A preference-labeling task for an RLHF reward model, with eight-axis rubric.
4. A tool-calling-trajectory correction task for an agent fine-tune.
5. An image-segmentation task for a vision classifier.
6. An audio-timestamp task for ASR training.

The answer differs across the six. The reason is the subject of this chapter.

## The four tools, briefly

### Label Studio (HumanSignal)

- **Repo / project:** <https://github.com/HumanSignal/label-studio>. Docs: <https://labelstud.io/>.
- **License:** Apache 2.0 (Community); a Starter / Enterprise tier exists with SSO, review workflows, and commercial support.
- **Modality coverage:** text, image, audio, video, time series, HTML, PDF, 3D point cloud, and structured JSON (for tool-calling trajectories). The broadest coverage of the four tools.
- **Configuration primitive:** an **XML "labeling config"** that declares input `<Object>` elements, output `<Control>` elements (`<Choices>`, `<Labels>`, `<Rating>`, `<Pairwise>`, `<TextArea>`, `<RectangleLabels>`, etc.), and a `<View>` layout. The same primitive supports every modality.
- **Review-queue model:** tasks → annotations → reviews → predictions. Annotator and reviewer are distinct roles. Consensus and adjudication are workflows on top of the review queue (Chapter 03).
- **SDK:** `label-studio-sdk` (Python). REST API is first-class and stable.

A minimal text-classification labeling config:

```xml
<View>
  <Text name="text" value="$text"/>
  <Choices name="label" toName="text" choice="single">
    <Choice value="support_request"/>
    <Choice value="complaint"/>
    <Choice value="feedback"/>
    <Choice value="other"/>
  </Choices>
</View>
```

The same XML grammar expresses a preference-pair task:

```xml
<View>
  <Text name="prompt" value="$prompt"/>
  <View style="display:flex;">
    <Text name="a" value="$response_a"/>
    <Text name="b" value="$response_b"/>
  </View>
  <Pairwise name="winner" toName="a,b" selectionStyle="background:#90EE90"/>
  <TextArea name="rationale" toName="prompt" placeholder="Why did you pick this one?"/>
</View>
```

This single XML dialect — covering single-choice, pairwise, span, rating, and free text — is Label Studio's defining property.

### Argilla (Hugging Face)

- **Repo / project:** <https://github.com/argilla-io/argilla>. Docs: <https://docs.argilla.io/>. Now hosted and maintained under Hugging Face.
- **License:** Apache 2.0.
- **Modality coverage:** text-first. Chat, span, classification, ranking, rating. Image support is improving but secondary.
- **Configuration primitive:** a **Python dataclass-style schema** declared through `argilla.Settings`, `argilla.Field`, `argilla.Question`, and `argilla.Metadata`. Everything is Python; no XML.
- **Review-queue model:** records → suggestions → responses. Multiple annotators per record is first-class. The `distribution` field controls consensus. Integrates natively with Hugging Face `datasets` for roundtripping.
- **SDK:** `argilla` (Python). The SDK *is* the interface; the UI mirrors it.

A minimal Argilla preference dataset schema:

```python
import argilla as rg

settings = rg.Settings(
    fields=[
        rg.TextField(name="prompt"),
        rg.TextField(name="response_a"),
        rg.TextField(name="response_b"),
    ],
    questions=[
        rg.LabelQuestion(name="winner", labels=["a", "b", "tie"]),
        rg.TextQuestion(name="rationale", required=False),
        rg.RatingQuestion(name="confidence", values=[1, 2, 3, 4, 5]),
    ],
    metadata=[rg.TermsMetadataProperty(name="model_pair")],
    distribution=rg.TaskDistribution(min_submitted=3),
)
dataset = rg.Dataset(name="rlhf-preference-v1", settings=settings)
```

Argilla's native integration with the Hugging Face stack makes it the default for teams whose labeling output is a training dataset in the HF `datasets` format.

### Prodigy (Explosion AI)

- **Repo / project:** <https://prodi.gy/>. Not open source — a **paid commercial licence**. Included here because it is a de-facto standard in the spaCy ecosystem and is the right tool for several common text / NLP labeling tasks.
- **License:** proprietary, per-seat.
- **Modality coverage:** text-primary; some image and audio support. NER, span, classification, dependency parsing, coreference, rules-based labeling.
- **Configuration primitive:** a **Prodigy "recipe"** — a Python function decorated with `@prodigy.recipe` that returns a stream, update callback, and UI config. Recipes are first-class code, not config.
- **Review-queue model:** session-oriented. Annotators run a CLI that pulls a stream of items; progress is measured in *decisions per minute*. Explicit support for **model-in-the-loop** active learning — Prodigy's defining property.
- **SDK:** integrated with spaCy and Python; recipe authoring is the primary surface.

A minimal Prodigy recipe stub:

```python
import prodigy
from prodigy.components.loaders import JSONL

@prodigy.recipe("classify-tickets")
def classify_tickets(dataset: str, source: str):
    stream = JSONL(source)
    return {
        "dataset": dataset,
        "stream": stream,
        "view_id": "choice",
        "config": {"choice_style": "single", "choices": ["support", "complaint", "feedback", "other"]},
    }
```

Prodigy's *active-learning-first* UI (bind-key-per-decision, rapid-fire) makes it the right tool when the task is a text classification or NER task whose labels a human can decide in under five seconds each, and when the stream is driven by a model-in-the-loop.

### Doccano (open-source community project)

- **Repo / project:** <https://github.com/doccano/doccano>. Docs: <https://doccano.github.io/doccano/>.
- **License:** MIT.
- **Modality coverage:** text-only. Classification, sequence labeling, sequence-to-sequence, intent detection, slot filling, speech-to-text transcription, image classification (limited).
- **Configuration primitive:** project templates chosen at creation time (DocumentClassification, SequenceLabeling, Seq2seq, etc.). Less flexible than Label Studio's XML or Argilla's schema.
- **Review-queue model:** basic — per-project user roles, approval workflow. Fewer out-of-the-box consensus and adjudication features than Label Studio or Argilla.
- **SDK:** `doccano-client` (Python). Also exposes a REST API.

Doccano's defining property is **simplicity and self-hosting**. For many classroom / small-team deployments it is the right call because the operational cost is near zero.

## Tool-to-modality mapping

The decision matrix below is the one you should be able to defend. There is no single "best" tool; the right pick is modality-conditional.

| Task modality | Default tool | Secondary | Why |
|---|---|---|---|
| Text classification (small, bounded, fast per-item decision) | **Prodigy** | Label Studio, Doccano | Rapid-fire UI, native model-in-the-loop |
| Text classification (large, many classes, review workflow) | **Label Studio** | Argilla | Review queues and adjudication built-in |
| Named entity recognition (contested boundaries) | **Prodigy** | Label Studio | Keyboard-driven span UI, patterns, suggestions |
| Relation extraction, event extraction | **Label Studio** | Argilla | Explicit relation / entity-link primitives |
| Preference pair for RLHF / DPO | **Argilla** | Label Studio | Distribution control + HF `datasets` roundtrip |
| Likert / rubric scoring, InstructGPT-style | **Argilla** | Label Studio | Multiple questions per record are first-class |
| Chatbot-arena-style ranking | **Label Studio** or custom | Argilla | `<Pairwise>` + custom front-end if needed |
| Tool-calling-trajectory correction | **Label Studio** | Argilla | JSON-structured input + per-step annotation |
| Image classification | **Label Studio** | Doccano (limited) | `<Image>` + `<Choices>` is well-supported |
| Image segmentation / bounding box / polygon | **Label Studio** | CVAT (out of scope) | `<Image>` + `<RectangleLabels>` / `<PolygonLabels>` |
| Audio timestamp / transcript | **Label Studio** | Doccano (STT only) | `<AudioPlus>` + `<Labels>` with time ranges |
| Video annotation | **Label Studio** | CVAT (out of scope) | `<Video>` + `<Labels>` with per-frame tracking |
| PDF document layout | **Label Studio** | — | `<Image>` + `<HyperText>` + region tools |
| Instruction-fine-tune SFT labels | **Argilla** | Label Studio | Chat and multi-turn fields are first-class |

Three principles come out of the matrix:

1. **If the task's output is a Hugging Face `datasets` row, default to Argilla.** The roundtrip overhead is zero. Chat-formatted SFT datasets and preference datasets are the canonical case.
2. **If the task is non-text or mixes multiple modalities, default to Label Studio.** Its XML config covers more modalities than the other three combined.
3. **If the task is fast per-item (< 5 seconds per decision) and benefits from model-in-the-loop, consider Prodigy.** Its UI is explicitly designed for this regime and will materially improve annotator throughput.

## What the four tools do **not** cover

Mark the boundary:

- **Vendor-managed labeling** — Scale AI, Surge AI, Appen, Sama, iMerit, Toloka, SuperAnnotate. These are managed services with their own UIs. Chapter 05 is the contracting / audit chapter. You still write the labeling guidelines and the calibration set; the vendor runs the UI. The four tools in this chapter are for **in-house** labeling programs and for **vendor-output audit** against your own calibration set.
- **Image computer-vision-specialist tools** — **CVAT** (<https://github.com/opencv/cvat>) and **Supervisely** are best-in-class for dense CV labeling (video, 3D). Label Studio covers a substantial subset; if your program is primarily CV, CVAT is worth a separate evaluation. Out of scope for this module.
- **LLM-as-judge rubric runs** — not labeling in this module's sense. See Chapter 09 and `model-evaluation-engineer`.
- **Synthetic-data generation** — [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md) covers it. Synthetic data is a *generation* program; labeling it (for quality filtering) can use the tools in this chapter, but creating it does not.

## The deployment choice: SaaS vs self-host

Each of the four tools has a self-host path and (for Label Studio and Argilla) a managed offering:

| Tool | Self-host | Managed |
|---|---|---|
| Label Studio | Docker, docker-compose, Helm chart. Reference: <https://labelstud.io/guide/install.html>. | HumanSignal cloud (paid). |
| Argilla | Docker, docker-compose, Hugging Face Spaces template. Reference: <https://docs.argilla.io/latest/getting_started/quickstart/>. | Hugging Face-hosted. |
| Prodigy | Python package, runs locally (per-seat licence). | — |
| Doccano | Docker, docker-compose. Reference: <https://doccano.github.io/doccano/>. | — (community-hosted only) |

The two deployment choices that bear on labeling-program quality:

1. **Where the data lives.** Self-host when the labeling slice contains sensitive data that cannot leave your infrastructure. A managed service for the labeling UI is still a data-sharing decision.
2. **Who holds the identity surface.** For in-house labeling, SSO (SAML, OIDC) against your IdP is a near-requirement in regulated settings. Label Studio Enterprise, Argilla with Hugging Face Spaces + OAuth, and Doccano all support this to varying depths.

Chapter 03 picks up the **project configuration** detail — review queues, consensus, adjudication — on whichever tool you selected here.

## A concrete tool-selection worked example

> *You are asked to label 20k (prompt, response_a, response_b) triples for an RLHF reward-model training run. Each item has an eight-axis rubric (helpfulness, honesty, harmlessness, tone, factuality, completeness, instruction-following, reasoning). Annotators are a vendor-provided pool of 50 workers plus 5 in-house reviewers. The output must land in Hugging Face `datasets` format for `fine-tuning-engineer`'s DPO pipeline.*

Decision path:

- Modality: preference-pair + Likert rubric. Argilla is the default.
- Scale: 20k × ~2 minutes each ≈ 70 person-hours. Three annotators per item for consensus ≈ 210 person-hours. One in-house reviewer at 10% sample ≈ 70 person-hours (slower than first-pass labeling). Total ≈ 280 person-hours; a 50-worker pool completes in ~1 week. (These numbers are illustrative; your pool's measured per-item time is the only number that actually plans the schedule.)
- Output format: Argilla → `rg.Dataset.to_hub()` → Hugging Face `datasets.Dataset` directly, one-line export. Label Studio → JSON export + transform script. The pipeline overhead difference is tangible.
- Review workflow: Argilla's `TaskDistribution(min_submitted=3)` + `rg.ResponseStatus.draft` filter gives you three-annotator consensus natively. Adjudication is a second pass by the in-house reviewers on `disagreement > threshold` items.

Verdict: **Argilla**, self-hosted (sensitive prompt data), SSO via your IdP, output roundtripped to HF `datasets` for `fine-tuning-engineer`.

If the same task had bounding boxes on the (prompt, response) context instead, the verdict flips to **Label Studio**.

## What this chapter is **not**

- It is not a tutorial on installing any one tool. The docs for each are good; follow them.
- It is not a review-queue / consensus / adjudication chapter — that is Chapter 03.
- It is not a vendor-tool chapter — that is Chapter 05.

## Summary

- Four tools dominate open-source labeling: **Label Studio** (broadest modality coverage, XML config), **Argilla** (HF-native, text-first, Python schema), **Prodigy** (paid, text + model-in-the-loop, rapid-fire UI), **Doccano** (simplest, text-only, self-host).
- Pick by modality, by output-format fit, and by per-item decision time. The tool-to-modality table above is the one you should be able to defend.
- Self-host or managed is a data-sharing and identity decision, not a feature decision.
- Chapter 03 is the natural follow-on — once a tool is chosen, project configuration, review queues, consensus, and adjudication are the operational depth.

# Chapter 03 — Project Configuration, Review Queues, Consensus, and Adjudication

## Why this chapter exists

Chapter 02 picked the tool. This chapter is the configuration that makes it a production program rather than a labeling toy. Four operational concepts sit under this chapter:

1. **Labeling guidelines** — the written contract that binds annotators, reviewers, vendors, and downstream consumers.
2. **Review queue** — the state-machine that moves an item through annotate → review → accept / reject.
3. **Consensus** — how many annotators per item, and how disagreements are summarised into one label.
4. **Adjudication** — the human-in-the-loop resolution path for disagreements that consensus cannot resolve.

Every one of these decisions is a lever on quality and cost. This chapter walks through each.

## The labeling-guidelines document

The guidelines document is the one artifact that — if written badly — no amount of annotator qualification, tool configuration, or IAA gymnastics will rescue. It is also the one artifact people routinely skip ("we'll just send examples" or "the vendor's project manager will write it"). Resist both.

### What a production labeling-guidelines document contains

A production guidelines document has, at minimum, these sections:

```
1. Scope and task definition
   - One-sentence task statement
   - Target downstream use (which model / benchmark consumes these labels)
   - Explicit out-of-scope list

2. Label taxonomy
   - Each label defined in one paragraph
   - 3–5 positive examples per label
   - 3–5 negative examples per label (near-misses that are NOT this label)
   - Explicit "none of the above" / "unsure" handling

3. Edge-case catalogue
   - Every ambiguity you discovered on the calibration set
   - The resolution decision for each — written, not implicit

4. Annotator safety and PII
   - What to do if PII appears in content
   - What to do if harmful content appears (CSAM, threats, hate speech)
   - Who to escalate to and the SLA for response
   - How to pause and resume

5. UI mechanics
   - Screenshot of the interface
   - Keyboard shortcuts
   - How to mark "defer to reviewer" vs "skip"

6. Quality expectations
   - Target IAA, with the reason you picked it (Chapter 04)
   - Calibration-set pass threshold for qualification
   - QA-sample policy (how many of your labels will be re-checked, and by whom)

7. Versioning and change log
   - Every guideline change is a version bump
   - Which labels are affected by the change
   - Whether re-labeling of prior work is required

8. Signatures
   - Program owner, annotator team lead, vendor PM (if applicable), legal (for sensitive content)
```

The document is **not** the UI config. The UI shows annotators the fields; the guidelines tell them how to decide. A labeling program that treats the UI as self-explanatory produces labels calibrated to whichever interpretation happens to be most common, which is routinely the wrong one.

### Version the guidelines like code

Every guideline revision gets a version string and a SHA. Labels in the output carry the guideline version they were made under. When the guidelines change mid-program:

- **Minor clarification** (wording, examples) — bump the patch version, no re-label.
- **Edge-case decision reversal** — bump the minor version, mark affected labels as "needs re-label", flag them in the output.
- **Taxonomy change** (adding, removing, or merging labels) — bump the major version, re-label all affected items.

This is a lineage decision — it will be recorded as a `mod-109` event. If your pipeline cannot answer "which guideline version produced this label", your preference-pair hand-off to `fine-tuning-engineer` is already broken.

## The calibration set

A **calibration set** is a small (typically 20–200 items), **gold-labeled** seed that serves three distinct purposes:

1. **Annotator qualification** — a new annotator labels the calibration set first. Agreement with gold determines whether they qualify for the production pool.
2. **Drift detection** — on cadence (daily / weekly), all active annotators re-label a small sample from the calibration set. A sudden agreement drop is a leading indicator that guidelines are being interpreted differently, that annotator fatigue is setting in, or that vendor rotation has brought in a new cohort with different training.
3. **IAA baseline** — the calibration set is where IAA numbers are easiest to interpret, because the "true" label is known.

Build the calibration set deliberately. A good set spans the edge-case catalogue from the guidelines document — not just the easy centre of each class. It should contain, roughly:

- 40% easy / canonical examples per class
- 40% edge cases from the guidelines catalogue
- 20% items where the gold label was itself contested and had to be adjudicated by the program owner

Gold labels are produced by the **program owner + a subject-matter expert**, not by annotators. The expert's decisions become the written edge-case catalogue in the guidelines document. The calibration set is, in effect, the materialised form of the guidelines.

## Review queue — the four-state machine

Every production labeling tool has a review queue with the same four states, even if they are named differently:

```
            annotate               review               ship
[unlabeled] --------> [pending_review] --------> [accepted] -----> output dataset
                         |
                         | reject
                         v
                    [needs_rework] ----> back to annotate
```

The labels for these states in each tool:

| Tool | unlabeled | pending_review | accepted | needs_rework |
|---|---|---|---|---|
| Label Studio | `created` | `annotation_complete` | `reviewed_accepted` | `reviewed_rejected` |
| Argilla | `pending` | `submitted` | `validated` | `discarded` |
| Prodigy | (stream) | `accept`/`reject` on each item | — | — (re-stream) |
| Doccano | `unlabeled` | `labeled`, `in_review` | `approved` | — |

Three operational decisions attach to the review queue:

1. **Review sampling rate.** 100% review is expensive and unnecessary for mature programs. A production rate is usually 10–25% random sample per annotator, plus 100% on items flagged by consensus disagreement or confidence signals. New annotators start at 100% and taper as they prove out on the calibration set.
2. **Reviewer-to-annotator ratio.** A reasonable ratio is 1 reviewer per 5–10 annotators for text; higher for complex modalities. Reviewers are more expensive than annotators (they are usually senior annotators or program leads).
3. **Reject-and-rework vs re-label.** A rejected item can either go back to the same annotator with a reviewer's note ("rework") or to a different annotator for a fresh pass ("re-label"). Rework is cheaper and keeps the annotator learning; re-label is more rigorous and avoids annotator-specific blind spots. Use rework for first failures, re-label for repeat failures.

## Consensus — multiple annotators per item

For any label whose downstream use is training-weight-bearing (preference pairs, safety labels, high-stakes classification), **multiple annotators per item** is the default, not an upgrade.

Four consensus structures you will see:

- **k-of-n unanimous** — if all `n` annotators agree, accept the label; otherwise send to adjudication. Expensive but highest quality. Default for high-stakes labels on small pools.
- **majority vote** — accept the modal label if `> n/2` annotators agree. The Snow 2008 default. Cheaper, usable for most tasks, well-studied.
- **Dawid-Skene weighted** (Dawid & Skene 1979) — fit a per-annotator confusion matrix and combine labels into a probabilistic estimate. Better than majority vote when annotators have systematic biases.
- **MACE** (Hovy, Berg-Kirkpatrick, Vaswani, Hovy 2013) — explicitly models "which annotator is spamming" with a latent competence variable. Useful on crowdsourced pools with high worker variance.

The number of annotators per item is a budget decision:

| n | When to pick it | Cost multiplier |
|---|---|---|
| 1 | Simple binary task, qualified annotators, downstream training-insensitive | 1× |
| 2 | Standard default; disagreements go to a reviewer | 2× |
| 3 | Preference pairs, safety labels, any task with known inter-annotator variance | 3× |
| 5+ | Research / benchmark construction, Chiang-2024-style arena, calibration rounds | 5×+ |

Three annotators per item is the InstructGPT-era default for preference labeling (Ouyang 2022). Three gives you a majority with one abstention; it is the smallest `n` that lets you use Fleiss' kappa or Krippendorff's alpha meaningfully (Chapter 04).

### Writing consensus into the tool config

In **Label Studio Enterprise**, consensus is configured on the project via `consensus` / `minimum_annotations_per_task`. In **Argilla**, it is `rg.TaskDistribution(min_submitted=N)`. In **Prodigy**, it is a recipe-level decision (re-stream the same item to multiple sessions). In **Doccano**, it is a workflow pattern rather than a first-class feature — you duplicate items across projects or use the SDK to allocate.

Argilla example:

```python
settings = rg.Settings(
    fields=[rg.TextField(name="prompt"), rg.TextField(name="response_a"), rg.TextField(name="response_b")],
    questions=[rg.LabelQuestion(name="winner", labels=["a", "b", "tie"])],
    distribution=rg.TaskDistribution(min_submitted=3),  # every record collected by 3 annotators
)
```

## Adjudication — the escalation path

**Adjudication** is the human-in-the-loop resolution of disagreements that consensus cannot resolve. In a well-run program, 5–20% of items reach adjudication. If the rate is much higher, the guidelines are under-specified; if it is near zero, your consensus check is too permissive.

An adjudication workflow has three pieces:

1. **Adjudicator role.** Usually the program owner, a senior annotator, or a domain expert. Always a named human, not a vote.
2. **Adjudication queue UI.** The adjudicator sees the item, all `n` annotator labels, all `n` rationales (if captured), and the guidelines-document edge-case catalogue. They choose the final label and (crucially) write a one-sentence rationale.
3. **Feedback into the guidelines.** Every adjudicated item that revealed a new edge case **updates the guidelines**. This is the one feedback loop that is most often missing in practice; without it, adjudicators re-decide the same ambiguities forever.

In Label Studio, the adjudication view is the `Reviews` tab filtered on `agreement < threshold`. In Argilla, it is a filter on records where response-agreement is below threshold. In both tools, the adjudicator can add a `metadata.adjudicated_by` field and a `metadata.adjudication_reason` field that becomes part of the output dataset.

### The adjudication-to-guidelines feedback loop in operational cadence

```
weekly cadence:
  1. program owner reviews adjudicated items
  2. groups them by root-cause ambiguity
  3. for each group with >= 3 items:
     - add an edge-case entry to guidelines.md
     - re-run the calibration IAA on affected items
  4. bump guidelines version if any entry was added
  5. communicate to annotator pool (and vendor PM) in the next session kickoff
```

This loop is the one operational habit that most distinguishes a mature labeling program from an immature one. It is cheap (one or two hours of program-owner time per week) and pays back in weeks.

## The consensus / review / adjudication cost model

Put the four knobs together and you have a cost model you can defend to a budget owner:

```
total_labels_produced = N
annotators_per_item = a            # e.g. 3
review_sample_rate = r             # e.g. 0.15
adjudication_rate = j              # e.g. 0.10 (measured after launch, estimated before)
cost_per_annotator_label = c_a
cost_per_review = c_r
cost_per_adjudication = c_j

total_cost ≈ N * (a * c_a + r * c_r + j * c_j)
```

For a 20k-item preference run at `a=3`, `r=0.15`, `j=0.10`, `c_a=$0.80`, `c_r=$1.20`, `c_j=$5.00`: `20000 * (3 * 0.8 + 0.15 * 1.2 + 0.10 * 5.0) = 20000 * (2.4 + 0.18 + 0.5) = $61,600`. The values are illustrative — your vendor's quoted per-item rates are the real inputs. The *shape* is what matters: consensus dominates cost at the scales we operate on, which is why Chapter 06 (active learning) is the primary budget lever.

## Worked example — a Label Studio text-classification project with review and consensus

```xml
<View>
  <Text name="text" value="$text"/>
  <Choices name="label" toName="text" choice="single">
    <Choice value="support_request"/>
    <Choice value="complaint"/>
    <Choice value="feedback"/>
    <Choice value="other"/>
  </Choices>
  <TextArea name="notes" toName="text" placeholder="Notes for reviewer (optional)" required="false"/>
</View>
```

Project settings (via the Label Studio SDK):

```python
from label_studio_sdk.client import LabelStudio

ls = LabelStudio(base_url="https://labelstudio.internal", api_key=API_KEY)

project = ls.projects.create(
    title="support-ticket-classification-v1",
    label_config=open("label_studio_project.xml").read(),
)

project.update(
    maximum_annotations=3,         # consensus = 3 annotators per item
    overlap_cohort_percentage=100, # 100% of items are co-labeled
    show_overlap_first=True,
    sampling="Uniform sampling",   # randomised order
    review_settings={
        "sample_rate": 0.15,       # 15% of accepted annotations reviewed
        "auto_reject_on_disagreement": False,
    },
)
```

The project is now ready to:

- accept 3 annotators per item,
- surface a disagreement filter for the adjudicator queue,
- sample 15% of accepted items for reviewer QA, and
- version-tag every annotation with `annotator_id`, `created_at`, and the project `label_config` SHA.

## What this chapter is **not**

- Not an IAA chapter. That is Chapter 04 — and until you have the IAA tooling, the "target agreement" field in your guidelines is a placeholder.
- Not a vendor-contract chapter. That is Chapter 05 — the vendor-managed project configures *their* tool with the *same* guidelines and calibration set, and the vendor QA audit reads against the same calibration items.
- Not a preference-labeling-UI chapter. Chapter 08 covers the RLHF-specific protocols that extend consensus and rubric design.

## Summary

- The **guidelines document** is the contract. Write it before configuring the tool, version it like code, and feed adjudication outcomes back into it weekly.
- The **calibration set** is the materialised form of the guidelines — gold-labeled, edge-case-weighted, used for qualification, drift, and IAA baseline.
- The **review queue** is a four-state machine: unlabeled → pending_review → accepted (or needs_rework). Review sampling rate and reviewer ratio are the tunable knobs.
- **Consensus** is the number of annotators per item (`n=3` is the typical production default) and the aggregation rule (majority vote, Dawid-Skene, MACE).
- **Adjudication** is the named-human escalation for unresolved disagreements, with a feedback loop into the guidelines document.
- Chapter 04 pins IAA — the statistic you use to judge whether the consensus and adjudication machinery is actually producing a quality signal.

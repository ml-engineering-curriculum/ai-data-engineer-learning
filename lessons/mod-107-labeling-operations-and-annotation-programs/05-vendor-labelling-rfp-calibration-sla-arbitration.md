# Chapter 05 — Vendor Labelling: RFP, Calibration Set, QA Sample, SLA, Arbitration

## Why this chapter exists

In-house annotation covers small, sensitive, or tightly-specified labeling tasks. Everything larger — tens of thousands of items, 24-hour turnarounds, multi-language coverage — is contracted out to a labeling vendor. The quality of the labels you get back is approximately proportional to how clearly you wrote the contract, not how much you paid per item.

This chapter is the vendor-side operational playbook: the shape of the **RFP**, the **labeling-guidelines contract**, the **calibration set**, the **per-item QA sample policy**, the **quality SLA**, and the **escalation and arbitration** process. It covers the seven vendors the role is accountable for being able to work with: **Scale AI**, **Surge AI**, **Appen**, **Sama**, **iMerit**, **Toloka**, and **SuperAnnotate**.

What you will have at the end of this chapter is a reusable vendor package (RFP + guidelines + calibration + SLA + arbitration) that you can send to any of the seven with minimal per-vendor customisation.

## The seven vendors — one-line positioning

Pin these positionings. They change year to year, but the shape persists.

- **Scale AI** (<https://scale.com/>) — enterprise-focused; strong on CV, autonomous vehicles, defense, and increasingly on RLHF / frontier-lab preference labeling. Largest global scale, highest typical minimum contract size.
- **Surge AI** (<https://www.surgehq.ai/>) — RLHF-first; smaller vetted expert pools, higher cost per item, and visible transparency on labeler recruitment and vetting.
- **Appen** (<https://www.appen.com/>) — long-established crowd + managed services. Broad modality coverage; multi-language reach via a global crowd.
- **Sama** (<https://www.sama.com/>) — managed services with an emphasis on ethically sourced workforce and East African labor pools. CV and text; often selected for ESG / supply-chain-transparency reasons.
- **iMerit** (<https://imerit.net/>) — managed services with specialist teams in medical imaging, geospatial, and financial-document labeling. High quality on bounded expert tasks.
- **Toloka** (<https://toloka.ai/>) — crowdsourcing platform (Yandex-origin, now independent); self-service, task-marketplace model. Lower cost per item, higher QA burden on the buyer.
- **SuperAnnotate** (<https://www.superannotate.com/>) — tooling-first vendor; strong on CV with an in-platform labeling tool. Hybrid managed-services + tools offering.

Three operational differences across the seven that bear on the RFP shape:

1. **Managed vs marketplace.** Scale, Surge, Appen, Sama, iMerit, and SuperAnnotate are managed services — you pay per item, they manage the pool. Toloka is a marketplace — you post a task and crowdworkers self-select. Managed services absorb more QA burden; marketplaces push it to you.
2. **Workforce transparency.** Surge and Sama lead on labeler-population transparency; Scale has been criticised in press and academic literature for opaque pool composition. Ask for it in the RFP regardless (see Perrigo 2023 *Time* reporting on Sama / OpenAI content-moderation labeling for the operational reason this now matters; <!-- needs-research: confirm Perrigo 2023 Time article URL at authoring time. -->).
3. **Minimum contract size.** Scale and iMerit typically require multi-seven-figure annual commitments; Surge, Sama, and SuperAnnotate accept smaller contracts; Toloka has no minimum. This shapes vendor choice for small / pilot programs.

## The RFP — what to include

A **labeling vendor RFP** is a document of 10–25 pages (short for an enterprise RFP) with the following sections. Treat it as a reusable template.

### 1. Program summary (half page)

- One paragraph: the task, the modality, the volume, the quality bar.
- One paragraph: downstream use (pre-training data, SFT, RLHF reward, eval) — vendors price differently depending on what your labels will be used for.
- Timeline: start date, go-live date, SLA deadline.
- Decision timeline: when you expect to pick a vendor, with a hard date.

### 2. Volume and modality

- **Items** — total expected, items per week at steady state, burst capacity required.
- **Modality** — text, image, audio, video, tool-calling trajectory, preference pair. Specify format: file types, average size, number of fields, example JSON.
- **Languages** — list, with expected share of volume per language. If you require native speakers in a specific locale, say so.
- **Platform** — do you require them to use *their* tool, *your* tool (Label Studio / Argilla instance you host), or an API integration?

### 3. Labeling task specification — the attached guidelines

The guidelines document (Chapter 03) is **attached**, not inlined. The RFP body refers to it by SHA and version. This matters: the vendor's quote is against a specific guideline version, and both sides must be able to point at the same document.

### 4. Quality requirements

- **IAA target** — written as a hard threshold with the coefficient named (e.g. "Krippendorff's α ≥ 0.60 on the nominal label, measured on a rolling 1,000-item window, with the vendor's pool against the program's gold labels").
- **Calibration-set pass threshold** — the per-annotator agreement threshold against the gold calibration set that qualifies them for the production pool.
- **QA sample rate** — the fraction of the vendor's output that will be reviewed by the program's reviewers (e.g. 10%), and the acceptable per-batch pass rate.
- **Rework policy** — what happens to items that fail QA.

### 5. Workforce transparency (ask for it; do not assume)

- Country / region distribution of annotators (approximate).
- Working-conditions statement — wage floor, maximum weekly hours, access to welfare support for exposure to harmful content.
- Recruitment and training process summary.
- Whether labelers are employees or contractors.
- Welfare policy for annotators labeling harmful / traumatic content. Required for safety labeling programs.

Vendors differ significantly in what they will disclose. Surge and Sama lead; others vary. The fact that a vendor declines to disclose is itself information for your procurement decision.

### 6. Security and compliance

- Where does the data live? Can the vendor process it in your region? Can it be streamed only, never persisted?
- Which compliance frameworks does the vendor support? (SOC 2 Type II, ISO 27001, HIPAA, GDPR DPA.)
- Which **sub-processor** list does the vendor use? List them.
- If PII can appear, what is the vendor's data-minimisation and redaction policy?
- Does the vendor sign a DPA that matches your organisation's standard?

### 7. Commercial

- Pricing model — per item, per hour, or hybrid.
- Volume pricing tiers, and at what volume.
- Minimum commitment.
- SLAs with teeth — service credits, early-termination rights, caps on monthly fees in exchange for missed quality or timeline.

### 8. Response format

- Fixed-length written responses (e.g. 2 pages per section).
- Three customer references, one of whom is on a comparable program.
- A **free pilot** — 500–2,000 items on your calibration set at the vendor's expense, so you can audit their output yourself. All serious vendors will agree to this; a refusal is a signal.

## The labelling-guidelines contract

The guidelines document (Chapter 03) is **the contract** between you and the vendor. When a vendor's output disagrees with your gold labels, the first question is "which clause of the guidelines did the vendor interpret differently?"

Three vendor-specific practices to add to the guidelines document when you are shipping it to a vendor:

1. **Vendor-visible change log.** Every guideline revision is announced to the vendor PM with a 24–48 hour read-in window before any in-flight labels are produced against the new version.
2. **Vendor-side edge-case log.** The vendor PM runs their own edge-case log mirroring yours. Weekly syncs reconcile both sides' logs into one.
3. **Named adjudicator on each side.** The vendor has a program manager who owns the pool-side adjudication; you have the program owner who owns the final verdict. Both are named in the contract.

## The calibration set — three uses in the vendor context

The same calibration set from Chapter 03 has three vendor-specific uses:

1. **Vendor qualification** — before signing, the vendor labels (part of) the calibration set at their expense during the pilot. Their IAA against your gold labels is the primary quality-gate signal.
2. **Annotator qualification** — every annotator the vendor assigns to the program passes the full calibration set with per-annotator agreement above a defined threshold (typically α ≥ 0.70 per annotator against gold) before entering the production pool.
3. **Drift detection** — on cadence, the vendor's active pool re-labels a sample of calibration-set items. Agreement drift against the baseline triggers a quality review.

The calibration set is **not shared with annotators ahead of time**. Vendors who allow annotators to memorise the calibration set — intentionally or not — will produce inflated IAA numbers and real-world quality failure. The calibration-set items are indistinguishable from production items in the stream.

## The per-item QA sample

Independent of the vendor's own QA, you sample the vendor's output on your side. Typical policy:

- **Random sample** of 10% of the vendor's output per batch is re-labeled by your reviewer pool.
- **Stratified sample** — oversample low-prevalence classes and items the vendor marked low-confidence.
- **Flagged sample** — 100% of items where (a) the vendor pool had internal disagreement, or (b) the item matched an "always-review" rule in the guidelines (e.g. anything involving harmful content, PII, legal claims).

Reviewer verdicts feed:

- A **per-batch pass rate** — the fraction of sampled items where your reviewer agreed with the vendor's label. Below the SLA threshold → batch rework / rejection.
- A **per-annotator agreement** — for vendors who disclose annotator IDs, agreement by annotator. This is the signal that drives the vendor-side "remove this annotator from the pool" conversation.

Keep the QA audit sample versioned and signed. It is part of the lineage record.

## The quality SLA — writing it with teeth

An SLA without consequences is marketing. The three dimensions a labeling SLA should bind:

1. **Quality** — IAA against gold ≥ X on a rolling window, with a measurement interval named.
2. **Volume / timeliness** — N items per week with ≤ Y% late, with a named reference clock.
3. **Responsiveness** — vendor PM responds to guideline questions within Z hours.

Each with a **service credit** attached when missed. A practical structure:

```
Quality SLA:
  metric: Krippendorff's α against gold calibration
  target: ≥ 0.60 on each rolling 1000-item window
  measurement: weekly, on the per-batch QA sample
  miss consequence:
    1st miss in rolling 60 days: written notice + joint root-cause within 5 business days
    2nd miss in rolling 60 days: 10% service credit on the affected batch
    3rd miss in rolling 60 days: right to terminate without penalty + full refund of last batch
```

Service credits should be material — single-digit-percent credits are not consequences. The right to terminate without penalty on repeated misses is the one clause that most reliably drives vendor behaviour.

## Escalation and arbitration — the named path

When you and the vendor disagree on a label and the labels are already affecting a downstream model, you need a path that is faster than "schedule a meeting". Write it into the contract:

```
Escalation path:
  Level 0 (within 4h):     program owner <-> vendor PM, async
  Level 1 (within 24h):    joint call, vendor PM + program owner + vendor QA lead
  Level 2 (within 3 days): written arbitration by the named arbiter
  Level 3 (contractual):   contract-change escalation

Named arbiter:
  First choice: the program's domain expert (named individual)
  Second choice: an agreed third-party reviewer
  Arbiter decisions on label disputes are binding for the current guideline version
  Guideline changes can supersede an arbiter decision only on the next version bump
```

The point is that **the arbiter is named before the dispute arises**. Choosing the arbiter in the middle of a dispute is itself a dispute.

## Workforce welfare — operational, not aspirational

Labeling programs that touch harmful content (safety classification, CSAM detection, content moderation, violence / abuse) carry direct psychological risk to the annotators. The Perrigo 2023 *Time* reporting on OpenAI's Sama-labeled safety data brought this into public discussion; the operational point is older and the obligation is not optional.

Minimum operational requirements, written into the contract:

- **Content-exposure disclosure** — annotators are told what categories of content they will see before being assigned.
- **Hourly caps** — explicit maximum per-day / per-week exposure to harmful content categories.
- **On-call welfare support** — access to a mental-health professional during and after the program.
- **Right of refusal** — annotators can decline specific content categories without penalty.
- **Vendor attestation** — the vendor signs a representation that these policies are implemented.

For programs that do not touch harmful content, these are still worth asking for. They are cheap in the baseline case and absent / expensive to retrofit when you need them.

<!-- needs-research: confirm the Perrigo 2023 Time article URL ("OpenAI Used Kenyan Workers on Less Than $2 Per Hour to Make ChatGPT Less Toxic", Time, 18 Jan 2023) and any follow-up reporting at authoring time. -->

## The vendor-audit cadence

A production vendor program is audited on a weekly / bi-weekly cadence:

```
weekly:
  - per-batch QA pass rate
  - rolling IAA against gold
  - items adjudicated, with root-cause summary
  - calibration-set drift re-measurement

monthly:
  - annotator population review (joiners, leavers, pool size)
  - commercial reconciliation (volume vs commitment, service credits)
  - guideline change log review

quarterly:
  - end-to-end vendor review against RFP response
  - renewal / exit decision point if approaching term end
```

The audit outputs are first-class records — stored, signed, and referenced from the dataset card.

## When to NOT use a vendor

Vendor labeling is the wrong choice when:

- The task is small (≤ 10k items) and finite. In-house + Argilla / Label Studio is usually cheaper and faster.
- The domain expertise required is not reproducible in a vendor pool (e.g. your company's internal product taxonomy, niche scientific domains). An in-house small expert team outperforms a vendor's generalist pool.
- The data cannot leave your environment under any sub-processor arrangement. Self-host Label Studio with internal annotators.
- The task is actively evolving and the guidelines change daily. Vendor cycle times are measured in days; internal cycle times are measured in hours.

Document this reasoning in the program kickoff. "We built this in-house because X" is a defensible decision; "we built this in-house because we never considered a vendor" is not.

## What this chapter is **not**

- Not a per-vendor tutorial. Each vendor's platform documentation covers the mechanics. The shape of the contract is what you own.
- Not a Legal / Procurement document. Your Legal team owns the final MSA. Your job is to write a labeling-specific RFP that Legal can wrap.
- Not a workforce ethics manifesto. The welfare section is operational: the attestations are deliverable artifacts, not moral endorsements.

## Summary

- A labeling vendor is contracted against a **written RFP + guidelines contract + calibration set + QA sample policy + SLA + arbitration path**. All six are reusable; author them once and tune per-engagement.
- The seven vendors in scope (**Scale, Surge, Appen, Sama, iMerit, Toloka, SuperAnnotate**) differ in managed vs marketplace, workforce transparency, and minimum contract size — those three dimensions drive the per-engagement pick.
- The **guidelines document** is the contract: version it, change-log it, name an adjudicator on each side.
- The **calibration set** has three vendor uses: vendor qualification, annotator qualification, drift detection. It is not shared with annotators ahead of time.
- The **SLA has teeth** — service credits for missed quality, right to terminate on repeated misses.
- The **arbitration path and the arbiter are named in the contract, before any dispute arises.**
- **Workforce welfare** is operational, not aspirational. Content-exposure disclosure, exposure caps, on-call support, right of refusal — all written in.

Exercise 02 is where you write the RFP and the vendor-audit flow for a real scenario. Chapter 06 picks up the opposite half of the budget lever — active learning.

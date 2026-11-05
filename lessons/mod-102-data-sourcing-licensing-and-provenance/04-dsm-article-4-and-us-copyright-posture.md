# Chapter 04 — EU DSM Article 4 Reservations, US Copyright Posture, and the Legal Deferral Contract

## Why this chapter exists

The signals in Chapter 03 are mechanical: a header, a tag, a file. What those signals *mean* legally is a separate question. In the EU, the DSM Copyright Directive Article 4 TDM exception gives the signals an operative legal effect. In the United States, there is no equivalent statute; training-data copyright exposure is governed by the common-law fair-use doctrine, which is actively being litigated. Other jurisdictions have their own approaches (the UK's abandoned statutory TDM proposal, Japan's Article 30-4, Singapore's Section 243, South Korea's proposed reforms, Brazil's draft AI legislation).

You are not the legal decision-maker. The point of this chapter is to make you *fluent* in the shape of the legal question so you can:

1. Build the admit function that enforces counsel's current posture.
2. Keep the factual record the pipeline contributes to the regulator-facing package.
3. Know when a change in posture is a code change you flip, not a re-curation you run.

Everything in this chapter is a description of the landscape, not legal advice. Routing legal opinion to counsel is itself the role contract — Section "The legal deferral contract" below codifies it.

## EU DSM Article 4 as the operative substrate

Directive (EU) 2019/790 Article 4 reads (paraphrased for engineering clarity — the authoritative text is on EUR-Lex):

- Member States shall provide for an exception to copyright and database rights for reproductions and extractions of lawfully accessible works for the purposes of text and data mining.
- The exception applies unless the rightsholder has expressly reserved the use in an appropriate manner, such as **machine-readable means** in the case of content made publicly available online.
- The exception applies to any purpose, including commercial.

This is the structural difference between Article 3 (research institutions only, not opt-outable) and Article 4 (anyone, opt-outable). For a commercial ML program, Article 4 is the operative one.

The practical implication for the pipeline:

- Article 4 gives you a *default-permit* position for EU-origin content, provided (a) access is lawful and (b) the rightsholder has not reserved their rights.
- "Lawful access" excludes content behind an unauthorised bypass of technical protection or access controls. Scraping past a paywall or a captcha without authorisation is not Article 4 territory.
- "Reserved in an appropriate manner" includes the machine-readable signals in Chapter 03. A reasonable baseline: a page-level `noai` meta, an `ai.txt` entry disallowing your user agent, a `robots.txt` entry disallowing AI crawlers, or a C2PA / IPTC `do-not-train` metadata tag.
- The EU AI Act Article 53(1)(c) imposes on GPAI providers an obligation to have a policy to comply with Article 4(3). "Policy" here is a document; "compliance" is the admit function.

### What the EU AI Office GPAI Code of Practice adds

The AI Office's General-Purpose AI Code of Practice — final version received by the Commission on 10 July 2025 and approved by the Commission and AI Board on 1 August 2025, organised into three chapters (Transparency, Copyright, Safety and Security) — operationalises Article 53 duties, including the copyright-policy duty (<https://digital-strategy.ec.europa.eu/en/policies/contents-code-gpai>). The Code expects, at minimum:

- A documented copyright policy that is published or made available to the Office.
- Honouring of the main machine-readable opt-out signals (robots.txt, ai.txt, etc.) at ingest.
- Dataset-level documentation of the signals honoured.
- A reasonable attempt to extend compliance to upstream aggregators (Common Crawl, Hugging Face Hub, Zenodo), including by relying on their stated compliance mechanisms.
- A point of contact for rightsholders to raise concerns.

Signatory GPAI providers publish summaries of their adherence to the Code; non-signatories are still bound by the Article 53 duty, they just carry it without the Code's safe-harbour-shaped guidance.

Your pipeline-side evidence — the ingest decision log from Chapter 03 plus the dataset-level documentation from Chapters 05–06 — is the artifact the Code expects you to be able to produce.

## The US copyright posture

The United States has no DSM-equivalent statute. Commercial training on copyrighted material rests on the fair-use doctrine, codified at 17 U.S.C. §107. The four-factor fair-use test (purpose and character of the use; nature of the copyrighted work; amount and substantiality of the portion used; effect of the use upon the potential market) is applied case-by-case.

Two things follow for an AI Data Engineer.

1. **The US Copyright Office's position.** The Office has been publishing a multi-part report on *Copyright and Artificial Intelligence*: Part 1 — Digital Replicas (July 2024); Part 2 — Copyrightability (January 2025); Part 3 — Generative AI Training (pre-publication version released May 2025) (<https://www.copyright.gov/ai/>). The Part 3 analysis speaks to training-data questions but is advisory, not binding. Its central analytical move is that fair use depends heavily on commercial purpose, memorisation / output similarity, and market harm; it stops short of a blanket position.
2. **The open litigation landscape.** Multiple active lawsuits have been brought by rightsholders against AI providers, covering text, code, images, and music; several have reached partial summary judgment, many remain on appeal, and some have settled on licensing terms. Headline rulings to date include *Thomson Reuters v. ROSS Intelligence* (D. Del. Feb. 2025; affirmed by the Third Circuit in 2026), rejecting fair use for training a non-generative legal-search tool on Westlaw headnotes, and *Bartz v. Anthropic* (N.D. Cal. June 2025), holding generative-model training on legally-acquired books to be transformative fair use while rejecting fair use for a permanent library of pirated books (case settled September 2025). The landscape evolves quarterly. Your organisation's legal team tracks a monitoring brief; your pipeline's job is to produce the *factual record* that a defence or a settlement would depend on — what was ingested, under what signals, with what licence tags, with what provenance — not to opine on which side has the better argument.

### The engineering consequence of an unsettled US position

Because US fair-use for training is unsettled, the admit function must be configurable along a *risk-tolerance* axis that counsel sets. In practice this becomes a `us_training_posture` setting on the policy document (Chapter 02) with three positions:

- **Conservative** — ingest only content that is licensed, in the public domain, or opt-out-respecting *and* where the licensor has a documented US training grant. The denylist swallows most web-crawl content.
- **Moderate** — ingest permissively-licensed content + CC-permissive variants + crawl content whose per-site signals permit. US copyright exposure is accepted on the fair-use theory for the remainder, with an explicit `fair_use_claim` tag that counsel has reviewed.
- **Aggressive** — only explicit rightsholder reservations (opt-outs) block ingest. Fair use is claimed broadly. Highest risk; highest training-data volume.

The posture is counsel's. The code that enforces it is yours.

## Other jurisdictions worth knowing exist

For a multi-national ML program you also meet:

- **UK** — a proposed broad TDM exception was abandoned in 2023 after rightsholder opposition. The current UK position is somewhere between the EU Article 4 (with stronger reservation posture) and US fair-use territory. A December 2024 government consultation on a TDM exception with an opt-out mechanism closed in February 2025; in its March 2026 *Report on Copyright and Artificial Intelligence* the UK government stated that the data-mining-exception-with-opt-out proposal is no longer its preferred way forward, with no immediate reform committed.
- **Japan** — Copyright Act Article 30-4 (as amended 2018) permits use of works for *information analysis* purposes, including training, broadly and without an opt-out mechanism. Japan is currently the most TDM-permissive jurisdiction among G7 members for commercial training.
- **Singapore** — Copyright Act 2021 Section 243 creates a computational data analysis exception with specific conditions (lawful access, non-redistribution).
- **South Korea** — a TDM exception is being debated in the context of amendments to the Copyright Act.
- **Brazil** — draft AI legislation and copyright-reform proposals are in motion; the current position is narrow.
- **China** — no statutory TDM exception; the Cyberspace Administration's generative-AI measures and related rules impose data-provenance and consent obligations that function as de facto restrictions.
- **Canada** — Copyright Act does not currently include a training-specific exception; fair-dealing doctrine applies.

For each jurisdiction your product serves, your source manifest carries a `jurisdiction_posture` tag that counsel defines and the admit function enforces. "Serves" is defined by counsel and may include the model's training location, the deployment location, and the user's location.

## The legal deferral contract

This is the single most important paragraph in this module for your day-to-day work.

**You do not decide the legal posture.** Counsel decides. The AI governance analyst (level 15) facilitates. The AI Data Engineer's job is to:

- Produce the *factual record* — what was ingested, under what signals, with what licence tags, with what provenance. The record is complete, signed, and reproducible.
- Build the *admit function* that enforces counsel's current posture. The posture is a policy document; the function is code that reads the policy.
- Hold the *switch* — when counsel's posture changes (a new case settles, a new regulation takes effect, a new Code of Practice provision is adopted), the change is a policy update, not a re-curation unless a historical slice must be filtered.
- Surface *edge cases* — when the admit function encounters a scenario the policy did not anticipate, raise it to counsel through the governance analyst, do not resolve it yourself.
- Keep the *audit trail* — the decisions the admit function made, the inputs it used, and the policy version in force at the time.

What you do **not** do:

- You do not opine on fair use, on the likely outcome of pending litigation, on whether a given work is transformatively used.
- You do not change the posture without counsel signing the policy change.
- You do not accept one-off exceptions ("just this slice") without a ticketed, counsel-signed override logged in the decision trail.
- You do not promise regulators that the system has a property you cannot verify from the artifact.

The deferral contract is the engineering shape of the division of labour that mod-101 Chapter 06 mapped on the level ladder. It applies here because training-data copyright is the single most legally-fraught surface in the pipeline.

## How Article 4 and the signals from Chapter 03 compose

The admit function evaluates each item against: licence (Chapter 02), opt-out signals (Chapter 03), privacy regime (mod-101 Chapter 04), and jurisdictional copyright posture (this chapter). The composite:

```python
# aicg/datapipeline/admit/composite.py
from dataclasses import dataclass
from typing import Literal

Jurisdiction = Literal["eu", "us", "uk", "jp", "sg", "other"]

@dataclass
class CompositeDecision:
    admit: bool
    licence_decision: str
    opt_out_decision: str
    privacy_decision: str
    jurisdiction_decision: str
    posture_in_force: str
    item_sha: str
    policy_version: str

def composite_admit(item, policy, counsel_posture: str) -> CompositeDecision:
    licence_result = policy.license_gate.evaluate(item.licence, item.training_objective)
    opt_out_result = policy.opt_out_evaluator.evaluate(item.url, item.bytes, item.html_head)
    privacy_result = policy.privacy_evaluator.evaluate(item)
    jurisdiction_result = policy.jurisdiction_evaluator.evaluate(
        item,
        counsel_posture=counsel_posture,
    )

    all_admit = all([
        licence_result.decision.name != "DENY",
        opt_out_result.admit,
        privacy_result.admit,
        jurisdiction_result.admit,
    ])

    return CompositeDecision(
        admit=all_admit,
        licence_decision=licence_result.decision.name,
        opt_out_decision=("admit" if opt_out_result.admit else "deny"),
        privacy_decision=("admit" if privacy_result.admit else "deny"),
        jurisdiction_decision=("admit" if jurisdiction_result.admit else "deny"),
        posture_in_force=counsel_posture,
        item_sha=item.sha,
        policy_version=policy.version,
    )
```

The composite is the admit function Chapter 01 of this module referenced and mod-101 Chapter 04 introduced. Its output per item is the audit evidence; its configuration is counsel's posture document.

## What the pipeline hands to counsel

When a legal question lands — a rightsholder complaint, a subpoena, a regulator inquiry, a pending-litigation question — the data you produce on demand:

- The policy document in force at ingest time (Chapter 02).
- The per-item decision log for the implicated items (this chapter's composite).
- The dataset card and Datasheet for each slice involved (Chapter 05).
- The ML-BOM and SPDX AI records tying slices to the model(s) that trained on them (Chapter 06).
- The opt-out signals observed at the time of ingest and the time of the inquiry (Chapter 03), so that any retroactive change in signal is visible.
- The crawl contract and user-agent identity (Chapter 03).

If you can produce these within hours, counsel can respond defensively; if you cannot, counsel must respond conservatively. The engineering difference is largely cadence: continuous logging vs. one-shot archaeology.

## Summary

- EU DSM Article 4 gives a default-permit for TDM on lawfully-accessed content subject to machine-readable reservation; honouring the signals in Chapter 03 is the pipeline-side compliance.
- EU AI Act Article 53(1)(c) imposes a documented copyright policy on GPAI providers; the EU AI Office GPAI Code of Practice operationalises it with expected evidence.
- US copyright posture is unsettled and depends on fair-use case law in active development. The US Copyright Office's AI report and the open litigation landscape move the practitioner baseline quarterly.
- Multiple other jurisdictions have their own regimes; the per-source `jurisdiction_posture` tag lets the pipeline enforce counsel's per-jurisdiction position.
- The legal deferral contract is firm: counsel sets the posture, you build the admit function and keep the record. You do not opine; you do not accept un-ticketed one-off exceptions; you do not promise regulators properties you cannot verify from the artifact.
- The admit function composes licence, opt-out, privacy, and jurisdiction decisions. Its per-item log is the audit record.
- When a legal question lands, the artifact — policy document, per-item log, dataset card, ML-BOM, opt-out history — is what counsel defends with.

Exercise 03 and Exercise 04 both touch this chapter (opt-outs honouring + documentation respectively). Chapter 05 turns to the per-source documentation templates — Datasheets, Data Statements, Croissant, HF Dataset Cards — that make the artifact real.

# Quiz 01 — Sourcing, Licensing, Opt-Outs, Documentation, and Provenance

**Module:** [mod-102 — Data Sourcing, Licensing, Consent, and Provenance Engineering](../README.md)
**Format:** 12 multiple-choice questions, one correct answer per question.
**Suggested time:** 25 minutes.
**Coverage:** Chapters 01–07.

These questions test whether you can *use* the module's artifacts — pick the right source type for an objective, translate a licence clause into a `LicenseGate` decision, honour a stack of opt-out signals correctly, route a legal question to the right role, and name the artifact that satisfies a peer's handoff — not whether you can recite wording. Answer key with references is at the bottom.

---

## Q1. Source type → distinguishing property

A candidate source has these properties: licence is governed by your product's Terms of Service; the strongest provenance in the module (you know the user, the session, and the timestamp); and deletion propagates downstream when a user exercises an Article 17 right. According to Chapter 01, which of the five canonical source types is this?

- **A.** Public web crawl.
- **B.** Licensed partner corpora.
- **C.** Owned user-generated content (UGC).
- **D.** Synthetic data.

---

## Q2. Objective → blend pattern

You are curating a red-team / adversarial evaluation set for a GPAI chat model. According to Chapter 01's objective-specific selection patterns, which source type dominates the blend, and why?

- **A.** Public web crawl — volume dominates; adversarial content is found by scaling the crawl.
- **B.** Owned UGC — judgment dominates; users' real failure cases are the adversarial set.
- **C.** Third-party public benchmarks — independence dominates; red-team sets must be unbiased.
- **D.** Purpose-built adversarial sets (licensed from specialised vendors, authored in-house by `ai-risk-engineer` peers, or distilled from known-bad examples) — specificity dominates; public web crawl is almost never used.

---

## Q3. CC-BY-SA's share-alike clause onto model weights

Chapter 02 characterises the "viral scope" of the CC-BY-SA share-alike clause onto a trained model's weights as:

- **A.** A settled licensing question — the organisation must propagate share-alike onto the model weights.
- **B.** A settled licensing question — the organisation may freely use CC-BY-SA with no propagation.
- **C.** An unresolved *legal* question. The pipeline carries the CC-BY-SA tag so the slice is identifiable if counsel's posture changes; different organisations take different postures.
- **D.** Not a question — CC-BY-SA is not permitted for commercial training under any posture.

---

## Q4. The `LicenseGate` default

Your crawl ingests a document whose source carries no identifiable licence string at all — not a CC code, not an SPDX short-form, not a product ToS tag, not a DLA reference. According to Chapter 02's allowlist / denylist policy, what does the `LicenseGate` return?

- **A.** `ADMIT` with a "warn — unlabelled" note; the pipeline backfills classification later.
- **B.** `ADMIT_WITH_CONDITION` and routes the item to counsel for a one-off opinion.
- **C.** `DENY` — the policy's `default-deny` rule treats an unlabelled source as not admissible. The policy's defaultness is itself a policy claim.
- **D.** `ADMIT` under an assumed CC-BY licence, because CC-BY is the most common licence on the open web.

---

## Q5. robots.txt — most-specific `User-agent` match

Your crawler identifies as `ExampleCorpCrawler/1.0`. The target's `/robots.txt` contains two records:

```
User-agent: GPTBot
Disallow: /

User-agent: *
Disallow: /admin/
Crawl-delay: 10
```

According to Chapter 03 and RFC 9309, what is the correct crawl behaviour for `ExampleCorpCrawler/1.0`?

- **A.** Deny all paths — the `GPTBot` block applies to every AI-adjacent crawler.
- **B.** Deny `/admin/`, honour the 10-second `Crawl-delay`, allow everything else — the `User-agent: *` record is the most-specific match for `ExampleCorpCrawler/1.0`.
- **C.** Deny nothing — `ExampleCorpCrawler/1.0` is not named in a record, so no record applies.
- **D.** Switch to identifying as a general-purpose browser so neither record matches, then crawl everything.

---

## Q6. `llms.txt` is not an opt-out

A publisher serves `/llms.txt` at the site root. According to Chapter 03, how should the ingest pipeline treat its presence?

- **A.** As a blocking signal — its presence denies training-data admission.
- **B.** As an inclusion signal oriented at LLM *reasoning* rather than *training*; it is not an opt-out. The normal licence and opt-out analysis still applies.
- **C.** As a direct replacement for `robots.txt`.
- **D.** As a DSM Article 4(3) machine-readable reservation on its own.

---

## Q7. EU DSM Article 3 vs Article 4

Chapter 04 distinguishes Article 3 of the EU DSM Copyright Directive from Article 4. Which statement correctly describes the distinction a commercial ML program cares about?

- **A.** Article 3 is for anyone and is opt-outable; Article 4 is for research institutions only and is not opt-outable.
- **B.** Article 3 is for research institutions only and is not opt-outable; Article 4 applies to any purpose including commercial and is opt-outable via machine-readable reservation on lawfully-accessed content. Article 4 is the operative one for a commercial ML program.
- **C.** Both articles are opt-outable; the distinction is strictly territorial.
- **D.** Article 4 is about privacy and Article 3 is about copyright; they do not compose.

---

## Q8. The legal deferral contract

A product manager asks you, the AI Data Engineer, to confirm in writing that the slice's use is "fair use under US copyright law". According to Chapter 04's legal deferral contract, what is the correct response?

- **A.** Perform the four-factor fair-use analysis yourself and record the opinion in the slice manifest.
- **B.** Confirm fair use because the US Copyright Office's Part 3 report acknowledges that generative-AI training can be fair use.
- **C.** Route the opinion to counsel. Your deliverable is the *factual record* — what was ingested, under what signals, with what licence tags, with what provenance — not a legal conclusion. You do not opine on fair use.
- **D.** Refuse to engage; the AI Data Engineer has no role in this question at all.

---

## Q9. Data Statements for NLP vs the rest of the documentation set

Your slice is a linguistic corpus (customer-support dialogue). According to Chapter 05, which artifact — specifically — should accompany the Datasheet so that the linguistic grounding an NLP evaluator needs is on file?

- **A.** A Model Card (Mitchell et al. 2019) — Model Cards subsume Data Statements.
- **B.** A Data Statement for NLP (Bender & Friedman 2018) covering curation rationale, language variety (BCP-47 plus prose), speaker and annotator demographic, speech situation, and text characteristics.
- **C.** A Dataset Nutrition Label only — a one-page summary is enough.
- **D.** Nothing additional — Datasheets for Datasets fully cover NLP corpora without supplement.

---

## Q10. Three provenance records

A procurement team at a customer asks for "asset-level provenance for the multimodal images in your training corpus, cryptographically signed and verifiable per asset". Chapter 06 names three supply-chain record families. Which one is the correct answer?

- **A.** CycloneDX ML-BOM — it is the OWASP-sponsored line, popular with security and procurement teams.
- **B.** SPDX 3.0 Dataset profile — it is ISO-standardised.
- **C.** C2PA Content Credentials — cryptographically-signed provenance records bound to multimodal assets (images, video, audio), signed by the publishing device or platform.
- **D.** The Hugging Face Dataset Card — it carries the catalog metadata.

---

## Q11. DPIA input bundle contents

The `ai-governance-analyst` is drafting a DPIA for a slice. According to Chapter 07, the `dpia_input_bundle.yaml` you hand them must reference which set of artifacts?

- **A.** The slice's Model Card — everything else is the governance analyst's to gather independently.
- **B.** Only the Croissant and Hugging Face card metadata.
- **C.** The selection matrix, the licence policy in force, the ingest decision log, the Datasheet + Data Statement + Nutrition Label, the PII report, the retention record, and the ML-BOM + SPDX AI records — seven artifacts keyed to the slice.
- **D.** The AI Act Annex XI Section 1 technical-documentation package and the model-side ML-BOM.

---

## Q12. Who signs the Article 53(1)(d) public training-content summary

Under EU AI Act Article 53(1)(d), GPAI providers must publish a sufficiently detailed summary of training content using the AI Office template. According to Chapter 07, who supplies what, and who signs?

- **A.** The AI Data Engineer signs off on the summary; counsel supplies the inputs.
- **B.** The AI Data Engineer supplies the input YAML (source-type breakdown, licensed-partner counterparties, third-party datasets, opt-out signals honoured, point of contact, copyright-policy pointer); the `ai-evaluation-engineer` merges across N slices and renders the summary; the organisation declares. The AI Data Engineer does not sign the summary itself.
- **C.** The AI Governance Analyst signs off; the Risk Engineer supplies the inputs.
- **D.** The DPO signs off; the AI Data Engineer supplies the inputs.

---

## Answer key

**Q1 — C.** Chapter 01 §"Owned user-generated content (UGC)" names these three distinguishing properties exactly: licence is the product T&Cs, provenance is strongest (user / session / timestamp), and deletion is a hard requirement propagated downstream when a user exercises an Article 17 right. Web crawl's licence is per-site; licensed-partner is contractual; synthetic's licence is downstream of the generator's.

**Q2 — D.** Chapter 01 §"Objective-specific selection patterns" is explicit: for red-team / adversarial, "specificity dominates. The blend is near 100 % purpose-built adversarial sets: licensed from specialised vendors, authored in-house by `ai-risk-engineer` peers, or distilled from known-bad examples. Public web crawl is almost never used." A, B, C are each the pattern for a different objective (pretraining, preference / RLHF, evaluation / benchmark).

**Q3 — C.** Chapter 02 §"Creative Commons (CC) licences" §caveats says exactly: "CC-BY-SA viral scope for model weights is a legal question, not a licensing question. The share-alike clause binds 'adaptations.' Whether a trained model is an adaptation of the input data is unsettled. Different organisations take different postures; counsel sets your organisation's. The pipeline still carries the tag so the slice is identifiable if the posture changes." A and B are both settled-position claims that the chapter explicitly rejects; D is wrong because CC-BY-SA is at worst conditionally allowlisted.

**Q4 — C.** Chapter 02 §"Default-deny is the only safe default": "The default behaviour for unlabelled sources must be *deny*, not *admit with warning*. … The gate's defaultness is itself a policy claim." The policy snippet in the chapter has `tag: unlabelled` as a `denylist` entry whose reason is "default-deny — a source with no identified licence is not admissible." The gate's final return path for a tag that matches neither list is also `DENY, "unlabelled-or-not-in-policy"`.

**Q5 — B.** Chapter 03 §"robots.txt per RFC 9309" works this exact example: a crawler identifying as anything other than the named AI agents "Deny `/admin/`, 10-second crawl delay, allow everything else." The most-specific matching `User-agent` wins, and for a crawler that is not `GPTBot` the `User-agent: *` record is the match. Option D (spoofing a browser to evade AI-specific denials) is explicitly called out as unsafe and unethical.

**Q6 — B.** Chapter 03 §"llms.txt" is explicit: "For training-data purposes, `llms.txt` is not an opt-out signal; it is an inclusion signal. … The presence of `llms.txt` alone does not grant training permission — the normal licence and opt-out analysis still applies."

**Q7 — C.** Chapter 04 §"EU DSM Article 4 as the operative substrate": "This is the structural difference between Article 3 (research institutions only, not opt-outable) and Article 4 (anyone, opt-outable). For a commercial ML program, Article 4 is the operative one." Article 4 applies to any purpose including commercial and requires machine-readable reservation on publicly-available online content.

**Q8 — C.** Chapter 04 §"The legal deferral contract" is the single most important paragraph in the module on this point: "You do not opine on fair use, on the likely outcome of pending litigation, on whether a given work is transformatively used." The engineer's job is the factual record and the admit function that enforces counsel's current posture. A and B both have the engineer opining; D is the opposite error — refusing to engage even to supply the record.

**Q9 — B.** Chapter 05 §"Data Statements for NLP (Bender and Friedman 2018)": "For linguistic sources, Bender and Friedman … is the complementary template. Where Datasheets cover the general lifecycle, Data Statements add the **linguistic-grounding** that NLP evaluation requires." The chapter lists exactly the sections named in B. Chapter 05 also notes Model Cards are *paired model-side* documentation authored by the model trainer, so A is wrong.

**Q10 — C.** Chapter 06 §"C2PA Content Credentials and the Content Authenticity Initiative": "The Coalition for Content Provenance and Authenticity (C2PA) publishes a technical specification for **Content Credentials** — cryptographically-signed provenance records bound to multimodal assets." A and B are the right answers for other questions (source-level SBOM for security / procurement; ISO-standardised SBOM); D is the catalog metadata, not a provenance record.

**Q11 — C.** Chapter 07 §"Handoff 1 — `ai-governance-analyst` and the DPIA" lists these seven artifacts in order: (1) selection matrix, (2) licence policy, (3) ingest decision log, (4) Datasheet + Data Statement + Nutrition Label, (5) PII report, (6) retention record, (7) ML-BOM + SPDX AI records. The `dpia_input_bundle.yaml` sample in the chapter enumerates these same files.

**Q12 — B.** Chapter 07 §"The AI Act Article 53(1)(d) summary" and §"What you do not do, in all three handoffs" together: "You do not sign off on the AI Act Article 53(1)(d) summary — you supply the input (evaluation engineer signs; the organisation declares)." The sample `article_53_1_d_summary_input.yaml` enumerates the data-side inputs the engineer owns: source breakdown, licensed-partner counterparties, third-party datasets, opt-out signals honoured, point of contact, and the copyright-policy pointer.

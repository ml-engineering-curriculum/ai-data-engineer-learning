# Quiz 01 — Governance Frameworks and the AI Data Engineer Role Scope

**Module:** [mod-101 — AI Training-Data Governance Frameworks and the AI Data Engineer Position](../README.md)
**Format:** 10 multiple-choice questions, one correct answer per question unless stated otherwise.
**Suggested time:** 20 minutes.
**Coverage:** Chapters 01–06.

These questions test whether you can *use* the frameworks as a data engineer — translating a clause into the pipeline artifact that satisfies it, picking the right regime for a scenario, routing a request to the right role — not whether you can recite wording. Answer key with references is at the bottom.

---

## Q1. NIST AI RMF sub-category → artifact

A stakeholder cites a MEASURE 2 sub-category paraphrased as *"representativeness of the training data vs. the deployment population is measured, logged, and attached to the slice."* Which concrete pipeline artifact does Chapter 01 say satisfies this sub-category?

- **A.** A GOVERN-function policy document signed by the AI governance council.
- **B.** A representativeness audit per source and per curated slice (Dodge-style — what filters strip, demographic composition pre- and post-filter).
- **C.** A SOC 2 Type II report from the cloud-storage provider that hosts the raw corpus.
- **D.** An Annex XI technical-documentation package drafted by the fine-tuning engineer.

---

## Q2. The three NIST documents and when to use each

A product partner asks for "NIST AI RMF evidence" for a new image-generation feature that the model team is adding. According to Chapter 01, which document should you re-read *because the model has crossed a new GenAI capability threshold*?

- **A.** AI 100-1 (AI RMF 1.0) — because the core framework needs to be re-read whenever a capability changes.
- **B.** AI 600-1 (Generative AI Profile) — it adds GenAI-specific risks and suggested actions that are additive.
- **C.** The NIST AI RMF Playbook — it is the normative source for new capabilities.
- **D.** NIST SP 800-53 — it covers AI security controls.

---

## Q3. ISO/IEC 42001 Annex A.7 scope

An internal auditor asks "where is your evidence for ISO/IEC 42001 A.7.4 Quality of data for AI systems?" According to Chapter 02, what do you point them at?

- **A.** The organisation's AIMS scope statement from clause 4.
- **B.** The CycloneDX ML-BOM for the deployed model.
- **C.** A quality-measure log per slice against ISO/IEC 25012 characteristics and ISO/IEC 5259-2 measures, produced by the mod-104 toolchain.
- **D.** The incident-response runbook from the SOC.

---

## Q4. ISO/IEC 25012 proxies

You are populating a 25012 scorecard for a labelled slice. For the *accuracy* characteristic, which of the following is a valid *measurable proxy* in the shape Chapter 02 recommends?

- **A.** "Accuracy = truth" — the dataset is accurate if it matches reality.
- **B.** "Accuracy = agreement with a gold calibration set labelled by two senior reviewers."
- **C.** "Accuracy = the slice passed internal review" — a human signoff is the proxy.
- **D.** "Accuracy = the model trained on this slice beats the previous baseline on the production KPI."

---

## Q5. EU AI Act Article 10 vs. Article 53

Your organisation is building two systems: (i) an internal CV-screening tool for hiring, deployed in the EU; (ii) a 7B open-weights chat model you intend to place on the market. For the training-data slice of each, which EU AI Act provisions apply?

- **A.** Article 10 for both — all training data falls under Article 10.
- **B.** Article 10 for (i) because it is a high-risk system under Annex III; Article 53 + Annex XI + Annex XII + the Article 53(1)(d) public training-data summary for (ii) because it is a GPAI model.
- **C.** Article 53 for both — Article 53 governs all providers.
- **D.** Neither — the EU AI Act does not reach training data.

---

## Q6. The DSM Article 4 admit decision

Your crawler encounters a web page that (i) has a `robots.txt` with `Disallow: /` for AI crawlers, (ii) contains personal data of an EU resident, and (iii) has no `ai.txt` or C2PA opt-out tag. What does the admit function from Chapter 04 do, and why?

- **A.** Admit the item — only `ai.txt` and C2PA tags count as Article 4(3) reservations.
- **B.** Admit the item — personal data is not relevant to a copyright decision.
- **C.** Deny the item — `robots.txt` per RFC 9309 is a machine-readable reservation that Article 4(3) honours, so the content is reserved; the admit function returns `False`.
- **D.** Admit the item but tag it for later review — robots.txt is advisory only.

---

## Q7. GDPR Article 6 — the lawful-basis tag

Your source manifest marks a web-crawl source as `lawful_basis: 6(1)(f) legitimate interests`. According to Chapter 04, what additional evidence pointer must accompany that tag for the source to be defensibly admitted?

- **A.** An EDPB-approved certificate issued per-source.
- **B.** A documented Legitimate Interests Assessment (LIA) ID, with the three-step test (purpose, necessity, balancing) per EDPB Opinion 28/2024.
- **C.** Explicit consent from each affected data subject recorded in the consent store.
- **D.** Nothing — legitimate interests is self-executing.

---

## Q8. SR 11-7 vocabulary at a US bank

You are the AI Data Engineer at a US bank. The model risk management team says the training-data slice needs "effective challenge" before the credit-decisioning model goes to production. According to Chapter 05, what does that translate to on the pipeline side?

- **A.** A penetration test of the training infrastructure.
- **B.** An independent, qualified reviewer — typically a non-developer — signs off on the slice before release; the reviewer's identity is tagged into the release gate.
- **C.** A new model version trained from scratch and compared head-to-head.
- **D.** Posting the dataset card publicly on the bank's website.

---

## Q9. The deferral contract — routing a request

A product manager asks you, the AI Data Engineer, to "fix the hallucination mode" by re-training on better examples. According to Chapter 06, what is the right response?

- **A.** Accept and start designing the SFT mix yourself — you own training data.
- **B.** Refuse outright with no routing — the request is not yours to handle.
- **C.** Route the modelling/SFT design to `fine-tuning-engineer` (L30); you supply the curated substrate and the per-slice evidence, but you do not design the SFT recipe.
- **D.** Escalate to `staff-ml-engineer` (L40) — only staff can decide on re-training.

---

## Q10. Who owns which Annex XI section slice?

Your organisation is placing a GPAI model on the EU market. The compliance team is assembling the Annex XI Section 1 technical-documentation package. According to Chapters 03 and 06, which slice of the Annex XI package is *yours* (the AI Data Engineer) to own and populate?

- **A.** Computational resources used and energy consumption.
- **B.** The model's intended use and acceptable use policy.
- **C.** The "information on the data used" slice — type and provenance of data, curation methodology, how obtained and selected, measures to detect unsuitable data sources, measures to detect identifiable biases — assembled from your per-slice source manifests, licence matrix, preparation log, dedup report, contamination firewall report, PII report, and bias audit.
- **D.** The Article 51 systemic-risk classification decision.

---

## Answer key

**Q1 — B.** Chapter 01's sub-category → artifact table maps MEASURE 2 representativeness directly to the *representativeness audit per source and per curated slice (Dodge-style)* produced by the mod-104 toolchain. A is a GOVERN-layer artifact owned by the governance council, not the data engineer; C and D are unrelated.

**Q2 — B.** Chapter 01's reading discipline explicitly says to *"re-read AI 600-1 whenever the model crosses a GenAI capability threshold (first text generator, first image generator, first agent)"* because the GenAI-specific risks and suggested actions are additive. The Playbook is non-normative; AI 100-1 is a reference index.

**Q3 — C.** Chapter 02's Annex A.7 table maps A.7.4 Quality of data to the *quality-measure log per slice against ISO/IEC 25012 characteristics and ISO/IEC 5259-2 measures*, produced by the mod-104 toolchain (Great Expectations / Soda / Deequ / pandera).

**Q4 — B.** Chapter 02 recommends *operational* proxies — the example given is literally *"Accuracy = agreement with a 200-item calibration set labelled by two senior reviewers."* A is explicitly rejected ("Accuracy = truth is not a proxy"); C and D measure something else.

**Q5 — B.** Chapter 03 distinguishes the two regimes: Article 10 applies to high-risk systems under Article 6 / Annex III (hiring is Annex III employment); Article 53 + Annex XI + Annex XII + the public training-data summary apply to GPAI model providers. A GPAI model is not itself a high-risk system but can be a component of one.

**Q6 — C.** Chapter 04 lists `robots.txt` per RFC 9309 as a form in which an Article 4(3) reservation can be expressed. The admit function denies items from reserved sources regardless of their personal-data status. (The personal-data question is separate and would require its own Article 6 basis.)

**Q7 — B.** Chapter 04 is explicit: legitimate interests requires a documented LIA, and cites EDPB Opinion 28/2024 (17 December 2024) for the three-step test (purpose, necessity, balancing). The `evidence_pointer` field in Exercise 04's lawful-basis record is where the LIA ID lives.

**Q8 — B.** Chapter 05 defines *effective challenge* as *"independent, qualified, critical review of models by parties not involved in development"* and translates it on the pipeline side to a *data-side second reviewer* who was not part of curation reviewing the slice before release, tagged into the release gate.

**Q9 — C.** Chapter 06's deferral contract explicitly routes *"SFT recipe design or DPO/ORPO objective design"* to `fine-tuning-engineer` (L30). The AI Data Engineer supplies the substrate and the per-slice evidence; the SFT designer decides the mix.

**Q10 — C.** Chapter 03 names the "information on the data used" slice of Annex XI Section 1 as the data engineer's — type and provenance of data, curation methodology, measures to detect unsuitable sources, measures to detect identifiable biases — and Chapter 06 reinforces that model-level description, compute, and energy are owned by other roles.

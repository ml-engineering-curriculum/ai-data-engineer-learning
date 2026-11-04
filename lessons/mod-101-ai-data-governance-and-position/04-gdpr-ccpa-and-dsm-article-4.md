# Chapter 04 — The Privacy and Copyright Substrate: GDPR, CCPA/CPRA, and EU DSM Article 4

## Why this chapter exists

The frameworks in Chapters 01–03 set *what good training-data documentation looks like*. This chapter sets *what you are legally allowed to ingest in the first place*. Two legal regimes dominate for an AI Data Engineer:

- **Personal-data regimes** — GDPR (EU) and CCPA/CPRA (California), with equivalents accumulating in other US states and in other jurisdictions.
- **Copyright regimes** — the EU DSM Copyright Directive (Directive (EU) 2019/790) Article 4 text-and-data-mining (TDM) exception with its opt-out mechanism; US fair-use doctrine (currently in active litigation for training data — do not rely on it without counsel); other jurisdictional variants.

You are not the legal decision-maker. The `ai-governance-analyst` (level 15), `ai-privacy-engineer` (in the Governance curriculum), and in-house counsel decide lawful basis and licence posture. Your job is to make those decisions **enforceable in the pipeline** — to have the switches, filters, and records that let legal say "yes, this slice is defensible" and to have the receipts if a data subject, rights-holder, or regulator asks.

This chapter walks the specific Articles and sections you must be able to recognise, and the pipeline switch each one creates.

## GDPR: the Articles that land on training data

GDPR (Regulation (EU) 2016/679) applies whenever personal data of EU/EEA data subjects is processed, regardless of where the processor is located. Training data pipelines routinely contain personal data (user text, forum posts, code commits, scraped web content with PII); treat GDPR as default-on until you have a documented basis to rely on exception or anonymisation.

### Article 5 — principles of processing

Article 5(1) lays out the principles: lawfulness/fairness/transparency, purpose limitation, data minimisation, accuracy, storage limitation, integrity and confidentiality, and accountability (5(2)).

Pipeline consequences:

- **Purpose limitation** — the dataset card must declare the purpose(s) for which the slice may be used, and re-use for substantially different purposes requires a fresh lawful-basis review.
- **Data minimisation** — collect only what you need. For a translation model, you do not need author identity; strip it at ingest.
- **Accuracy** — tie-in to ISO/IEC 25012's accuracy characteristic and the mod-104 quality log.
- **Storage limitation** — slices have a documented retention window; the lineage system enforces deletion at expiry.
- **Accountability (5(2))** — you must be able to *demonstrate* compliance. Everything is logged.

### Article 6 — lawful basis

Processing personal data requires a lawful basis. The relevant ones for training data are:

- **6(1)(a) Consent** — opt-in from the data subject. Useful for first-party training data, user-submitted content with explicit consent, and research panels. Unworkable for web-scale crawl.
- **6(1)(b) Contract** — necessary for performance of a contract with the subject. Rare for pretraining; relevant for first-party data where the product T&Cs explicitly cover training.
- **6(1)(f) Legitimate interests** — balancing test. Commonly cited for training data, but requires a documented Legitimate Interests Assessment (LIA) and does not work for Article 9 special categories. The EDPB has issued opinions constraining how legitimate interests can be relied on for LLM training. <!-- needs-research: cite the current EDPB opinion on legitimate interests for AI training (Opinion 28/2024 and any superseding guidance). -->

Pipeline consequence: every source in the manifest carries a `lawful_basis` tag. Legitimate-interests sources are gated behind an LIA ID; consent-based sources are gated behind the consent store.

### Article 9 — special categories of personal data

Special categories (racial/ethnic origin, political opinions, religious beliefs, trade-union membership, genetic/biometric data, health, sex life/sexual orientation) have a tougher regime. Processing is prohibited unless a 9(2) exception applies. The exceptions relevant to training data include explicit consent (9(2)(a)), manifestly-public data (9(2)(e) — narrow), and processing necessary for reasons of substantial public interest (9(2)(g) — rarely applicable to private AI training).

Pipeline consequence:

- Special-category detectors run at ingest. Presidio (mod-110) can flag some categories; others require purpose-built classifiers.
- A slice containing special-category data must have a documented 9(2) basis attached, verified by legal.
- EU AI Act Article 10(5) permits special-category processing *to detect and correct bias* under strict conditions — but this is a narrow exception you do not invoke unilaterally.

### Article 14 — transparency when data is not collected from the subject

When personal data is collected from a source other than the data subject (e.g. a web crawl), Article 14 requires the controller to provide the subject with specific information including identity of the controller, purposes of processing, categories of data, source of the data, and data-subject rights. The information must be provided within one month unless an exception in Article 14(5) applies — notably 14(5)(b) where provision proves impossible or would involve a disproportionate effort, in which case appropriate measures must be taken (e.g. public notice).

Pipeline consequence: your organisation's privacy notice must cover what training data you ingest and how a subject can exercise their Article 15 (access), 16 (rectification), 17 (erasure), 18 (restriction), 20 (portability), and 21 (objection) rights. You maintain a **data-subject-request pipeline** — the technical ability to look up whether a particular subject's identifier appears in a slice, and to remove or restrict it. For large corpora this is non-trivial and often relies on hashed indices and signed-manifest lookups.

### Article 22 — automated decisions

Article 22 gives data subjects the right not to be subject to a decision based solely on automated processing which produces legal effects. For data engineers this is primarily downstream — the model's eventual use determines whether Article 22 applies — but the slice should carry a tag indicating whether it has been curated for an Article 22-adjacent use case (e.g. credit decisioning, hiring), because such use cases pick up additional obligations (e.g. meaningful information about the logic involved, Article 13(2)(f) / 14(2)(g) / 15(1)(h)).

## CCPA / CPRA: the sale-or-sharing axis

The California Consumer Privacy Act (as amended by the California Privacy Rights Act — effective 1 January 2023) governs personal information of California residents. For training-data pipelines two concepts matter:

- **Sale** — exchange of personal information for monetary or other valuable consideration.
- **Sharing** — disclosure of personal information for cross-context behavioural advertising, whether or not for consideration.

The CPRA added **"sharing"** as a distinct obligation, and extended consumer rights to include correction and limitation of use of sensitive personal information. The California Consumer Privacy Agency (CPPA) issues implementing regulations, including specific rules relevant to automated decisionmaking technology (ADMT). <!-- needs-research: cite current CPPA ADMT regulations status; this is evolving. -->

Pipeline consequence for CCPA/CPRA:

- A **Global Privacy Control (GPC)** signal from a California resident is an opt-out of sale/sharing. If you ingest first-party data, the pipeline must respect the opt-out.
- The source manifest for California-resident data carries a `sale_sharing_status` field. Downstream contracts with labelling vendors and third-party dataset providers propagate the status.
- Sensitive personal information (SPI) under CPRA has limitation-of-use rights. Treat SPI with the same gating as GDPR Article 9 data.

Other US states (Virginia, Colorado, Connecticut, Utah, Texas, Oregon, Montana, and growing) have broadly similar regimes; the `ai-governance-analyst` maintains the state-by-state matrix.

## EU DSM Copyright Directive Article 4 — the TDM exception and opt-out

EU Directive 2019/790 (the DSM Copyright Directive) created two text-and-data-mining exceptions:

- **Article 3** — TDM for research purposes by research organisations and cultural heritage institutions. Not usable by for-profit AI providers.
- **Article 4** — TDM for any purpose by anyone, **subject to a reservation (opt-out)** by rightsholders expressed in machine-readable form for content made publicly available online.

Article 4 is the operative one for commercial AI training. It gives you a positive right to mine lawfully accessible content, **unless** the rightsholder has reserved their rights. The reservation can be expressed via:

- **robots.txt** per RFC 9309 — directives against general web crawlers, which can be read as reservations.
- **ai.txt** / **llms.txt** / `noai` / `noimageai` — AI-specific opt-out signals pushed by publishers and industry groups.
- **Platform terms** (e.g. per-site Terms of Service).
- **Per-asset metadata** — IPTC, C2PA Content Credentials, XMP tags.

Pipeline consequence:

- A **crawl-contract compiler** reads opt-out signals at ingest and marks content as *reserved*. Reserved content is either dropped or routed into a licensed-only partition depending on your organisation's posture.
- EU AI Act Article 53(1)(c) obliges GPAI providers to have a policy to comply with Article 4(3). "Compliance" means your pipeline honours the reservation; "policy" means the dataset card references the opt-out matrix.
- The IETF **AI Preferences** working group is developing standards-track opt-out signalling. <!-- needs-research: verify current status, published RFCs, and interaction with existing robots.txt / ai.txt practice. -->

### US copyright for training data: what you can and cannot say

US copyright does not have an Article 4 analogue. Providers currently rely on *fair use* for training on copyrighted material, which is actively being litigated (multiple cases pending as of this writing). Your organisation's legal position on US training data is something counsel declares; you do not. <!-- needs-research: note the status of key pending US fair-use training cases at read time, without opining. -->

Pipeline consequence: the per-source `licence` tag and `jurisdiction_posture` tag lets you build a slice that matches the organisation's current legal risk tolerance. When counsel's position changes, the change is a filter update, not a re-curation.

## How the privacy and copyright layers combine

A single piece of content can be simultaneously:

- Copyrighted (DSM Article 4 or US fair-use context applies).
- Personal data of a data subject (GDPR / CCPA apply).
- Special-category personal data (GDPR Article 9 applies).
- Opted-out by the rightsholder (robots.txt / ai.txt / Article 4(3) reservation).
- Opted-out by the data subject (GDPR Article 21 objection, GPC signal).
- Marked up with provenance (C2PA Content Credentials).

Your pipeline must be able to evaluate all of these at the per-item level. The typical shape:

```python
# ingest_filter.py — per-item lawful-use evaluation at crawl time
from aicg.datapipeline import Item, CrawlContract

def admit(item: Item, contract: CrawlContract) -> bool:
    if item.source_reserved(contract):         # Art 4(3) reservation / robots.txt / ai.txt
        return False
    if item.contains_pii and item.lawful_basis is None:
        return False                            # GDPR Art 6 — no basis, no ingest
    if item.special_category and not item.article_9_exception:
        return False                            # GDPR Art 9
    if item.california_resident and item.gpc_signalled:
        return False                            # CCPA/CPRA sale-sharing opt-out
    if item.c2pa_opt_out:
        return False                            # per-asset provenance opt-out
    return True
```

The admit function is the compiled form of the organisation's legal posture. Legal signs off on the posture and the test; you own the code. The test's output is logged per item so a later audit can reconstruct *why* an item was admitted or dropped.

## Interaction with the frameworks in Chapters 01–03

- NIST AI RMF's GenAI Profile *Data Privacy* risk and *Intellectual Property* risk map directly to this chapter. Your admit function is the pipeline-side mitigation.
- ISO/IEC 42001 Annex A.7.3 (acquisition of data) is implemented by this admit function plus the crawl contract.
- EU AI Act Article 53(1)(c) copyright-policy obligation is enforced by this admit function.
- The GDPR / CCPA lawful-basis tags feed Article 10(2)(b) data-collection documentation for high-risk systems.

Everything fits together.

## Summary

- GDPR Articles 5, 6, 9, 14, 22 are the training-data-relevant provisions. Each one becomes a tag or a filter in the per-item admit function.
- CCPA/CPRA adds sale-sharing opt-out and SPI limitation. Honour GPC signals and propagate status through vendor contracts.
- EU DSM Article 4 gives you a positive TDM right subject to opt-out reservation. Honour the reservation at ingest; it is also the EU AI Act Article 53(1)(c) obligation.
- US copyright for training data is unsettled; your pipeline carries a `jurisdiction_posture` tag and legal owns the posture.
- The pipeline's admit function is the compiled form of the legal posture. Legal signs the posture; you own the code and the log.

Exercise 04 walks you through drafting the lawful-basis record for a mixed-source training corpus and building the admit function. Chapter 05 adds the sector-specific regimes.

# Chapter 05 — Sector and State Regimes: SR 11-7, FDA GMLP/PCCP, Colorado AI Act

## Why this chapter exists

The frameworks and regulations in Chapters 01–04 are the baseline. On top of them, specific sectors and US states add their own flavour of the same training-data-documentation obligation. The three you will meet most often as an AI Data Engineer are:

- **SR 11-7 / OCC 2011-12** — joint Federal Reserve and Office of the Comptroller of the Currency guidance on **model risk management** for US banks and bank holding companies.
- **FDA Good Machine Learning Practice (GMLP) and Predetermined Change Control Plans (PCCP)** — US FDA guidance for AI/ML-enabled **medical devices**.
- **Colorado AI Act (Senate Bill 24-205)** — the first US state-level **deployer/developer** regime for high-risk AI systems, effective 1 February 2026.

Each one is a *flavour* of the same artifact set: documented training-data practices, sufficient quality and representativeness, lifecycle controls, monitoring. The pipeline you built for Chapters 01–04 can serve all three with the right tagging; this chapter teaches you the vocabulary each regime uses and the specific delta each one imposes.

## SR 11-7 and OCC 2011-12 — model risk management in US banking

SR 11-7 is the Federal Reserve's 2011 supervisory letter on model risk management, issued jointly with OCC Bulletin 2011-12. It applies to all models used by US banks and bank holding companies — risk, pricing, fraud, AML, consumer-credit decisioning, and increasingly AI-based models. The FDIC adopted equivalent guidance in 2017 ("Adoption of Supervisory Guidance on Model Risk Management", FIL-22-2017, 7 June 2017). In April 2026 the Federal Reserve, OCC, and FDIC rescinded SR 11-7, OCC 2011-12, and FIL-22-2017 and issued revised interagency model risk management guidance (<https://www.fdic.gov/news/financial-institution-letters/2026/agencies-revise-interagency-model-risk-management-guidance>); the underlying principles remain the lingua franca. Verify which document is in force at read time.

SR 11-7 is pre-AI-era, but its vocabulary has become the lingua franca for model governance in US financial services. Its structure:

- **Model definition** — any quantitative method that applies statistical, economic, financial, or mathematical theories to transform input data into outputs.
- **Three pillars of model risk management**: robust model development/implementation/use; model validation; and governance/policies/controls.
- **Model inventory** — every model in scope is enumerated, with owners and review cadence.
- **Effective challenge** — independent, qualified, critical review of models by parties not involved in development.
- **Ongoing monitoring** — process and performance verification over the model's life.

Training-data obligations inside SR 11-7:

- *Data used to develop a model should be appropriate for the type of model and the use to which it will be put.*
- *Processes should exist to verify the integrity, completeness, accuracy, timeliness, and consistency of data used to develop or operate a model.*
- *Data limitations should be understood and considered in the model development, validation, and use.*

The vocabulary matches ISO/IEC 25012 almost one-for-one. For an AI Data Engineer at a US bank:

| SR 11-7 expectation | Pipeline artifact |
|---|---|
| Data integrity / completeness / accuracy / timeliness / consistency | ISO/IEC 25012 scorecard + 5259-2 measures (Chapter 02) |
| Documented data limitations | "Known gaps" section of the dataset card |
| Independent review of data | Signoff by a non-developer reviewer tagged into the slice's release gate |
| Data in model inventory | Dataset card ID referenced from the model's SR 11-7 inventory entry |

Effective challenge for training data usually means a *data-side second reviewer* who was not part of curation reviews the slice before release. This is a role, not a role level — it can be anyone with the skill, so long as independence is preserved.

## FDA GMLP and PCCP — medical devices

The FDA regulates AI/ML-enabled medical devices through the same pathways as other devices (510(k), De Novo, PMA), plus two guidance tracks specific to ML:

- **Good Machine Learning Practice (GMLP) Guiding Principles** — ten principles published October 2021 jointly by FDA, Health Canada, and the UK MHRA. Non-binding but widely followed. The principles cover the entire ML lifecycle including training data.
- **Predetermined Change Control Plans (PCCP)** — final FDA guidance issued December 2024 (and earlier draft and discussion papers) enabling device makers to pre-specify how an AI/ML-enabled device will be modified post-market without requiring a new 510(k) each time.

GMLP training-data principles (paraphrased — read the primary source for exact wording):

- Training datasets are *independent of test datasets* — contamination control.
- Datasets are *representative of the intended patient population* — Article-10-style representativeness.
- Training data is *characterized, with quality-control and bias-mitigation efforts documented*.
- Focus is on the *performance of the human-AI team*, not just the model.

Engineering output for a GMLP-aligned pipeline:

| GMLP principle (paraphrased) | Pipeline artifact |
|---|---|
| Training / test independence | Contamination firewall report (mod-106) + lineage separation (mod-109) |
| Representative of intended patient population | Representativeness audit scoped to demographic variables declared in the Device Description + sufficiency statement |
| Characterised training data, bias-mitigation documented | Dataset card + bias audit + mitigation records |
| Human-AI team performance | Tag on the slice connecting it to the use-case's human-in-the-loop evaluation plan |

A PCCP pre-specifies *what changes to the device* the maker can make without new submission — including changes to training data under declared conditions. Your pipeline contributes by:

- Making every slice reproducible and signed, so a modification is a *new, declared slice* rather than an in-place change.
- Emitting a diff between slices that matches the PCCP's declared modification profile.
- Retaining the pre-modification slice for the regulator-mandated retention window.

## Colorado AI Act — the US state-level deployer/developer regime

Colorado SB 24-205 (the "Colorado AI Act") was signed by Governor Polis in May 2024 and takes effect **1 February 2026**. It is the first comprehensive US state-level AI law targeting both **developers** and **deployers** of **high-risk artificial intelligence systems** — systems that make, or are a substantial factor in making, a *consequential decision* (one with material effect on education, employment, financial services, government services, health, housing, insurance, or legal services access or terms).

The developer / deployer obligations most relevant to a data engineer:

- Developers must provide deployers with documentation including the *type of data used to train* the system, *known or reasonably foreseeable limitations of the system including risks of algorithmic discrimination*, and *information on how the system was evaluated for performance and risks of algorithmic discrimination*.
- Developers must make a publicly available statement summarising the types of high-risk AI systems the developer has currently made available and how known/foreseeable risks are managed.
- Developers must disclose known or foreseeable risks of algorithmic discrimination to the Attorney General within 90 days of discovery.

The Attorney General issues rules on implementation; the law contains defences for compliance with recognised AI risk management frameworks including NIST AI RMF. Note that the original 1 February 2026 effective date has moved: SB25B-004 (signed 28 August 2025) pushed it to 30 June 2026, and SB 26-189 (signed 14 May 2026) substantially revised the regime with a new 1 January 2027 effective date. AG rulemaking under the original SB 24-205 did not complete before it was superseded. Verify the operative statute and rules at <https://leg.colorado.gov/bills/sb24-205> and <https://coag.gov/> at read time.

For the pipeline:

| Colorado AI Act requirement | Pipeline artifact |
|---|---|
| Developer documentation of "type of data used to train" | Source manifest + licence matrix (same artifacts as EU AI Act Art 10) |
| Documentation of "known limitations including risks of algorithmic discrimination" | Bias audit + known-gaps section of the dataset card |
| Documentation of performance evaluation and discrimination evaluation | Representativeness audit + downstream fairness eval tagged to the slice |
| Deployer-facing summary | Model card excerpt + dataset card excerpt packaged for deployers |

The law has a safe-harbor for compliance with recognised frameworks — so the NIST AI RMF evidence you emit under Chapter 01 is directly reusable here. The shape of the obligation is ever more similar to EU AI Act Article 10; you build one artifact set and tag it for multiple regulators.

### Other US states

Other US states have passed or are considering AI-specific laws (e.g. Utah Artificial Intelligence Policy Act 2024, Illinois HB 3773 2024, Texas Data Privacy and Security Act). These are primarily *deployer-facing* and not primarily on training-data content; the `ai-governance-analyst` tracks them. Your pipeline's tagging should be extensible to new regimes without re-plumbing.

## New-York City Local Law 144 — a narrow but concrete example

NYC Local Law 144 (effective July 2023) regulates **Automated Employment Decision Tools (AEDTs)** used for NYC-based hiring and promotion. It requires an independent bias audit within one year prior to use, publication of the audit summary, and candidate notice.

Why it matters for an AI Data Engineer: *bias audit* under LL 144 is scoped to outcomes, not to training data directly. But to perform an LL 144 audit the auditor needs the slice's demographic composition and the model's performance by demographic group — artifacts you own. Tag any slice used to train an AEDT so the LL 144 package can be assembled quickly.

DCWP's implementing rules were adopted on 6 April 2023 (Notice of Adoption) with enforcement beginning 5 July 2023; the rules text is at <https://rules.cityofnewyork.us/rule/automated-employment-decision-tools-updated/>. No substantive amendments have been adopted since.

## How the sector/state regimes reuse the pipeline

The point of this chapter is that **you do not build three pipelines**. You build one pipeline that emits the artifacts in Chapters 01–04, and you tag slices so the same artifact flows into the SR 11-7 package, the FDA 510(k) submission, the Colorado developer documentation, or the LL 144 audit as needed.

A tagging shape that works:

```yaml
# slice.yaml — per-slice regulatory posture
slice_id: credit-ltv-features-2026-11-04
reg_posture:
  eu_ai_act:
    high_risk: true                      # Annex III financial services
    article_10_package: path/to/art10/
    gpai_summary_entry: null             # not a GPAI model training set
  nist_ai_rmf:
    profile: generic                     # or genai
    subcategories: [MEASURE 2.11, MEASURE 3.3]
    playbook_version: 2024-01
  sr_11_7:
    model_inventory_id: MI-1823
    second_reviewer: j.smith@...
  fda:
    applicable: false
  colorado_ai_act:
    high_risk_system: true
    consequential_decision_domain: financial_services
    developer_doc_pack: path/to/colorado/
  nyc_ll144:
    applicable: false
```

Downstream, each regulator-facing packager reads `reg_posture` and pulls the artifacts into the shape that regulator expects. Legal, the governance analyst, and sector-specific compliance staff own the mapping; you own the tags and the packagers.

## Summary

- SR 11-7 / OCC 2011-12 is the US banking regime. Its vocabulary (model inventory, effective challenge, ongoing monitoring) wraps your ISO/IEC 25012 and 5259 artifacts. Add an independent second reviewer and a model-inventory ID.
- FDA GMLP + PCCP is the US medical-device regime. Add representativeness scoped to patient population, train/test independence evidence, and reproducibility sufficient to be a PCCP-declared modification.
- Colorado AI Act is the US state-level deployer/developer regime, effective 1 February 2026. It leans heavily on documentation similar to EU AI Act Article 10; you reuse the same artifacts.
- The point is that one pipeline, correctly tagged, feeds every regulator-facing package. Tag the slice; let downstream packagers assemble.

Exercise 05 asks you to draw the role-scope and deferral contract, which brings Chapters 01–05 together with Chapter 06's level-ladder view.

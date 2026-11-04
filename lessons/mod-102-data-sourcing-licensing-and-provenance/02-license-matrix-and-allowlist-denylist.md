# Chapter 02 — The Data-Licence Matrix and the Allowlist / Denylist Policy

## Why this chapter exists

Every slice carries a licence posture. If someone — an auditor, a regulator's technical reviewer, opposing counsel in a dispute — asks "under what terms are you allowed to train on this?", the right answer is the signed licence tag on each source and the policy that admitted it. The wrong answers are "fair use probably", "we think it was open", and "the dataset card said CC-BY so we figured".

This chapter makes licence a first-class pipeline property. You will learn to read the licences you will actually meet, map each to the training-use obligations it imposes, and compile the organisation's position on each into an **allowlist / denylist policy** that the pipeline enforces at ingest. The policy itself is a signed artifact — counsel writes it, you make it executable.

## The licence families you will actually meet

For the training-data pipeline there are five families. Everything else is a specialisation.

### 1. Creative Commons (CC) licences

Creative Commons publishes a family of licences covering most public web content that is explicitly open-licensed. The licences are modular: **CC-BY** (attribution), **CC-BY-SA** (attribution, share-alike), **CC-BY-NC** (attribution, non-commercial), **CC-BY-ND** (attribution, no derivatives), **CC-BY-NC-SA**, **CC-BY-NC-ND**, and **CC0** (public-domain dedication). The currently recommended version is 4.0; earlier versions (3.0, 2.5, 2.0, 1.0) remain in force wherever the work is still licensed under them.

For training-data purposes, the matrix compresses to a handful of decisions.

| Licence | Can you ingest? | Can you train a commercial model? | Does the trained model inherit share-alike? | Attribution burden |
|---|---|---|---|---|
| CC0 / public domain | yes | yes | no | optional (courtesy) |
| CC-BY 4.0 | yes | yes (attribution) | no | yes — per-source or batched in dataset card |
| CC-BY-SA 4.0 | yes | **disputed** for model output; safe for data use with share-alike propagation; counsel call | **disputed** — some read the viral clause onto model weights | yes |
| CC-BY-NC 4.0 | yes | **no** for commercial training | n/a — rejected upstream | n/a |
| CC-BY-ND 4.0 | yes for data inclusion but **no modification** | **unsafe** — training mutates content; counsel call | n/a | n/a |
| CC-BY-NC-SA / NC-ND | yes | **no** | n/a | n/a |

Three important caveats.

- **CC-BY-SA viral scope for model weights is a legal question, not a licensing question.** The share-alike clause binds "adaptations." Whether a trained model is an adaptation of the input data is unsettled. Different organisations take different postures; counsel sets your organisation's. The pipeline still carries the tag so the slice is identifiable if the posture changes.
- **CC-BY-ND ("no derivatives") is the awkward one.** Reading training as "derivative" is plausible enough that risk-averse organisations deny-list it; permissive organisations allow it with a counsel opinion on file. Mark the slice; let counsel decide.
- **"Non-commercial" (NC) has no bright-line definition.** CC does not define commercial; case law varies by jurisdiction. Most ML programs read NC as *any model that generates revenue, directly or indirectly* and reject outright.

### 2. Open-source software licences applied to data

Code corpora and some data releases use software licences (MIT, BSD, Apache 2.0, GPL family, MPL, LGPL). Software licences were not written for training data, but their terms still bind when the licensor chose them.

| Licence | Can you ingest? | Can you train a commercial model? | Weight / derivative-output concerns | Attribution burden |
|---|---|---|---|---|
| MIT / BSD-2 / BSD-3 | yes | yes | low — permissive | yes (preserve notices) |
| Apache 2.0 | yes | yes | low — permissive; carries patent grant | yes (preserve NOTICE file) |
| GPL-2.0 / GPL-3.0 | yes | **disputed** — copyleft scope onto model weights is unresolved | **disputed** — some claim derivative-work status of generations trained on GPL code | yes |
| AGPL-3.0 | yes | **disputed with network clause** — AGPL's SaaS trigger adds exposure if model-weighted outputs are served | high | yes |
| MPL-2.0 / LGPL | yes | yes, but **file-level copyleft** complicates attribution on generated outputs | medium | yes |

The copyleft questions on GPL / AGPL for model weights are the code-corpus analogue of the CC-BY-SA question. Keep the tag, let counsel set the posture. Note that specialised code-training licences — for example, the licences chosen by some large code-training corpora like The Stack v1/v2 (BigCode), which explicitly filter to permissive licences and provide an opt-out — are *data* licences on top of the underlying code licences; both apply.

### 3. Proprietary / licensed-partner contracts

Every licensed partner corpus (Chapter 01 type 2) comes with a bespoke contract. The matrix for proprietary contracts is not CC-shaped; it is a set of clauses you must extract and tag per source.

Clauses that *always* appear in a well-drafted training-data licence:

- **Permitted uses** — what training objectives are in scope (pretraining, SFT, RLHF, evaluation, red-team). Rarely are all included.
- **Commercial use** — whether training is permitted for commercial deployment.
- **Model output scope** — whether model outputs may be commercialised, redistributed, published.
- **Redistribution** — whether the raw data or derived slices may be redistributed outside your organisation (e.g. to a partner lab, to subprocessors, to a published dataset card).
- **Territorial scope** — jurisdictions covered.
- **Retention window** — how long you may hold the data.
- **Subject-request passthrough** — whether the licensor handles data-subject requests or you do.
- **Audit rights** — whether the licensor may audit your use.
- **Termination and effect** — what you must do with ingested content if the agreement terminates.
- **Warranty and indemnity** — the licensor's warranty that the content is lawfully licensable.

Clauses to *look for* that routinely trip up a pipeline:

- **Model-training carve-out** — some licences are permissive for research but exclude training foundation models or include a per-token fee.
- **"Competing product" prohibition** — common in code licences and in some licensed partner corpora; blocks using the data to train a product that competes with the licensor.
- **Attribution-at-inference** — some licences require attribution in model outputs, not just in documentation. (This is a product-design constraint, not a data-pipeline one, but you flag it so product knows.)
- **No-opt-out guarantees** — some older licences do not contemplate GDPR subject requests; you need explicit written assurance.

Each clause becomes a boolean or typed field in the licensed-partner source record.

### 4. Platform ToS (owned UGC)

Your own product's Terms of Service functions as the licence for owned UGC. The only two things you need, once legal has read the version in force at content-creation time, are:

- **A content-creation-time licence string** — the ToS version in force when the content was produced. "UGC-v2024-02-01" as a tag with a URL to the archived ToS text. Older content keeps its older tag; it does not automatically pick up newer, broader terms.
- **A per-user consent modifier** — whether the user explicitly opted in or out of training use beyond what the ToS implies. Honour the opt-out.

The subtle failure mode is a ToS update that broadens training rights: content created before the update still carries the old tag and the old scope. Make sure your lineage system (mod-109) records the ToS version at ingest, not at re-use.

### 5. Public-domain dedications and government works

Some content is not licensed because it has no copyright to begin with: US federal government works (17 U.S.C. §105), public-domain works whose copyright has expired, works dedicated to the public domain via CC0 or public-domain mark. These are the cleanest sources but are not automatic:

- US federal works are public domain; state and local government works are not (they can and sometimes do assert copyright).
- Foreign government works have no US-federal-equivalent rule; each jurisdiction varies.
- "Public domain in the US but not elsewhere" is routine for pre-1929 works; tag the jurisdiction.
- CC0 dedications are revocable in some jurisdictions (CC provides a fallback permissive licence for this reason).

Public-domain content is the only category where licence tagging is "N/A"; everything else gets a tag.

## Commercial-use carve-outs, redistribution, and the clauses that bite

Across all families, three clause categories are disproportionately responsible for post-ingest surprises.

### Commercial-use carve-outs

A licence may permit non-commercial training but deny commercial training (CC-BY-NC is the common case; most research-only academic releases are equivalent). The pipeline rule is: **mark every slice with a commercial-use posture at construction, and reject sources whose licence does not match**. If a slice is marked `research-only`, it must never be fed into a model slated for commercial deployment. The posture is enforced by the training orchestrator reading the slice's manifest.

### Redistribution restrictions

A licence may permit you to train on content but forbid you to redistribute the raw content or the derived slices. This matters for:

- Published dataset cards that include samples.
- Transferring the slice to a subprocessor (a third-party labelling vendor, a compute partner).
- Sharing a decontamination index with an evaluation partner.

The pipeline rule is: **every slice carries a `redistribution_scope` tag** — internal-only, subprocessor-with-DPA, public-with-attribution, public-without-attribution. Dataset-card samples use the most restrictive member of the slice.

### Attribution burden

A licence may require attribution — sometimes per-asset, sometimes aggregated. Attribution for a billion-document pretraining slice is not practicable per-document; it is aggregated by source domain and listed in the dataset card. For licensed partner content with per-record attribution, the attribution is maintained in a sidecar index tied to the record ID. When the model is served, product decides whether attribution appears in user-facing surfaces.

## The allowlist / denylist policy

The policy document is the compiled legal position. It is signed by counsel and the AI governance council, and it binds the pipeline. Its minimum structure:

```yaml
# policies/data_license_policy_v2026-11-04.yaml
policy_id: data-license-policy-v2026-11-04
effective: 2026-11-04
approved_by:
  counsel: counsel@example.com
  governance_council: ai-governance-council@example.com
reviews_on_or_before: 2027-05-04   # every 6 months, or on legal-position change

allowlist:
  - tag: cc0 / public-domain
    scope: all-training-objectives
    attribution: optional-courtesy
  - tag: cc-by-4.0
    scope: all-training-objectives
    attribution: per-domain-aggregated
  - tag: mit
    scope: all-training-objectives
    attribution: preserve-notices
  - tag: apache-2.0
    scope: all-training-objectives
    attribution: preserve-notice-file
  - tag: bsd-2-clause / bsd-3-clause
    scope: all-training-objectives
    attribution: preserve-notices
  - tag: owned-ugc-tos-v2024-02-01-or-later
    scope: all-training-objectives-subject-to-user-opt-out
    attribution: n-a
  - tag: licensed-dla-with-training-clause
    scope: per-contract-defined
    attribution: per-contract-defined

allowlist-with-conditions:
  - tag: cc-by-sa-4.0
    scope: all-training-objectives
    conditions:
      - counsel-opinion-on-weight-share-alike-propagation-attached
      - slice-tagged-share-alike-propagated for downstream consumers
    attribution: per-domain-aggregated
  - tag: cc-by-nd-4.0
    scope: research-only-unless-counsel-opinion-attached
    conditions:
      - counsel-opinion-on-derivative-work-status-attached
    attribution: per-domain-aggregated
  - tag: gpl-2.0 / gpl-3.0
    scope: all-training-objectives
    conditions:
      - counsel-opinion-on-copyleft-model-output-scope-attached
      - no-direct-generation-of-licensed-source-files-in-product (mod-106 contamination enforced)
    attribution: preserve-notices
  - tag: agpl-3.0
    scope: all-training-objectives
    conditions:
      - counsel-opinion-on-network-clause-scope-attached
      - product-side-saas-trigger-reviewed
    attribution: preserve-notices

denylist:
  - tag: cc-by-nc-* (any non-commercial CC variant)
    reason: non-commercial clause incompatible with commercial model deployment
  - tag: cc-*-nd (derivative restriction)
    scope: denied-for-non-research-objectives
    reason: training mutates content; derivative status unresolved
  - tag: proprietary-no-training-clause
    reason: licensor expressly reserved
  - tag: platform-tos-pre-v2024-02-01
    reason: pre-training-clause ToS version
  - tag: unlabelled
    reason: default-deny — a source with no identified licence is not admissible

special-case:
  - tag: synthetic-from-third-party-model
    rule: generator-licence-governs; see per-generator addendum
    addendum: policies/generator_license_addendum_v2026-11-04.yaml

enforcement:
  ingest_gate: aicg.datapipeline.policy.LicenseGate@v3
  per_item_log: ingest_decisions.jsonl (fields: item_sha, source_id, licence_tag, decision, reason)
  override_authority: counsel + governance_council (co-sign); each override logged with a ticket ID
```

The policy is executable. The `LicenseGate` component reads the YAML, checks each ingest item against the lists, and either admits, admits-with-condition, or denies. "Admits-with-condition" means the admission depends on an artifact — a counsel opinion PDF, a share-alike propagation tag — that must be attached to the slice for the slice to be releasable. The gate writes an ingest decision log that becomes audit evidence.

### Default-deny is the only safe default

The default behaviour for unlabelled sources must be *deny*, not *admit with warning*. The cost of an occasional friction-ticket is dramatically lower than the cost of discovering, a year in, that 3 % of the corpus has no licence at all. The gate's defaultness is itself a policy claim.

## A worked example: evaluating a candidate source

You have a candidate source: a 500 GB collection of code published on GitHub, filtered to repositories whose top-level licence file is a permissive OSS licence. Someone proposes using it for a code fine-tune.

Walk the matrix.

1. **Source type** — public web crawl (Chapter 01 type 1), refined; or third-party public-release dataset if you source it via BigCode's The Stack rather than crawling directly. Different downstream burdens.
2. **Licence** — mixed. Each repository carries its own licence. The slice as a whole is "assumes-permissive"; the pipeline must verify per-repository.
3. **Allowlist decisions per licence** — MIT / BSD / Apache 2.0 are allowlisted outright. GPL-2 / GPL-3 are `allowlist-with-conditions` and may be routed into a GPL-only partition or filtered out. AGPL is a stricter gate. Non-OSS licences (or missing licence) are denied.
4. **Author opt-out posture** — the Stack project provides an opt-out mechanism (am-i-in-the-stack). Honour it; Chapter 03 covers how.
5. **Attribution** — per-repository attribution in a sidecar; MIT / BSD / Apache 2.0 require preserving notices; this becomes a pipeline-side obligation.
6. **Commercial-use posture** — permissive OSS licences generally permit commercial training; GPL copyleft propagation remains a counsel question.
7. **Redistribution** — a dataset card listing per-repository samples complies with attribution; redistributing the whole slice is a scope decision.

The outcome is a `license_decision` per repository — admit (permissive), admit-with-partition (GPL / AGPL conditional partition), deny (non-OSS, missing). The per-repository decisions feed the aggregate slice; the slice inherits the strictest member's constraints.

## The licence gate in code

A minimal version of the gate:

```python
# aicg/datapipeline/policy/license_gate.py
from dataclasses import dataclass
from enum import Enum
import yaml

class Decision(Enum):
    ADMIT = "admit"
    ADMIT_WITH_CONDITION = "admit-with-condition"
    DENY = "deny"

@dataclass
class GateResult:
    decision: Decision
    reason: str
    conditions_required: list[str]

class LicenseGate:
    def __init__(self, policy_path: str):
        self.policy = yaml.safe_load(open(policy_path))

    def evaluate(self, licence_tag: str, objective: str) -> GateResult:
        tag = licence_tag.lower().strip()
        for row in self.policy.get("denylist", []):
            if self._match(tag, row["tag"]):
                return GateResult(Decision.DENY, row["reason"], [])
        for row in self.policy.get("allowlist", []):
            if self._match(tag, row["tag"]) and self._in_scope(objective, row.get("scope")):
                return GateResult(Decision.ADMIT, "allowlisted", [])
        for row in self.policy.get("allowlist-with-conditions", []):
            if self._match(tag, row["tag"]) and self._in_scope(objective, row.get("scope")):
                return GateResult(Decision.ADMIT_WITH_CONDITION, "conditional", list(row["conditions"]))
        return GateResult(Decision.DENY, "unlabelled-or-not-in-policy", [])

    def _match(self, tag: str, pattern: str) -> bool:
        # ... glob-style match with family prefixes; omitted for brevity
        ...

    def _in_scope(self, objective: str, scope: str | None) -> bool:
        if scope is None or scope == "all-training-objectives":
            return True
        # ... parse per-objective scope lines; omitted for brevity
        ...
```

The gate is testable with unit tests whose inputs are hypothetical `(licence_tag, objective)` pairs and whose expected outputs are policy-derived `GateResult`s. The tests belong in the mod-104 quality-measure suite.

## What this chapter hands to the rest of the module

- The `license` cell of the Chapter 01 selection matrix is populated by this chapter's tags.
- The ingest decision log produced by the gate is one input to the datasheet (Chapter 05).
- The ML-BOM entry (Chapter 06) carries the licence tag and the gate decision as metadata.
- The peer-coordination handshake with counsel and `ai-governance-analyst` (Chapter 07) is mediated by this policy document.

## Summary

- Five licence families: Creative Commons, open-source software, proprietary / licensed-partner contracts, platform ToS for owned UGC, and public-domain / government works. Each has its own matrix.
- Commercial-use carve-outs, redistribution restrictions, and attribution burdens are the clauses that bite; every slice carries a posture for each.
- The allowlist / denylist policy is a signed, executable artifact. Counsel writes it; the `LicenseGate` enforces it; the ingest decision log is the audit evidence.
- Default-deny for unlabelled sources is non-negotiable. The gate must refuse to admit what it cannot classify.
- CC-BY-SA, CC-BY-ND, and GPL / AGPL copyleft onto model weights are unresolved legal questions. Keep the tags; let counsel set the posture; make the posture switchable.

Exercise 02 walks you through authoring a complete policy document and a test suite for its gate. Chapter 03 turns to the opt-out signals that modify admission irrespective of licence.

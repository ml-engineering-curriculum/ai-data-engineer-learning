# Lab 02 — Implement and Test the Composite Admit Function

**Estimated duration:** 90 minutes
**Chapter dependency:** [Chapter 04 — EU DSM Article 4 Reservations, US Copyright Posture, and the Legal Deferral Contract](../04-dsm-article-4-and-us-copyright-posture.md). Chapters [02](../02-license-matrix-and-allowlist-denylist.md) and [03](../03-crawling-and-training-opt-out-contracts.md) supply the child evaluators the composite chains.
**Relationship to the exercises:** Exercise 02 builds the `LicenseGate` and Exercise 03 builds the `OptOutEvaluator`. This lab is the *composition pass* — the Chapter 04 `composite_admit` function that chains licence, opt-out, privacy, and jurisdiction decisions into a single per-item verdict, writes each verdict to a reproducible log, and pins the ordering so a rightsholder reservation is recorded as a *copyright* denial, never as an incidental privacy denial. Do not re-author the `LicenseGate` or `OptOutEvaluator` internals here; stub them and focus on composition, ordering, logging, and tests.

## Objective

By the end of this lab you will have:

1. A `composite.py` module implementing the Chapter 04 §"How Article 4 and the signals from Chapter 03 compose" `composite_admit` function. The function takes an item, a policy, and a `counsel_posture` string and returns a `CompositeDecision` with four sub-decisions, the posture in force, the item SHA, and the policy version.
2. Stub `LicenseGate`, `OptOutEvaluator`, `PrivacyEvaluator`, and `JurisdictionEvaluator` child components whose only job is to return the pre-programmed decision the test case needs. The stubs reflect the Chapter 02 / 03 / 04 shapes but carry no production parsing logic — the composition is what is under test.
3. A `test_composite.py` pytest module with: one test per child-gate denial path; one admit path per the four Chapter 01 source types ("public web crawl", "licensed partner", "owned UGC", "synthetic"); and one *ordering* test per the Chapter 04 rule that a DSM Article 4(3) reservation is recorded as a copyright denial even when the item also fails the privacy or licence check.
4. A `decision_log.jsonl` the composite writes with one line per decision, carrying the full four-way sub-decision breakdown plus the policy version.

The point: *the composite is what you point at when a regulator asks "why was this item admitted or denied?"* Chapter 04 says the composite per-item log is the audit evidence. This lab builds that composite and that log.

## Prerequisites

- You have read Chapter 04 (and skimmed Chapters 02 and 03 for the child-gate vocabulary).
- You have Python 3.11+ and can install packages. Only `pytest` is required.
- You understand pytest at the "writes a test function, runs it, passes or fails" level.
- You have about 90 minutes.

## Step 1 — Make the scratch directory and install pytest

```bash
mkdir -p lab02-composite-admit && cd lab02-composite-admit
python -m venv .venv
. .venv/bin/activate
pip install pytest
```

**Checkpoint:** `pytest --version` prints a version.

## Step 2 — Draft the data model

Create `models.py`. The item fields mirror the Chapter 04 `composite_admit` signature; the child decision shapes mirror Chapter 02's `GateResult`, Chapter 03's `OptOutDecision`, and the Chapter 04 privacy / jurisdiction evaluators.

```python
"""Per-item data model for the composite admit function.

The dataclasses are deliberately thin. Field semantics are documented in
Chapter 04. The composite is what is under test here; the child-gate
internals live in Exercises 02 (licence) and 03 (opt-out), and in mod-101
(privacy) and mod-104 (jurisdiction).
"""
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Literal

Jurisdiction = Literal["eu", "us", "uk", "jp", "sg", "other"]


class LicenceDecision(Enum):
    ADMIT = "admit"
    ADMIT_WITH_CONDITION = "admit-with-condition"
    DENY = "deny"


@dataclass
class LicenceResult:
    decision: LicenceDecision
    reason: str
    conditions_required: list[str] = field(default_factory=list)


@dataclass
class OptOutResult:
    admit: bool
    signals_consulted: list[str] = field(default_factory=list)
    signals_blocking: list[str] = field(default_factory=list)


@dataclass
class PrivacyResult:
    admit: bool
    reason: str = ""


@dataclass
class JurisdictionResult:
    admit: bool
    reason: str = ""


@dataclass
class Item:
    sha: str
    url: str
    source_id: str
    source_type: Literal["public-web-crawl", "licensed-partner", "owned-ugc",
                         "third-party-public", "synthetic"]
    licence: str
    training_objective: str
    created_at: datetime
    bytes_: bytes = b""
    html_head: str = ""


@dataclass
class Policy:
    version: str


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
```

**Checkpoint:** `python -c "import models; print(models.LicenceDecision.ADMIT)"` prints `LicenceDecision.ADMIT`.

## Step 3 — Draft the child-gate stubs

Create `gates.py`. The stubs record the decisions the test wants them to return; the composition is what we are exercising.

```python
"""Stub child gates. In production, each is a real evaluator — Chapter 02's
LicenseGate, Chapter 03's OptOutEvaluator, mod-101 Chapter 04's privacy
admit, and Chapter 04's jurisdiction_evaluator. Here we stub each one so the
composite is isolatable under test.
"""
from __future__ import annotations

from dataclasses import dataclass, field

from models import (
    JurisdictionResult,
    LicenceDecision,
    LicenceResult,
    OptOutResult,
    PrivacyResult,
)


@dataclass
class StubLicenseGate:
    """Return LicenceResult keyed by (licence_tag, training_objective)."""
    table: dict[tuple[str, str], LicenceResult] = field(default_factory=dict)

    def evaluate(self, licence_tag: str, objective: str) -> LicenceResult:
        if (licence_tag, objective) in self.table:
            return self.table[(licence_tag, objective)]
        # Chapter 02 §"Default-deny is the only safe default".
        return LicenceResult(LicenceDecision.DENY, "unlabelled-or-not-in-policy")


@dataclass
class StubOptOutEvaluator:
    """Return OptOutResult keyed by URL."""
    table: dict[str, OptOutResult] = field(default_factory=dict)

    def evaluate(self, url: str, asset_bytes: bytes, html_head: str) -> OptOutResult:
        if url in self.table:
            return self.table[url]
        return OptOutResult(admit=True, signals_consulted=["robots-txt"], signals_blocking=[])


@dataclass
class StubPrivacyEvaluator:
    """Return PrivacyResult keyed by item.sha."""
    table: dict[str, PrivacyResult] = field(default_factory=dict)

    def evaluate(self, item) -> PrivacyResult:
        return self.table.get(item.sha, PrivacyResult(admit=True))


@dataclass
class StubJurisdictionEvaluator:
    """Return JurisdictionResult keyed by item.sha, parameterised by counsel_posture."""
    table: dict[str, JurisdictionResult] = field(default_factory=dict)

    def evaluate(self, item, counsel_posture: str) -> JurisdictionResult:
        return self.table.get(item.sha, JurisdictionResult(admit=True, reason=counsel_posture))
```

**Checkpoint:** `python -c "import gates; print(gates.StubLicenseGate())"` prints an empty-table stub.

## Step 4 — Implement the composite

Create `composite.py`. The ordering matters: Chapter 04 §"How Article 4 and the signals from Chapter 03 compose" chains the four child decisions, and Chapter 03 §"Specific behaviours for the production crawler bots" plus the deferral contract in Chapter 04 together say that an opt-out-driven denial must be recorded as such and not reclassified.

```python
"""Composite admit function — the Chapter 04 §"How Article 4 and the signals
from Chapter 03 compose" shape.

Ordering rule: licence first, then opt-out, then privacy, then jurisdiction.
Chapter 04's composite returns a CompositeDecision with every sub-decision
visible; the overall admit is the conjunction. If any one deny is present, the
item is denied, but the sub-decisions remain legible so a rightsholder opt-out
is not reclassified as a privacy denial under adversarial review.
"""
from __future__ import annotations

import json
from dataclasses import asdict
from datetime import datetime, timezone

from models import CompositeDecision, Item, LicenceDecision, Policy


class CompositePolicy:
    """Small holder for the four child gates plus the policy version."""

    def __init__(self, license_gate, opt_out_evaluator, privacy_evaluator,
                 jurisdiction_evaluator, version: str):
        self.license_gate = license_gate
        self.opt_out_evaluator = opt_out_evaluator
        self.privacy_evaluator = privacy_evaluator
        self.jurisdiction_evaluator = jurisdiction_evaluator
        self.version = version


def composite_admit(item: Item, policy: CompositePolicy, counsel_posture: str) -> CompositeDecision:
    """The Chapter 04 composite admit function."""
    licence_result = policy.license_gate.evaluate(item.licence, item.training_objective)
    opt_out_result = policy.opt_out_evaluator.evaluate(item.url, item.bytes_, item.html_head)
    privacy_result = policy.privacy_evaluator.evaluate(item)
    jurisdiction_result = policy.jurisdiction_evaluator.evaluate(item, counsel_posture=counsel_posture)

    all_admit = all([
        licence_result.decision is not LicenceDecision.DENY,
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


def write_decision(decision: CompositeDecision, path: str = "decision_log.jsonl") -> None:
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps(asdict(decision)) + "\n")
```

Two deliberate design choices worth noticing:

- The composite does **not** short-circuit. All four child gates are evaluated every time, so the per-item log always shows all four sub-decisions. This is what makes the log reproducible under adversarial review — a rightsholder asking "did you check the opt-out signal?" sees the opt-out line on every item, admit or deny.
- The overall `admit` is the conjunction of the four. `LicenceDecision.ADMIT_WITH_CONDITION` is *not* a deny; the chapter is explicit that the condition is attached as an artifact on the slice, not enforced at the per-item gate.

**Checkpoint:** `python -c "import composite; print(composite.composite_admit.__doc__)"` prints the docstring.

## Step 5 — Draft the pytest fixtures

Create `conftest.py`:

```python
"""Shared test fixtures for the composite admit tests."""
from datetime import datetime, timedelta, timezone

import pytest

from composite import CompositePolicy
from gates import (
    StubJurisdictionEvaluator,
    StubLicenseGate,
    StubOptOutEvaluator,
    StubPrivacyEvaluator,
)
from models import (
    Item,
    JurisdictionResult,
    LicenceDecision,
    LicenceResult,
    OptOutResult,
    PrivacyResult,
)


NOW = datetime(2026, 11, 17, 9, 0, 0, tzinfo=timezone.utc)


def _item(sha: str, source_type: str, licence: str, url: str,
          source_id: str = "src", objective: str = "sft-customer-support") -> Item:
    return Item(
        sha=sha, url=url, source_id=source_id, source_type=source_type,
        licence=licence, training_objective=objective,
        created_at=NOW - timedelta(days=1),
    )


@pytest.fixture
def crawl_item():
    return _item("sha-crawl-01", "public-web-crawl", "CC-BY-4.0",
                 "https://example.com/post-1")


@pytest.fixture
def licensed_item():
    return _item("sha-lic-01", "licensed-partner", "dla-2026-04",
                 "licensed://vendor-x/record-1", source_id="licensed-qa-vendor-x")


@pytest.fixture
def owned_ugc_item():
    return _item("sha-ugc-01", "owned-ugc", "product-tos-v2024-02-01",
                 "product://session-1/turn-3")


@pytest.fixture
def synthetic_item():
    return _item("sha-syn-01", "synthetic", "generator-tos-permissive",
                 "synthetic://run-04/pair-17")


@pytest.fixture
def allow_all_policy():
    """A permissive composite that admits every item against the fixture objective."""
    gate = StubLicenseGate(table={
        ("CC-BY-4.0", "sft-customer-support"): LicenceResult(LicenceDecision.ADMIT, "allowlisted"),
        ("dla-2026-04", "sft-customer-support"): LicenceResult(LicenceDecision.ADMIT, "licensed-partner-dla"),
        ("product-tos-v2024-02-01", "sft-customer-support"): LicenceResult(LicenceDecision.ADMIT, "owned-ugc-tos"),
        ("generator-tos-permissive", "sft-customer-support"): LicenceResult(LicenceDecision.ADMIT, "generator-ok"),
    })
    return CompositePolicy(
        license_gate=gate,
        opt_out_evaluator=StubOptOutEvaluator(),
        privacy_evaluator=StubPrivacyEvaluator(),
        jurisdiction_evaluator=StubJurisdictionEvaluator(),
        version="composite-v2026-11-17",
    )
```

**Checkpoint:** `pytest --collect-only -q` lists the fixtures without errors.

## Step 6 — Write the composite tests

Create `test_composite.py`:

```python
"""Composite admit tests — four happy paths, four denial paths, and the
Chapter 04 ordering rule."""
from composite import composite_admit, write_decision
from gates import StubOptOutEvaluator
from models import (
    JurisdictionResult,
    LicenceDecision,
    LicenceResult,
    OptOutResult,
    PrivacyResult,
)


# ---- happy paths: one admit per Chapter 01 source type ---------------------

def test_crawl_admitted(crawl_item, allow_all_policy):
    d = composite_admit(crawl_item, allow_all_policy, counsel_posture="moderate")
    assert d.admit is True
    assert d.licence_decision == "ADMIT"
    assert d.opt_out_decision == "admit"
    assert d.privacy_decision == "admit"
    assert d.jurisdiction_decision == "admit"
    assert d.policy_version == "composite-v2026-11-17"


def test_licensed_admitted(licensed_item, allow_all_policy):
    d = composite_admit(licensed_item, allow_all_policy, counsel_posture="moderate")
    assert d.admit is True


def test_owned_ugc_admitted(owned_ugc_item, allow_all_policy):
    d = composite_admit(owned_ugc_item, allow_all_policy, counsel_posture="moderate")
    assert d.admit is True


def test_synthetic_admitted(synthetic_item, allow_all_policy):
    d = composite_admit(synthetic_item, allow_all_policy, counsel_posture="moderate")
    assert d.admit is True


# ---- deny paths: one per child-gate --------------------------------------

def test_licence_deny_blocks(crawl_item, allow_all_policy):
    allow_all_policy.license_gate.table[("CC-BY-4.0", "sft-customer-support")] = \
        LicenceResult(LicenceDecision.DENY, "cc-by-nc-rejected")
    d = composite_admit(crawl_item, allow_all_policy, counsel_posture="moderate")
    assert d.admit is False
    assert d.licence_decision == "DENY"
    # Other sub-decisions still recorded, even on licence deny.
    assert d.opt_out_decision == "admit"
    assert d.privacy_decision == "admit"
    assert d.jurisdiction_decision == "admit"


def test_opt_out_deny_blocks(crawl_item, allow_all_policy):
    allow_all_policy.opt_out_evaluator.table[crawl_item.url] = OptOutResult(
        admit=False, signals_consulted=["robots-txt"], signals_blocking=["robots-txt"]
    )
    d = composite_admit(crawl_item, allow_all_policy, counsel_posture="moderate")
    assert d.admit is False
    assert d.licence_decision == "ADMIT"
    assert d.opt_out_decision == "deny"


def test_privacy_deny_blocks(crawl_item, allow_all_policy):
    allow_all_policy.privacy_evaluator.table[crawl_item.sha] = PrivacyResult(admit=False, reason="pii-no-basis")
    d = composite_admit(crawl_item, allow_all_policy, counsel_posture="moderate")
    assert d.admit is False
    assert d.privacy_decision == "deny"


def test_jurisdiction_deny_blocks(crawl_item, allow_all_policy):
    allow_all_policy.jurisdiction_evaluator.table[crawl_item.sha] = JurisdictionResult(
        admit=False, reason="conservative-posture-eu-only"
    )
    d = composite_admit(crawl_item, allow_all_policy, counsel_posture="conservative")
    assert d.admit is False
    assert d.jurisdiction_decision == "deny"
    # posture_in_force is the counsel_posture passed in — the composite
    # records it on every decision regardless of outcome.
    assert d.posture_in_force == "conservative"


# ---- admit-with-condition is not a deny ----------------------------------

def test_admit_with_condition_is_not_a_deny(crawl_item, allow_all_policy):
    """Chapter 02: the ADMIT_WITH_CONDITION path is admission. The condition is
    an artifact attached to the slice, not enforced at the per-item gate."""
    allow_all_policy.license_gate.table[("CC-BY-4.0", "sft-customer-support")] = \
        LicenceResult(LicenceDecision.ADMIT_WITH_CONDITION, "counsel-opinion-on-weight-share-alike-attached",
                      conditions_required=["counsel-opinion-pdf"])
    d = composite_admit(crawl_item, allow_all_policy, counsel_posture="moderate")
    assert d.admit is True
    assert d.licence_decision == "ADMIT_WITH_CONDITION"


# ---- ordering: an opt-out denial is recorded as such, not reclassified --

def test_opt_out_denial_is_recorded_as_opt_out_even_when_privacy_also_fails(
        crawl_item, allow_all_policy):
    """Chapter 04: when both the opt-out signal and the privacy basis are
    missing, the composite records *both* denials. The opt-out denial does
    not disappear just because the privacy denial would also block.

    This matters for rightsholder audit: a rightsholder asking "did you honour
    my opt-out?" must see the opt-out line on the item's log, not the
    item marked only as a privacy denial. Chapter 04 §"How Article 4 and the
    signals from Chapter 03 compose" treats every sub-decision as separately
    legible for this reason."""
    allow_all_policy.opt_out_evaluator.table[crawl_item.url] = OptOutResult(
        admit=False, signals_consulted=["robots-txt"], signals_blocking=["robots-txt"]
    )
    allow_all_policy.privacy_evaluator.table[crawl_item.sha] = PrivacyResult(
        admit=False, reason="pii-no-basis"
    )
    d = composite_admit(crawl_item, allow_all_policy, counsel_posture="moderate")
    assert d.admit is False
    # Both denials visible — that is the whole point.
    assert d.opt_out_decision == "deny"
    assert d.privacy_decision == "deny"


# ---- the posture_in_force is recorded on every decision ------------------

def test_posture_in_force_is_recorded_on_every_decision(crawl_item, allow_all_policy):
    """Chapter 04: `us_training_posture` is set on the policy; the composite
    records whichever posture was passed in, so that if counsel changes
    posture later, the log explains which items were decided under which."""
    d_mod = composite_admit(crawl_item, allow_all_policy, counsel_posture="moderate")
    d_cons = composite_admit(crawl_item, allow_all_policy, counsel_posture="conservative")
    assert d_mod.posture_in_force == "moderate"
    assert d_cons.posture_in_force == "conservative"


# ---- log round-trip ------------------------------------------------------

def test_decision_round_trips_through_jsonl(crawl_item, allow_all_policy, tmp_path):
    import json
    d = composite_admit(crawl_item, allow_all_policy, counsel_posture="moderate")
    log = tmp_path / "decision_log.jsonl"
    write_decision(d, path=str(log))
    lines = log.read_text().splitlines()
    assert len(lines) == 1
    record = json.loads(lines[0])
    # Every field in CompositeDecision must round-trip.
    for field in ("admit", "licence_decision", "opt_out_decision", "privacy_decision",
                  "jurisdiction_decision", "posture_in_force", "item_sha", "policy_version"):
        assert field in record
```

Run the tests:

```bash
pytest -q
```

**Expected output (shape — 11 tests passing in well under a second):**

```
...........                                                      [100%]
11 passed in 0.08s
```

**Checkpoint:** `pytest -q` exits 0.

If a test fails, read the failing assertion and fix `composite.py`. Do not change the test to make it pass; the test is the specification. In particular `test_opt_out_denial_is_recorded_as_opt_out_even_when_privacy_also_fails` pins the chapter's ordering-and-legibility rule — do not short-circuit the composite to make it faster at the cost of losing sub-decision visibility.

## Step 7 — Run the composite over a sample and inspect the log

Create `sample_run.py`:

```python
"""Run a handful of items through composite_admit() and emit decision_log.jsonl."""
from datetime import datetime, timedelta, timezone

from composite import CompositePolicy, composite_admit, write_decision
from gates import (
    StubJurisdictionEvaluator,
    StubLicenseGate,
    StubOptOutEvaluator,
    StubPrivacyEvaluator,
)
from models import (
    Item,
    JurisdictionResult,
    LicenceDecision,
    LicenceResult,
    OptOutResult,
    PrivacyResult,
)


def main():
    now = datetime(2026, 11, 17, 9, 0, 0, tzinfo=timezone.utc)

    license_gate = StubLicenseGate(table={
        ("CC-BY-4.0", "sft"): LicenceResult(LicenceDecision.ADMIT, "cc-by-allowlisted"),
        ("CC-BY-NC-4.0", "sft"): LicenceResult(LicenceDecision.DENY, "nc-denylisted"),
        ("product-tos-v2024-02-01", "sft"): LicenceResult(LicenceDecision.ADMIT, "owned-ugc"),
    })
    opt_out = StubOptOutEvaluator(table={
        "https://opted-out.example/page": OptOutResult(
            admit=False, signals_consulted=["robots-txt"], signals_blocking=["robots-txt"]
        ),
    })
    privacy = StubPrivacyEvaluator(table={
        "sha-04": PrivacyResult(admit=False, reason="pii-no-basis"),
    })
    jurisdiction = StubJurisdictionEvaluator()

    policy = CompositePolicy(license_gate, opt_out, privacy, jurisdiction, version="v2026-11-17")

    items = [
        Item("sha-01", "https://ok.example/1", "src-crawl", "public-web-crawl",
             "CC-BY-4.0", "sft", created_at=now - timedelta(days=1)),
        Item("sha-02", "https://ok.example/2", "src-crawl", "public-web-crawl",
             "CC-BY-NC-4.0", "sft", created_at=now - timedelta(days=1)),
        Item("sha-03", "https://opted-out.example/page", "src-crawl", "public-web-crawl",
             "CC-BY-4.0", "sft", created_at=now - timedelta(days=1)),
        Item("sha-04", "product://session-9/turn-1", "src-ugc", "owned-ugc",
             "product-tos-v2024-02-01", "sft", created_at=now - timedelta(days=1)),
    ]

    # Clear old log
    open("decision_log.jsonl", "w").close()
    for item in items:
        write_decision(composite_admit(item, policy, counsel_posture="moderate"))

    print("Done. Inspect decision_log.jsonl.")


if __name__ == "__main__":
    main()
```

Run and inspect:

```bash
python sample_run.py
cat decision_log.jsonl
```

**Expected output (shape — one line per item, all four sub-decisions visible on every line):**

```
{"admit": true,  "licence_decision": "ADMIT", "opt_out_decision": "admit", "privacy_decision": "admit", "jurisdiction_decision": "admit", "posture_in_force": "moderate", "item_sha": "sha-01", "policy_version": "v2026-11-17"}
{"admit": false, "licence_decision": "DENY",  "opt_out_decision": "admit", "privacy_decision": "admit", "jurisdiction_decision": "admit", "posture_in_force": "moderate", "item_sha": "sha-02", "policy_version": "v2026-11-17"}
{"admit": false, "licence_decision": "ADMIT", "opt_out_decision": "deny",  "privacy_decision": "admit", "jurisdiction_decision": "admit", "posture_in_force": "moderate", "item_sha": "sha-03", "policy_version": "v2026-11-17"}
{"admit": false, "licence_decision": "ADMIT", "opt_out_decision": "admit", "privacy_decision": "deny",  "jurisdiction_decision": "admit", "posture_in_force": "moderate", "item_sha": "sha-04", "policy_version": "v2026-11-17"}
```

(Your JSON will not be pretty-printed; the shape is what matters.)

**Checkpoint:** `wc -l decision_log.jsonl` prints **4**, `grep -c '"admit": true' decision_log.jsonl` prints **1**, and every line parses as JSON with all eight `CompositeDecision` fields present.

## Step 8 — Confirm the log is machine-parseable per Chapter 04

```bash
python -c "
import json
outcomes = []
for line in open('decision_log.jsonl'):
    r = json.loads(line)
    outcomes.append((r['item_sha'], r['admit'],
                     r['licence_decision'], r['opt_out_decision'],
                     r['privacy_decision'], r['jurisdiction_decision']))
for row in outcomes:
    print(row)
"
```

**Expected output:**

```
('sha-01', True,  'ADMIT', 'admit', 'admit', 'admit')
('sha-02', False, 'DENY',  'admit', 'admit', 'admit')
('sha-03', False, 'ADMIT', 'deny',  'admit', 'admit')
('sha-04', False, 'ADMIT', 'admit', 'deny',  'admit')
```

**Checkpoint:** every row carries all four sub-decisions explicitly; no sub-decision is omitted or defaulted.

## What you have at the end

```
lab02-composite-admit/
├── models.py                 # Item, CompositeDecision, child result types
├── gates.py                  # StubLicenseGate / StubOptOutEvaluator / StubPrivacy / StubJurisdiction
├── composite.py              # composite_admit + CompositePolicy + write_decision
├── conftest.py               # pytest fixtures
├── test_composite.py         # happy paths, deny paths, ordering, log round-trip
├── sample_run.py             # exercises composite_admit over four fixture items
└── decision_log.jsonl        # one JSON record per decision with all four sub-decisions
```

## Discussion

- The composite's return value carries all four sub-decisions on every item, admitted or denied. That is not accidental: Chapter 04 §"When a legal question lands" lists the "per-item decision log for the implicated items" as one of the artifacts counsel defends with. If the log loses an opt-out line when a privacy denial also fires, the rightsholder-facing audit story weakens. `test_opt_out_denial_is_recorded_as_opt_out_even_when_privacy_also_fails` pins the rule.
- `LicenceDecision.ADMIT_WITH_CONDITION` is admission, not denial. Chapter 02 is explicit: the condition is an artifact (counsel opinion PDF, share-alike propagation tag) attached to the slice before release, not enforced per item. `test_admit_with_condition_is_not_a_deny` keeps that contract visible.
- The `posture_in_force` field is the counsel-posture axis Chapter 04 introduces. Flip from `moderate` to `conservative` and the per-item log records the flip; if counsel re-postures later, the log explains which items were decided under which posture. `test_posture_in_force_is_recorded_on_every_decision` keeps that contract visible.
- The stubs are *only* stubs. In production, each is a real evaluator — Chapter 02's `LicenseGate` (Exercise 02), Chapter 03's `OptOutEvaluator` (Exercise 03), the mod-101 Chapter 04 privacy admit, and a Chapter 04 jurisdiction evaluator parameterised on counsel's posture per jurisdiction. The composite's shape is stable across those substitutions; that is the point of chaining through typed child-result dataclasses.
- Chapter 04's `composite_admit` pseudocode is tight — the lab's implementation is intentionally close. Where you would extend in production: structured logging (OpenTelemetry attributes per sub-decision), per-counterparty throttling, and the retroactive-revalidation path described in Chapter 03 §"Caching, revalidation, and TTL".

## If you want to go further (optional)

- Replace the stub `OptOutEvaluator` with Exercise 03's evaluator — pass it a fixture directory of `robots.txt` / `ai.txt` / meta-noai pages and watch the composite's `opt_out_decision` column populate from real signals.
- Replace the stub `LicenseGate` with Exercise 02's gate reading `policy/data_license_policy_v2026-11-04.yaml`; add tests that exercise the `admit-with-condition` path for CC-BY-SA with a counsel-opinion PDF attached to the slice.
- Add a `jurisdiction_posture` parameter that reads from a per-source `jurisdiction_posture` tag (Chapter 04 §"Other jurisdictions worth knowing exist"). Write tests that flip the posture from `moderate` to `conservative` for the same item and watch the composite's verdict flip.
- Swap the JSONL log for an OpenLineage emitter (preview of mod-109) — each composite decision becomes a lineage event whose outputs are the admitted items.
- Add a retroactive-revalidation subcommand that reads an older `decision_log.jsonl`, re-runs each item through the current composite, and emits a filter-list of items whose status flipped. Chapter 03 §"Caching, revalidation, and TTL" names this behaviour as the opt-out revalidation step.

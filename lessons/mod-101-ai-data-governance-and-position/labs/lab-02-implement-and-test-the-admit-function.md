# Lab 02 — Implement and Test the Per-Item Admit Function

**Estimated duration:** 90 minutes
**Chapter dependency:** [Chapter 04 — The Privacy and Copyright Substrate](../04-gdpr-ccpa-and-dsm-article-4.md). Chapters [02](../02-iso-suite-for-training-data.md) and [03](../03-eu-ai-act-articles-10-53-72.md) are helpful for the vocabulary.
**Relationship to the exercises:** Exercise 04 asks you to *design* the lawful-basis record and sketch the admit function as pseudocode. This lab asks you to *implement it as runnable Python* with a decision log and pytest coverage for each deny reason. Do not repeat Exercise 04's design work — this lab is about the code and the tests.

## Objective

By the end of this lab you will have:

1. A `admit.py` module that implements the per-item admit function from Chapter 04 (GDPR Articles 5–9, CCPA/CPRA sale-sharing opt-out, EU DSM Article 4(3) reservation, C2PA opt-out).
2. A `decision_log.jsonl` emitted by the function with one line per decision, including a stable `deny_reason` code.
3. A `test_admit.py` pytest module with one test per deny reason, plus happy-path admits for each of the four Chapter 04 source categories.
4. All tests passing (`pytest -q` exits 0).

The point: *legal signs the posture; you own the code and the log* (Chapter 04). This lab is the code and the log.

## Prerequisites

- You have read Chapter 04.
- You have Python 3.11+ and can install packages. Only `pytest` is required.
- You understand pytest at the "writes a test function, runs it, passes or fails" level.
- You have about 90 minutes. This lab is longer in step count than Lab 01 because it is implementation.

## Step 1 — Make the scratch directory and install pytest

```bash
mkdir -p lab02-admit && cd lab02-admit
python -m venv .venv
. .venv/bin/activate
pip install pytest
```

**Checkpoint:** `pytest --version` prints a version.

## Step 2 — Draft the data model

Create `models.py`:

```python
"""Per-item data model for the admit function.

These dataclasses are deliberately thin. Field semantics are documented in
Chapter 04. A real pipeline would layer in validators and persistence — this
lab keeps them inert so the admit function is testable in isolation.
"""
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Literal

LawfulBasis = Literal[
    "6(1)(a)-consent",
    "6(1)(b)-contract",
    "6(1)(f)-legitimate-interests",
    "n/a-no-personal-data",
]

Article9Basis = Literal[
    "9(2)(a)-explicit-consent",
    "9(2)(e)-manifestly-public",
    "9(2)(g)-substantial-public-interest",
    "10(5)-bias-correction",          # EU AI Act narrow exception
]


@dataclass(frozen=True)
class CrawlContract:
    """The compiled form of the organisation's legal posture (Chapter 04)."""
    allow_legitimate_interests: bool = True
    allow_special_categories: bool = False
    retention_window_days: int = 365


@dataclass
class Item:
    item_sha: str
    source_id: str
    category: Literal["first-party", "crawl", "licensed", "synthetic"]
    created_at: datetime                        # when item entered the pipeline

    # GDPR
    contains_pii: bool = False
    lawful_basis: LawfulBasis | None = None
    lia_id: str | None = None                   # required for 6(1)(f)
    consent_record_id: str | None = None        # required for 6(1)(a)
    contract_id: str | None = None              # required for 6(1)(b)
    special_category: bool = False
    article_9_basis: Article9Basis | None = None

    # CCPA / CPRA
    california_resident: bool = False
    gpc_signalled: bool = False

    # EU DSM Article 4(3) opt-out signals
    robots_txt_reserved: bool = False           # RFC 9309 directive
    ai_txt_reserved: bool = False
    meta_noai: bool = False
    c2pa_opt_out: bool = False

    # Synthetic data inheritance
    parent_items: list[str] = field(default_factory=list)
```

**Checkpoint:** `python -c "import models; print(models.CrawlContract())"` prints a dataclass repr with no error.

## Step 3 — Implement the admit function

Create `admit.py`:

```python
"""Per-item admit function — the compiled form of the organisation's legal posture.

Each return path carries a stable `deny_reason` code. Legal reviews the code list;
the engine never admits an item without emitting one record either way.

Every decision is written to decision_log.jsonl (one JSON object per line).
"""
from __future__ import annotations

import json
from dataclasses import dataclass, asdict
from datetime import datetime, timezone

from models import CrawlContract, Item


# Stable deny-reason codes. Add here; never remove. Legal reviews this list.
DENY_NO_GDPR_BASIS = "deny_no_gdpr_basis"
DENY_LEGITIMATE_INTERESTS_WITHOUT_LIA = "deny_legitimate_interests_without_lia"
DENY_LEGITIMATE_INTERESTS_FORBIDDEN_BY_CONTRACT = "deny_legitimate_interests_forbidden_by_contract"
DENY_ARTICLE_9_NO_EXCEPTION = "deny_article_9_no_exception"
DENY_CCPA_GPC_OPT_OUT = "deny_ccpa_gpc_opt_out"
DENY_DSM_ARTICLE_4_RESERVED = "deny_dsm_article_4_reserved"
DENY_C2PA_OPT_OUT = "deny_c2pa_opt_out"
DENY_RETENTION_EXPIRED = "deny_retention_expired"

ADMIT = "admit"


@dataclass
class Decision:
    item_sha: str
    source_id: str
    outcome: str          # "admit" or one of the DENY_* codes
    decided_at: str
    details: dict


def _requires_basis(item: Item) -> bool:
    return item.contains_pii or item.special_category


def _basis_evidence_present(item: Item) -> bool:
    if item.lawful_basis == "6(1)(a)-consent":
        return item.consent_record_id is not None
    if item.lawful_basis == "6(1)(b)-contract":
        return item.contract_id is not None
    if item.lawful_basis == "6(1)(f)-legitimate-interests":
        return item.lia_id is not None
    if item.lawful_basis == "n/a-no-personal-data":
        return not item.contains_pii and not item.special_category
    return False


def admit(item: Item, contract: CrawlContract, now: datetime | None = None) -> Decision:
    """Return a Decision. Side effect: none — the caller appends to the log."""
    now = now or datetime.now(timezone.utc)

    # DSM Article 4(3) — rightsholder reservation (any of robots.txt, ai.txt, <meta noai>)
    if item.robots_txt_reserved or item.ai_txt_reserved or item.meta_noai:
        return Decision(item.item_sha, item.source_id, DENY_DSM_ARTICLE_4_RESERVED,
                        now.isoformat(),
                        {"robots_txt": item.robots_txt_reserved,
                         "ai_txt": item.ai_txt_reserved,
                         "meta_noai": item.meta_noai})

    # Per-asset provenance opt-out
    if item.c2pa_opt_out:
        return Decision(item.item_sha, item.source_id, DENY_C2PA_OPT_OUT,
                        now.isoformat(), {})

    # CCPA/CPRA sale-sharing opt-out via GPC
    if item.california_resident and item.gpc_signalled:
        return Decision(item.item_sha, item.source_id, DENY_CCPA_GPC_OPT_OUT,
                        now.isoformat(), {})

    # GDPR Article 9 — special categories require a 9(2) exception
    if item.special_category:
        if item.article_9_basis is None or not contract.allow_special_categories:
            return Decision(item.item_sha, item.source_id, DENY_ARTICLE_9_NO_EXCEPTION,
                            now.isoformat(),
                            {"article_9_basis": item.article_9_basis,
                             "contract_allows": contract.allow_special_categories})

    # GDPR Article 6 — personal data requires a lawful basis with evidence
    if _requires_basis(item):
        if item.lawful_basis is None or not _basis_evidence_present(item):
            return Decision(item.item_sha, item.source_id, DENY_NO_GDPR_BASIS,
                            now.isoformat(),
                            {"lawful_basis": item.lawful_basis})
        if item.lawful_basis == "6(1)(f)-legitimate-interests":
            if not contract.allow_legitimate_interests:
                return Decision(item.item_sha, item.source_id,
                                DENY_LEGITIMATE_INTERESTS_FORBIDDEN_BY_CONTRACT,
                                now.isoformat(), {})
            if item.lia_id is None:
                return Decision(item.item_sha, item.source_id,
                                DENY_LEGITIMATE_INTERESTS_WITHOUT_LIA,
                                now.isoformat(), {})

    # Retention — Chapter 04 Article 5 storage limitation
    age_days = (now - item.created_at).days
    if age_days > contract.retention_window_days:
        return Decision(item.item_sha, item.source_id, DENY_RETENTION_EXPIRED,
                        now.isoformat(), {"age_days": age_days})

    return Decision(item.item_sha, item.source_id, ADMIT, now.isoformat(), {})


def write_decision(decision: Decision, path: str = "decision_log.jsonl") -> None:
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps(asdict(decision)) + "\n")
```

**Checkpoint:** `python -c "import admit, models; print(admit.ADMIT)"` prints `admit`.

## Step 4 — Draft the pytest fixtures

Create `conftest.py`:

```python
"""Shared test fixtures for the admit tests."""
from datetime import datetime, timedelta, timezone

import pytest

from models import CrawlContract, Item


@pytest.fixture
def now():
    return datetime(2026, 11, 16, 9, 0, 0, tzinfo=timezone.utc)


@pytest.fixture
def contract():
    return CrawlContract(
        allow_legitimate_interests=True,
        allow_special_categories=False,
        retention_window_days=365,
    )


@pytest.fixture
def first_party_item(now):
    return Item(
        item_sha="sha-fp-001",
        source_id="first-party-chat",
        category="first-party",
        created_at=now - timedelta(days=10),
        contains_pii=True,
        lawful_basis="6(1)(a)-consent",
        consent_record_id="consent-2026-001",
    )


@pytest.fixture
def crawl_item(now):
    return Item(
        item_sha="sha-crawl-001",
        source_id="example-tech-blog",
        category="crawl",
        created_at=now - timedelta(days=10),
        contains_pii=False,
        lawful_basis="n/a-no-personal-data",
    )


@pytest.fixture
def licensed_item(now):
    return Item(
        item_sha="sha-lic-001",
        source_id="stackx-partner",
        category="licensed",
        created_at=now - timedelta(days=10),
        contains_pii=True,
        lawful_basis="6(1)(b)-contract",
        contract_id="C-2026-AX-07",
    )


@pytest.fixture
def synthetic_item(now):
    return Item(
        item_sha="sha-syn-001",
        source_id="teacher-distill-run-04",
        category="synthetic",
        created_at=now - timedelta(days=1),
        contains_pii=False,
        lawful_basis="n/a-no-personal-data",
        parent_items=["sha-fp-001", "sha-crawl-001"],
    )
```

**Checkpoint:** `pytest --collect-only -q` lists the fixtures without errors.

## Step 5 — Write one test per deny reason, plus happy paths

Create `test_admit.py`:

```python
"""One test per deny reason, plus one admit per source category."""
from dataclasses import replace
from datetime import timedelta

from admit import (
    admit,
    ADMIT,
    DENY_NO_GDPR_BASIS,
    DENY_LEGITIMATE_INTERESTS_WITHOUT_LIA,
    DENY_LEGITIMATE_INTERESTS_FORBIDDEN_BY_CONTRACT,
    DENY_ARTICLE_9_NO_EXCEPTION,
    DENY_CCPA_GPC_OPT_OUT,
    DENY_DSM_ARTICLE_4_RESERVED,
    DENY_C2PA_OPT_OUT,
    DENY_RETENTION_EXPIRED,
)


# -------- happy paths: one admit per Chapter 04 source category -----------

def test_first_party_with_consent_admitted(first_party_item, contract, now):
    d = admit(first_party_item, contract, now=now)
    assert d.outcome == ADMIT

def test_crawl_with_no_personal_data_admitted(crawl_item, contract, now):
    d = admit(crawl_item, contract, now=now)
    assert d.outcome == ADMIT

def test_licensed_with_contract_admitted(licensed_item, contract, now):
    d = admit(licensed_item, contract, now=now)
    assert d.outcome == ADMIT

def test_synthetic_derived_from_cleared_parents_admitted(synthetic_item, contract, now):
    d = admit(synthetic_item, contract, now=now)
    assert d.outcome == ADMIT


# -------- deny paths: one per code ---------------------------------------

def test_deny_no_gdpr_basis(crawl_item, contract, now):
    # crawl item that accidentally has PII but no lawful basis
    item = replace(crawl_item, contains_pii=True, lawful_basis=None)
    d = admit(item, contract, now=now)
    assert d.outcome == DENY_NO_GDPR_BASIS

def test_deny_legitimate_interests_without_lia(crawl_item, contract, now):
    item = replace(crawl_item,
                   contains_pii=True,
                   lawful_basis="6(1)(f)-legitimate-interests",
                   lia_id=None)
    d = admit(item, contract, now=now)
    assert d.outcome == DENY_NO_GDPR_BASIS  # LIA missing surfaces as no-basis evidence

def test_deny_legitimate_interests_forbidden_by_contract(crawl_item, contract, now):
    # Contract posture: LI forbidden for this slice
    stricter = replace(contract, allow_legitimate_interests=False)
    item = replace(crawl_item,
                   contains_pii=True,
                   lawful_basis="6(1)(f)-legitimate-interests",
                   lia_id="LIA-2026-04")
    d = admit(item, stricter, now=now)
    assert d.outcome == DENY_LEGITIMATE_INTERESTS_FORBIDDEN_BY_CONTRACT

def test_deny_article_9_no_exception(first_party_item, contract, now):
    item = replace(first_party_item, special_category=True, article_9_basis=None)
    d = admit(item, contract, now=now)
    assert d.outcome == DENY_ARTICLE_9_NO_EXCEPTION

def test_deny_ccpa_gpc_opt_out(first_party_item, contract, now):
    item = replace(first_party_item, california_resident=True, gpc_signalled=True)
    d = admit(item, contract, now=now)
    assert d.outcome == DENY_CCPA_GPC_OPT_OUT

def test_deny_dsm_article_4_reserved_via_robots_txt(crawl_item, contract, now):
    item = replace(crawl_item, robots_txt_reserved=True)
    d = admit(item, contract, now=now)
    assert d.outcome == DENY_DSM_ARTICLE_4_RESERVED

def test_deny_dsm_article_4_reserved_via_ai_txt(crawl_item, contract, now):
    item = replace(crawl_item, ai_txt_reserved=True)
    d = admit(item, contract, now=now)
    assert d.outcome == DENY_DSM_ARTICLE_4_RESERVED

def test_deny_c2pa_opt_out(crawl_item, contract, now):
    item = replace(crawl_item, c2pa_opt_out=True)
    d = admit(item, contract, now=now)
    assert d.outcome == DENY_C2PA_OPT_OUT

def test_deny_retention_expired(first_party_item, contract, now):
    item = replace(first_party_item, created_at=now - timedelta(days=400))
    d = admit(item, contract, now=now)
    assert d.outcome == DENY_RETENTION_EXPIRED


# -------- ordering: DSM reservation is checked before GDPR --------------

def test_dsm_reservation_wins_over_missing_basis(crawl_item, contract, now):
    # Both deny conditions present: DSM reservation and missing GDPR basis.
    # Chapter 04's admit function checks the source reservation first.
    item = replace(crawl_item, robots_txt_reserved=True,
                   contains_pii=True, lawful_basis=None)
    d = admit(item, contract, now=now)
    assert d.outcome == DENY_DSM_ARTICLE_4_RESERVED
```

**Checkpoint:** `pytest -q` prints `14 passed` (or whatever the count is after your ordering choices) in **under 1 second**, and **exits 0**.

**Expected output (shape):**

```
..............                                               [100%]
14 passed in 0.08s
```

If any test fails, read the failing assertion and fix `admit.py`. Do not change the test to make it pass; the test is the specification.

## Step 6 — Run the function over a sample and inspect the log

Create `sample_run.py`:

```python
"""Run a handful of items through admit() and emit decision_log.jsonl."""
from datetime import datetime, timedelta, timezone

from admit import admit, write_decision
from models import CrawlContract, Item


def main():
    now = datetime(2026, 11, 16, 9, 0, 0, tzinfo=timezone.utc)
    contract = CrawlContract()

    items = [
        Item("sha-01", "first-party-chat", "first-party",
             created_at=now - timedelta(days=5),
             contains_pii=True, lawful_basis="6(1)(a)-consent",
             consent_record_id="C-2026-001"),
        Item("sha-02", "example-tech-blog", "crawl",
             created_at=now - timedelta(days=5),
             robots_txt_reserved=True, lawful_basis="n/a-no-personal-data"),
        Item("sha-03", "stackx-partner", "licensed",
             created_at=now - timedelta(days=5),
             contains_pii=True, lawful_basis="6(1)(b)-contract",
             contract_id="C-2026-AX-07"),
        Item("sha-04", "first-party-chat", "first-party",
             created_at=now - timedelta(days=5),
             contains_pii=True, lawful_basis=None),           # no basis
        Item("sha-05", "first-party-chat", "first-party",
             created_at=now - timedelta(days=5),
             california_resident=True, gpc_signalled=True,
             contains_pii=True, lawful_basis="6(1)(a)-consent",
             consent_record_id="C-2026-002"),
    ]

    # Clear old log
    open("decision_log.jsonl", "w").close()

    for item in items:
        write_decision(admit(item, contract, now=now))

    print("Done. Inspect decision_log.jsonl.")


if __name__ == "__main__":
    main()
```

Run it and inspect:

```bash
python sample_run.py
cat decision_log.jsonl
```

**Expected output (shape — one line per item, deny codes stable):**

```
{"item_sha": "sha-01", "source_id": "first-party-chat", "outcome": "admit", "decided_at": "2026-11-16T09:00:00+00:00", "details": {}}
{"item_sha": "sha-02", "source_id": "example-tech-blog", "outcome": "deny_dsm_article_4_reserved", "decided_at": "2026-11-16T09:00:00+00:00", "details": {"robots_txt": true, "ai_txt": false, "meta_noai": false}}
{"item_sha": "sha-03", "source_id": "stackx-partner", "outcome": "admit", "decided_at": "2026-11-16T09:00:00+00:00", "details": {}}
{"item_sha": "sha-04", "source_id": "first-party-chat", "outcome": "deny_no_gdpr_basis", "decided_at": "2026-11-16T09:00:00+00:00", "details": {"lawful_basis": null}}
{"item_sha": "sha-05", "source_id": "first-party-chat", "outcome": "deny_ccpa_gpc_opt_out", "decided_at": "2026-11-16T09:00:00+00:00", "details": {}}
```

**Checkpoint:** `wc -l decision_log.jsonl` prints **5**, and `grep -c '"outcome": "admit"' decision_log.jsonl` prints **2**.

## Step 7 — Convince yourself the log is machine-parseable

```bash
python -c "
import json
seen = set()
for line in open('decision_log.jsonl'):
    r = json.loads(line)
    seen.add(r['outcome'])
print('outcomes seen:', sorted(seen))
"
```

**Expected output:**

```
outcomes seen: ['admit', 'deny_ccpa_gpc_opt_out', 'deny_dsm_article_4_reserved', 'deny_no_gdpr_basis']
```

**Checkpoint:** every outcome in the log parses as valid JSON and every `outcome` value is either `admit` or one of the deny codes exported from `admit.py`.

## What you have at the end

```
lab02-admit/
├── models.py              # Item + CrawlContract dataclasses
├── admit.py               # admit() + deny-reason codes + write_decision()
├── conftest.py            # pytest fixtures
├── test_admit.py          # 1 test per deny reason + 1 happy path per source category
├── sample_run.py          # exercises admit() over sample items
└── decision_log.jsonl     # one JSON record per decision
```

## Discussion

- Notice the *ordering* of checks in `admit()` — DSM reservation is tested before GDPR basis. `test_dsm_reservation_wins_over_missing_basis` pins that ordering. Chapter 04's `admit` function pseudocode uses this order; if you flip it, an item that is both rightsholder-reserved and missing a GDPR basis is logged as a privacy denial, which misleads a rightsholder audit later.
- The deny codes are a *stable public interface*. Legal signs off on the list. If you ever need a new reason, add to the list — never silently reclassify.
- The `contract` parameter is the compiled form of legal's posture. `allow_legitimate_interests=False` is how you flip the pipeline when counsel changes its posture. The test `test_deny_legitimate_interests_forbidden_by_contract` exercises that flip.
- `parent_items` on synthetic items is a hook for inheritance: a stricter check would walk `parent_items` and re-run admit. The lab leaves that as a stretch; the chapter calls it out explicitly ("a synthetic pair derived from a (2) source that was opted out at crawl time should never have been generated").

## If you want to go further (optional)

- Extend `admit()` to walk `parent_items` and refuse a synthetic item whose any ancestor would be denied under the current contract. Add a test `test_synthetic_inherits_parent_denial`.
- Add an `article_14_notice_pointer` check: refuse any first-party item whose source's privacy notice URL is null, reflecting GDPR Article 14 obligations.
- Add a per-source *data-subject-objection* registry (GDPR Article 21): a `DENY_SUBJECT_OBJECTED` code that keys off a subject-identifier hash.
- Swap the JSONL log for an OpenLineage emitter (see mod-109 preview) — the decision becomes a lineage event.
- Add a CLI (`python -m admit path/to/items.jsonl`) that reads items from stdin/JSONL and emits decisions to stdout/JSONL, suitable for piping through a dataflow step.

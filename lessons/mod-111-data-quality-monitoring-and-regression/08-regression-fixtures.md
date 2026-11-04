# Chapter 08 — Regression Fixtures: Turning Incidents into Permanent Guards

## Why this chapter exists

A monitor fires. The on-call rotation triages the alert. The fix lands. Four months later, a different engineer merges a well-intentioned refactor that re-introduces the same bug. The monitor fires again. The on-call rotation triages again. The fix lands again. The team's institutional memory of why that monitor exists has lasted exactly one personnel rotation.

The remedy is a **regression fixture**: a minimal reproducible slice plus a pass/fail check that re-runs on every pipeline change. Each resolved incident produces one; the fixture library grows; the next re-introduction of the bug is caught in the PR, not in production. By the end of the module this will be one of the two cumulative assets you own (the other being the monitoring contract itself).

This chapter teaches the fixture-authoring discipline, the determinism and minimality constraints that make fixtures survive, and the CI integration that makes them *mandatory* instead of optional.

## What a regression fixture is (and isn't)

A regression fixture has four properties. All four are required; dropping any one makes the fixture less useful than nothing.

1. **Minimal.** The smallest input that reproduces the issue. Ten rows is better than ten thousand; one document is better than a hundred — if either can still trigger the behaviour.
2. **Reproducible.** Fixed inputs, fixed random seeds, fixed tool versions, fixed reference data. Running the fixture twice produces byte-identical outputs.
3. **Pass/fail-checked.** An explicit assertion, not "it runs". The assertion names the observable that the incident was about ("the quality filter keeps Spanish forum text at the same rate as English forum text") and compares it to a threshold.
4. **CI-gating.** Runs on every PR to the pipeline repo. A failing fixture blocks merge, not "shows a warning".

A regression fixture is *not*:

- A unit test for a function. Unit tests catch code-level regressions; fixtures catch *data-shape* regressions.
- A full integration test. Fixtures run on the minimal slice; integration tests exercise the full pipeline at operational scale.
- A benchmark. Fixtures assert correctness, not performance.
- A monitor. Monitors watch slices in production; fixtures watch PRs before they become slices.

The two together — monitors in production, fixtures in CI — form a closed loop. Every monitor alert that led to a fix should spawn a fixture; every fixture addition is tied to the monitor id that caught the original incident.

## The fixture directory layout

One fixture per directory. The directory is the unit; it owns its inputs, its expected outputs, its assertion code, and its provenance note.

```
fixtures/
├── f.es_forum_text_quality_filter.v1/
│   ├── README.md                       # one-page incident narrative
│   ├── incident.yaml                   # link to the alert, the PR, the version-pin
│   ├── input/
│   │   ├── documents.parquet           # the minimal input slice
│   │   ├── documents.parquet.sha256
│   │   └── config.yaml                 # any pipeline config overrides
│   ├── reference_outputs/
│   │   ├── quality_filter_output.parquet
│   │   └── quality_filter_output.parquet.sha256
│   ├── assertions/
│   │   └── test_quality_filter.py      # the pytest file
│   └── fixture.yaml                    # the fixture's own metadata
```

The `fixture.yaml` is the fixture's identity card:

```yaml
# fixture.yaml
id: f.es_forum_text_quality_filter.v1
version: 1
authored_at: 2026-11-05
author: ai-data-engineer@acme
incident:
  id: INC-2026-11-04-es-balance
  monitor_that_caught_it: m.population.lang_psi.v1
  version_pin_rejected: cc_en.v3.2026-11-04T08:17:00Z
pipeline_stage: quality_filter
input_sha256: 7b19d…
expected_output_sha256: 2c88f…
determinism:
  rng_seed: 42
  pinned_dependencies:
    - package: pandera
      version: 0.19.0
    - package: fasttext
      version: 0.9.2
on_failure: block_pr
cadence: every_pr
```

## The incident narrative — writing `README.md`

A fixture lives for years. The engineer who re-reads it four years later needs the *why*, not the mechanics. Keep the README short, factual, and link-rich.

```markdown
# fixture: f.es_forum_text_quality_filter.v1

## Incident

On 2026-11-04 the slice `cc_en.v3.2026-11-04T08:17Z` fired `m.population.lang_psi.v1`
at retrain-blocking severity. The Spanish cell had dropped from 2.57% to 0.41% of
the corpus (relative delta −84%).

## Root cause

The `quality_filter` stage used a length / formatting heuristic tuned against
English-anchored proxies. Mexican-Spanish forum text whose paragraph structure
and token-per-sentence distribution differed from English was over-filtered.

Per-language filter removal-rate monitor (`m.population.quality_filter_removal_rate_by_language.v1`)
showed `es` removal rate jumped from 0.41 → 0.61 while `en` stayed at 0.18.

## Fix

PR [#1234] replaced the single-language threshold with a per-language threshold
calibrated against a per-language reference distribution.

## What this fixture guards

The fixture feeds 500 Spanish-forum documents + 500 English-forum documents
through the quality filter. The assertion is that the per-language removal rate
matches reference ± 5 percentage points.

A future refactor that returns the quality filter to an English-anchored heuristic
(whether by accident or by well-meaning simplification) fails this fixture.

## Primary monitor
`m.population.quality_filter_removal_rate_by_language.v1`

## Related
- PR #1234 — the fix
- incident.yaml — machine-readable incident record
```

Every fixture's README follows the same shape: incident, root cause, fix, what the fixture guards, references. Readers four years later arrive at the fixture through a failed CI run — the README is their first stop.

## Minimality — how to shrink a slice

A production incident is usually caught on a 10M-row slice. The fixture that reproduces it is 100–1,000 rows. Minimality is engineering discipline; here are the moves.

1. **Bisect by partition.** Take the offending slice; split by `source` or `language` or `url_host_tld`; find the smallest partition that still triggers the assertion.
2. **Delta-debug within a partition.** Quartile-halve the documents in the triggering partition; keep the half that still fails; repeat until halving stops reproducing.
3. **Shrink per-document.** For rows that survive minimisation, trim the text to the minimum that still triggers. For very large documents this cuts fixture size dramatically without affecting the bug.
4. **Mix in a control.** Add a comparably sized negative-case sample (documents that *should not* trigger) to the fixture so the assertion has both sides. A fixture that only exercises the positive case overfits the test to the bug.
5. **Anonymise.** If the input contains any licensable or personal content, redact or synthesise before committing. The fixture lives forever in the repo; so do its inputs.

Record the minimisation pedigree in `incident.yaml` — original slice SHA, partition path, minimisation seed. If a reader wants to confirm the fixture still reproduces the original issue on the full slice, the breadcrumb makes it possible.

## Determinism — the three axes

A fixture that is non-deterministic is worse than nothing: it produces intermittent CI failures that teams learn to re-run until green. Pin determinism on three axes.

- **Input determinism.** The input Parquet is content-addressed; its SHA-256 is in `fixture.yaml`. Changing the input is a version bump.
- **Dependency determinism.** The pipeline stage's dependencies (pandera, fasttext, Presidio, etc.) are version-pinned at fixture-authoring time and recorded in `fixture.yaml`. The CI job installs exactly those versions.
- **Runtime determinism.** The pipeline stage's RNG seed is pinned. If the stage uses a model (classifier, embedding model), the model artifact SHA is recorded.

The assertion itself must then be able to hit the pinned expected output. One pattern — "the stage's output SHA matches `expected_output_sha256`" — is strict and useful, but brittle (a reordering of non-semantic fields fails). A gentler pattern — "the stage's output, after projection to the observable column, has property P" — is what the assertion checks against.

## The assertion — what the fixture actually asserts

An assertion is a single, named, diffable statement about the observable the incident was about. Not "the pipeline runs without error". Not "the output is the same". *The observable* behind the original alert.

For the Spanish-forum-text example, the assertion is:

```python
# assertions/test_quality_filter.py
import pandas as pd
import pathlib
import yaml

FIXTURE_DIR = pathlib.Path(__file__).resolve().parent.parent

def test_per_language_removal_rate_parity_es_en():
    fixture = yaml.safe_load((FIXTURE_DIR / "fixture.yaml").read_text())
    docs = pd.read_parquet(FIXTURE_DIR / "input" / "documents.parquet")
    filtered = run_quality_filter(docs, config_path=FIXTURE_DIR / "input" / "config.yaml")

    removal_rate = (
        (len(docs) - len(filtered))
        / len(docs)
    )
    per_lang_removal = (
        docs.groupby("language").size() - filtered.groupby("language").size()
    ) / docs.groupby("language").size()

    es_rate = per_lang_removal.loc["es"]
    en_rate = per_lang_removal.loc["en"]
    assert abs(es_rate - en_rate) <= 0.05, (
        f"per-language removal-rate parity broken: "
        f"es={es_rate:.3f} vs en={en_rate:.3f}, delta={abs(es_rate - en_rate):.3f} > 0.05 "
        f"(monitor m.population.quality_filter_removal_rate_by_language.v1 would fire)"
    )
```

The assertion names the monitor it guards. That is the link that tells future-you "when this fails, go look at the monitor's history". The failure message is precise enough that the CI log alone is diagnostic.

## pytest integration

Fixtures are pytest tests. They live next to the pipeline code. The CI config is one line:

```yaml
# .github/workflows/ci.yml (illustrative)
- name: Run regression fixtures
  run: pytest fixtures/ -v
```

Three operational notes:

- **Fail-fast is wrong here.** Run *all* fixtures to collect every failure, not just the first. A broken refactor often breaks multiple fixtures; the engineer needs the full list.
- **Mark fixtures slow if they are.** Fixtures that touch GPU / large models can be slow. Split into `@pytest.mark.slow` (ran on PRs touching their stage) and default (ran on every PR).
- **Record fixture IDs in the test output.** JUnit XML emitted by pytest should carry the fixture id; the CI job routes a failing fixture to the responsible area (quality filter, dedup, PII).

## The fixture library as a cumulative asset

Over time, the `fixtures/` directory grows: one fixture per incident, each one guarding a specific observable the pipeline has been wrong about. The library has three properties worth cultivating:

1. **It describes the pipeline's failure surface.** A new engineer joining the team should read every fixture's README to learn how the pipeline *has* failed historically. That is institutional memory in executable form.
2. **It constrains refactors.** A proposed simplification that cannot preserve every fixture is a proposed loss of resolved institutional learning. The fixtures are a load-bearing constraint on the refactor budget.
3. **It is auditable.** For regulated use cases (EU AI Act Article 15 — robustness; NIST AI RMF MANAGE 4.3), a fixture library is a defensible answer to "how do you ensure resolved quality issues stay resolved?". Record the library's size, growth rate, and failure history in the monitoring contract's preamble.

The anti-pattern to avoid: treating fixtures as "temporary tests that will be deleted when the bug is forgotten". **Fixtures do not expire.** A fixture is retired only when the pipeline stage it guards is removed, and even then the fixture's `fixture.yaml` is archived with a `retired:` annotation rather than deleted.

## What you ship from this module to the paired projects repo

Each project in the paired `ai-data-engineer-learning-solutions` repo will include at least one fixture under its `fixtures/` directory, with the exact layout defined above. The acceptance criterion for a project is not just "the pipeline runs" but "at least one regression fixture exists, is CI-gated, and asserts a named observable".

Exercise 05 in this module walks you through authoring a fixture against a pipeline stage you build in [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) or [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md).

## The rare cases where a fixture is the wrong tool

Three situations where a fixture is not the answer:

1. **The incident was about operational substrate, not data behaviour.** A networking failure, a storage-backend outage, a CI config break — these are not fixture material; they are runbook material for `ai-infra-mlops`.
2. **The observable is inherently non-deterministic.** A fixture that relies on a hosted API with changing outputs cannot be a fixture. Replace with a mock or route the test elsewhere.
3. **The scale is irreducible.** Some behaviour emerges only at corpus scale (MinHash-LSH behaviour on 10 M documents, embedding-drift tests that need 10 k samples per side). For these, a fixture at full scale is not possible; instead, author a sampling-based *property test* (hypothesis / fast-check–style) that exercises the invariant on randomised small inputs, and record the full-scale check as a scheduled production monitor instead.

## Summary

- A regression fixture is a minimal, reproducible, pass/fail-checked, CI-gating artifact. All four properties are required.
- Each fixture has its own directory: `README.md` with incident narrative, `fixture.yaml` with metadata, `input/` with the minimal slice, `reference_outputs/` with pinned expected output, `assertions/` with the pytest file.
- The assertion names an observable (not "runs without error") and names the monitor it guards.
- Minimality is engineering discipline: partition-bisect, delta-debug, per-document shrink, mix in a control, anonymise. Record the pedigree.
- Determinism is enforced on three axes: input, dependencies, runtime.
- Fixtures are cumulative; the library describes the pipeline's historical failure surface. It is auditable and constrains refactors.
- Three edge cases justify not writing a fixture: substrate incidents, non-deterministic observables, irreducibly-large scale.

Chapter 09 is the operational layer — how alerts get routed, how the evidence is attached to the dataset version-pin, and how the role boundaries this module has been drawing all along get wired into the on-call reality.

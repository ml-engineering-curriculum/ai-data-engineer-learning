# Labs for mod-101-ai-data-governance-and-position

Guided, step-by-step hands-on sessions for the module. Each lab has a stated duration, prerequisites, numbered steps with commands or code, expected output, and checkpoints. Labs are *guided practice*; they differ from the module's `exercises/`, which are open-ended design work.

- [Lab 01 — Build a Cross-Framework `reg_posture` Tag for a Single Slice](lab-01-build-a-cross-framework-reg-posture-tag.md) — 90 min. Stitch NIST AI RMF, ISO/IEC 42001 Annex A.7, EU AI Act Article 10 or 53, SR 11-7, and Colorado AI Act into a single per-slice `reg_posture.yaml` plus a flat auditor-grade cross-reference CSV, and validate that reusable artifacts (source manifest, preparation log) appear across regimes. Based on Chapters 01, 02, 03, and 05.
- [Lab 02 — Implement and Test the Per-Item Admit Function](lab-02-implement-and-test-the-admit-function.md) — 90 min. Implement the Chapter 04 admit function in runnable Python with stable deny-reason codes, write pytest coverage for every deny path plus one happy-path admit per Chapter 04 source category, and emit a machine-parseable `decision_log.jsonl`. Based on Chapter 04.

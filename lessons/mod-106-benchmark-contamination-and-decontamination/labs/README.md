# Labs for mod-106-benchmark-contamination-and-decontamination (Benchmark Contamination Detection and Decontamination Engineering)

These labs are guided practice sessions. Each is a small, self-contained hands-on exercise anchored to one chapter and *distinct* from the module's [exercises](../exercises). Pick the lab whose chapter you want to internalise before attempting the longer exercises.

| # | Title | Chapter | Duration |
|---|---|---|---|
| [01](lab-01-scoring-harness-alignment-roundtrip.md) | The Scoring-Harness Alignment Round-Trip | [06](../06-lm-eval-harness-vs-dolma-practice.md) (+ [02](../02-the-eval-surface-the-firewall-defends.md), [03](../03-n-gram-overlap-detection.md)) | 60 min |
| [02](lab-02-min-k-prob-null-distribution.md) | Plot the Min-K% Prob Null Distribution on a Small Open Model | [07](../07-defending-against-min-k-prob-and-oren-exchangeability.md) | 75 min |
| [03](lab-03-emit-the-one-page-decontamination-summary.md) | Emit the One-Page Decontamination Summary From `decontamination_log.yaml` | [08](../08-the-decontamination-log-as-audit-evidence.md) | 60 min |

The labs compose loosely with the exercises: Lab 01's `recall_table.md` and `policy_note.md` are drop-in evidence for Exercise 02's round-trip test; Lab 02's `min_k_prob.py` is a drop-in starter for Exercise 03's Min-K% implementation; Lab 03's emitter is the summary shape Exercise 05 drives against your real Exercise 01–04 outputs. Running the three back-to-back is a tour of the module's three most load-bearing engineering surfaces — the normalisation contract between firewall and scoring harness, what a null Min-K% distribution looks like, and the audit-evidence handoff that turns the firewall into a first-class quality control.

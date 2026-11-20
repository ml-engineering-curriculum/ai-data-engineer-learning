# Labs for mod-105-deduplication-at-scale (Deduplication at Trillion-Token Scale)

These labs are guided practice sessions. Each is a small, self-contained hands-on exercise anchored to one chapter and *distinct* from the module's [exercises](../exercises). Pick the lab whose chapter you want to internalise before attempting the longer exercises.

| # | Title | Chapter | Duration |
|---|---|---|---|
| [01](lab-01-plot-the-lsh-s-curve.md) | Plot and Read the LSH S-Curve | [04](../04-minhash-lsh-at-scale.md) | 60 min |
| [02](lab-02-normalisation-policy-ablation.md) | Normalisation Policy Ablation for Exact Dedup | [02](../02-exact-dedup-sha256-and-xxhash.md) | 60 min |
| [03](lab-03-emit-the-one-page-executive-summary.md) | Emit the One-Page Executive Summary From `dedup_report.yaml` | [08](../08-dedup-report-as-audit-evidence.md) | 60 min |

The labs compose loosely with the exercises: Lab 01's S-curve plot and empirical table are the reasoning sub-skill Exercise 02 pairs with a labelled-sample calibration; Lab 02's chosen-policy block is the `policy_note.md` Exercise 01 demands be written before any dedup code; Lab 03's emitter is the report-handoff shape Exercises 01–05 progressively fill in. Running the three back-to-back is a tour of the module's three most load-bearing engineering surfaces — the S-curve, the normalisation policy, and the audit-evidence handoff.

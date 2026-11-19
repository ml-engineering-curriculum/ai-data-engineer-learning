# Labs for mod-104-data-quality-measurement-and-audit (Data Quality Measurement, Representativeness, and Corpus Audit)

These labs are guided practice sessions. Each is a small, self-contained hands-on exercise anchored to one chapter and *distinct* from the module's [exercises](../exercises). Pick the lab whose chapter you want to internalise before attempting the longer exercises.

| # | Title | Chapter | Duration |
|---|---|---|---|
| [01](lab-01-deequ-and-pandera-at-a-stage-boundary.md) | Deequ and pandera at a Stage Boundary | [03](../03-deequ-and-pandera.md) | 75 min |
| [02](lab-02-disaggregated-counters-and-divergence.md) | Disaggregated Counters and Divergence Against a Reference Distribution | [06](../06-representativeness-and-disaggregated-counters.md) | 75 min |
| [03](lab-03-assemble-the-per-slice-quality-report.md) | Assemble the Per-Slice Quality Report From Evidence Stubs | [08](../08-per-slice-quality-report-handoff.md) | 60 min |

The labs compose: Lab 01's `deequ_result.json` and Lab 02's `representativeness.json` are drop-in replacements for the corresponding stubs in Lab 03's emitter. Running all three back-to-back ends with a `per_slice_quality.yaml` populated by two real evidence blobs and six stubs — the shape Chapters 02–07's exercises will later fill in completely.

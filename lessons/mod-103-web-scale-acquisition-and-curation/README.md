# mod-103 — Web-Scale Data Acquisition and Curation Pipelines

> **Role:** AI Data Engineer · **Family:** ML Engineering · **Level:** L30

**Estimated effort:** 14 hours

## What this module is about

This is the module where you stop reading other people's pipelines and run one. You pull a real slice of Common Crawl, run it through the CCNet-style sequence (language identification → perplexity filtering → deduplication), compare two main-content extractors on the same HTML, and reproduce a scoped subset of a public reference pipeline (Dolma, RedPajama-Data v2, FineWeb / FineWeb-Edu, or The Pile).

The goal is **muscle memory for the canonical pipeline shape**:

```
acquire → filter-by-URL → extract-main-content → language-id → deboilerplate
       → quality-filter (perplexity, heuristics) → deduplicate
       → decontaminate against benchmarks → emit shards + manifest
```

Every published open pretraining corpus you will cite — C4, Dolma, RedPajama, FineWeb, RefinedWeb, The Pile — is a specific choice of operators arranged in that order. Once you have run one end-to-end you can read the others.

This module is explicitly about the **authoring side** of the pipeline. Cluster-side execution — shard layouts, WebDataset / Mosaic Streaming, tokenisation throughput on a training node — is the next-up [`training-pipeline-engineer`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning) role at level 35. Chapter 07 defines the hand-off contract.

## Learning objectives

- Pull a non-trivial slice of Common Crawl (one WARC / WAT / WET snapshot) and run the CCNet-style pipeline (Wenzek et al. 2019) end-to-end on a workstation or small cluster — FastText `lid.176`, a KenLM-style perplexity filter, and hash-based deduplication — producing a documented reproducible slice.
- Run HTML main-content / boilerplate extraction with `trafilatura` and compare against `resiliparse` on the same slice; measure extraction quality against the C4 (Dodge et al. 2021) and RefinedWeb (Penedo et al. 2023) methodologies.
- Reproduce a scoped subset of a public reference pipeline — Dolma (Soldaini et al. 2024), RedPajama-Data v2 (quality-signal-per-document), FineWeb / FineWeb-Edu, The Pile — enough that `filter → extract → language-id → deboilerplate → dedup → quality-filter → decontaminate` is muscle-memoried; cite each public reference as the design precedent.
- Use a working operator library (Alibaba Data-Juicer, NVIDIA NeMo Curator, Dolma toolkit, Ray Data) rather than hand-rolling one — know what is 80+ operators of utility and what the custom-code surface really is.
- Keep the crawling-ethics contract from [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md) enforced in code — robots.txt / opt-out signals checked at the **acquire** stage, not at the publish stage.
- Hand off to [`training-pipeline-engineer`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning) (peer / next-up, level 35) on cluster-side sharding, WebDataset / Mosaic Streaming layout, and tokenisation throughput — this module covers the authoring side of the pipeline, not the cluster execution side.

## Chapters

| # | Title | Objective it anchors |
|---|---|---|
| [01](01-common-crawl-anatomy.md) | Common Crawl anatomy — WARC, WAT, WET, and the index | Objective 1 (pull a slice) |
| [02](02-ccnet-pipeline-end-to-end.md) | The CCNet-style pipeline end-to-end | Objective 1 (run it) |
| [03](03-boilerplate-extraction-trafilatura-and-resiliparse.md) | Main-content extraction — trafilatura vs resiliparse | Objective 2 |
| [04](04-public-reference-pipelines.md) | Public reference pipelines — Dolma, RedPajama-Data v2, FineWeb, The Pile | Objective 3 |
| [05](05-operator-libraries.md) | Operator libraries — Data-Juicer, NeMo Curator, Dolma toolkit, Ray Data | Objective 4 |
| [06](06-crawling-ethics-in-code.md) | The crawling-ethics contract, enforced in code | Objective 5 |
| [07](07-handoff-to-training-pipeline.md) | Hand-off to `training-pipeline-engineer` | Objective 6 |

## Exercises

| # | Title | Effort |
|---|---|---|
| [01](exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md) | Common Crawl slice extraction and language-id | 3h |
| [02](exercises/exercise-02-trafilatura-vs-resiliparse-extraction-benchmark.md) | trafilatura vs resiliparse extraction benchmark | 2h |
| [03](exercises/exercise-03-dolma-or-redpajama-v2-pipeline-reproduction.md) | Dolma or RedPajama-Data v2 pipeline reproduction (scoped) | 3h |
| [04](exercises/exercise-04-data-juicer-operator-chain-authoring.md) | Data-Juicer operator-chain authoring | 3h |

Labs and quizzes stub out — see `labs/` and `quizzes/`. Primary references are in [`resources.md`](resources.md).

## Prerequisites

- You have completed [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md). In particular, you have an internalised crawling-ethics contract (robots.txt per RFC 9309, `ai.txt` / `llms.txt` / `noai` / Google-Extended / GPTBot / CCBot / ClaudeBot / PerplexityBot, EU DSM Article 4 TDM opt-out). Chapter 06 of this module expects you to arrive with that contract in hand.
- You can read a WARC file with `warcio` or an equivalent library.
- You have a workstation with Python 3.11+, at least 100 GB of free disk, and either a single GPU or a small multi-node cluster. The exercises assume a workstation; the stretch goals extend to a cluster via Ray.

## How to use this module

Read chapters 01–02 and run exercise 01 before touching chapters 03+. Everything downstream — extractors, reference pipelines, operator libraries — composes on top of the CCNet-style skeleton, and the mental model of *"what is a WARC and how do I get documents out of it"* is load-bearing.

The exercises are deliberately scoped so you are **not** rebuilding Dolma or RefinedWeb from scratch. You are running the canonical shape against a slice small enough to iterate on in minutes, with reproducible seeds and pinned source SHAs, so that when you later contribute to a real corpus you have already met every operator in isolation.

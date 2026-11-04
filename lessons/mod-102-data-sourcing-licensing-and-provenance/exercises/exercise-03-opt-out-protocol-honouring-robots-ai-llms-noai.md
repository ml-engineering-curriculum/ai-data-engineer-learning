# Exercise 03 — Opt-Out Protocol: Honouring robots.txt, ai.txt, llms.txt, and the noai family

**Estimated effort:** 3 hours
**Chapter dependency:** [Chapter 03 — Honouring Crawling and Training Opt-Out Contracts](../03-crawling-and-training-opt-out-contracts.md)

## Objective

Implement and test the ingest-side opt-out evaluator. You will parse `robots.txt` per RFC 9309, honour the AI-specific user-agent records, parse `ai.txt`, read page-level `noai` / `noimageai` meta tags, recognise edge-platform AI-blocks, and treat `llms.txt` correctly as an *inclusion* signal rather than an opt-out. The deliverable is the evaluator in code and a fixture suite that exercises every signal path.

## Prerequisites

- You have read Chapter 03.
- You have read RFC 9309 (*Robots Exclusion Protocol*): <https://www.rfc-editor.org/rfc/rfc9309>.
- You have skimmed the publisher-maintained documentation for: GPTBot, ClaudeBot, Google-Extended, CCBot, PerplexityBot.
- You have reviewed the ai.txt convention (e.g. <https://site.spawning.ai/spawning-ai-txt>) and the llms.txt convention (<https://llmstxt.org/>).
- You are comfortable writing a parser or using a maintained one (e.g. `urllib.robotparser`, `reppy`, or a hand-rolled implementation).

## Problem statement

Your organisation operates a focused crawler identifying as `ExampleCorpCrawler/1.0 (+https://example.com/crawler)` and ingests from multiple upstream aggregators (Common Crawl, Hugging Face Hub datasets, in-house crawl output). The ingest pipeline must evaluate opt-out signals per URL before admitting content. Your task:

1. Parse `robots.txt` per RFC 9309. Identify the most-specific matching `User-agent` record. Honour `Disallow` and `Allow` directives. Record `Crawl-delay` as a soft contract.
2. Parse `ai.txt` and apply its directives as an additional denial layer.
3. Read page-level `<meta name="robots" content="...">` tags; treat `noai` and `noimageai` as blocking signals; record `noindex` / `nofollow` as weaker signals.
4. Handle `llms.txt` correctly — recognise its presence and treat it as inclusion signalling for retrieval contexts, *not* as an opt-out.
5. Detect edge-platform AI blocks (hypothetically modelled as a `Cloudflare-Ai-Block: 1` response header).
6. Emit a decision per URL with the signals consulted, the signals blocking, the user agent used, and a timestamp.

## Starter guidance

- Treat the evaluator as a *chain* of signal checks. Each check returns `admit` or `deny` and contributes to the decision log; admission requires every signal to permit.
- `robots.txt` parsing is subtler than it looks: the "most specific user agent match" rule, path prefix matching, `$` anchor, and `*` wildcard semantics all have corner cases. Use (or re-implement) a tested parser; do not roll your own regex without tests.
- `ai.txt` is not RFC-standardised; treat its format conservatively. Reject a file whose syntax you cannot parse unambiguously and log the rejection.
- For meta tags, parse the HTML head only; do not pull in a full HTML renderer. `beautifulsoup4` or `lxml` are fine.
- The IETF ai-pref working group's output may introduce a `/.well-known/ai-preferences` resource with structured HTTP headers. Design the chain so adding it is a new link, not a rewrite. <!-- needs-research: verify ai-pref status at implementation time. -->
- Distinguish your own user agent from the AI-specific tokens in robots.txt tests: a crawler identifying as `ExampleCorpCrawler/1.0` is *not* automatically denied by a `User-agent: GPTBot` block.
- Do **not** spoof a browser user agent to bypass an AI-specific denial. The honour contract is the point of the exercise.

## Deliverables

1. **`opt_out_evaluator.py`** — the evaluator implementing Chapter 03's interface. `evaluate(url, asset_bytes, html_head, response_headers) -> OptOutDecision` with `admit: bool`, `signals_consulted: list[str]`, `signals_blocking: list[str]`, `user_agent_as: str`, `evaluated_at: str`.
2. **`robots_parser.py`** — RFC 9309-conformant parser (yours or a thin wrapper around a maintained library — cite the library's version if you use one).
3. **`fixtures/`** — at minimum:
   - `fixtures/robots/allow-all.txt`
   - `fixtures/robots/disallow-all-ai-agents.txt` (blocks GPTBot, ClaudeBot, Google-Extended, CCBot, PerplexityBot)
   - `fixtures/robots/specific-paths.txt`
   - `fixtures/robots/crawl-delay.txt`
   - `fixtures/ai_txt/opt-out.txt`
   - `fixtures/ai_txt/allow-some.txt`
   - `fixtures/pages/page-with-meta-noai.html`
   - `fixtures/pages/page-with-meta-noimageai.html`
   - `fixtures/pages/page-clean.html`
   - `fixtures/pages/page-with-llms-txt-referenced.html`
   - `fixtures/response_headers/cloudflare-ai-block.json`
4. **`test_opt_out_evaluator.py`** — pytest cases covering:
   - robots.txt block for `GPTBot` does not block `ExampleCorpCrawler`.
   - robots.txt `User-agent: *` block applies to `ExampleCorpCrawler`.
   - A URL that is allowed by robots.txt but disallowed by ai.txt is denied.
   - A page with `meta name="robots" content="noai"` is denied even when robots.txt allows.
   - `llms.txt` presence does not admit a URL that robots.txt denies, and does not deny a URL that robots.txt allows.
   - `Cloudflare-Ai-Block: 1` header leads to deny and does not retry.
   - Per-asset C2PA `do-not-train` assertion leads to deny.
   - Signals-blocking list is non-empty in exactly the deny cases.
5. **`crawl_contract.md`** — a one-page public-facing document stating your crawler's identity, the signals it honours, the point of contact for rightsholder concerns, and the revalidation cadence. Mirror the shape of Google's or Anthropic's public crawler documentation.

## Acceptance criteria

- Parser passes the RFC 9309 examples (at minimum the examples in §5 of the RFC).
- Evaluator correctly chooses the *most specific* matching `User-agent` record.
- `llms.txt` is **not** treated as an opt-out signal; a test asserts this.
- Every decision records both `signals_consulted` and `signals_blocking`.
- The evaluator is deterministic given its inputs and policy; the decision log is reproducible.
- No test case requires spoofing a user agent to pass.
- At least one test exercises the ai-pref future-signal extension point (even if the implementation is a stub marked `<!-- needs-research: ... -->`).
- The crawl-contract document names a real contact address (a mailbox, not an individual) and a revalidation cadence.

## Stretch goals

- Add a revalidation job that re-evaluates opt-outs on a historical slice against current signals; emit a filter-list of items whose status flipped.
- Extend the evaluator with a `well-known-ai-preferences` reader stubbed to the current ai-pref draft spec. Keep the stub behind a feature flag.
- Add an observability counter: emit per-authority `signals_blocking` histograms so the data-engineering team can see which opt-out families are most common across the crawl.
- Compare your evaluator's decisions on a sampled Common Crawl WARC against the WARC's `is-blocked-by-robots` notes and reconcile any divergence.

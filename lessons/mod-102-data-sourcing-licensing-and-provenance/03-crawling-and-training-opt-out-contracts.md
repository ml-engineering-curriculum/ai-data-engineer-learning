# Chapter 03 — Honouring Crawling and Training Opt-Out Contracts

## Why this chapter exists

A licence tag (Chapter 02) tells you whether you *may* use content. A opt-out signal tells you whether the publisher *reserves* their rights against AI training specifically. The two are independent: a CC-BY-licensed page may still carry an `ai.txt` opt-out; a page without any declared licence may still be crawlable under a jurisdiction's TDM exception subject to its machine-readable reservation.

The population of opt-out signals you must honour has grown quickly and is not yet converged. This chapter catalogues the signals as they stand today, shows how to implement the honour contract per RFC 9309 and the current community conventions, and sets up how the pipeline evaluates them at scale. The IETF AI Preferences (ai-pref) working group is actively drafting what should become the standards-track successor; watch its output.

## The signals you must honour

A production crawler and a production ingest pipeline today check, for each URL, up to seven classes of signal.

### 1. robots.txt per RFC 9309

RFC 9309 (IETF, September 2022) is the standards-track definition of the Robots Exclusion Protocol. It canonicalises four decades of deployed practice. Key properties:

- The file lives at `/robots.txt` on the origin's URI authority.
- Directives are grouped into *records*, each introduced by one or more `User-agent:` lines and followed by `Allow:` / `Disallow:` lines.
- The most specific matching `User-agent` wins. A record keyed `User-agent: *` is the fallback.
- RFC 9309 defines `Allow` and `Disallow`; it does **not** standardise `Crawl-delay` or `Sitemap`, which are widely deployed but non-normative.
- Path matching is prefix-based; `$` anchors the end; `*` is a wildcard with widely but not universally supported semantics.

For AI crawlers specifically, publishers increasingly add AI-specific `User-agent` records. The common tokens are enumerated below. Your crawler must identify itself by a stable `User-agent`, honour the most specific matching record, and respect `Crawl-delay` even though it is non-normative (treat it as a soft contract).

A concrete `robots.txt` the crawler must parse correctly:

```
User-agent: GPTBot
Disallow: /

User-agent: ClaudeBot
Disallow: /

User-agent: Google-Extended
Disallow: /

User-agent: CCBot
Disallow: /archives/

User-agent: PerplexityBot
Disallow: /

User-agent: *
Disallow: /admin/
Crawl-delay: 10
```

Correct behaviour for each of these agents:

- A crawler identifying as `GPTBot`, `ClaudeBot`, `Google-Extended`, `PerplexityBot`: deny all paths.
- A crawler identifying as `CCBot`: deny `/archives/`, allow everything else.
- A crawler identifying as anything else: deny `/admin/`, 10-second crawl delay, allow everything else.

### 2. AI-specific crawler user agents

These are the production tokens you must recognise as of 2026. All are maintained by their operators; verify the current list and their published `User-agent` strings at read time.

- **GPTBot** — OpenAI crawler for training data. OpenAI publishes its identification string and a dedicated opt-out mechanism.
- **ChatGPT-User** — OpenAI agent that fetches on behalf of a ChatGPT user action (not training).
- **OAI-SearchBot** — OpenAI crawler for SearchGPT indexing (not training).
- **CCBot** — Common Crawl Foundation crawler. Common Crawl output is widely re-used; opting out of CCBot opts out of most downstream aggregators.
- **ClaudeBot** — Anthropic crawler for training.
- **Claude-User** — Anthropic agent that fetches on behalf of a Claude user action (not training).
- **Google-Extended** — product token Google exposes so publishers can opt out of Google's generative-AI training without opting out of Google Search (whose crawler is `Googlebot`).
- **Googlebot** — the general Google Search crawler; distinct from `Google-Extended`.
- **Applebot-Extended** — Apple's analogue of `Google-Extended` for Apple Intelligence training.
- **Bytespider** — ByteDance's crawler, widely blocked.
- **PerplexityBot** — Perplexity's crawler.
- **Meta-ExternalAgent** / **FacebookBot** — Meta's AI-training user agents; verify current token.
- **Amazonbot** — Amazon's crawler used for AI training and other purposes.
- **Diffbot**, **Omgilibot**, **cohere-ai**, **DataForSeoBot**, **Kagibot**, and others — smaller or specialised crawlers.

Your own crawler must choose a stable, publicly-documented `User-agent` string and keep it documented. Spoofing another vendor's token to bypass opt-outs is unethical and, in many jurisdictions, legally exposed.

### 3. ai.txt

`ai.txt` is a community convention popularised by Spawning and others. It lives at `/ai.txt` and declares opt-outs targeted at AI training specifically — distinct from `robots.txt`, which was never intended to express training-use intent. There is no RFC; conventions vary slightly across proponents. A representative shape:

```
User-Agent: *
Disallow: /

Allow: /press-releases/
```

If your pipeline consumes ai.txt, consume it conservatively: opt-out lines bind; allow lines do not override more specific `robots.txt` denials. Treat ai.txt as an additional denial layer, not a substitute.

### 4. llms.txt

`llms.txt` is a separate community convention (see llmstxt.org) oriented primarily at LLM *reasoning* rather than *training*: it is a succinct, LLM-friendly site summary the publisher wishes an LLM to use when answering queries about the site. For training-data purposes, `llms.txt` is not an opt-out signal; it is an inclusion signal. Your pipeline should recognise `llms.txt` so that the content inside is correctly weighted for canonical representations, but the presence of `llms.txt` alone does not grant training permission — the normal licence and opt-out analysis still applies. Keep the two conventions mentally separate.

### 5. Page-level and asset-level metadata — noai / noimageai / META robots

Per-page HTML meta tags extend `robots.txt` to the asset granularity. The two that matter for training-data purposes:

- `<meta name="robots" content="noai">` — a Microsoft / DeviantArt–originated page-level opt-out for AI training. Widely deployed on art and photo sites.
- `<meta name="robots" content="noimageai">` — the image-specific analogue. Relevant for image corpora.
- `<meta name="robots" content="noindex, nofollow">` — standard search-indexing opt-outs; not training-specific but a reasonable default-deny signal if the page is behind `noindex`.

Image and video assets may carry provenance metadata (EXIF, IPTC XMP, C2PA Content Credentials — see Chapter 06) with an AI-training reservation. The pipeline must read embedded metadata on binary assets, not only the HTML wrapper.

### 6. Cloudflare and other platform-level AI opt-outs

Cloudflare operates a platform-level AI bot management feature that lets origin operators block AI crawlers at the edge. For the crawler author this presents as 403s or challenges when identifying as a known AI bot; your crawler must identify honestly and back off.

Cloudflare has also published a per-request "AI bot" signal and a pay-per-crawl programme (announced July 2025). For an ingest pipeline, the practical obligation is: when Cloudflare's edge responds with an AI-bot block, do not retry with a different user agent, and record the block as an opt-out event in the manifest.

Other CDNs (Fastly, Akamai, AWS CloudFront via Managed Rules) expose similar features. Your pipeline's observed-response record is itself part of the opt-out audit trail.

### 7. DSM Article 4(3) machine-readable reservations

The EU DSM Copyright Directive Article 4 TDM exception requires the reservation be expressed in a *machine-readable* manner for content made publicly available online. Chapter 04 handles the legal substrate; for this chapter, the practical signals that the pipeline recognises as Article 4(3) reservations:

- `robots.txt` with AI-specific user-agent denials.
- `ai.txt` opt-outs.
- Page-level `<meta name="robots" content="noai">`.
- Asset-level C2PA / IPTC / XMP `do-not-train` metadata.
- Site-level Terms of Service that assert reservation in a machine-readable location (e.g. a well-known URI) — less uniform in practice.

Which of these count as Article 4(3) reservation is a legal question — see Chapter 04. For the pipeline's purposes, honouring the full list is the safe default.

## The IETF ai-pref working group

The IETF **AI Preferences (ai-pref)** working group was chartered to develop standards-track mechanisms for expressing AI opt-in / opt-out preferences in a uniform way. At the time of this chapter's writing the WG has produced drafts covering a vocabulary of AI purposes, HTTP-header expression, and a `/.well-known/ai-preferences` URI. The direction of travel is toward a single, standards-track signal with explicit purpose-category granularity (training vs. retrieval vs. search indexing, for example).

<!-- needs-research: at read time verify the current list of published RFCs, active I-Ds, and WG status for IETF ai-pref — https://datatracker.ietf.org/wg/aipref/about/ -->

Your pipeline should be ready to consume whatever ai-pref produces as an *additional* signal, not a replacement for the existing ones, until the industry converges. Design the opt-out evaluator as a chain: each honoured signal is one input, and admission requires all signals to permit.

## How the pipeline evaluates opt-outs at scale

Opt-out evaluation is a per-URL question, but its inputs are coarser:

- `robots.txt` and `ai.txt` are per-authority (scheme + host + port). Cache per authority with a TTL.
- `llms.txt` is per-authority.
- Meta tags are per-page. Parse the HTML head on fetch.
- Embedded image metadata is per-asset.
- Edge-platform blocks surface as HTTP responses.
- DSM Article 4(3) reservation is evaluated as the composite of the above.

A simplified structure for the pipeline's opt-out evaluator:

```python
# aicg/datapipeline/optout/evaluator.py
from dataclasses import dataclass
from typing import Literal

Signal = Literal[
    "robots-txt",
    "ai-txt",
    "meta-noai",
    "meta-noimageai",
    "image-asset-c2pa-do-not-train",
    "edge-platform-block",
    "well-known-ai-preferences",  # ai-pref
]

@dataclass
class OptOutDecision:
    admit: bool
    signals_consulted: list[Signal]
    signals_blocking: list[Signal]
    user_agent_as: str
    evaluated_at: str  # ISO-8601 timestamp; stored in manifest

class OptOutEvaluator:
    def __init__(self, user_agent: str, cache_ttl_hours: int = 24):
        self.user_agent = user_agent
        self.cache_ttl_hours = cache_ttl_hours

    def evaluate(self, url: str, asset_bytes: bytes | None, html_head: str | None) -> OptOutDecision:
        signals_consulted: list[Signal] = []
        signals_blocking: list[Signal] = []

        if (robots := self._robots_for(url)) is not None:
            signals_consulted.append("robots-txt")
            if not robots.allowed(self.user_agent, url):
                signals_blocking.append("robots-txt")

        if (ai_txt := self._ai_txt_for(url)) is not None:
            signals_consulted.append("ai-txt")
            if not ai_txt.allowed(self.user_agent, url):
                signals_blocking.append("ai-txt")

        if html_head is not None:
            if self._meta_has(html_head, "noai"):
                signals_consulted.append("meta-noai")
                signals_blocking.append("meta-noai")
            if self._meta_has(html_head, "noimageai"):
                signals_consulted.append("meta-noimageai")
                signals_blocking.append("meta-noimageai")

        if asset_bytes is not None and self._c2pa_do_not_train(asset_bytes):
            signals_consulted.append("image-asset-c2pa-do-not-train")
            signals_blocking.append("image-asset-c2pa-do-not-train")

        if (ai_pref := self._well_known_ai_pref_for(url)) is not None:
            signals_consulted.append("well-known-ai-preferences")
            if not ai_pref.allowed(self.user_agent, url, purpose="train"):
                signals_blocking.append("well-known-ai-preferences")

        return OptOutDecision(
            admit=len(signals_blocking) == 0,
            signals_consulted=signals_consulted,
            signals_blocking=signals_blocking,
            user_agent_as=self.user_agent,
            evaluated_at=self._now_iso(),
        )
```

Every evaluation emits a decision that lands in the per-item log. The log is the audit evidence; it answers "did we check, which signals were present, and which blocked" for every ingested or rejected item.

## Caching, revalidation, and TTL

Opt-out signals change. A site operator who adds `Disallow: /` for `GPTBot` today expects the pipeline to honour it tomorrow, not a year from now because of a stale cache.

Practical caching rules:

- `robots.txt` TTL: at most 24 hours. Revalidate on each fetch for actively-crawled authorities.
- `ai.txt` TTL: at most 24 hours.
- `llms.txt`: cache aggressively (7 days) — it is inclusion signalling, not opt-out.
- Per-page meta tags: re-evaluate on every fetch; no cross-fetch cache.
- Asset-level metadata: re-evaluate on every ingest.
- Edge-platform blocks: track by authority; back off and do not retry under a different identity.

Separately, the pipeline must be able to **re-apply opt-outs to historical slices**. If a site adds an opt-out next week and your slice from last month contains that site's content, your organisation's policy will typically be to filter the content on next slice rebuild. Keep the mapping from item to source URL and a daily re-evaluation job that can shadow-filter a slice against current signals. This is the "opt-out revalidation" step in mod-109.

## Specific behaviours for the production crawler bots

For an organisation running its own crawler, you will almost certainly be asked to honour the AI-opt-out signals for both inbound and outbound reasons:

- **Inbound** — your own public sites publish `robots.txt` entries for third-party AI crawlers. The decisions there are a product-and-legal question your team may advise on but does not own.
- **Outbound** — your own crawler honours the signals above for every URL it fetches.

There are three patterns that are unsafe and recur:

1. **Identifying as a browser to evade AI-specific opt-outs.** Spoofing a general-purpose user agent to evade an AI-specific denial inverts the signal. Do not do this even if some sites permit general crawling but deny AI crawling; honour the AI denial.
2. **Carrying over data from a historical crawl after a new opt-out appears.** If the site added `GPTBot: Disallow` after your crawl, your slice contains content whose current posture is opt-out. Either filter on rebuild or document a position on retroactive application. Default is to filter.
3. **Using a shared corpus (Common Crawl, HTTP Archive) without re-evaluating opt-outs.** Common Crawl itself honours `robots.txt` at fetch time but older archives may contain content whose current opt-out posture is different. Re-evaluate on ingest.

The stakeholder-facing form of these rules is a one-page **crawl contract** that documents your organisation's policy and the fact that the pipeline enforces it. Publish it.

## Summary

- The pipeline honours seven classes of signal: `robots.txt` (RFC 9309), AI-specific user agents, `ai.txt`, `llms.txt` (an inclusion signal, not an opt-out), page-level `noai` / `noimageai` meta, platform-level edge blocks (Cloudflare and similar), and — increasingly — a `/.well-known/ai-preferences` resource standardised by the IETF ai-pref WG.
- Each ingest item carries a decision: `admit` or `reject`, the signals consulted, the signals blocking, the user-agent identity, and the evaluation timestamp.
- Caches have short TTLs on authority-level signals and revalidate on every fetch for page-level signals. Opt-outs are re-applied to historical slices on rebuild.
- Spoofing user agents is unsafe and unethical. The crawler identifies honestly and honours all specific records that match.
- The IETF ai-pref working group's standards-track evolution is the direction of travel. Design the evaluator as a signal chain so a new signal is additive.
- DSM Article 4(3) reservation is the composite legal effect of these signals in the EU; US doctrine is unsettled. Chapter 04 handles the legal substrate.

Exercise 03 walks you through building and testing the opt-out evaluator against a fixture set of realistic signals. Chapter 04 turns to the copyright-regime substrate and the legal deferral contract.

# Chapter 06 — The Crawling-Ethics Contract, Enforced in Code

## Why this chapter exists

In [`mod-102-data-sourcing-licensing-and-provenance`](../mod-102-data-sourcing-licensing-and-provenance/README.md) you learned the shape of the crawling-and-training opt-out contract: `robots.txt` per RFC 9309 for crawling, `ai.txt` / `llms.txt` / `noai` / Cloudflare AI-opt-out / Google-Extended / GPTBot / CCBot / ClaudeBot / PerplexityBot / Applebot-Extended for AI-use opt-outs, the IETF `aipref` working group's standards-track evolution of those signals, EU DSM Article 4 TDM-exception rightholder reservations, C2PA Content Credentials where upstream supplies them.

This chapter is the companion operational chapter: **where in the pipeline do you check those signals, and what artifact proves you did?**

The short answer — and the one that fails most audits — is that the contract **must be enforced at the acquire stage**, not at the publish stage. Checking `robots.txt` on the way out is not "doing crawling ethics"; it is removing evidence of a violation. Checking it at acquire time is.

## The three-moment principle

Every URL in your corpus passes through three moments in time:

1. **The moment it was crawled.** Whoever collected the page (Common Crawl, your own crawler, a partner) was acting under some crawler-identity and some set of robots-signals as they existed at that time.
2. **The moment you acquired it.** You read the bytes into your pipeline, possibly months or years after the original crawl.
3. **The moment you use it for training.** Your model actually sees the text.

The opt-out signals that control each moment are different:

| Moment | Primary signals | Who checks |
|---|---|---|
| Crawl | `robots.txt` (RFC 9309), 4xx/5xx HTTP responses, host rate limits | The crawler — Common Crawl's `CCBot`, your `Scrapy` run, a partner |
| Acquire | AI-use opt-outs: `ai.txt` at host root, `llms.txt`, `User-agent: GPTBot/Google-Extended/CCBot/ClaudeBot/PerplexityBot/Applebot-Extended` lines in `robots.txt`, Cloudflare AI-opt-out header, `noai` / `noimageai` meta tags; EU DSM Article 4 rightholder reservations | **You** (the pipeline) |
| Use | Downstream dataset-card license metadata, training-time filters | Legal + the training team |

For a Common-Crawl-derived pipeline the *crawl* step has already happened under CCBot; you inherit CCBot's robots.txt compliance posture. The step you own is **acquire** — and acquire-time checking of AI-use opt-outs is where the training-use contract actually becomes real.

## Why "check at publish" fails

A tempting but wrong design: "we pull everything, filter to training-use-allowed URLs right before we emit the final shards". Reasons it fails:

- **You still processed the opted-out content.** Every operator in Chapters 02–05 — language ID, perplexity, dedup, decontamination — ran against documents whose publishers said *don't*. You can't un-process them; you can only not-emit.
- **Upstream statistics are polluted.** Your dedup clusters, quality-tier distributions, and language histograms were computed over a population that included opted-out content; your publicly reported statistics no longer match the final slice.
- **Legal exposure is proportional to processing, not publishing**, under several regimes. Specifically, EU DSM Article 4 TDM-exception reservations are **reservations against reproduction and extraction**, not just against publication; GDPR Article 6 lawful-basis questions turn on *processing*, which includes the acquire-and-transform step.
- **The audit trail doesn't exist.** If an auditor asks "did you honour the opt-out at this host on this date", your answer must be "yes, we never acquired the document" — not "we acquired it and then dropped it later". The first is a record in the acquire-stage log; the second is an admission.

Enforce the contract at the earliest stage where the signal is legible. For AI-use opt-outs that stage is acquire.

## The acquire-stage check

Minimal shape:

```python
# conceptual, not production: acquire-stage AI-use opt-out check
import httpx
from urllib.parse import urlsplit
from functools import lru_cache
from datetime import datetime, timezone

OPTED_OUT_AGENTS = {
    "GPTBot", "Google-Extended", "CCBot", "ClaudeBot",
    "PerplexityBot", "Applebot-Extended", "Bytespider",
    # verify current list at read time; the set grows
}

@lru_cache(maxsize=100_000)
def robots_disallows_ai(host: str) -> dict:
    """Fetch robots.txt once per host; parse AI-agent lines. Returns which agents are disallowed."""
    try:
        r = httpx.get(f"https://{host}/robots.txt", timeout=10, follow_redirects=True)
        if r.status_code != 200:
            return {}
        return parse_robots_ai_agents(r.text, OPTED_OUT_AGENTS)
    except Exception:
        return {}  # fail-open on fetch error; decide your policy

@lru_cache(maxsize=100_000)
def host_has_ai_txt(host: str) -> bool:
    try:
        r = httpx.get(f"https://{host}/ai.txt", timeout=10, follow_redirects=True)
        return r.status_code == 200 and "User-Agent:" in r.text
        # parsing ai.txt is more subtle; this is illustrative
    except Exception:
        return False

def ai_use_allowed(url: str, as_of: datetime | None = None) -> dict:
    """Decision function at acquire time. Returns (allowed, evidence)."""
    host = urlsplit(url).netloc
    agents = robots_disallows_ai(host)
    ai_txt = host_has_ai_txt(host)
    # project policy: deny if ANY AI-agent is disallowed OR ai.txt exists with a deny rule
    denied_agents = [a for a, disallowed in agents.items() if disallowed]
    allowed = not denied_agents and not ai_txt
    return {
        "url": url,
        "host": host,
        "allowed": allowed,
        "robots_disallowed_agents": denied_agents,
        "ai_txt_present": ai_txt,
        "checked_at": (as_of or datetime.now(timezone.utc)).isoformat(),
    }
```

What this is **not**:

- It is not a complete implementation. Parsing `ai.txt` and `llms.txt` correctly, honouring `Allow` + `Disallow` ordering per RFC 9309, handling `Sitemap` lines, and dealing with `<meta name="robots" content="noai">` in HTML head are all finicky details. Use a tested library — `reppy` (<https://github.com/seomoz/reppy>), Google's `robots-txt-parser`, or Common Crawl's own `robotstxt-java` — rather than writing a parser.
- It does not handle the Common Crawl angle — CCBot already honoured the publisher's robots.txt at crawl time, but the *AI-use* overlay (GPTBot, Google-Extended, etc.) was almost certainly not expressed yet when the page was crawled. Your acquire-stage check is an overlay on top of what CCBot already did.

What it **is**: a per-URL decision function that runs *before* any downstream operator sees the content, whose output is a structured record attached to the per-record manifest.

## Per-record evidence

The output of the acquire-stage check is not "allowed = yes/no"; it is a **record** attached to the per-record manifest from Chapter 01. Shape:

```json
{
  "url": "https://example.com/article/abc",
  "warc_filename": "crawl-data/CC-MAIN-2024-30/segments/.../warc/CC-MAIN-...warc.gz",
  "warc_record_offset": 10254873,
  "warc_record_length": 8321,
  "observed_sha256": "8f3b...",
  "acquire_decision": {
    "allowed": false,
    "checked_at": "2026-11-04T18:22:01Z",
    "robots_disallowed_agents": ["GPTBot", "Google-Extended"],
    "ai_txt_present": false,
    "policy_version": "aicg-policy-v1.3",
    "evidence_url": "https://example.com/robots.txt",
    "evidence_sha256": "4d9b..."
  }
}
```

The `evidence_sha256` is the hash of the robots.txt (or `ai.txt`) you fetched at check time. Someone six months from now can re-fetch the host's robots.txt, compare hashes, and either confirm the decision or note that the publisher has since changed their stance. **Hash the evidence.** Without it the audit trail is "we claimed we checked"; with it the audit trail is "here is exactly what the publisher said at 18:22 UTC on 4 November 2026".

## Where this lives in the pipeline

Operationally the acquire-stage check sits between the Chapter 01 slice-selection predicate and the Chapter 02 pipeline:

```
CC index query (Chapter 01)
  ↓ URL list + (warc, offset, length) triples
Group by host (one robots.txt fetch per host, not per URL)
  ↓
Acquire-stage ethics check  ← THIS CHAPTER
  ↓ per-URL decision records written to manifest
Range-fetch the allowed URLs only
  ↓ document stream
CCNet-style pipeline (Chapter 02+)
```

Two operational notes:

1. **Group by host before checking.** Robots.txt is per-host, so one fetch serves every URL on that host. A 100M-URL slice over 1M distinct hosts means 1M robots.txt fetches, not 100M. Cache aggressively and persist the cache across pipeline runs — robots.txt rarely changes within a day but does change over months.
2. **Rate-limit politely.** You are now making HTTP requests directly to publishers. Limit to a few requests per host per second, back off on 429/503, and use a clear `User-Agent` string that identifies your pipeline.

## Signal-by-signal reference

| Signal | Where | How to check | Spec |
|---|---|---|---|
| `robots.txt` crawler ban | `/robots.txt` at host root | Standard robots parser with your identified `User-Agent` | RFC 9309 (<https://www.rfc-editor.org/rfc/rfc9309>) |
| AI-agent bans (GPTBot, Google-Extended, CCBot, ClaudeBot, PerplexityBot, Applebot-Extended, Bytespider) | `/robots.txt` lines with those `User-agent:` names | Same parser, check those specific agents | Each operator publishes its agent name; e.g. OpenAI <https://platform.openai.com/docs/gptbot> |
| `ai.txt` | `/ai.txt` at host root | Line-oriented parser similar to robots.txt | <https://site.spawning.ai/spawning-ai-txt> (community spec) |
| `llms.txt` | `/llms.txt` at host root | Line-oriented; semantics are community-evolving | <https://llmstxt.org/> (community spec) |
| `noai` / `noimageai` meta tag | `<meta name="robots" content="noai">` in HTML head | HTML parse post-extraction, or `<head>`-only pre-extraction | Community convention |
| Cloudflare AI-opt-out | Response header `X-Robots-Tag` or Cloudflare AI-opt-out header | Check response headers on fetch | Cloudflare docs (verify current form) |
| EU DSM Article 4 TDM reservation | Rightholder opt-out in machine-readable form (typically `robots.txt` or `ai.txt` per the EU AI Office GPAI Code of Practice) | Jurisdictionally-conditioned; see mod-102 for the legal shape | Directive (EU) 2019/790 Art. 4; EU AI Office guidance |
| C2PA Content Credentials | Embedded manifest in file metadata (images primarily; text support emerging) | C2PA SDK verification | C2PA specification <https://c2pa.org/specifications/> |

The IETF `aipref` working group (<https://datatracker.ietf.org/wg/aipref/about/>) is formalising the above as standards-track; verify the published RFCs and drafts at read time.

<!-- needs-research: once IETF aipref produces an RFC, cite it directly; current form is draft. -->

## The "my slice is CC, CC already handled robots" fallacy

You will hear this argument. It is wrong for AI-use opt-outs and here is why:

- Common Crawl's CCBot honours `robots.txt` **for crawling**. At the time a page was crawled, if the publisher had `Disallow: /` for `CCBot`, CCBot would skip it. That is the crawl-time signal.
- But `GPTBot`, `Google-Extended`, `ClaudeBot`, `Applebot-Extended`, etc. are *AI-use* signals that largely did not exist before 2023. Many pages in older Common Crawl snapshots were crawled before their publishers had expressed an AI-use preference.
- When you acquire a 2021 CC snapshot in 2026, the publisher has had time to express their AI-use preference. Your acquire-stage check reads **today's** signals against **today's** acquire; the historical crawl-time robots.txt is not sufficient.

The right posture: CCBot handled historical crawling-time robots; **you** handle present acquire-time AI-use opt-outs. Two different signals at two different moments, both must be checked by somebody.

## Deletion requests after the fact

Even with a correct acquire-stage check, you will eventually get a post-hoc request to remove a URL or a host from your slice. Design for this:

- Every emitted shard embeds, in its manifest, the full list of URLs it contains.
- A "removal request" is a diff on the manifest: a list of URLs or hosts to redact.
- Your pipeline has a documented **redact-and-re-shard** operation that takes the diff and produces a new shard set with a new manifest.
- You never destroy the old manifests; you mark them superseded and point to the successor.

This is the deliverable that satisfies deletion-request handling for GDPR Articles 17 / 21, CCPA/CPRA consumer-request obligations, and the DMCA / equivalent takedown shape. The engineering work is tedious but well-scoped. [`mod-109-dataset-versioning-lineage-reproducibility`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) covers the versioning substrate (DVC / lakeFS / Pachyderm) you build this on.

## What this chapter is not

- **It is not a legal determination.** You are enforcing signals in code; legal counsel makes the call on whether a given signal has legal force in a given jurisdiction. The chapter is "run the check and keep the evidence"; the legal weight of the evidence is counsel's domain. See `ai-governance-analyst` (level 15) and `ai-risk-engineer` (peer, level 25) on the risk-model side; see legal counsel on jurisdiction.
- **It is not a substitute for a license matrix.** The mod-102 license matrix is upstream; this chapter is *acquire-stage opt-out enforcement*, which is a different check. A permissively-licensed source can still carry a `GPTBot: Disallow` and both facts are true simultaneously.
- **It is not complete without PII scrubbing** (mod-110). An allowed URL can still contain personal data that needs handling. The opt-out signal is about the *publisher's* consent to AI-use; PII is about *data-subject* consent, which is a separate axis.

## Summary

- The crawling-ethics contract from mod-102 must be enforced **at acquire time**, not at publish time. Processing opted-out content and then dropping it is not compliance; it is evidence of a violation.
- The per-URL acquire-stage decision returns `(allowed, evidence)` and is attached to the per-record manifest. Hash the evidence (robots.txt, `ai.txt`) so the audit trail can be re-verified later.
- Signals to check: `robots.txt` per RFC 9309, AI-agent lines (`GPTBot`, `Google-Extended`, `CCBot`, `ClaudeBot`, `PerplexityBot`, `Applebot-Extended`, `Bytespider`), `ai.txt`, `llms.txt`, `<meta name="robots" content="noai">`, Cloudflare AI-opt-out, EU DSM Article 4 rightholder reservations, C2PA. The IETF `aipref` working group is formalising the standards-track shape.
- Group by host before checking: one robots.txt fetch per host, not per URL. Cache persistently across runs; rate-limit politely; identify yourself in the `User-Agent`.
- CC's CCBot handled crawl-time robots; you handle present-moment AI-use opt-outs. These are different signals at different moments.
- Design the slice manifest and shard emission so that post-hoc deletion requests can be answered with a redact-and-re-shard operation. The versioning substrate is mod-109.

Chapter 07 picks up the output end of the pipeline — what exactly do you hand to the `training-pipeline-engineer` at level 35, and what does their reciprocal expectation look like?

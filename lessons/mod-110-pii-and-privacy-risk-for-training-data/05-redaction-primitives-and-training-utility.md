# Chapter 05 — Redaction Primitives and Training Utility

## Why this chapter exists

Detecting PII is one half of the control. The other half is deciding *what to do with each finding*. Choose badly and you either:

- **Preserve the PII** (hashing with no key management, or tokenisation with a reversible vault that is itself leaked), or
- **Destroy training utility** (replace every name with `<PERSON>` and the model loses the signal that proper nouns exist in the distribution at all).

This chapter pins the vocabulary and the engineering trade-offs. Every entity type in your privacy-risk report is annotated with a **primitive choice** (mask / tokenise / surrogate / FPE / hash) and a **utility note**. This chapter tells you how to pick each one.

The vocabulary used here follows **ISO/IEC 20889:2018** *Privacy enhancing data de-identification terminology and classification of techniques*, which is the standards reference for this chapter.

## The primitives

| Primitive | What it does | Reversible? | Preserves length? | Preserves distribution? | Training-utility impact |
|---|---|---|---|---|---|
| **Masking** | Replace PII span with a constant (`<PERSON>`, `***`) | No | No | No | Low — model learns the placeholder |
| **Tokenisation** | Replace PII span with an opaque token, with a (optionally reversible) vault mapping | Yes (with vault) | No | No | Low — model learns the token |
| **Surrogate generation** | Replace PII span with a plausible fake of the same type | No (one-way) | Approximately | Yes — synthetic distribution | Moderate — preserves grammar, distorts facts |
| **Format-preserving encryption (FPE)** | Encrypt the PII while preserving its format (digits→digits, chars→chars) | Yes (with key) | Yes | Partially | High — preserves "shape" but still PII-under-encryption |
| **Hashing** | One-way hash the PII, optionally with HMAC + secret | No | No | No | Low — but useful for linkage without reversal |

A redaction policy is a **per-entity-type mapping** from entity type to primitive, plus the parameters for that primitive (vault location, key reference, secret salt, surrogate generator). Commit the mapping. The privacy-risk report cites it.

### Masking — the default

Masking replaces the PII span with a constant string. In Presidio terms:

```python
OperatorConfig("replace", {"new_value": "<PERSON>"})
# or
OperatorConfig("mask", {"masking_char": "*", "chars_to_mask": 4, "from_end": True})
```

**When masking is right:**

- You have no need to recover the original value. Pretraining-data cleaning is almost always this case.
- The span does not need to be identifiable later for correlation.
- You are willing to accept that the model learns the placeholder as a distributional feature.

**When masking is wrong:**

- You need to re-link findings to the source (for internal analytics or for audit response).
- The entity type carries distributional signal the model should learn (e.g., proper-name distribution in natural language).

The choice of placeholder matters. Models trained on text with `<PERSON>` tokens frequently generate `<PERSON>` tokens at inference time — a learnt artifact. If you mask, use a placeholder that is unlikely to appear in normal text, is consistent across the corpus, and ideally is pre-declared in the tokenizer vocabulary so it is not shattered into subwords.

### Tokenisation — the vault model

Tokenisation replaces each PII span with an opaque, consistent token backed by a **vault**: a key-value store mapping tokens to the original values.

```
"Jane Doe" → "PERSON_01"   (vault: PERSON_01 → "Jane Doe")
"Jane Doe" → "PERSON_01"   (same input → same token, consistent)
"John Smith" → "PERSON_02" (different input → different token)
```

Tokenisation is **reversible**: anyone with vault access can recover the original values. This is a feature and a liability:

- **Feature:** structured data pipelines can re-correlate token values across datasets without reissuing the raw PII.
- **Liability:** the vault is itself a database of raw PII. Its protection is exactly the protection of the PII it indexes.

**When tokenisation is right:**

- Structured analytics / feature-engineering pipelines where downstream joins need consistency.
- Scenarios where the detection pipeline is in a different trust zone than the analytics pipeline, and the vault is held only in the trust zone that needs it.

**When tokenisation is wrong:**

- Pretraining-data cleaning. The model does not need to re-link; the vault adds a high-value target without upside.
- Any case where consistency across runs matters more than consistency within a run — unless the vault is explicitly persisted and access-controlled.

The vault is **not** a solved problem — it is a database of secrets with its own access control, encryption-at-rest, backup, and audit log requirements. If you use tokenisation, you own the vault's threat model.

### Surrogate generation — "realistic fakes"

Surrogate generation replaces each PII span with a **plausible fake** of the same type. "Jane Doe" becomes "Emma Johnson". "jane.doe@example.com" becomes "emma.johnson@example.net". "(415) 555-0199" becomes "(312) 555-0287".

**When surrogates are right:**

- You care about preserving the **shape** of the natural-language distribution — proper-name frequency, sentence grammar, token-level realism. Fine-tuning corpora for conversational agents are a common case.
- The content matters to the model's downstream task (e.g., a resume-screening model needs to see names in resumes; a `<PERSON>` placeholder teaches it nothing).
- You want to defeat extraction attacks: an attacker who recovers "Emma Johnson" from the model has not recovered anyone's PII.

**When surrogates are wrong:**

- Audit-grade pipelines where the redaction must be deterministic and inspectable per span. Surrogate generation adds a randomness source; reproducibility requires seed management.
- Multilingual corpora where the surrogate generator does not have per-language distributions — you will introduce Western-sounding names into non-Western corpora.

Surrogate generators come with trade-offs:

- **Consistency vs randomness** — if "Jane Doe" always becomes "Emma Johnson" (consistent), you retain the privacy harm of linkability (an attacker who knows the mapping can recover). If it is random per occurrence, grammar and co-reference break ("Jane Doe said she would …" becomes "Emma said Olivia would …").
- **Faithfulness** — a realistic surrogate must match the demographic distribution of the source. Faker's default locale is US-English; non-US corpora need a locale configuration.
- **Reversibility** — surrogates are one-way (there is no mapping back by design). That is a feature.

Libraries to know: **Faker** (<https://faker.readthedocs.io/>) is the standard Python surrogate generator. Presidio's `AnonymizerEngine` ships a `replace` operator that can be driven by a Faker call.

### Format-preserving encryption (FPE)

FPE encrypts a PII value while preserving its **format** — digits stay digits, characters stay characters, length is preserved. A 16-digit credit card encrypts to a 16-digit string that is still Luhn-valid (with an FFX variant that preserves the checksum).

The canonical reference is **NIST SP 800-38G** *Recommendation for Block Cipher Modes of Operation: Methods for Format-Preserving Encryption*, which defines **FF1** and **FF3** modes. <https://csrc.nist.gov/pubs/sp/800/38/g/final>. The final publication from 2016 still formally specifies both modes, but NIST has stated since April 2017 that FF3 is no longer suitable as a general-purpose FPE method; the **second public draft of SP 800-38G Rev. 1** (3 February 2025, <https://csrc.nist.gov/pubs/sp/800/38/g/r1/ipd>) withdraws FF3 entirely and specifies FF1 only, with a larger minimum domain size and tightened implementation requirements. Rev. 1 was still in draft as of this writing — treat FF1 as the only standards-track mode to adopt in new systems.

**When FPE is right:**

- Structured-data pipelines (database columns of credit-card numbers, SSNs) where downstream systems expect the format and length to be unchanged.
- Reversible redaction with strong cryptographic guarantees — stronger than tokenisation's vault, because the key is the only secret, not a whole database.

**When FPE is wrong:**

- Training-data pipelines. FPE'd values are still high-entropy strings that look like the original type; the model learns nothing useful from them, and if the key is ever compromised the original PII is recoverable from the training data. This is a strict loss versus masking.
- Small domains. FF3 has known cryptanalysis for small domains (phone number last-4 digits, short-numeric IDs). Use masking or surrogates for small-domain entities.

FPE is a strong primitive **for the wrong use case** in a training-data pipeline. Know that it exists, know when operations teams use it, and know not to apply it to corpus text just because it sounds sophisticated.

### Hashing

Hashing replaces each PII span with the hash of the value, typically with a **keyed hash** (HMAC-SHA256 with a per-corpus secret salt) so dictionary attacks against the unsalted hash space do not work.

```python
import hmac, hashlib
def keyed_hash(value: str, key: bytes) -> str:
    return hmac.new(key, value.encode("utf-8"), hashlib.sha256).hexdigest()
```

**When hashing is right:**

- You need **linkability without reversibility** — the same input always produces the same hash, which supports consistent cross-document references without exposing the plaintext.
- You are confident the key is managed with the same discipline as the raw PII.

**When hashing is wrong:**

- Pretraining data. Hash values are 64-character hex strings that destroy natural-language grammar. The model learns nothing useful. Masking is strictly better.
- Any case where the attacker can **brute-force the preimage**. Phone numbers (small space), SSNs (small space with validation), truncated emails — the preimage space is small enough that even HMAC with a salt is defensible only if the salt itself stays secret and rotates per corpus.

Hashing is the right primitive for structured-data pipelines that need join keys; it is almost always the **wrong** primitive for text-corpus scrubbing.

## The per-entity-type policy table

For a pretraining corpus, a reasonable default policy looks like this:

| Entity type | Primitive | Rationale |
|---|---|---|
| `EMAIL_ADDRESS` | Mask `<EMAIL>` | No downstream re-correlation; natural-language signal preserved by placeholder consistency. |
| `PHONE_NUMBER` | Mask `<PHONE>` | Same reasoning. |
| `CREDIT_CARD` | Mask `<CARD>` | Never useful as training signal; always redact. |
| `US_SSN` / national IDs | Mask `<ID>` | Never useful as training signal. |
| `PERSON` (high-confidence) | **Surrogate** (consistent within-doc) | Preserves grammar and co-reference; preserves name distribution. |
| `PERSON` (low-confidence) | Mask `<PERSON>` or pass-through | High precision floor needed to avoid destroying signal on non-PII. |
| `LOCATION` (cities, countries) | **Pass-through** | Commonly not PII in text; destroying location signal breaks training. |
| `LOCATION` (street addresses) | Mask `<ADDRESS>` | Specific street addresses are PII. |
| `URL` | **Pass-through** (unless internal) | Public URLs are not PII; internal URLs can be masked. |
| `IP_ADDRESS` | Mask `<IP>` | IP addresses are PII under GDPR; redact. |
| `MEDICAL_RECORD_NUMBER` / HIPAA identifiers | Mask or drop entire document | HIPAA Safe Harbor requires removal of all 18 identifiers; consider document-level quarantine instead. |
| Internal customer IDs | Mask `<CUSTOMER_ID>` | Rarely useful as training signal. |

For a fine-tuning corpus (chat / resume / email) where entity-shape matters, the `PERSON` and `LOCATION` policy may shift toward **surrogate generation** more aggressively. Author the table for your specific slice; do not cargo-cult.

## When the right answer is "drop the document"

Some documents have so much PII that scrubbing them destroys the signal entirely. Clinical notes, legal filings with named parties, incident reports — scrubbing a document where 40% of the tokens are PII leaves you with gibberish, and the memorisation risk of what remains is high because the surrounding context still identifies the subject.

The pipeline-side answer is **document-level quarantine**:

- Any document with a PII density above a threshold (e.g., 5% of tokens or 10 findings per 1,000 tokens) is routed to a quarantine bucket.
- Quarantine documents are not released to training. They sit in a separately-audited store with their own access control.
- The quarantine manifest is a field in the privacy-risk report.

Chapter 06 covers the quarantine wiring. Chapter 09 covers the manifest shape. The point here is that **redaction is not the only answer** — sometimes the right primitive is no primitive at all, because the document should not be in the corpus.

## Utility evaluation — closing the loop

Every redaction choice has a downstream cost. The pipeline should include a **small-slice training probe** that:

- Trains a tiny model (e.g., a 125M-parameter LM) on the scrubbed corpus and on an unscrubbed-but-dedup'd baseline.
- Measures perplexity on a held-out validation set.
- Reports the delta.

A high delta (perplexity much worse on scrubbed) is a signal that the redaction policy destroyed too much signal. A low delta is reassurance. This is a `fine-tuning-engineer` deliverable in practice (training a probe is their craft); your contribution is the scrubbed slice and the clear statement of the redaction policy used. Chapter 06 wires in the handoff.

## Summary

- Redaction primitives are **mask, tokenise, surrogate, FPE, hash**. Each has a distinct reversibility, length-preservation, and distribution-preservation profile.
- For **pretraining-data pipelines**: masking is the default, surrogates are right for high-signal entities like `PERSON`, tokenisation / FPE / hashing are usually wrong.
- For **structured-data pipelines**: FPE and tokenisation have real roles, with explicit key-management discipline.
- Author a **per-entity-type policy table** and commit it. The privacy-risk report cites it.
- When redaction would destroy a document's signal, the right answer is **document-level quarantine**, not aggressive redaction.
- Close the utility loop with a **small-slice training probe**; the delta is the evidence that the redaction policy did not cost more than it gained.

Chapter 06 picks up with pipeline integration — where in the pipeline each stage runs, and how dedup (per the Kandpal 2022 result) carries privacy load alongside the detector.

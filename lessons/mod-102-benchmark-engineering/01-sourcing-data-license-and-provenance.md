# Sourcing Raw Data: Licensing and Provenance

A benchmark is not just a file of items and labels. It is a claim about where those items came from, what the license permits, and what can be reproduced by someone else. If any of the three is missing, the benchmark cannot be redistributed, cannot be cited without a caveat, and — as later chapters will show — cannot be versioned or decontaminated in a defensible way.

This chapter is the upstream half of the sourcing question: how to pull raw data with a documented license and a provenance chain that survives a legal or reproducibility audit. Chapter 2 covers the downstream half: converting that raw data into a benchmark-ready dataset.

## Why license and provenance are load-bearing, not paperwork

Two reasons the eval engineer cares.

**Redistribution and downstream use.** If your benchmark ends up inside a `datasets` load script, an academic paper, or a public leaderboard, the terms attached to each item's *source* propagate to your users. An eval built on Common Crawl text, on Stack Overflow answers, on GitHub code, on Reddit threads, on YouTube transcripts, or on a purchased corpus each carries a different set of restrictions. The failure mode is not "you get sued" — it is "your benchmark cannot be used by the organizations you built it for," and you find out after the release.

**Contamination reasoning depends on knowing the source.** Chapter 5 covers contamination detection. Every one of those detectors — n-gram overlap against a pretraining corpus, embedding search, likelihood probes — requires you to know what corpora the items came from, when they were collected, and whether they were ever posted on the public web. An eval item scraped from a Stack Overflow question posted in 2019 has to be treated as memorized-until-proven-otherwise for any model whose pretraining cutoff is 2020 or later. An eval item written from scratch by a paid contractor under an NDA has a very different contamination story. You cannot even *ask* the contamination question without knowing which case you are in.

Getting the license and provenance right at sourcing time is orders of magnitude cheaper than reconstructing them from a data lake six months later.

## The provenance record: what to capture per item

For every raw item that enters the pipeline, record the following. Capture it in a structured file (JSONL, Parquet) that lives alongside the raw data and moves with it — not in a wiki page that will drift.

- **`source_id`** — a stable identifier for the origin corpus (e.g. `stackoverflow-2024-01-dump`, `internal-support-tickets-q1-2026`, `contractor-batch-42`). Not free text; a controlled vocabulary maintained in a single file in the repo.
- **`source_uri`** — the specific URL, S3 path, contract PO, or ticket ID the item came from. For scraped items, the exact URL; for a purchased corpus, the vendor and delivery reference.
- **`retrieved_at`** — ISO-8601 UTC timestamp of when your pipeline fetched or received the item. This is the anchor for the "was this in the pretraining cutoff" question later.
- **`content_hash`** — SHA-256 of the raw item bytes. Lets you detect silent upstream changes and lets a downstream consumer verify they got the same item you scored.
- **`license`** — a SPDX identifier (`MIT`, `CC-BY-4.0`, `CC-BY-SA-4.0`, `CC0-1.0`, `ODbL-1.0`, ...) plus any non-standard extensions. Use `NOASSERTION` explicitly when you do not know — never leave the field blank; a blank license is a silent claim of public domain.
- **`license_source`** — where you observed the license (e.g. `https://stackoverflow.com/help/licensing`, `contract-2025-08.pdf#section-4`, `README.md#L3`). A license without a source is a rumor.
- **`attribution_required`** — boolean; if true, the derivative benchmark must ship an attribution manifest. Most Creative Commons licenses require attribution; MIT and Apache-2.0 require the notice; CC0 and public domain do not.
- **`allows_commercial_use`** — boolean; some CC variants (`-NC`) and some purchased corpora forbid it. If any item in your benchmark is `-NC`, the whole benchmark is `-NC` unless you segregate it.
- **`allows_derivatives`** — boolean; `-ND` variants forbid remixing. Labelling an item is a derivative work in most jurisdictions.
- **`share_alike`** — boolean; `-SA` variants require the derivative to be released under the same license (viral). This constrains your benchmark's release license.
- **`consent_status`** — for human-produced text or user data, whether you have consent to use it for benchmark purposes. `explicit-optin` / `broad-tos-consent` / `no-consent-known`. If the last, the item usually cannot be used at all for eval, regardless of technical license.
- **`pii_status`** — `screened-and-clean` / `contains-pii-redacted` / `pii-unknown`. Chapter 2 covers the screening pipeline.
- **`ingest_pipeline_version`** — the version of your ingestion tool that produced this record. When you find a bug in your PII scrubber later, this is how you decide which items to reprocess.

The point is not to fill every field for every item; it is to make each field a required part of the schema so an item cannot enter the pipeline without an answer, even if the answer is `unknown`. `unknown` is queryable; a missing column is not.

## License classes you will encounter, in decreasing ease of use

- **Public domain / CC0.** No restrictions on redistribution, derivatives, or commercial use. Attribution is polite but not required. Easiest case.
- **Permissive open (MIT, Apache-2.0, BSD).** Common on code corpora (GitHub) and some datasets. Redistribution and derivatives allowed with a notice; commercial use fine.
- **Creative Commons attribution (CC-BY).** Common on Wikipedia, government open data, some scientific publications. Redistribution allowed; you must credit the source. If you keep an attribution manifest, safe.
- **CC-BY-SA (share-alike).** Wikipedia is the canonical example. Derivatives must be released under CC-BY-SA. This is fine for a public benchmark; it is a problem if your organization intends to keep the benchmark private and commercial.
- **CC-BY-NC (non-commercial).** Common on academic datasets. Cannot be used in a benchmark that gates production shipping decisions inside a for-profit organization, at least without legal review of the specific "non-commercial" definition the license uses.
- **CC-BY-ND (no derivatives).** Rarely compatible with an eval benchmark — labelling is a derivative.
- **Corporate ToS with silence on downstream use.** Common on scraped web content (Reddit, Twitter/X, Stack Exchange, forums). Silence is not permission. Treat as `NOASSERTION` and get legal to look at it before shipping.
- **Restrictive research licenses (e.g. LDC, some enterprise corpora).** Usually forbid redistribution outright. You can use them internally; you cannot publish the benchmark items themselves. Publish task definitions and evaluator code instead, and describe the corpus for reproducibility.
- **Purchased or contractor-produced data.** Governed by the contract, not by a public license. Read the contract for the derivative-rights clause and the redistribution clause; these vary and are the load-bearing ones.

The failure mode common to the last four categories is that someone downstream — a partner, a regulator, an academic collaborator — asks for the benchmark file, and you cannot give it to them. Plan for that at sourcing time.

## Common sources, with the license and contamination story attached

- **GitHub public code.** License is per-repository. Many repositories have no explicit license (default: all rights reserved). GitHub's ToS grants specific rights to GitHub and to viewers, but *not* automatic redistribution rights to third parties. Contamination story: assume any file present in a public GitHub commit before your model's training cutoff is in the pretraining corpus.
- **Stack Overflow / Stack Exchange.** User content is licensed CC-BY-SA (version varies by post date; check the site's licensing page). Attribution to the author and the URL is required. Contamination story: heavily present in pretraining corpora; use for eval only with a strong contamination check.
- **Wikipedia.** CC-BY-SA-4.0 (with GFDL history for older content). Attribution required. Contamination story: near-universal presence in pretraining; suitable for eval only with post-cutoff timestamping (edits after the model cutoff are safer) and canary items.
- **Common Crawl.** Not itself a license — a redistribution of web pages under fair-use / ToS-of-origin. Each page has its own status. Contamination story: extensive pretraining source; the same pages you would source eval from are the ones the model saw.
- **arXiv.** License varies per paper (typically CC-BY, CC-BY-NC, or arXiv's non-exclusive license). Check `arxiv_license` field in the API response. Contamination story: present in pretraining for major LLMs.
- **News datasets (RCV1, Reuters, NYT).** Restrictive research licenses; usually cannot be redistributed. Use the standard splits and describe rather than republishing.
- **Government open data (US federal, EU open data portal, UK data.gov.uk).** Usually public domain or a permissive open-government license (OGL v3, CC-BY equivalents). Attribution required for some.
- **Human contractor labelling / generation.** Contract-governed. Ensure the contract grants your organization derivative rights and includes an assignment or license broad enough for your intended benchmark release.
- **Internal production data (user prompts, agent traces, support tickets).** Governed by your organization's privacy policy and the user's ToS consent. Almost always requires a PII screening pipeline (Chapter 2) and consent review before it can be used in an eval whose outputs leave the production perimeter.

For each raw source you use, write a one-paragraph "source dossier" in the repo that names the license, the SPDX identifier, the attribution requirements, and the contamination risk for the models you plan to evaluate. Cite the specific license URL. The dossier is what a partner asks for when they want to reuse the benchmark; write it once, at source-selection time.

## Sourcing checklist

Before an item enters the raw store, verify:

1. The source has a written license or contractual grant that permits the use you intend (internal eval, published benchmark, commercial use).
2. The license identifier and license source URL are recorded per item, not per corpus, unless the corpus-level statement is unambiguous.
3. If attribution is required, an attribution manifest is being maintained in parallel with the raw store.
4. If share-alike is required, the eventual benchmark license is compatible.
5. If the data contains user content, a privacy / consent review is on file for the intended use.
6. `retrieved_at` and `content_hash` are recorded, so you can prove what you fetched and when.
7. The source is captured in a source dossier that names the contamination risk for the models under evaluation.

Items that fail any of these do not go into the raw store. They go into a `quarantine/` directory with a note explaining what is missing, so a later reviewer can decide whether to obtain the missing piece or drop the item.

## What documented provenance actually costs

The overhead is real: the sourcing pipeline is longer, the schema is stricter, and items get quarantined that a scrappier pipeline would just ingest. Two things make it worth it.

First, every downstream defense in this module — contamination reports, versioning, deprecation, canary sets — is faster and more defensible when the raw data has a provenance record. You can filter the eval to "items sourced before model X's training cutoff" in a query, not in a rediscovery project.

Second, provenance is a one-way ratchet. You can add it at sourcing time in a straight line. You cannot add it retroactively without doing the sourcing again. Teams that skip this and later need it (typically because a legal question came up, or a partner asked for the manifest, or a contamination probe returned a hit and no one could tell what corpus the item came from) spend far more reconstructing it than they would have spent doing it right up front.

## Summary

Documented licensing and provenance are the entry gate for any raw data that will become a benchmark. Per-item you record the source identifier, the retrieval URL and timestamp, a content hash, the license (SPDX + source URL), the attribution and share-alike constraints, the consent and PII status, and the ingest pipeline version. Choose sources whose license permits your intended downstream use, write a per-source dossier that captures the contamination risk, and quarantine — not silently ingest — items that fail the checks. Chapter 2 picks up from the raw store and turns it into a benchmark-ready dataset.

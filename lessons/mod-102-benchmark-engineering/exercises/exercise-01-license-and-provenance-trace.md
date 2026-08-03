# exercise-01: License and Provenance Trace

**Estimated effort:** 2 hours

## Objective

Take a real, publicly available raw data source and build the per-item provenance record and per-source dossier that Chapter 1 requires. The output is a small artifact — a JSONL file of ~500 rows plus a markdown dossier — that a colleague could hand to a partner or a lawyer without embarrassment.

The point is not to build a full ingestion pipeline. The point is to force through, item by item, the fields the schema demands and to notice where an under-documented source stops being usable.

## Prerequisites

- Chapter 01 of this module.
- Python 3.11+ with `requests`, `pandas` (or `polars`), and `pyarrow` for Parquet if you prefer it over JSONL.
- A GitHub or Hugging Face account for reading the source metadata (no authentication write scopes required).

## Corpora (pick one)

Choose exactly one of the following as your raw source. All are publicly documented and have primary license statements you can cite.

- **Wikipedia article revisions.** Use the Wikipedia REST API or the `wikitextparser` package on a public dump. Reasonable slice: 500 recent revisions of high-traffic articles in a single language. License: CC-BY-SA-4.0 (with GFDL history).
- **Stack Overflow question–answer pairs.** Use a public Stack Exchange data dump (`archive.org` or the Stack Exchange Data Dump). Slice to 500 accepted-answer pairs from a single tag. License: CC-BY-SA (version dependent — check post date and Stack Exchange's licensing page).
- **arXiv paper abstracts + license fields.** Use the arXiv OAI-PMH API or the Kaggle arXiv metadata dump. Slice to 500 recent papers. License: per-paper, listed in metadata; expect a mix of CC-BY, CC-BY-NC, and arXiv non-exclusive.
- **GitHub code snippets from permissively-licensed repositories.** Use the GitHub search API restricted to repositories with `license:mit` or `license:apache-2.0`; slice to 500 files under a size threshold. License: repository-declared.
- **A US-government open dataset from data.gov.** Pick one dataset with a documented license (e.g. a public-use microdata file, an EPA measurement series). Slice to 500 rows or 500 records. License: usually public-domain-in-the-US or an OGL-equivalent.

If your organization has an internal corpus you are allowed to write about, you may substitute it with the internal license statement and access log playing the role of the public license.

## Requirements

### Part A — the source dossier

Write `SOURCE_DOSSIER.md` (target length: 400–800 words). It must contain:

1. **Source identification.** Name, canonical URL, dump/version identifier, and the date you accessed it.
2. **License statement.** SPDX identifier (or `NOASSERTION` if none applies), the URL of the license text you are relying on, and a short paraphrase of the specific clauses relevant to redistributing an eval derived from this data. If per-item licenses vary, describe the distribution.
3. **Attribution requirements.** Whether attribution is required and what a compliant attribution manifest looks like for this corpus.
4. **Share-alike / non-commercial / no-derivative constraints.** Which apply, and what they mean for a benchmark you might publish under (a) a permissive open license and (b) an internal-only license.
5. **Consent and PII considerations.** What kinds of personal data the source might contain, what the source's ToS says about downstream use, and whether human-subjects consent is on file for eval-derivative use.
6. **Contamination story.** Which pretraining corpora likely contain this source, and for which model release cutoffs you would consider items from this source memorized-until-proven-otherwise. Cite corpora inclusion documentation (e.g. C4, The Pile, RedPajama papers) where possible.
7. **A go/no-go for two intended uses:** an internal, non-published benchmark that gates production shipping, and a public benchmark that ships to a leaderboard. Argue each with reference to the constraints above.

### Part B — the per-item provenance JSONL

Produce `provenance.jsonl`, one row per item, with at minimum these fields (from Chapter 1):

- `item_id` (UUIDv4 you generate at ingest time; never reuse)
- `source_id` (controlled vocabulary; declare in a `sources.yaml`)
- `source_uri` (the exact URL, dump path, or record ID)
- `retrieved_at` (ISO-8601 UTC)
- `content_hash` (SHA-256 of the raw item bytes)
- `license` (SPDX identifier or `NOASSERTION`)
- `license_source` (URL or file path where you observed the license)
- `attribution_required` (bool)
- `allows_commercial_use` (bool)
- `allows_derivatives` (bool)
- `share_alike` (bool)
- `consent_status` (`explicit-optin` / `broad-tos-consent` / `no-consent-known` / `not-applicable`)
- `pii_status` (`screened-and-clean` / `contains-pii-redacted` / `pii-unknown` / `not-applicable`)
- `ingest_pipeline_version` (a version string for your fetcher/cleaner)

Target 500 rows. Items that fail any of the checks go into a separate `quarantine.jsonl` with a `quarantine_reason` field explaining what is missing. Report the split (accepted vs. quarantined) in your submission.

### Part C — the attribution manifest

If your chosen source requires attribution, produce `ATTRIBUTION.md` in a format a downstream user could ship with a derivative benchmark. Include:

- One entry per unique upstream author or per source page, whichever the license demands.
- Per-entry: author name (or handle, or `Anonymous` if unnamed), work title (if any), source URL, license identifier.
- A short header block naming the derivative work and the license it is being released under.

If your source is public-domain or does not require attribution, write a one-paragraph statement in `SOURCE_DOSSIER.md` explaining why the attribution manifest is empty, and produce an empty `ATTRIBUTION.md` file with just that statement as a comment.

### Part D — the ingestion script

Write `ingest.py` (or `ingest.ipynb`) that regenerates `provenance.jsonl` and `quarantine.jsonl` from the raw source given only the source URL and a seed. The script must:

- Be deterministic (same input → same output) given the seed and a pinned snapshot of the source.
- Log the tool versions it depends on (`ingest_pipeline_version` in the output rows must match).
- Fail loudly (nonzero exit) if any required field is unfillable for an item that would otherwise be accepted.

## Starter guidance

- Do not skip Part A because it is the boring one. Part B without Part A is a spreadsheet with a plausible-looking `license` column; Part A is what makes it defensible.
- Use SPDX identifiers (`CC-BY-SA-4.0`, `MIT`, `Apache-2.0`, `CC0-1.0`, `NOASSERTION`) from the current SPDX License List. Do not invent identifiers.
- For sources where per-item license varies (arXiv), read the license field from the API response and record it verbatim; do not aggregate to a corpus-level claim you cannot support.
- For Stack Overflow / Stack Exchange, be careful: the license version (CC-BY-SA-3.0 vs. CC-BY-SA-4.0) depends on the post date; see the Stack Exchange licensing page for the transition timeline.
- Wikipedia's attribution requirement is satisfied by a link to the article's revision history; you do not need to list every editor.
- If a source's license page is silent on redistribution and you cannot tell, set `license` to `NOASSERTION`, quarantine the affected items, and explain in the dossier what would resolve it (contact the source, ask legal, find a follow-on statement).
- Store the raw bytes you hashed alongside your submission (or a URL that resolves to them), so a reviewer can verify the `content_hash` field.

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- `SOURCE_DOSSIER.md` names the SPDX identifier, cites the license URL, and answers all seven required questions with source references — not from memory.
- `provenance.jsonl` has all required columns for every accepted row and is joinable by `item_id` to the raw items.
- `quarantine.jsonl` is not empty *unless* the dossier explicitly argues that no item in your sample could fail any check.
- `ATTRIBUTION.md` exists and complies with the source's attribution requirement (or contains the explanation of why none is required).
- `ingest.py` is deterministic and re-runnable; a reviewer running it with the documented seed and source snapshot reproduces the JSONL files byte-identically.
- No fields are silently blank. `NOASSERTION` / `not-applicable` / `unknown` are used explicitly where relevant.
- The go/no-go section explicitly names which of the two intended uses is permitted and why.

## Stretch goals

- **Extend to two sources with different license classes** (e.g. Wikipedia CC-BY-SA and a public-domain government dataset) and produce a combined dossier that explains how the joint benchmark inherits constraints. Show which constraints (share-alike, attribution, non-commercial) are contagious across the union.
- **Build a per-item SPDX SBOM-style manifest** (analogous to a software bill of materials, but for data). Follow the SPDX 3.0 dataset profile or the Croissant metadata format from MLCommons.
- **Wire the ingest script into a CI job** that recomputes `content_hash` on the raw source and flags items whose upstream content has changed since ingest. This is the "silent upstream drift" detector.
- **Write a one-page policy document** for a hypothetical organization ("what our ingestion pipeline must guarantee before any benchmark is shipped to the leaderboard") that generalizes from your specific corpus to the standing shape of the ingestion pipeline.

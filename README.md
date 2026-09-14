# UK Compliance Knowledge Base

> ### What this repository is
>
> An engineering write-up of a retrieval system over the UK retail
> financial-services rulebook, not the running code. The scrapers pull from FCA,
> legislation.gov.uk, JMLSG and BIS, and the corpus is redistributable only under
> each publisher's own terms, so neither the scraped content nor the pipeline is
> published here.
>
> File paths named below (`scripts/scrape-fca-handbook-pdf.ts`,
> `lib/source-urls.ts` and the rest) describe how that system is organised. They
> are not files in this repository. Do not look for them here.
>
> The numbers (887 source documents, 16,829 chunks, the citation and
> cite-or-refuse behaviour) are measurements from that internal system.
>
> What you can take from it: the source-acquisition strategy, the page-precise
> citation design, and the refusal discipline, which is the transferable part. For
> code you can actually run, see
> [spa-automation-toolkit](https://github.com/sayandip1987/spa-automation-toolkit),
> [amazon-connect-flow-tools](https://github.com/sayandip1987/amazon-connect-flow-tools)
> and [humanise-plugin](https://github.com/sayandip1987/humanise-plugin).
>
> Nothing here is legal or regulatory advice.

A second RAG platform, built on the same architecture as
[enterprise-rag-knowledge-base](https://github.com/sayandip1987/enterprise-rag-knowledge-base)
and scaled about 7 times, adapted to the UK financial-services regulatory corpus.
Uploading a PRD produces a structured regulatory gap analysis instead of a
platform map.

> Live: [redacted].netlify.app · JWT-protected
> Build effort: about 4 to 6 hours, single owner, AI-assisted, architecture reused
> Internal-team users: about 25 to 30 compliance officers, product owners, legal liaisons

## What it does

- Indexes 16,829 chunks across 887 source documents spanning the UK retail-financial-services rulebook:
  - 42 FCA Handbook sourcebooks: PRIN, COND, GEN, FIT, COCON, APER, TC, SYSC, CONC, COBS, BCOBS, MCOB, ICOBS, CMCOB, PROD, ESG, MIPRU, MIFIDPRU, GENPRU, INSPRU, IPRU-INV/FSOC/INS, FUND, COLL, PRR, MAR, REC, RCB, FINMAR, BENCH, DTR, SUP, DISP, COMP, DEPP, CASS, FEES, CREDS, FCG, WDPG, UNFCOG
  - Consumer Credit Act 1974, every section, part and schedule
  - Payment Services Regulations 2017, every regulation and schedule
  - JMLSG AML/CTF guidance
  - BIS Consumer Credit Directive guidance
  - FCA's Payment Services and E-Money Approach Document
- PRD-to-regulatory-perimeter mapping. Compliance and product teams upload a PRD and get a structured map: which regimes apply and why, applicable rules and obligations with cited sections and PDF page numbers, required compliance workflows, identified gaps with recommendations, and open questions for legal counsel
- Page-precise source citations. Every source badge resolves to the exact page of the official PDF (`...sourcebook/CONC.pdf#page=42`), so a click puts a specialist on the rule text rather than a homepage
- Routes around bot protection through official content channels. Three of six target sites block direct access: legislation.gov.uk uses AWS WAF, jmlsg.org.uk uses Akamai, and handbook.fca.org.uk is a JS-only Angular SPA. The pipeline routes around all three, via Wayback Machine `id_` snapshots for the first two and the per-sourcebook PDF API at `api-handbook.fca.org.uk` for the third. Zero failures across all 42 FCA sourcebooks
- Isolated from the sibling knowledge base. Same Qdrant cluster, separate collection (`uk_compliance_kb`), separate JWT secret, separate frontend, separate Netlify project. Payloads, indexes and auth tokens cannot cross between the two platforms
- 8/10 sanity-check accuracy across in-domain queries. Both misses return defensible alternative sources: FCA SYSC for AML questions is valid AML guidance, and the FCA Approach Document for unauthorised-payment-allocation questions explains the underlying PSR 2017 rule
- Every generated answer ends with "This is informational and not legal advice", and the system prompt instructs the model to escalate when the context is silent rather than guess

## Architecture

```
                       User Question / PRD Upload
                                 |
                                 v
                  +-------------------------------------------+
                  | Intent Router (GPT-4o-mini)               |  Returns up to 3 source filters
                  +-------------------------------------------+  (e.g. ["JMLSG Guidance", "FCA Handbook"])
                                 |
                                 v  (parallel with embedding)
                  +-------------------------------------------+
                  | OpenAI Embeddings                         |  text-embedding-3-small (1536-dim)
                  +-------------------------------------------+
                                 |
                +---------------+----------------+
                |                                |
                v                                v
       +------------------+         +------------------------+
       | Q&A Flow         |         | PRD Analysis Flow      |
       | Embed + filter   |         | GPT-4o extracts 8-12   |
       | -> retrieve top-15|         | regulatory queries ->  |
       +------------------+         | parallel vector search |
                |                   +------------------------+
                v                                |
       +------------------+                      |
       | Reranker         |  GPT-4o-mini scores  |
       | (0-10 per chunk) |  each passage        |
       | -> top-5         |                      |
       +------------------+                      |
                |                                |
                +---------------+----------------+
                                |
                                v
                  +-------------------------------------------+
                  | Qdrant Cloud — uk_compliance_kb           |  16,829 points · 1536-dim · cosine
                  | payload indexes on source + section_id    |  Filtered search over 6 isolated sources
                  +-------------------------------------------+
                                |
                                v
                  +-------------------------------------------+
                  | GPT-4o                                    |  Grounded answer with page-precise citations
                  +-------------------------------------------+  + "not legal advice" footer
                                |
                                v
                  +-------------------------------------------+
                  | Next.js Chat UI                           |  Markdown + page-deep-linked source badges
                  +-------------------------------------------+  + structured PRD reports
```

## How it was built, phase by phase

**Phase 1, source discovery and access strategy.** The six target sources span three different access postures:

| Source | Live access | Working route |
|---|---|---|
| BIS Consumer Credit Directive Guidance | Public PDF | direct download (`assets.publishing.service.gov.uk` PDF) |
| FCA Payment Services & E-Money Approach | Public PDF | direct download (`fca.org.uk` PDF) |
| Consumer Credit Act 1974 | AWS WAF JS challenge blocks `curl` | Wayback Machine `id_` snapshots |
| Payment Services Regulations 2017 | AWS WAF JS challenge blocks `curl` | Wayback Machine `id_` snapshots |
| JMLSG Guidance | Akamai bot protection (403) | Wayback Machine `id_` snapshots |
| FCA Handbook | Angular SPA, empty HTML, no JSON API | Discovered the per-sourcebook PDF API at `api-handbook.fca.org.uk/files/sourcebook/<CODE>.pdf` |

The FCA Handbook route mattered most. Probing the URL pattern revealed 42 sourcebooks downloadable as authoritative PDFs, about 80 MB in total, which bypassed the SPA and avoided the multi-hour headless-browser crawl otherwise required.

**Phase 2, scrapers, per-source and checkpointed.** Each scraper is filesystem-checkpointed, so re-running skips sourcebooks already cached on disk:

- `scripts/scrape-bis-pdf.ts` downloads the BIS guidance PDF, parses with `pdf-parse`, splits by form-feed page break, one JSON per page
- `scripts/scrape-fca-psr-pdf.ts` does the same for the 296-page FCA Approach Document
- `scripts/scrape-cca-1974.ts` discovers all parts, sections and schedules from `/contents`, fetches each via `web.archive.org/web/2024id_/<url>` to bypass AWS WAF, survives per-page Wayback timeouts, and is rate-limited at about 700 ms between requests
- `scripts/scrape-psr-2017.ts` takes the same Wayback approach for PSR 2017 (UKSI 2017/752)
- `scripts/scrape-jmlsg.ts` is a Wayback-based crawler discovering JMLSG pages from the home page and current-guidance index
- `scripts/scrape-fca-handbook-pdf.ts` downloads each of 42 sourcebooks via the FCA's PDF API, parses with `pdf-parse`, chunks by page (form-feed delimited)
- `scripts/scrape-fca-handbook.ts` is the original fallback scraper using Wayback, kept as belt and braces after the PDF route superseded it
- `scripts/scrape-all.ts` orchestrates all six scrapers in sequence, surviving individual failures

Scrapers write JSON of shape:

```json
{
  "source": "FCA Handbook",
  "section_id": "CONC/page/0042",
  "section_url": "https://www.handbook.fca.org.uk/handbook/CONC",
  "section_heading": "CONC: 5.2A.34 G — creditworthiness assessment",
  "parent_heading": "FCA Handbook — CONC (Consumer Credit sourcebook)",
  "content": "...",
  "scraped_at": "2026-04-27T12:55:00.000Z",
  "scraper": "scrape-fca-handbook-pdf"
}
```

**Phase 3, knowledge ingestion pipeline.** `scripts/ingest.ts` walks `scraped_content/<source>/*.json`, chunks long docs at paragraph boundaries with sentence-level overlap (`MAX_CHUNK = 1800` chars, about 450 tokens), embeds in batches of 32 via OpenAI, and upserts into the `uk_compliance_kb` Qdrant collection. Two payload indexes are created at ingest time for server-side filtering: `source` (keyword) and `section_id` (keyword). Re-running with `RESET=1`, the default, drops and rebuilds; `RESET=0` appends. Result: 887 source docs to 16,829 chunks, averaging 1360 chars, about 7 times the chunk count of the sibling knowledge base.

**Phase 4, RAG chat application.** `/api/chat` runs intent-classify, embed, filtered vector retrieve, rerank, generate, with module filter and embedding in parallel, falling back to unfiltered if the classifier returns fewer than 3 results. Sources come back with page-precise URLs (`#page=N`) and short citations, for example `FCA Handbook — CONC, p. 42`. `/api/analyze-prd` is a multi-query RAG: GPT-4o extracts 8 to 12 regulatory queries across CCA, FCA Handbook, JMLSG/MLR 2017, PSR 2017 and the FCA Approach, runs parallel vector searches, and generates a structured gap-analysis report (PRD summary, regulatory perimeter table, applicable rules and obligations table, required compliance workflows, identified gaps and risks with recommendations, open questions for counsel, and the "not legal advice" disclaimer). `/api/auth` is JWT with 24-hour expiry, admin email and password seeded from env. `/api/feedback` embeds user feedback combined with the original Q&A context and upserts it under a "User Feedback" source.

**Phase 5, page-precise source resolution.** The `resolveSource()` helper in `lib/source-urls.ts` maps each chunk's `(source, section_id, section_url, section_heading)` to deep-linked URLs:

| Input source pattern | Resolved URL | Citation example |
|---|---|---|
| FCA Handbook chunk with `section_id = "CONC/page/42"` | `https://api-handbook.fca.org.uk/files/sourcebook/CONC.pdf#page=42` | `FCA Handbook — CONC, p. 42` |
| BIS Guidance chunk with `section_id = "page/57"` | `…/bis-10-1053-consumer-credit-directive-guidance.pdf#page=57` | `BIS Consumer Credit Directive Guidance, p. 57` |
| FCA Approach Doc chunk | `…/payment-services-electronic-money-approach.pdf#page=275` | `FCA Payment Services & E-Money Approach, p. 275` |
| CCA 1974 chunk with `section_id = "section/8"` | legislation.gov.uk section URL | `Consumer Credit Act 1974, s.8` |
| PSR 2017 chunk with `section_id = "regulation/71"` | legislation.gov.uk regulation URL | `PSR 2017, reg. 71` |
| JMLSG chunk | Wayback-resolved page URL | `JMLSG — <heading>` |

Resolved citation labels feed into the GPT-4o context as `[1] <citation>` markers, so generated answers cite real page numbers rather than fabricated rule numbers.

**Phase 6, production deployment.** Separate GitHub repo, separate Netlify project, auto-deploy on push to `main`. First build completed in 47 seconds. All env vars imported as Netlify secret values via `.env` paste at deploy time.

**Phase 7, live bug fixes after launch.** Two bugs surfaced in the first user session and were fixed within minutes:

1. **GFM table renderer mangling cells.** `splitTableRow` used a literal space as the sentinel for escaped pipes, so every space inside a cell got replaced with `|` on the way back. Fixed with a `line.split(/(?<!\\)\|/)` negative-lookbehind split, unit-tested before push.
2. **FCA Handbook badges pointing at the SPA root.** Chunk IDs already carried PDF page numbers, but badges resolved to `/handbook/CONC`, the sourcebook overview with no page. The new `resolveSource()` helper routes them to `…/CONC.pdf#page=N`, and the badge UI now shows `FCA Handbook — CONC, p. 42`.

Both fixes shipped via `git push`, Netlify auto-rebuilt, and the new bundle was live within about 50 seconds.

## Knowledge base coverage

| Source | Source Docs | Chunks | Coverage |
|---|---|---|---|
| FCA Handbook (42 sourcebooks) | 10,033 pages | ~13,500 | Full: all 42 sourcebooks |
| FCA Payment Services & E-Money Approach | 296 pages | 664 | Full document (FCA-FG17/3) |
| Payment Services Regulations 2017 | 173 sections | 548 | Full SI 2017/752, every regulation and schedule |
| Consumer Credit Act 1974 | 276 sections | 641 | Full UKPGA 1974/39, every part, section, schedule |
| BIS Consumer Credit Directive Guidance | 111 pages | 203 | Full BIS/10/1053 |
| JMLSG Guidance | 23 pages | 39 | Top-level guidance plus linked AML/CTF pages |
| User Feedback (live) | grows with use | n/a | Embedded into the same collection at runtime |
| **Total** | **~10,918 docs** | **16,829 chunks** | **All 6 target sources fully indexed** |

Re-running `npm run scrape:fca-pdf` refreshes the FCA Handbook PDFs to current published versions. Re-running `npm run ingest` re-embeds and re-upserts the full corpus in about 7 minutes.

## Performance and quality, pipeline choices

| Dimension | Choice | Trade-off / Justification |
|---|---|---|
| Top-5 in-source precision | Two-stage retrieve then rerank (15 to 5) | 100% top-5 from correct source on in-domain queries; rerank exposes noise (scores 0–1) for a principled "not found" |
| Cross-regime bleed | Eliminated | Intent classifier scopes search to 3 sources or fewer; falls back to unfiltered if the filter is too restrictive |
| Avg latency | ~1–3 s (chat), ~25 s (PRD analysis) | +2 LLM calls (classifier + reranker) for chat; +N embedding and search calls for PRD multi-query. Acceptable for a compliance workload |
| Qdrant chunks | 16,829 | All 6 sources fully indexed; payload indexes on `source` + `section_id` enable server-side filtering |
| Source citation precision | Page number embedded in URL via `#page=N` | Click jumps to the rule page directly, which removes manual navigation in 1000+ page PDFs |
| "Not legal advice" discipline | Enforced in system prompt | Every nuanced answer ends with the disclaimer; the model is instructed to escalate when context is silent |
| Data safety posture | Full JSON backup before any destructive op | `npm run backup` snapshots all 16,829 points (vectors and payloads, about 350 MB) |

## Context engineering

A regulatory corpus punishes sloppy context construction harder than a product corpus does, because the reader needs to land on a specific rule rather than get the gist of one.

**The chunk boundary is the page, because the citation is a page.** FCA sourcebooks and the BIS and FCA PDFs are chunked on form-feed page breaks rather than on a character window. That is not the most semantically tidy split, and it is the right one anyway: `section_id` of `CONC/page/0042` is what later becomes `CONC.pdf#page=42`. Choosing the chunk boundary to match the citation unit is what makes a click land on the rule instead of near it.

**`MAX_CHUNK = 1800` characters, about 450 tokens.** Small enough that top-5 passages leave room for a real answer, large enough that a regulation and its qualifying subsection usually survive together. Long documents split at paragraph boundaries with sentence-level overlap for the same reason as the sibling system: a rule that straddles a split stays recoverable.

**Citations enter the context as resolved labels, not raw metadata.** `resolveSource()` runs before generation, and what reaches GPT-4o is `[1] FCA Handbook — CONC, p. 42` rather than a URL or a chunk ID. The model is never asked to construct a citation, only to reference one already in front of it. That single ordering choice is most of why answers cite real page numbers.

**Scope the context before filling it.** The intent classifier narrows to at most 3 of 6 sources before retrieval runs, so a CONC question does not spend window space on CASS passages. When the classifier returns nothing, the filter is dropped rather than guessed at, because a wrong scope is worse than a wide one on a corpus where the answer may genuinely sit in an unexpected sourcebook.

## Grounding, judging and refusal

**Grounding is a citation that resolves, not a better prompt.** The anti-hallucination work here is almost entirely plumbing. Chunk IDs carry the PDF page number, `resolveSource()` turns that into a `#page=N` deep link, and the resolved citation label goes into the model's context as a `[1] <citation>` marker. The model cites a page because a real page number is the only thing in front of it. That is a cheaper and more reliable guarantee than any instruction not to invent rule numbers.

**Refusal is a feature, and it is built rather than requested.** The system prompt instructs the model to escalate when the retrieved context is silent, and the reranker gives that instruction something to act on: out-of-domain questions score 0 to 1 across every candidate, so "the rulebook does not say" is a measurable state rather than a judgement call. On a compliance corpus this matters more than coverage. A confident paraphrase of a rule that does not exist is worse than no answer.

**LLM-as-judge, kept narrow.** GPT-4o-mini reranks on a bounded 0 to 10 scale for one question at a time. It is not asked whether an answer is correct, or whether a regime applies, because those need a specialist. The 8/10 sanity-check result is reported with both misses examined rather than as a headline, and both turned out to return defensible alternative sources, which is the kind of detail an aggregate score hides.

**No fine-tuning.** The corpus changes when the FCA republishes a sourcebook, so `npm run scrape:fca-pdf` followed by a re-ingest is the update path. A fine-tuned model would need retraining on the same cadence and would still not produce a page number you can click.

## Operational costs

| Service | Free Tier | Estimated Monthly (Production) |
|---|---|---|
| OpenAI Embeddings | — | ~$0.10 (one-shot ingest), then near-zero for queries |
| OpenAI GPT-4o + GPT-4o-mini | — | ~$5–$15 (answer generation + classifier + reranker) |
| Qdrant Cloud | 1 GB free | $0 (16,829 × 1536-dim points fits comfortably) |
| Netlify | 100 GB bandwidth, 125 K functions | $0 |

**Why this build took 4 to 6 hours despite about 7 times the data:** the architecture, Next.js scaffolding, intent-classifier prompt, reranker and JWT layer were lifted directly from the sibling knowledge base. New work was scoped to source discovery and per-regulator scrapers, the page-precise citation resolver, the PRD analysis prompt re-targeted at regulatory gap analysis, and the "not legal advice" discipline. Reuse was the multiplier.

# UK Compliance Knowledge Base — UK Financial Regulation Search
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
> are not files in this repository — do not look for them here.
>
> The numbers — 887 source documents, 16,829 chunks, the citation and
> cite-or-refuse behaviour — are measurements from that internal system.
>
> What you can take from it: the source-acquisition strategy, the page-precise
> citation design, and the refusal discipline, which is the transferable part. For
> code you can actually run, see
> [spa-automation-toolkit](https://github.com/sayandip1987/spa-automation-toolkit),
> [amazon-connect-flow-tools](https://github.com/sayandip1987/amazon-connect-flow-tools)
> and [humanise-plugin](https://github.com/sayandip1987/humanise-plugin).
>
> Nothing here is legal or regulatory advice.


A second RAG platform built off the same architecture as Project 1, scaled ~7× and adapted to the UK financial-services regulatory corpus. PRD upload produces a structured regulatory gap analysis instead of a platform map.

> Live: [redacted].netlify.app · JWT-protected
> Build effort: ~4–6 hours, single owner, AI-assisted (architecture reused from Project 1)
> Internal-team users: ~25–30 compliance officers, product owners, legal liaisons

## Project Outcome

- Indexed **16,829 knowledge chunks** across **887 source documents** spanning the UK retail-financial-services rulebook:
  - **42 FCA Handbook sourcebooks** — PRIN, COND, GEN, FIT, COCON, APER, TC, SYSC, CONC, COBS, BCOBS, MCOB, ICOBS, CMCOB, PROD, ESG, MIPRU, MIFIDPRU, GENPRU, INSPRU, IPRU-INV/FSOC/INS, FUND, COLL, PRR, MAR, REC, RCB, FINMAR, BENCH, DTR, SUP, DISP, COMP, DEPP, CASS, FEES, CREDS, FCG, WDPG, UNFCOG
  - **Consumer Credit Act 1974** — every section, part, and schedule
  - **Payment Services Regulations 2017** — every regulation and schedule
  - **JMLSG AML/CTF guidance**
  - **BIS Consumer Credit Directive guidance**
  - **FCA's Payment Services & E-Money Approach Document**
- **PRD-to-Regulatory-Perimeter Mapping** — Compliance and product teams upload a PRD and instantly get a structured map: regulatory perimeter (which regimes apply, with reasoning), applicable rules and obligations (with cited sections and PDF page numbers), required compliance workflows, identified gaps with concrete recommendations, and open questions for legal/compliance counsel
- **Page-precise source citations** — every source badge resolves to the exact page of the official PDF (`...sourcebook/CONC.pdf#page=42`) so a click jumps a specialist straight to the rule text rather than a homepage
- **Bot-protection bypass via official content channels** — three of six target sites are aggressively bot-protected (legislation.gov.uk uses AWS WAF; jmlsg.org.uk uses Akamai; handbook.fca.org.uk is a JS-only Angular SPA). Pipeline routes around all three: legislation.gov.uk via Wayback Machine `id_` snapshots; FCA Handbook via the discovered per-sourcebook PDF API at `api-handbook.fca.org.uk`; JMLSG via Wayback. **Zero failures across all 42 FCA sourcebooks.**
- **Watertight isolation from sibling KB** — same Qdrant cluster, separate collection (`uk_compliance_kb`), separate JWT secret, separate frontend, separate Netlify project — no possibility of cross-contamination of payloads, indexes, or auth tokens between the two platforms
- **8/10 sanity-check accuracy** across in-domain queries; the two "misses" surface defensible alternative sources (FCA SYSC for AML questions also returns valid AML guidance; FCA Approach Doc for unauthorised-payment-allocation questions also explains the underlying PSR 2017 rule)
- **Built-in disclaimers** — every generated answer ends with "This is informational and not legal advice"; system prompt instructs the LLM to escalate when context is silent rather than guess

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

## How It Was Built — Phase-by-Phase

**Phase 1 — Source Discovery & Access Strategy.** The six target sources span three radically different access postures:

| Source | Live access | Working route |
|---|---|---|
| BIS Consumer Credit Directive Guidance | Public PDF | direct download (`assets.publishing.service.gov.uk` PDF) |
| FCA Payment Services & E-Money Approach | Public PDF | direct download (`fca.org.uk` PDF) |
| Consumer Credit Act 1974 | AWS WAF JS challenge blocks `curl` | Wayback Machine `id_` snapshots |
| Payment Services Regulations 2017 | AWS WAF JS challenge blocks `curl` | Wayback Machine `id_` snapshots |
| JMLSG Guidance | Akamai bot protection (403) | Wayback Machine `id_` snapshots |
| FCA Handbook | Angular SPA — empty HTML, no JSON API | Discovered the per-sourcebook PDF API at `api-handbook.fca.org.uk/files/sourcebook/<CODE>.pdf` |

The FCA Handbook discovery was the breakthrough: probing the URL pattern revealed 42 sourcebooks downloadable as authoritative PDFs (~80 MB total), bypassing the SPA entirely and avoiding the multi-hour headless-browser crawl that would otherwise have been required.

**Phase 2 — Scrapers (per-source, checkpointed).** Each scraper is filesystem-checkpointed — re-running skips sourcebooks already cached on disk:

- `scripts/scrape-bis-pdf.ts` — downloads BIS guidance PDF, parses with `pdf-parse`, splits by form-feed (page break), one JSON per page
- `scripts/scrape-fca-psr-pdf.ts` — same flow for the 296-page FCA Approach Document
- `scripts/scrape-cca-1974.ts` — discovers all parts/sections/schedules from `/contents`, fetches each via `web.archive.org/web/2024id_/<url>` to bypass AWS WAF; resilient to per-page Wayback timeouts; rate-limited at ~700 ms between requests
- `scripts/scrape-psr-2017.ts` — same Wayback approach for PSR 2017 (UKSI 2017/752)
- `scripts/scrape-jmlsg.ts` — Wayback-based crawler discovering JMLSG pages from home + current-guidance index
- `scripts/scrape-fca-handbook-pdf.ts` — downloads each of 42 sourcebooks via the FCA's PDF API, parses with `pdf-parse`, chunks by page (form-feed delimited)
- `scripts/scrape-fca-handbook.ts` — original fallback scraper using Wayback (kept as belt-and-braces; PDF route superseded it)
- `scripts/scrape-all.ts` — orchestrator running all six scrapers in sequence, surviving individual failures

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

**Phase 3 — Knowledge Ingestion Pipeline.** `scripts/ingest.ts` walks `scraped_content/<source>/*.json`, chunks long docs at paragraph boundaries with sentence-level overlap (`MAX_CHUNK = 1800` chars / ~450 tokens), embeds in batches of 32 via OpenAI, upserts into the `uk_compliance_kb` Qdrant collection. Two payload indexes created at ingest time for server-side filtering: `source` (keyword), `section_id` (keyword). Re-running with `RESET=1` (default) drops and rebuilds; `RESET=0` appends. **Result: 887 source docs → 16,829 chunks (avg 1360 chars) — ~7× the chunk count of Project 1.**

**Phase 4 — RAG Chat Application.** `/api/chat` runs intent-classify → embed → filtered vector retrieve → rerank → generate, with module filter and embedding in parallel; fallback to unfiltered if classifier returns <3 results. Sources returned with page-precise URLs (`#page=N`) and short citations (e.g. "FCA Handbook — CONC, p. 42"). `/api/analyze-prd` is a multi-query RAG: GPT-4o extracts 8–12 regulatory queries (across CCA, FCA Handbook, JMLSG/MLR 2017, PSR 2017, FCA Approach), runs parallel vector searches, generates a structured gap-analysis report (PRD summary, regulatory perimeter table, applicable rules/obligations table, required compliance workflows, identified gaps & risks with recommendations, open questions for counsel, "not legal advice" disclaimer). `/api/auth` is JWT with 24-hour expiry; admin email/password seeded from env. `/api/feedback` embeds user feedback combined with original Q&A context, upserts under a "User Feedback" source.

**Phase 5 — Page-Precise Source Resolution.** `lib/source-urls.ts` `resolveSource()` helper maps each chunk's `(source, section_id, section_url, section_heading)` to deep-linked URLs:

| Input source pattern | Resolved URL | Citation example |
|---|---|---|
| FCA Handbook chunk with `section_id = "CONC/page/42"` | `https://api-handbook.fca.org.uk/files/sourcebook/CONC.pdf#page=42` | `FCA Handbook — CONC, p. 42` |
| BIS Guidance chunk with `section_id = "page/57"` | `…/bis-10-1053-consumer-credit-directive-guidance.pdf#page=57` | `BIS Consumer Credit Directive Guidance, p. 57` |
| FCA Approach Doc chunk | `…/payment-services-electronic-money-approach.pdf#page=275` | `FCA Payment Services & E-Money Approach, p. 275` |
| CCA 1974 chunk with `section_id = "section/8"` | legislation.gov.uk section URL | `Consumer Credit Act 1974, s.8` |
| PSR 2017 chunk with `section_id = "regulation/71"` | legislation.gov.uk regulation URL | `PSR 2017, reg. 71` |
| JMLSG chunk | Wayback-resolved page URL | `JMLSG — <heading>` |

Resolved citation labels feed into the GPT-4o context as `[1] <citation>` markers, so generated answers cite real page numbers rather than fabricated rule numbers.

**Phase 6 — Production Deployment.** Separate GitHub repo, separate Netlify project, auto-deploy on push to `main`, first build completed in 47 seconds. All env vars imported as Netlify "secret values" via `.env` paste at deploy time.

**Phase 7 — Live Bug Fixes (post-launch).** Two bugs surfaced in the first user session and were fixed within minutes:

1. **GFM table renderer mangling cells** — `splitTableRow` used a literal space as the sentinel for escaped pipes, so every space inside a cell got replaced with `|` on the way back. Fixed with `line.split(/(?<!\\)\|/)` negative-lookbehind regex split, unit-tested before push.
2. **FCA Handbook badges pointing at the SPA root** — chunk IDs already carried PDF page numbers, but badges resolved to `/handbook/CONC` (sourcebook overview, no page). New `resolveSource()` helper routes them to `…/CONC.pdf#page=N`; badge UI now shows "FCA Handbook — CONC, p. 42".

Both fixes shipped via `git push`; Netlify auto-rebuilt and the new bundle was live within ~50 seconds.

## Knowledge Base Coverage

| Source | Source Docs | Chunks | Coverage |
|---|---|---|---|
| FCA Handbook (42 sourcebooks) | 10,033 pages | ~13,500 | Full — all 42 sourcebooks |
| FCA Payment Services & E-Money Approach | 296 pages | 664 | Full document (FCA-FG17/3) |
| Payment Services Regulations 2017 | 173 sections | 548 | Full SI 2017/752 — every regulation + schedule |
| Consumer Credit Act 1974 | 276 sections | 641 | Full UKPGA 1974/39 — every part, section, schedule |
| BIS Consumer Credit Directive Guidance | 111 pages | 203 | Full BIS/10/1053 |
| JMLSG Guidance | 23 pages | 39 | Top-level guidance + linked AML/CTF pages |
| User Feedback (live) | grows with use | n/a | Embedded into the same collection at runtime |
| **Total** | **~10,918 docs** | **16,829 chunks** | **All 6 target sources fully indexed** |

Re-running `npm run scrape:fca-pdf` refreshes the FCA Handbook PDFs to current published versions. Re-running `npm run ingest` re-embeds and re-upserts the full corpus in ~7 minutes.

## Performance & Quality — Pipeline Choices

| Dimension | Choice | Trade-off / Justification |
|---|---|---|
| Top-5 in-source precision | Two-stage retrieve→rerank (15→5) | 100% top-5 from correct source on in-domain queries; rerank exposes noise (scores 0–1) for principled "not found" |
| Cross-regime bleed | Eliminated | Intent classifier scopes search to ≤3 sources; fallback to unfiltered if filter is too restrictive |
| Avg latency | ~1–3 s (chat), ~25 s (PRD analysis) | +2 LLM calls (classifier + reranker) for chat; +N embedding+search calls for PRD multi-query — acceptable for a compliance workload |
| Qdrant chunks | 16,829 | All 6 sources fully indexed; payload indexes on `source` + `section_id` enable server-side filtering |
| Source citation precision | Page number embedded in URL via `#page=N` | Click jumps to the rule page directly — eliminates manual navigation in 1000+ page PDFs |
| "Not legal advice" discipline | Enforced in system prompt | Every nuanced answer ends with the disclaimer; LLM instructed to escalate when context is silent |
| Data safety posture | Full JSON backup before any destructive op | `npm run backup` snapshots all 16,829 points (vectors + payloads, ~350 MB) |

## Operational Costs

| Service | Free Tier | Estimated Monthly (Production) |
|---|---|---|
| OpenAI Embeddings | — | ~$0.10 (one-shot ingest), then near-zero for queries |
| OpenAI GPT-4o + GPT-4o-mini | — | ~$5–$15 (answer generation + classifier + reranker) |
| Qdrant Cloud | 1 GB free | $0 (16,829 × 1536-dim points fits comfortably) |
| Netlify | 100 GB bandwidth, 125 K functions | $0 |

**Why this build was 4–6 hours despite ~7× the data:** the architecture, Next.js scaffolding, intent-classifier prompt, reranker, and JWT layer were lifted directly from Project 1. New work was scoped to (a) source-discovery / scrapers per-regulator, (b) page-precise citation resolver, (c) PRD analysis prompt re-targeted at regulatory gap analysis, (d) "not legal advice" discipline. Reuse was the multiplier.

---


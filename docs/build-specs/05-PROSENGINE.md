# Build-Spec 05 — ProsEngine

> Full-stack replication spec for ProsEngine's 4 tools: Company Intel, Person
> Intel, Website Intel, Data Seeder. Grounded in the live codebase (paths + line
> numbers real).

---

## 0. What this subsystem is

Deep, single-target intelligence. Four tools share a parallel-research →
synthesis → verdict → humanize pipeline:
- **Company Intel** — 11 parallel agents → CompanyReport.
- **Person Intel** — 9 parallel agents → PersonProfile (auto-seeds a watchlist).
- **Website Intel** — multi-page BFS crawl + per-page AI analysis.
- **Data Seeder** — real 4-phase pipeline EVAL → APPROVE → HARVEST → ENRICH.

```
target → parallel research (Gemini×N + Perplexity×N + Claude + GPT-4o + Web Seeder)
        → aggregate context → synthesis (Claude→Gemini→GPT-4o→Nexus)
        → verdict layer (§7) → humanizer (§8) → report (+ Activepieces webhook)
```

---

## 1. Frontend — `pages/prospecting/`

- **company.tsx** — 5-step wizard (Name → Seller Context → Goals → Known Facts → Generate); expandable report sections; VerdictPills; humanized summary; Save/Delete.
- **person.tsx** — 5-step wizard (Identity → ...); auto-fill from URL params / `websiteIntelContext`; quick-fill execs; auto-seed into "ProsEngine Watchlist".
- **website.tsx** — Target URL → Scan → Configure questionnaire → Extract → Results grid; export; inline ProsEngineChat; writes `websiteIntelContext` for Person Intel.
- **seeder.tsx** — text mode (prompt+filters+count) OR URL mode embedding `<SeederWizard />` (4-phase EVAL/APPROVE/HARVEST/ENRICH).
- **components/VerdictPill.tsx** — lavender `🛡 <trustScore>` pills, color by certainty (verified/likely/unverified/estimated), hover shows rationale+sources.
- **components/SeederWizard.tsx** — drives the 4 seed endpoints; polls rows every 3s.

---

## 2. API endpoints

### Company Intel — `routes/company-intel.ts`
- `POST /profile` `{companyName, website?, crNumber?, city?, sellerContext?, intelligenceGoals?[], knownFacts?}` → `CompanyReport` (profile/financials/ownership/leadership/operations/market/approach/news/intelligence + `executiveSummary`, `humanizedSummary`, `verdicts[]`).
- `POST /save` · `GET /saved` · `DELETE /saved/:id`.
- `POST /web-seed` `{rootUrl, maxPages?<=50, seedMode?, companyName?}` → `WebSeederResult`.

### Person Intel — `routes/person-intel.ts`
- `POST /profile` `{name, company?, title?, linkedinUrl?, websiteUrl?, sellerContext?, intelligenceGoals?[], knownFacts?}` → `PersonProfile` (profile/career/education/company_analysis/wealth_profile/personal_profile/approach_strategy/intelligence_notes + `_pipelineStats`, `humanizedProfile`, `verdicts`).
- `POST /save` · `GET /saved` · `DELETE /saved/:id` · `POST /quick` (Claude-only fast enrich).

### ProsEngine Chat — `routes/prosengine-chat.ts`
- `POST /prosengine/chat` `{messages?|message, context?, mode?, model?}` → `{reply, profileUpdate?, researchSteps?}`. Intent classes: answer_from_context / perplexity_search / crawl_url / deep_research (o4-mini-deep-research, Perplexity fallback).
- `POST /prosengine/chat/stream` (SSE: `intent | agent_start | agent_done | synthesising | reply | done`).
- `POST /prosengine/seed` (legacy text generator).
- `POST /prosengine/analyze-url` `{url, description?}` → `{siteType, companiesDetected, questions[]}`.
- `POST /prosengine/seed-from-url` `{url, answers?, description?}` → `{records[], summary, market_insight}` (11 parallel agents).

### Data Seeder (4-phase) — `routes/seeder.ts`
- `POST /prosengine/seed/eval` `{url, maxPages?=25}` → `{planId, seedPlan{entities[],fields[],pagesScanned}}` (crawl4ai + Nexus extractor).
- `POST /prosengine/seed/approve` `{planId, approvedFields[]}`.
- `POST /prosengine/seed/harvest` `{planId}` → background ScrapeGraphAI extract → `seeder_rows`.
- `GET /prosengine/seed/rows?planId=` → `{rows}` (≤500).
- `POST /prosengine/seed/enrich` `{stagingIds[]}` → marks enriched (deep enrich via Lead Factory is the wired-in TODO).

All seed/harvest/enrich run inside `runInJob`; eval uses `enterJob` (paid-API budget).

---

## 3. Backend engines

### Company Intel pipeline
Phase 1 (parallel `Promise.allSettled`): Web Seeder (8 pages) + Gemini×5 (profile/ownership/leadership/market/financials, 25s) + Perplexity×4 (25s) + Claude (30s) + GPT-4o (25s).
Phase 2: aggregate ≤14k chars + seller context.
Phase 3: synthesis Claude Sonnet 4.6 (120s) → Gemini 2.5 Flash → GPT-4o (90s) → Nexus (OpenRouter DeepSeek/Qwen) → minimal report.
Phase 4: verdict layer. Phase 5: humanize. Phase 6: Activepieces webhook (non-blocking). Cost-gated via `enterJob`.

### Person Intel pipeline
Phase 1 (9 parallel agents): Perplexity×4 (career/company/wealth/news, 25s) + company crawl (15s, 5 pages) + Gemini A/B (20s) + Claude (12s) + GPT-4o (12s).
Phase 2: LinkedIn discovery (regex across sources). Phase 3: aggregate ≤18k. Phase 4: synthesis Gemini→Claude→GPT-4o (35s each, first non-null). Phase 5: inject LinkedIn. Phase 6–7: verdict + humanize. Phase 8: auto-seed "ProsEngine Watchlist" (`lead_list_items`, source=prosengine, score 80). Phase 9: webhook.

### Website Intel — `lib/web-seeder.ts`
PowerScraper auto-escalation (Cheerio→Playwright→Stealth→BeautifulSoup) + BFS crawl (priority about>team>contact>news, ≤maxPages cap 50, depth 3, pagination detect) → per-page type classify → email/Saudi-phone extraction → per-page Claude analysis (20s) → aggregated `WebSeederResult`.

### Data Seeder 4-phase
EVAL: crawl4ai (≤25 pages, 12k chars) → Nexus extraction tier infers entities+fields+confidence → `seeder_plans` (status eval).
APPROVE: store `approvedFields`, status approved.
HARVEST: ScrapeGraphAI extract with approved schema (fallback fullStackCrawl) → `seeder_rows` (pending), status done.
ENRICH: mark rows enriched (deep enrich via Lead Factory = TODO).

---

## 4. Output layering

1. LLM synthesis → JSON report.
2. **Verdict layer** (`lib/credibility/verdict.ts`): `scoreFact()` per fact. Certainty: verified (≥2 primary, 90–100) / likely (1 primary or ≥2 secondary, 60–89) / unverified (single weak, 20–59) / estimated (LLM-only, 0–40). Tier weights primary 90 / secondary 65 / inferred 30. Primary = Tadawul/registry/filings; secondary = Perplexity/Gemini/scrapers; inferred = Claude/GPT-4o training.
3. **Humanizer** (`lib/report/humanize.ts`): strip markdown/code artifacts, optional Nexus writer rewrite, fact-preservation guard (numbers/dates/proper-nouns must survive, else retry, else return stripped raw). Returns `{humanized, removedCount, factPreserved}`.

---

## 5. Database tables

`lib/db/src/schema/`: `company_intel_research.ts`, `prosengine_research.ts`, `seeder_staging.ts`, `research_jobs.ts`, `prospecting_results.ts` (+ `prospecting_jobs/sessions/exports`).

- **company_intel_research** — `company_name, website, cr_number, city, seller_context, intelligence_goals, known_facts, report (JSON text), tags, notes, timestamps`.
- **prosengine_research** — `person_name, company, title, linkedin_url, seller_context, intelligence_goals, known_facts, report (JSON text), tags, notes, created_at`.
- **seeder_plans** — `root_url, status (eval|approved|harvesting|done|failed), entities jsonb, fields jsonb, approved_fields jsonb, pages_scanned, created_at`.
- **seeder_rows** — `plan_id, entity_type (company|person|product|contact), data jsonb, source_url, enrichment_status (pending|enriched), created_at`.
- **research_jobs** — `query, status, progress, report jsonb, sources jsonb, findings jsonb, agent_results jsonb, timestamps`.
- **prospecting_results** — `job_id, company_data jsonb (FastEnrichmentResult), enrichment_status, source_url, enrichment_report_id, created_at`.

---

## 6. Dependencies & env vars

**Required:** `DATABASE_URL`; `ANTHROPIC_API_KEY` (primary synthesis); `GEMINI_API_KEY` (searches+synthesis+Arabic); `OPENAI_API_KEY` (synthesis/knowledge); `PERPLEXITY_API_KEY` (web search, budget-gated; `DISABLE_PERPLEXITY` kill switch).

**Optional LLM (Nexus fallback):** `OPENROUTER_API_KEY`, `GROQ_API_KEY`, `DEEPSEEK_API_KEY`, `MOONSHOT_API_KEY`, `OLLAMA_BASE_URL`.

**Scrapers/data:** `SCRAPEGRAPH_API_KEY` (Seeder HARVEST), `CRAWL4AI_API_KEY` (optional/self-host), `APOLLO_API_KEY`, `EXPLORIUM_API_KEY`, `SCOUT_URL`/`SCOUT_API_KEY`. PowerScraper needs a Chromium (Playwright) + Python BeautifulSoup subprocess.

Models: Claude `claude-sonnet-4-6`; OpenAI `gpt-4o` + `o4-mini-deep-research-2025-06-26`; Gemini `gemini-2.5-flash`; Perplexity `sonar`; Nexus DeepSeek/Qwen/Llama/Mistral.

---

## 7. Key timeouts

Gemini search 25s · Perplexity 25s · company crawl ≤18s · Company synthesis Claude 120s / GPT-4o 90s · Person synthesis 35s each · person crawl 15s · Claude/GPT knowledge 12s · per-page web-seeder analysis 20s · Seeder EVAL ~20s.

---

## 8. Minimum build to replicate

1. Postgres + the 6 tables above.
2. PowerScraper (Cheerio/Playwright + Python bs4) + crawl4ai; ScrapeGraphAI for the Seeder HARVEST phase (or fallback to crawl+regex).
3. Express routes `/api/company-intel/*`, `/api/person-intel/*`, `/api/prosengine/*`, `/api/prosengine/seed/*` (+ chat SSE).
4. The parallel-research → synthesis → verdict → humanize pipeline on the Nexus router (needs Anthropic + Gemini keys; Perplexity optional but big quality lift).
5. React 4-tool pages + VerdictPill + SeederWizard.
6. Person Intel auto-seed into the Lead Genome "ProsEngine Watchlist" list; verdict/humanizer shared with all report endpoints.

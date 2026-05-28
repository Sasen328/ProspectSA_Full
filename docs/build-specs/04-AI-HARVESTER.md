# Build-Spec 04 — AI Harvester (Harvest AI)

> Full-stack replication spec for Harvest AI: the Masaar Engine (CR intelligence),
> Masaar Database, AI Database Builder, the unified source registry, the power
> scraper, and plugins. Grounded in the live codebase (paths + line numbers real).

---

## 0. What this subsystem is

The data-acquisition + enrichment backbone. Four cooperating parts:
- **Masaar Engine** — multi-agent Saudi CR intelligence pipeline (mc.gov.sa stealth + CAPTCHA → CR parse → AOA → deep research → compliance → bilingual report).
- **Masaar Database** — keyword/sector harvest from open-data + Amaaly AOA + custom URLs; enrich/dedup/export.
- **AI Database Builder** — harvest from built-in Saudi directories (Wikidata/CMA/Tadawul/Yellow Pages/...) + bulk enrich + push to main DB.
- **Unified source registry** (`harvest_sources` + `source_enforcement`) — one source-of-truth with per-engine whitelist/blacklist.

```
CR# → /masaar/start → 5 agents (SSE, CAPTCHA overlay) → MasaarReport (EN+AR)
keyword → /masar/database/harvest → power-scraper L1-L4 → masar_companies → enrich/dedup/export
sources → /builder/harvest → directory harvesters → builder_companies → enrich → push-to-database
registry → /api/sources/* → harvest_sources + per-engine enforce
```

---

## 1. Frontend

| File | Role |
|---|---|
| `pages/masaar/index.tsx` (1–849) | Masaar Engine: CR input, stealth toggle, agent timeline, CAPTCHA overlay, bilingual report |
| `pages/masaar/database.tsx` (1–850) | Masaar DB: harvest form, company grid, bulk actions, export |
| `pages/database-builder/index.tsx` (1–500) | Builder: source cards, harvest, results table, bulk enrich/dedup/clean/push |

### Action buttons → endpoints
- **Masaar Engine:** Start → `POST /api/masaar/start`; Submit CAPTCHA → `POST /api/masaar/captcha/:jobId`; stream → `GET /api/masaar/stream/:jobId`.
- **Masaar DB:** Harvest, Re-enrich, Pipeline-enrich, Enrich All, Deduplicate, Delete/Bulk-delete, Export (csv/excel/word/pdf/pptx), Analyze-source, Cancel — see §2.3.
- **Builder:** Harvest one/all, Enrich/Bulk-enrich/Re-enrich-all, Deduplicate, Auto-clean, Push-to-database, Export, Cancel — see §2.4.

---

## 2. API endpoints

### 2.1 Harvest AI consolidation — `routes/harvest-ai.ts`
- `GET /api/harvest-ai/masaar-database` · `GET /builder-database` · `GET /stats` (feeds Lead Factory suggest).

### 2.2 Masaar Engine — `routes/masaar.ts`
- `POST /start` `{crNumber(7–12), stealthMode?}` → `{jobId,...}`.
- `POST /captcha/:jobId` `{captchaText, captchaFor}`.
- `GET /stream/:jobId` → SSE `AgentEvent`: `agent_start|agent_log|agent_complete|agent_error|captcha_required|captcha_solved|stealth_solving|stealth_solved|stealth_session|job_complete|job_error` (with `captchaScreenshot` base64, `stealthMethod`, `report`).

### 2.3 Masaar Database — `routes/masar-database.ts` (1–1017)
- Harvest: `POST /harvest` `{companyName?, keyword?, sector?, parameters?, sources?[], customUrls?[]}`; `GET /stream/:jobId` (SSE `HarvestEvent`); `GET /jobs`; `POST /jobs/:jobId/cancel`.
- Companies: `GET /companies?page&limit&search&city&enrichmentStatus&source`; `GET /companies/:id`; `DELETE /companies/:id`; `DELETE /companies/bulk {ids[]}`.
- Enrich: `POST /companies/:id/re-enrich`; `POST /companies/:id/pipeline-enrich` (full 5-agent pipeline); `POST /enrich-all {mode?:"pending"|"all"}`.
- `POST /deduplicate {threshold?=0.88}`.
- `GET /export?format=csv|excel|word|pdf|pptx&search&ids`.
- Custom sources: `GET/POST/DELETE /custom-sources`; `POST /analyze-source {url}` (Claude inspects site).
- `GET /stats`.

### 2.4 AI Database Builder — `routes/builder.ts` (1–797)
- Sources: `GET/POST/DELETE /sources`; `POST /sources/:id/harvest {batchSize?, enrichmentDepth?}`.
- Harvest: `POST /harvest {sourceIds?[], batchSize?=5, enrichmentDepth?}`.
- Jobs: `GET /jobs`, `GET /jobs/:jobId`, `POST /jobs/:jobId/cancel`.
- Results: `GET /stats`, `GET /results?page&limit&status&industry&hideDuplicates&jobId`, `POST /results/:id/enrich`, `POST /results/:id/save-enrichment`, `POST /bulk-enrich {ids?}`, `POST /re-enrich/:id`, `POST /re-enrich-all`, `POST /deduplicate`, `POST /clean`.
- `GET /export?format=...`, `POST /push-to-database {ids?}`.

### 2.5 Unified source registry — `routes/sources.ts` (1–143)
- `GET /api/sources?category=` · `POST /api/sources` · `PATCH /:id` · `DELETE /:id` (user-only).
- `POST /:id/test` (HEAD probe → status ok/degraded/down, updates lastSynced).
- `POST /enforce {engineName, requiredIds[], excludedIds[]}` (upsert per engine).
- `GET /health` (dashboard feed).
- Helper `resolveEnginesSources(engineName)` → `{all, required, excluded}`.

---

## 3. Backend engines

### 3.1 Masaar Engine — `lib/masaar-engine.ts` (2000+ lines)
5 agents in sequence:
1. **MC.gov.sa stealth browser + Claude CR parser** — StealthBrowser navigates the CR e-service, handles Cloudflare + CAPTCHA (Claude Vision auto-solve → human fallback → session reuse), fills CR + code, extracts HTML; Claude (+ NEXUS extractor) parses fields (Perplexity fallback if empty).
2. **Amaaly AOA** — stealth search emagazine.aamaly.sa, download AOA PDF, OCR/extract shareholders+board+management.
3. **Deep research** — Perplexity×5 + Gemini×4 + Claude + GPT-4o (+ OpenRouter/Groq stubs), skips down providers.
4. **Compliance** — OFAC/UNSC/EU + CMA/SAMA/ZATCA/Maroof/Najiz → `ComplianceResult` (overallRisk low/medium/high).
5. **Bilingual report compiler** — Claude EN + GPT-4o AR → `MasaarReport`.

Job mgmt: `createJob/getJobEmitter/submitCaptcha`, 45-min auto-cleanup. CAPTCHA wait up to 180s. Budget-guarded Perplexity via `paid-api-guard`.

`MasaarReport` shape: `crNumber, fetchedAt, stealthMode, sources{mcGovSa,emagazine,najiz}, parsed{names, legalForm, city, capital, shareholders[], managers[], board, rules...}, aoa, compliance, legalAgencies[], conflicts[], reportAr, reportEn`.

### 3.2 Masaar Harvester — `lib/masar-harvester.ts`
`runHarvest(jobId, keyword, sources, customUrls, options)` dispatches per source (open-data CKAN, amaaly-aoa stealth, custom URLs via power-scraper); inserts to `masar_companies` (status pending); `enrichMasarCompany(id)` (Crawl4AI+Perplexity+Claude); dedup by CR → normalized EN → normalized AR (keep highest enrichment score).

### 3.3 Builder Engine — `lib/builder-engine.ts`
`runHarvest({sourceIds, batchSize, enrichmentDepth})` spawns concurrent source harvesters → `builder_companies`. `reEnrichCompany`→`enrichCompanyWithAI`; `deduplicateAll()`; `autoClean()`. Depth basic→standard(AI)→deep(+Crawl4AI+Perplexity).

### 3.4 Power Scraper — `lib/power-scraper.ts` (4 layers)
L1 Cheerio (static) → L2 Playwright (JS) → L3 Playwright+Stealth (camoufox-js if present) → L4 BeautifulSoup subprocess (`bs4_extract.py`, Arabic RTL/malformed). `scrapePage(url, {engine, timeoutMs, followPagination, pageClassifier})` → `PageData`. Auto-pagination (≤20 pages), UA + viewport rotation, 800–2500ms human delays.

### 3.5 Stealth Browser — `lib/stealth-browser.ts`
Anti-fingerprint (webdriver/plugins/languages/WebGL/canvas noise), HumanBehavior (mouse Bézier, scroll, jitter), `autoSolveCaptcha()` (Claude Vision, 3 attempts, human fallback), session persistence (cookies+localStorage per domain).

---

## 4. Source registry & enforcement

`harvest_sources` unifies the legacy `composer_user_sources` / `masar_custom_sources` / `builder_custom_sources`.
Enforcement: load `enabled` sources → look up `source_enforcement` for the engine → if `requiredIds` non-empty use ONLY those, else use all minus `excludedIds`. `trustWeight` (0–100) feeds the credibility verdict layer. `visibility`: system (read-only) / user (deletable) / team.

---

## 5. Plugins / built-in sources

- `lib/data-sources.ts` — `SAUDI_DATA_SOURCES` (Wikidata 726, Saudi Open Data, MoC, CMA, Tadawul, Yellow Pages, Daleel, Kompass, chambers...). Per-source harvester dispatch (SPARQL / scraper / API).
- `lib/free-sources.ts` — `harvestGleifSA`, `harvestOpenCorporatesSA`, `harvestWikidataSA`, `harvestTadawulSA`, `harvestHunterIO`, `harvestGithubOrg` (waterfall priority).
- Scraper libs: Cheerio (L1), Playwright (L2), camoufox-js (L3), BeautifulSoup4 (L4); Scout-side Crawl4AI, ScrapeGraphAI, Sherlock, TheHarvester, Scrapy, Selenium.

---

## 6. Database tables

`lib/db/src/schema/`: `masar_companies.ts`, `masar_harvest_jobs.ts`, `masar_custom_sources.ts`, `builder_companies.ts`, `builder_custom_sources.ts`, `builder_jobs.ts`, `harvest_sources.ts`.

- **masar_companies** — names, `cr_number` (unique), legal form, city/region, capital, founding/registration/expiry, `shareholders jsonb`, `board_of_directors jsonb`, `management jsonb`, activity, status, source, `enrichment_status`, contact, employees, revenue estimate+rationale, `news_headlines jsonb`, `analysis_en/ar`, `analysis_data jsonb`, timestamps, `enriched_at`.
- **masar_harvest_jobs** — `job_id (unique), keyword, source, status, companies_found, companies_enriched, error_message, timestamps`.
- **builder_companies** — `job_id, source_id, names, industry, city/region/country, contact, description(_ar), employee_count int, revenue, founding_year int, cr_number, capital, entity/company type, owner_*, estimated_wealth, shareholders/key_executives (JSON text), market_positioning, recent_news, linkedin_url, enrichment_score, enrichment_status, is_duplicate, is_validated, timestamps`.
- **harvest_sources** — `label, url, type, category, language, countries jsonb, industries jsonb, credibility, trust_weight, enabled, visibility, required_for_engines jsonb, last_synced, status, created_at`.
- **source_enforcement** — `engine_name, required_ids jsonb, excluded_ids jsonb, updated_at`.

---

## 7. Export formats (both DBs)
CSV (quoted) · XLSX (3 sheets: Companies / Shareholders / Management+Board) · Word (HTML→DOC full profiles) · PDF (dark print theme) · PPTX (1 slide/company).

---

## 8. Dependencies & env vars

**Required:** `DATABASE_URL`; `ANTHROPIC_API_KEY` (CR parse, report, CAPTCHA vision); `SCOUT_URL` (Python Scout).
**Strongly recommended:** `OPENAI_API_KEY` (AR report, synthesis), `GEMINI_API_KEY` (research ×4), `PERPLEXITY_API_KEY` (research ×5, budget-guarded; `PERPLEXITY_JOB_BUDGET`, `DISABLE_PERPLEXITY`).
**Optional LLM:** `OPENROUTER_API_KEY`, `GROQ_API_KEY`.
**Scrapers:** `CAMOUFOX_ENABLED`, `THEHARVESTER_BIN`, `CHROMIUM_EXECUTABLE_PATH`; Scout pip deps (crawl4ai, scrapegraphai, sherlock, scrapy, playwright, beautifulsoup4, feedparser...).
**Proxy/CAPTCHA (optional):** `*PROXY*`+`NEXUS_PROXY_ENABLED`, `*CAPTCHA*`+`NEXUS_CAPTCHA_ENABLED`.

Registries scraped (no key, public/CAPTCHA-gated): mc.gov.sa, emagazine.aamaly.sa, Najiz, CMA, SAMA, ZATCA, Maroof.

---

## 9. Gotchas
- CAPTCHA wait soft-fails after timeout → job status `failed`.
- Masaar jobs auto-expire after 45 min in memory.
- `stealthMode` defaults true.
- Dedup threshold currently exact-normalization at 0.88.
- Only `visibility:"user"` sources deletable.
- Builder polls every 3s.

---

## 10. Minimum build to replicate
1. Postgres + the 7 tables above.
2. Python Scout + a Chromium (Playwright) for stealth; BeautifulSoup subprocess for L4.
3. Express routes `/api/masaar/*`, `/api/masar/database/*`, `/api/builder/*`, `/api/sources/*`, `/api/harvest-ai/*` with SSE.
4. Masaar engine (stealth browser + CAPTCHA vision needs `ANTHROPIC_API_KEY`), harvester, builder engine, power-scraper, on the Nexus router.
5. The unified `harvest_sources` + `source_enforcement` with `resolveEnginesSources` consulted at every engine's job start.
6. React Masaar/Builder pages consuming the SSE shapes in §2.

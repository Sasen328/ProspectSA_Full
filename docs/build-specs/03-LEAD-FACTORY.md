# Build-Spec 03 — Lead Factory (+ Signal Intelligence + Relationship Tree)

> Full-stack replication spec for the Lead Factory 7-agent pipeline, the Signal
> Intelligence engine, and the 4-agent Relationship Tree. Grounded in the live
> codebase (file paths + line numbers real).

---

## 0. What this subsystem is

The heavy lead-generation engine. Three surfaces share one backend fabric:
- **Lead Factory** — 7-agent pipeline: ICP brief → harvest → enrich → signals → validate → score+copywrite → publish.
- **Signal Intelligence** — event-driven scoring (news/sanctions/contracts/regulatory) → `recommendedAction`.
- **Relationship Tree** — 4-agent org-chart + network + outreach sequencer.

```
brief → /lead-factory/start → 7 agents (SSE stream) → results table → publish → companies pool (+ optional downstream signals/rel-intel)
company → /signals/scan → Scout + sanctions + RSS → scored signals → recommendedAction
company → /relationship-intel/start → 4 agents (SSE) → org chart + connections + ranked outreach
```

---

## 1. Frontend

### Lead Factory — `pages/lead-factory/`
- `index.tsx` — tab nav (Company Hunt / Person Hunt / Results + Signals + Relationship).
- `company.tsx` (1–85) — Company Hunt filter panel + agent stream; `brief` (mode=company), `useLeadFactoryStream(jobId)`; "Hunt" → `POST /api/lead-factory/start`.
- `person.tsx` (1–88) — Person Hunt (mode=person) + tenure/experience/education filters.
- `results.tsx` (1–150) — results table (CompanyName, domain, phone, email, city, industry, employees, revenue, ICP score, tier, validation, CR, LinkedIn), row drawer → Company/Person/Relationship Intel buttons; bulk Publish/Reject/Export.

### Signal Intelligence — `pages/signal-intelligence/`
- `index.tsx` (1–150) — company scanner (EN+AR) → `POST /api/signals/scan`; signal cards by category; live feed.
- `tree.tsx` — live push view from `/api/signals/feed` (SSE).

### Relationship Tree — `pages/relationship-intel/`
- `index.tsx` (1–150) — org-chart builder → `POST /api/relationship-intel/start`; SVG org chart colored by seniority; ranked outreach cards.
- `tree.tsx` — org network SVG graph.

---

## 2. API endpoints

### Lead Factory — `routes/lead-factory.ts`
- `POST /start` (body = `LeadFactoryBrief`) → `{ok, jobId}`; runs pipeline in background.
- `GET /stream/:jobId` → SSE: `agent_start | agent_log | agent_progress | lead_found | lead_enriched | lead_scored | lead_published | lead_rejected | agent_complete | agent_error | pipeline_complete | heartbeat`.
- `GET /jobs` (last 50) · `GET /jobs/:jobId` · `GET /results/:jobId` (by icpScore desc).
- `POST /results/:jobId/publish` `{autoEnrichDownstream?}` — publish A/B tiers to companies pool.
- `POST /results/:jobId/bulk-action` `{action:"publish"|"reject", rowIds[]<=500, autoEnrichDownstream?}`.
- `POST /results/:jobId/export?format=csv|xlsx|json|pdf|ppt`.
- `POST /jobs/:jobId/cancel`.
- `GET /company-suggest?q=` · `GET /person-suggest?q=` (autocomplete; max 8).

### Signals — `routes/signals.ts`
- `POST /scan` `{company_name, company_name_ar?, company_id?, domain?, run_llm?, save_to_db?}` → `SignalScanResult` (totals, sanctioned, topPositive/topNegative, recommendedAction, overall buying/risk).
- `GET /:companyId` · `GET /by-name/:name` · `GET /recent` · `GET /feed` (SSE, 3s tick).
- `POST /news` · `POST /sanctions` · `POST /individual` · `POST /regulatory`.
- `POST /:id/push-to-genome` → inserts a lead from a signal.

### Relationship Intel — `routes/lead-factory.ts` (`/relationship-intel/*`)
- `POST /start` (body = `RelationshipIntelBrief`) → `{ok, jobId}`.
- `GET /stream/:jobId` → SSE: `agent_start | agent_log | org_node_found | stakeholder_enriched | network_connection | outreach_contact | agent_complete | agent_error | pipeline_complete | heartbeat`.
- `GET /jobs` (last 20) · `GET /jobs/:jobId` · `POST /jobs/:jobId/cancel`.

---

## 3. Lead Factory — 7 agents (`lib/lead-factory-engine.ts`)

1. **ICP Mapper & Source Orchestrator** (269–450) — NEXUS synthesis selects from 40+ sources (Saudi gov: Maroof/Bluepages/Aamaly/MOCI/MISA/MODON/NIC/HRDF/Etimad/NCBE/SAMA/CMA/ZATCA/GASTAT/Najiz; GCC exchanges + registries; global open data GLEIF/OpenCorporates/Wikidata/GitHub/Clearbit; 12 media RSS; exec directories; job boards; AI: Perplexity/Scout/Gemini). Emits source priority + 5–7 EN/AR queries each.
2. **Lead Harvester** (474–1300+) — executes per-source harvest, dedup by domain/CR/phone → `RawLead[]`.
3. **Deep Enrichment** — Scout site-intel + GLEIF LEI + OpenCorporates + Wikidata + Gemini/NEXUS field synthesis (parallel, 20s timeouts).
4. **Signal Intelligence** — `scanCompanySignals()` per company → `signalData` (buying/risk, sanctioned, top signals).
5. **Validate, Verify & Deduplicate** (1331–1514) — phone/email/name/CR rules → `pass|warn|reject`; dedup vs `lead_fingerprints` (domain/CR/phone/email exact, name ≥0.88).
6. **ICP Scoring + AI Copywriter** (1514+) — composite 0–100 → tier A(80+)/B(60+)/C(40+)/D; NEXUS generates outreach email + LinkedIn + opening angle + cultural note + conversation hook.
7. **Publish & Seed** — insert `lead_factory_results`; publish A/B to `companies`+`executives`; optional downstream signals + rel-intel.

`LeadFactoryBrief` (Zod, 71–116): `inputMode (segment|list)`, `mode (person|company)`, `icpDescription`, industries/cities/companySizes/entityTypes/targetTitles/seniority/departments, signal filters, data-requirement flags (hasExecutives/hasWebsite/hasVerifiedEmail), `targetCount<=500`, `enrichmentDepth (basic|standard|deep)`, `autoEnrichDownstream`.

---

## 4. Signal engine (`lib/signal-engine.ts`)

Score weights (36–58): positive — ipo 10, funding 9, contract 8/9, acquisition 8, expansion 7, revenue 7; negative — bankruptcy 10, sanctions 10, fraud 9, fine 8, lawsuit 7, investigation 7, downgrade 7, layoff 6.
`scanCompanySignals()` (138–176): parallel Scout news/sanctions/contracts + local Google/Saudi RSS + free sanctions screen; dedup by URL; per-article NEXUS extraction-tier 1-sentence summary.
recommendedAction: `risk≥9→disqualify`, `risk≥7→hold`, `buying≥8→prioritize`, `buying≥6→monitor`.

**Signal monitor** (`lib/signal-monitor.ts`) — push stream over Argaam/Arab News/Saudi Gazette/Mubasher RSS (+ optional Perplexity/Gemini), NEXUS extraction → `SignalAlert` events + heartbeat.

---

## 5. Relationship engine — 4 agents (`lib/relationship-intel-engine.ts`)

1. **Org Mapper** (134–300+) — Scout site-intel + Gemini research (Tadawul/CMA/Argaam/SaudiCEOs/Forbes/CNBC/Zawya) → `OrgNode[]` (executive/board/shareholder; seniority; email/linkedin/nationality).
2. **Stakeholder Enricher** — `scoutSignalsIndividualFull()` per node + NEXUS profile.
3. **Network Expander** — shared board members + parent/subsidiary + JV → `NetworkConnection[]` (strength strong/medium/weak).
4. **Outreach Sequencer** — rank by seniority + ICP relevance; top 10 get WhatsApp opener + email + LinkedIn + best time + conversation hook + cultural note → `OutreachContact[]`.

---

## 6. External dependency — Python Scout (FastAPI, port 8099)

`artifacts/python-scout/main.py`. Endpoints used here:
- `/scout/site-intel`, `/scout/osint/harvest`, `/scout/osint/social` (Sherlock 15+ platforms), `/scout/crawl`, `/scout/ai-extract` (Gemini), `/scout/full-scan`.
- Signals: `/scout/signals/news`, `/signals/sanctions` (OFAC SDN/UN/EU), `/signals/contracts`, `/signals/full`, `/signals/individual(-full)`, `/signals/regulatory` (CMA/SAMA/ZATCA/NCBE/Maroof/Najiz/Tadawul).
Deps: BeautifulSoup4, Requests, Scrapy, Playwright, dnspython, whois, feedparser, google-genai, Sherlock. Sanctions screening also available natively in TS via `lib/sanctions-screen.ts`.

---

## 7. Database tables

`lib/db/src/schema/lead_factory.ts`, `company_signals.ts`.

- **lead_factory_jobs** — `id, status, input_mode, brief jsonb, target_count, agent_progress jsonb, total_discovered/enriched/validated/published/rejected, error_message, created_at, completed_at`.
- **lead_factory_results** — company fields + `key_executives jsonb, raw_data jsonb, enriched_data jsonb, signal_data jsonb, icp_score, priority_tier, buying_score, risk_score, quality_score, validation_status, validation_reasons jsonb, is_duplicate, duplicate_of, outreach_email/linkedin/whatsapp, opening_angle, cultural_note, conversation_hook, published_lead_id, published_company_id, created_at`.
- **lead_fingerprints** — dedup index (see Build-Spec 02).
- **relationship_intel_jobs** — `id, target_company_name(_ar), target_cr_number, target_website, status, org_chart_data jsonb, network_data jsonb, outreach_plan jsonb, total_contacts/connections, adjacent_companies, error_message, timestamps`.
- **company_signals** — `id, company_id, company_name(_ar), domain, category, event_types jsonb, primary_event_type, title, summary, source_url/name, published_at, llm_summary, buying_signal_score, risk_score, relevance_score, recommended_action, is_sanctioned, sanctions_hits jsonb, raw_signals jsonb, created_at`.

---

## 8. Dependencies & env vars

**Required:** `DATABASE_URL`; `SCOUT_URL` (Scout sidecar) for enrichment/signals; at least one LLM key (Gemini strongly recommended as Nexus fallback).

**LLM (Nexus waterfall OpenRouter→Groq→Gemini→Claude→Ollama):** `OPENROUTER_API_KEY`, `GROQ_API_KEY`, `GEMINI_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OLLAMA_BASE_URL`+`OLLAMA_MODEL`.

**Realtime/search:** `PERPLEXITY_API_KEY` (+ `DISABLE_PERPLEXITY` kill switch), `TAVILY_API_KEY`.

**Contact data (optional):** `APOLLO_API_KEY`+`APOLLO_CLIENT_SECRET`+`APOLLO_ACCESS_TOKEN`, `EXPLORIUM_API_KEY`, `HUNTER_API_KEY`, `WAPPALYZER_API_KEY`.

**Scraping/proxy/captcha (optional):** `APIFY_API_KEY`, `CHROMIUM_EXECUTABLE_PATH`, `IPROYAL_*`/`LUNAPROXY_*`/`WEBSHARE_PROXY_LIST`+`NEXUS_PROXY_ENABLED`, `*CAPTCHA*`+`NEXUS_CAPTCHA_ENABLED`.

**Automation (optional):** `ACTIVEPIECES_URL/API_KEY/FLOW_*`.

---

## 9. Minimum build to replicate

1. Postgres + the 5 tables above.
2. Python Scout sidecar (or stub the `/scout/*` calls) — without it, enrichment/signals degrade to free sources.
3. Express routes `/api/lead-factory/*`, `/api/signals/*`, `/api/relationship-intel/*` with SSE.
4. The engine libs (7-agent, signal, relationship) on the Nexus router with ≥1 LLM key.
5. React pages consuming the SSE event shapes in §2; `useLeadFactoryStream` hook.
6. `lead_fingerprints` dedup shared with Lead Genome.

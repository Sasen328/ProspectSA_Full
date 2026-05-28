# ProspectSA — Build-Spec Replication Pack

Standalone, plug-and-build specifications for the five core subsystems. Each doc
is grounded in the live codebase (real file paths + line numbers) and covers the
**full stack**: frontend → API endpoints → backend engines → database tables →
dependencies/env vars → a "minimum build to replicate" checklist.

> These specs deliberately exclude the agent **Swarm** (you are building that on
> Kimi). They describe everything the Swarm would plug into.

## The five subsystems

| # | Spec | Surface |
|---|---|---|
| 01 | [AI Chat / Composer](./01-AI-CHAT-COMPOSER.md) | Orchestrator tool-loop, 6-stage Composer wizard, Customize drawer, Behavior Agent (AI observer) |
| 02 | [Lead Genome](./02-LEAD-GENOME.md) | Saved-lead bucket + AI-scored list builder + deep enrich |
| 03 | [Lead Factory](./03-LEAD-FACTORY.md) | 7-agent pipeline + Signal Intelligence + Relationship Tree |
| 04 | [AI Harvester](./04-AI-HARVESTER.md) | Masaar Engine + Masaar DB + AI DB Builder + source registry + power-scraper |
| 05 | [ProsEngine](./05-PROSENGINE.md) | Company / Person / Website Intel + 4-phase Data Seeder |

## Shared backbone (read these once; every subsystem uses them)

- **Nexus LLM router** (`lib/nexus/llm-router.ts`, `roles.ts`) — tiers
  `extraction · arabic · synthesis · bulk · realtime`; roles map to tiers; ordered
  attempt chain per tier with fallthrough (OpenRouter free → Groq → DeepSeek →
  Gemini → Claude → GPT-4o → Ollama). `nexusRunRole(role, task)` is the entry point.
- **Verdict layer** (`lib/credibility/verdict.ts`) — `scoreFact()` tags every fact
  `verified|likely|unverified|estimated` with a 0–100 trustScore. Consumes
  `trust_weight` from the source registry.
- **Humanizer** (`lib/report/humanize.ts`) — strips machine artifacts + optional
  Nexus-writer rewrite with a fact-preservation guard. Every report endpoint pipes
  through it.
- **Source registry** (`harvest_sources` + `source_enforcement`, `routes/sources.ts`)
  — one source-of-truth; `resolveEnginesSources(engine)` returns required/excluded
  per engine; trust_weight feeds the verdict layer.
- **Paid-API guard** (`lib/paid-api-guard.ts`) — AsyncLocalStorage job context +
  per-job budget; gates Perplexity/Apollo/Explorium so background fan-out can't
  burn spend without an explicit job (`enterJob`/`runInJob`/`canSpend`/`recordSpend`).
- **Python Scout sidecar** (`artifacts/python-scout`, port 8099) — site-intel,
  OSINT (Sherlock/TheHarvester), deep crawl (Scrapy/Crawl4AI), AI extract
  (ScrapeGraphAI/Gemini), and the signals suite (news/sanctions/contracts/
  individual/regulatory).
- **PowerScraper** (`lib/power-scraper.ts`) — 4-layer escalation: Cheerio →
  Playwright → Playwright+Stealth (camoufox) → BeautifulSoup subprocess.

## Global environment reference

**Boot-critical:** `DATABASE_URL`, plus at least `ANTHROPIC_API_KEY` (and ideally
`GEMINI_API_KEY`). `SCOUT_URL` for enrichment/signals.

**LLM providers (fallback fabric):** `OPENROUTER_API_KEY`, `GROQ_API_KEY`,
`OPENAI_API_KEY`, `GEMINI_API_KEY`, `PERPLEXITY_API_KEY`, `DEEPSEEK_API_KEY`,
`MOONSHOT_API_KEY`/`KIMI_API_KEY`, `OLLAMA_BASE_URL`+`OLLAMA_MODEL`.

**Search/scrape:** `TAVILY_API_KEY`, `SEARXNG_URL`, `SCRAPEGRAPH_API_KEY`,
`CRAWL4AI_API_KEY`, `CAMOUFOX_ENABLED`, `THEHARVESTER_BIN`, `CHROMIUM_EXECUTABLE_PATH`.

**Contact/data (optional):** `APOLLO_API_KEY`+`APOLLO_CLIENT_SECRET`+`APOLLO_ACCESS_TOKEN`,
`EXPLORIUM_API_KEY`, `HUNTER_API_KEY`, `WAPPALYZER_API_KEY`.

**Cost/feature flags:** `NEXUS_PREFER_FREE_MODELS` (default true), `DISABLE_PERPLEXITY`,
`PERPLEXITY_JOB_BUDGET`.

**Infra/security:** `API_TOKEN`, `FRONTEND_ORIGIN`, `PORT`, `NODE_ENV`,
`SELF_URL` (for the `lead_factory_run` tool), `ACTIVEPIECES_*` (optional automation).

## How to stand it up

```bash
# 1. Postgres + migrations
docker compose up -d db && pnpm --filter @workspace/db run push --force

# 2. Python Scout sidecar (signals/OSINT/scrape) on :8099
docker compose up -d scout

# 3. API server (Express) on :3000  — reads DATABASE_URL, SCOUT_URL, LLM keys
pnpm --filter @workspace/api-server run dev

# 4. Frontend (Vite) on :5173
pnpm --filter @workspace/prospect-sa run dev

# health
curl localhost:3000/api/healthz && curl localhost:3000/api/readyz
```

Each subsystem's doc ends with a **"Minimum build to replicate"** checklist — start
there if you're rebuilding one tool at a time.

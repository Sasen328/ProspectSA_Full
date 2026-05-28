# Build-Spec 01 — AI Chat / Composer

> Full-stack replication spec for the AI Chat orchestrator, the 6-stage Composer
> wizard, the Customize drawer, and the Behavior Agent (AI observer). Everything
> below is grounded in the live codebase — file paths + line numbers are real.

---

## 0. What this subsystem is

A conversational research cockpit. The user builds a structured research brief
through a wizard, the brief is enhanced into a prompt, an **agent tool-loop**
(Claude + 9 tools) executes it with live SSE streaming, the raw answer is parsed
into typed report blocks, and the result can be exported or pushed to Lead Genome.
A floating **Behavior Agent** watches the session and suggests next actions.

```
Composer wizard → /enhance → Clarify → /ai-chat/stream (tool loop) → render-blocks → Report → Enrich/Export
                                                          ▲
                                          Behavior Agent (observer + plug-actions)
```

---

## 1. Frontend

| File | Role |
|---|---|
| `artifacts/prospect-sa/src/pages/ai-chat/index.tsx` (1–267) | Page shell; plain-chat vs composer toggle; SSE parsing |
| `components/composer/ChatLayout.tsx` (1–260) | 6-stage orchestration wrapper |
| `components/composer/Composer.tsx` (1–353) | 4-layer wizard (Pick / Scope / Tools / Ask) |
| `components/composer/ClarifyView.tsx` (1–123) | Report-shape picker + clarifying chips |
| `components/composer/ReportView.tsx` (1–127) | Renders typed `ReportBlock[]` + export buttons |
| `components/composer/EnrichView.tsx` | Stage 6 — filter/save results |
| `components/composer/CustomizeDrawer.tsx` (1–335) | 6-tab CRUD drawer |
| `components/composer/BehaviorAgent.tsx` (1–116) | Floating AI observer |

### 6 stages (`ChatLayout`)
`compose → enhance → clarify → run → report → enrich`. Stages unlock as reached;
a "mega-mind" banner shows the live route summary (target · countries · industry ·
modes · report shape). History bar loads recent runs via `GET /api/composer/runs?limit=3`.

### `ComposerState` (the brief)
```ts
interface ComposerState {
  templateId?: string;
  modes: string[];                 // e.g. ["leadgen","enrich"]
  target: "person" | "company" | "both";
  countries: string[];             // ISO-2
  industry: string;
  listing: string;                 // "Any", "Tadawul main", ...
  subFilters: Record<string,string>;
  askFilters: Record<string,string>;
  sources: string[];               // source IDs
  connectors: string[];
  skills: string[];
  freeText: string;                // the question
  enhance: boolean;                // run prompt enhancer?
}
```

### Customize drawer — 6 tabs
`skills` · `templates` · `sources` · `connectors` · `plugins` · `modes`.
Skills/Templates/Sources are full CRUD (DB-backed). Connectors show live OAuth
status. Plugins toggle the 9 agent tools per session. Modes are read-only builtins.

### Behavior Agent (AI observer)
Floating bottom-right panel; foldable/closeable. Keeps a **10-event ring buffer**
(`compose`/`nav`/`filter`/`run`/`save`). Produces suggestions two ways:
1. **Deterministic rules** (instant, zero-cost) — e.g. `target==="both"` → suggest both hunts; `Arabic-first` chip → "prefer Arabic sources" hint.
2. **LLM fallback** — `POST /api/behavior/suggest` returns `{suggestion, oneLineHint, plugActions[]}`. Plug-actions are ready-to-fire buttons (`{label, endpoint, body}`), max 3.

---

## 2. API endpoints

### AI Chat stream — `routes/ai-chat.ts` (1–93)
`POST /api/ai-chat/stream` → **SSE**.
Request: `{ message, history?: {role,content}[], system? }` (history capped at 10).
SSE events:
```
{event:"agent_start", data:{agent,description}}
{event:"agent_done",  data:{agent,found,summary?}}
{event:"token",       data:"<text chunk>"}      // data is a raw string
{event:"intent",      data:{plan}}
{event:"final",       data:{text}}
{event:"error",       data:{message}}
```
**Plan B:** if `ANTHROPIC_API_KEY` missing, falls back to Nexus (Kimi planner) and
emits a `🧠 Planner … Nexus fallback` agent_start.

### Composer CRUD — `routes/composer.ts` (1–366)
- `GET /modes` · `GET /sources` (`?reco=1&industry=&countries=&listing=&target=`) · `GET /connectors` · `GET /skills` · `GET /templates`
- Skills: `POST/PATCH/DELETE /skills[/:id]`
- Templates: `POST/PATCH/DELETE /templates[/:id]`
- User sources: `GET/POST /user-sources`, `DELETE /user-sources/:id`
- Connectors OAuth: `GET /connectors/:id/auth-url`, `GET /connectors/:id/callback`, `DELETE /connectors/:id`
- Runs: `POST/GET /runs`, `GET/PATCH /runs/:id`, `GET /runs/:id/export?format=`
- `POST /render-blocks` `{rawText, shape}` → `{blocks}`
- `POST /export` `{blocks, rawText, shape, format, title}` → binary (xlsx/pdf/html/jsx/pptx/csv)
- `POST /enhance` (body = ComposerState) → `{enhancedPrompt, ...}`
- Plugins: `GET /plugins?sessionId=`, `POST /plugins {sessionId, enabled[]}`

### Behavior — `routes/behavior.ts` (1–106)
- `POST /api/behavior/event` `{sessionId, kind, payload}` → `{ok}` (fire-and-forget, persists to `behavior_events`)
- `POST /api/behavior/suggest` `{sessionId, state, history[]}` → `{suggestion, oneLineHint, plugActions[]}`

---

## 3. Orchestrator (the tool loop)

`lib/agents/orchestrator.ts` (1–169) — `runAgentChat(userMessage, history, onEvent, opts)`.
- `maxIterations` default **8** tool rounds.
- `model` default **`claude-sonnet-4-6`** (override via `AI_CHAT_ORCHESTRATOR_MODEL`).
- Loop: call Claude with tools → if `stop_reason==="tool_use"` run each tool, emit
  `agent_start`/`agent_done`, append `tool_result` (truncated to 12k chars), repeat;
  else stream tokens and finish.
- Friendly emoji labels per tool keep the SSE privacy-safe.

### System prompt (strict contract)
Honor every constraint; cite every fact with a source URL; never invent
contact data (leave blank + flag); halt-and-ask on conflict; prefer Arabic
sources for Tadawul companies; output export-ready Markdown.

---

## 4. The 9 agent tools — `lib/agents/tools.ts` (1–289)

| Tool | Input | Backend |
|---|---|---|
| `nexus_run` | `{role, task}` | `nexusRunRole(role, task)` |
| `web_search` | `{query, limit?=8}` | `freeWebSearch` (Tavily→SearXNG→Google) |
| `url_crawl` | `{url}` | axios+Cheerio static (12s, 8k chars) |
| `deep_scrape` | `{url}` | `scrapeWithPlaywright` (stealth, 12k chars) |
| `harvester_run` | `{query, limit?=12}` | `lib/harvester` generator (GLEIF/OpenCorporates/Wikidata/news/sanctions/OSINT) |
| `sanctions_screen` | `{name}` | `screenName` (OFAC/UN/EU/OpenSanctions) |
| `scout_osint` | `{query}` | `POST {SCOUT_URL}/osint` (Sherlock, 30s) |
| `lead_factory_run` | `{industry, country?, targetCount?=25}` | `POST {SELF_URL}/api/lead-factory/start` |
| `signal_monitor` | `{company}` | `harvester_run` w/ "hiring OR funding OR leadership" |

`toAnthropicTools()` strips handlers for the Claude API; `getToolByName()` resolves handlers.

---

## 5. Backend engine — Nexus LLM router

`lib/nexus/llm-router.ts` (1–723) + `lib/nexus/roles.ts`.

**Tiers:** `extraction · arabic · synthesis · bulk · realtime`.
**Roles → tier:** planner→synthesis, researcher→realtime, extractor→extraction,
arabic→arabic, writer→synthesis, validator→bulk, bulk→bulk, signal→extraction, tree→extraction.

`nexusGenerate(prompt, {tier, forceModel?, forceProvider?, ...})` builds an ordered
**attempt chain** per tier and falls through on timeout/failure. `nexusRunRole(role, task, opts)`
is the role abstraction used by the tools. Cost tracked per model via `COST_PER_MTOK`;
`NEXUS_PREFER_FREE_MODELS=true` prepends OpenRouter `:free` variants.

Provider clients: OpenRouter, Groq, Ollama (`OLLAMA_BASE_URL`), DeepSeek native,
Kimi/Moonshot, Perplexity, Gemini, OpenAI, Anthropic.

---

## 6. Database tables

`lib/db/src/schema/composer.ts`, `behavior_events.ts`, `conversations.ts`, `messages.ts`.

- **composer_skills** — `id, builtin_id, name, description, system_prompt, tool_whitelist text[], report_schema, model_tier, visibility, enabled, created_at, updated_at`
- **composer_templates** — `id, builtin_id, name, description, default_question, default_modes text[], default_target, default_countries text[], default_industry, default_sources text[], default_skills text[], required_schema, visibility, timestamps`
- **composer_user_sources** — `id, label, url, category, language, countries text[], industries text[], enabled, created_at`
- **composer_runs** — `id, state jsonb, enhanced_prompt, report_shape, blocks jsonb, raw_text, status, error_message, created_at, completed_at`
- **behavior_events** — `id, session_id, kind, payload jsonb, created_at`
- **conversations** — `id, title, created_at`
- **messages** — `id, conversation_id (fk cascade), role, content, created_at`

---

## 7. Builtin registries — `lib/composer/`

- **sources.ts** — 80+ sources in 6 categories (ksa-market, arabic-press, regional-press, global-open, sector, social) + a recommendation algorithm (max 8).
- **modes.ts** — 9 modes (leadgen, enrich, research, extract, compare, deepdive, signal, tree, custom) each with `promptSuffix`, `defaultTools`, `reportSchema`.
- **templates.ts** — 6 quick-start templates.
- **skills.ts** — 13 pre-baked skills (finance/real-estate/energy/people/compliance).
- **connectors.ts** — 28 connectors (scraping ready; productivity/files/CRM/messaging require OAuth or key).

---

## 8. Dependencies & env vars (to make it real)

**Required to boot:** `ANTHROPIC_API_KEY` (orchestrator + synthesis).

**LLM providers (unlock tiers/fallback):** `OPENAI_API_KEY`, `GEMINI_API_KEY`,
`OPENROUTER_API_KEY`, `GROQ_API_KEY`, `PERPLEXITY_API_KEY`, `DEEPSEEK_API_KEY`,
`MOONSHOT_API_KEY`/`KIMI_API_KEY`, `OLLAMA_BASE_URL`+`OLLAMA_MODEL`.

**Search/web:** `TAVILY_API_KEY`, `SEARXNG_URL`, `SEARXNG_INSTANCES`.

**Sidecar:** `SCOUT_URL` (Python Scout, default `http://localhost:8099`) for `scout_osint`.

**Self-call:** `SELF_URL` (default `http://localhost:3000`) for `lead_factory_run`.

**Feature flags:** `NEXUS_PREFER_FREE_MODELS` (default true), `DISABLE_PERPLEXITY`.

**Infra:** `DATABASE_URL`, `API_TOKEN`, `FRONTEND_ORIGIN`.

---

## 9. Minimum build to replicate

1. Postgres + the 7 tables above.
2. Express server exposing `/api/ai-chat/stream` (SSE) + `/api/composer/*` + `/api/behavior/*`.
3. The orchestrator loop with the 9 tool handlers (stub `scout_osint`/`lead_factory_run` if the sidecar/self engines aren't ready).
4. Nexus router with at least one provider key (Anthropic) — others optional fallbacks.
5. React Composer wizard + ChatLayout + ReportView consuming the SSE event shapes in §2.
6. Behavior Agent posting `/behavior/event` and rendering `/behavior/suggest` plug-actions.

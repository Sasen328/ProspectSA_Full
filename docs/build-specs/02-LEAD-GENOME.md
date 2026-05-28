# Build-Spec 02 — Lead Genome

> Full-stack replication spec for Lead Genome: the saved-lead bucket + AI-scored
> list builder. Grounded in the live codebase (file paths + line numbers real).

---

## 0. What this subsystem is

Lead Genome is the **destination bucket + segmentation engine** for saved leads.
Other engines (Lead Factory, ProsEngine, AI Chat, Harvest AI) push leads into it
via one endpoint. It then provides: a searchable bucket, per-source stats, an
AI-scored **list builder** (10-step ICP wizard), and one-click **deep enrichment**
(re-runs the Lead Factory pipeline on a saved lead).

```
Engines → /lead-genome/save → bucket → list wizard (/api/lead-lists) → AI hunt (fetch+preScore+aiScore) → list items → export
                                                                              └ enrich/:leadId → Lead Factory job
```

---

## 1. Frontend

| File | Role |
|---|---|
| `pages/leads/index.tsx` (1–1278) | Main page: "AI Lead Hunts" + "ProsEngine Research" tabs; 10-step wizard; list detail |
| `components/LeadGenomePanel.tsx` (1–105) | Embedded saved-bucket widget |
| `components/SaveToLeadGenome.tsx` (1–60) | Drop-in push button for any engine page |
| `lib/lead-genome-client.ts` (1–131) | API client |

### 10-step hunt wizard (`WizardModal`, 187–505)
1 name · 2 industries · 3 cities (28) · 4 revenue range · 5 employee min/max ·
6 person types (executive/owner/shareholder/board_member/management) · 7 compensation ·
8 required person fields (phone|email|linkedin — any) + company fields (revenue/employees/CR — each mandatory) ·
9 data sources (orcbase/masaar/builder/sa_market) · 10 max leads (10–500) + notes.
Submit → `POST /api/lead-lists` → background hunt.

### Key components
- **LeadCard** (617–717): avatar, name/title/company, type+source badges, score bar.
- **LeadProfileDialog** (508–614): full detail; "Generate Intel Profile" → `/prospecting/person`; "Remove".
- **LeadGenomePanel**: queries `GET /api/lead-genome/hunt`, source dropdown (9 sources), empty-state prompt.
- **SaveToLeadGenome**: `<SaveToLeadGenome source="lead-factory" lead={{firstName,lastName,title,email}} />` → `POST /api/lead-genome/save`.

---

## 2. API endpoints

### Lead Genome — `routes/lead-genome.ts` (1–224)
- `POST /save` — insert one lead; prepends `[from:{source}]` to notes; `status="new"`.
- `POST /hunt` `{q?, title?, department?, seniority?, source?, limit?<=500}` → `{leads, count}` (q→ILIKE on name/email; source→notes ILIKE).
- `GET /stats` → `{total, bySource{...}, listCount}`.
- `POST /lists` `{name, criteria?, sourcesSearched?}` (simple create).
- `GET /lists` · `GET /lists/:id` (list+items).
- `POST /lists/:id/items` `{leadIds[]}` — bulk add saved leads to a list.
- `POST /enrich/:leadId` → creates a Lead Factory **person**-mode job → `{jobId}`; stream at `/api/lead-factory/stream/:jobId`.

### Lead Lists (full AI hunt) — `routes/lead-lists.ts` (1–976)
- `POST /api/lead-lists` (696–777) — wizard target. Body = `LeadCriteria` (name, industries[], cities[], revenueRange, employeeMin/Max, personTypes[], compensationRange, requiredPersonFields[], requiredCompanyFields[], sources[], maxLeads, freeText?). Returns `{id, status:"running", name}`; runs hunt in `setImmediate`.
- `POST /:id/retry` (780–845) — clear items, re-run.
- `GET /` · `GET /:id` · `GET /:id/items` · `DELETE /:id/items/:itemId` · `DELETE /:id`.
- `GET /stats/all` (854–871) → per-source counts.
- `GET /:id/export?format=csv|xlsx|json` (907–973) — 24-column export.

### Leads (supporting) — `routes/leads.ts`
- `GET /api/leads` (paginated, search/filter) · `POST /api/leads` (gate-validated) · `POST /push-from-company/:companyId` (bulk push execs) · `PATCH/DELETE /:id`.

---

## 3. Hunt pipeline (the engine)

`POST /api/lead-lists` background flow:
1. **fetchPeople(criteria)** — parallel query of 6 sources: OrcBase (executives⨝companies), Masaar (masar_companies JSON arrays), Builder (builder_companies JSON), SA Market (Tadawul/NOMU shareholders+execs), ProsEngine (prosengine_research). Deduped by `name|company`, top `2×maxLeads`.
2. **preScore** (73–142) — rule-based 1–100: base 30; industry ±20/−5; city ±15/−3; employee +10; revenue +8; compensation +12; contact +5 each; required fields = hard reject (structured) or −15 (CR-sourced).
3. Dedup by `name|company` (keep highest).
4. **aiScoreBatch** (177–218) — batches of 10; Claude Sonnet 4.5 primary → GPT-4o-mini fallback → preScore. Returns `{aiScore, aiReasoning}`.
5. Merge, sort by `aiScore ?? matchScore` desc, insert into `lead_list_items`, set list `status="done", totalFound=N`.

---

## 4. Validation gate (manual pushes)

`lib/lead-gate.ts` + `lib/lead-validator.ts`: `insertLeadWithGate()` returns
`{status:"pass"|"warn"|"reject", reasons[], isDuplicate, confidence, fingerprint}`.
- Format checks (phone/email/CR), fingerprint, dedup vs `lead_fingerprints` (exact domain/CR/phone/email; fuzzy name ≥0.88).
- Active verify: DNS/MX lookup (4s, 1h cache), domain liveness (HEAD, cached), dummy detection, cross-source corroboration (TRUSTED_SOURCES). Confidence 0–100; dummy → reject; <35 → warn.

---

## 5. Database tables

`lib/db/src/schema/leads.ts`, `lead_lists.ts`.

- **leads** — `id, company_id, first_name, last_name, first_name_ar, last_name_ar, title, title_ar, email, phone, linkedin_url, twitter_url, department, seniority, notes (tagged [from:SOURCE]), status (default new), created_at, updated_at`.
- **lead_lists** — `id, name, criteria (JSON), status (pending|running|done|failed), total_found, sources_searched (JSON), timestamps`.
- **lead_list_items** — person fields (`person_name(_ar)`, `person_title(_ar)`, `person_type`, `seniority`, `department`, `nationality`, `linkedin`, `estimated_salary`, `biography`) + company context (`company_name(_ar)`, `industry`, `city`, `company_revenue`, `company_employees`, `cr_number`, `ownership_pct`) + contact (`phone`, `email`, `website`) + scoring (`source`, `source_id`, `match_score`, `ai_score`, `ai_reasoning`), `created_at`.
- **lead_fingerprints** — `normalized_name, domain, phone_normalized, email_normalized, cr_number, source_table, source_id, created_at`.

Source tables read by the hunt: `companies`, `executives`, `masar_companies`, `builder_companies`, `sa_market_*`, `prosengine_research`.

---

## 6. Dependencies & env vars

**Required:** `ANTHROPIC_API_KEY` (AI scoring), `DATABASE_URL`. `OPENAI_API_KEY` (fallback scoring).

**Enrich path:** the Lead Factory engine + its deps (Scout sidecar `SCOUT_URL`, Nexus providers) — see Build-Spec 03.

**Optional:** `GEMINI_API_KEY`, `OPENROUTER_API_KEY`, `GROQ_API_KEY`, `PERPLEXITY_API_KEY`; contact APIs `APOLLO_*`, `EXPLORIUM_API_KEY`, `HUNTER_API_KEY`; `TAVILY_API_KEY`.

---

## 7. Minimum build to replicate

1. Postgres + `leads`, `lead_lists`, `lead_list_items`, `lead_fingerprints` (+ the source tables the hunt reads, or stub `fetchPeople` to one source).
2. Express routes `/api/lead-genome/*` + `/api/lead-lists/*`.
3. The hunt pipeline (fetch → preScore → aiScore → persist). AI scoring needs one LLM key; degrade to preScore-only without it.
4. The gate (`insertLeadWithGate`) for manual `POST /api/leads`.
5. React `/leads` page (wizard + list detail) + `LeadGenomePanel` + `SaveToLeadGenome` button on every engine page.
6. `POST /lead-genome/enrich/:leadId` wired to the Lead Factory job runner.

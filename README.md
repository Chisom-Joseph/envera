<div align="center">

# ENVERA

### Cross-Agency Financial Regulatory Enforcement Intelligence

**One MCP endpoint. Six US federal regulators. One normalized schema.**

`SEC`  ·  `CFPB`  ·  `FTC`  ·  `FINRA`  ·  `FinCEN`  ·  `OCC`

[![Node](https://img.shields.io/badge/node-%E2%89%A520-3C873A?logo=node.js&logoColor=white)](https://nodejs.org/)
[![TypeScript](https://img.shields.io/badge/typescript-5.x-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![MCP](https://img.shields.io/badge/MCP-Streamable%20HTTP-FF6B35)](https://modelcontextprotocol.io/)
[![Postgres](https://img.shields.io/badge/postgres-14%2B-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Railway](https://img.shields.io/badge/deploys%20on-Railway-0B0D0E?logo=railway&logoColor=white)](https://railway.app/)
[![License](https://img.shields.io/badge/license-MIT-22C55E)](#license)

</div>

---

ENVERA replaces the fragile manual workflow compliance teams rely on today — six agency websites, ten-plus RSS feeds, twenty-plus spreadsheet tabs — and the enterprise tools that monetize that pain: **Bloomberg Law (~$5k/yr)**, **Intelligize (~$10k/yr)**, **Thomson Reuters RI (~$8k/yr)**.

It ships as a [Context Protocol](https://ctxprotocol.com) MCP server with two billing modes:

| Mode | Surface | Price | Use case |
|:-----|:--------|:------|:---------|
| **Query** | Context app | `$0.10` / response | Curated enforcement intelligence — entity resolution, risk summaries, narrative memos. |
| **Execute** | SDK | `$0.001` / call | Normalized typed enforcement records for programmatic agent workflows. |

---

## Contents

**Get started**  ·  [Quickstart](#quickstart)  ·  [Environment](#environment-variables)  ·  [Deploy](#deploy-to-railway)
**Reference**  ·  [MCP Tools](#mcp-tools)  ·  [Architecture](#architecture)  ·  [Ingestion](#ingestion-pipeline)  ·  [Entity Resolution](#entity-resolution)
**Meta**  ·  [Product Contract](#product-contract)  ·  [Project Layout](#project-layout)  ·  [Error Contract](#error-contract)  ·  [License](#license)

---

## Quickstart

> **Prereqs** — Node 20+, PostgreSQL 14+, (optional) Redis.

```bash
# 1. Clone & configure
cp .env.example .env         # edit DATABASE_URL, DATA_GOV_API_KEY, OCC_API_KEY, ADMIN_INGEST_TOKEN

# 2. Install & migrate
npm install
npm run db:migrate:dev       # applies schema + pg_trgm extension

# 3. Seed data (one-shot ingestion from all six agencies)
npm run ingest all

# 4. Run
npm run dev                  # MCP server on $PORT (default 3000)
```

<details>
<summary><strong>PowerShell equivalent (Windows)</strong></summary>

```powershell
Copy-Item .env.example .env
npm install
npm run db:migrate:dev
npm run ingest all
npm run dev
```

</details>

<details>
<summary><strong>Docker</strong></summary>

A production-minded [`Dockerfile`](Dockerfile) is included — build and run against your own Postgres:

```bash
docker build -t envera .
docker run --rm -p 3000:3000 --env-file .env envera
```

</details>

### Verify the MCP surface

```bash
curl -s http://localhost:3000/health | jq

curl -s -X POST http://localhost:3000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"1"}}}'
```

> `tools/list` is unauthenticated (MCP discovery). `tools/call` requires a Context JWT when `CONTEXT_AUTH_ENABLED=true`.

### npm scripts

| Script | Purpose |
|:-------|:--------|
| `npm run dev`             | Start the MCP server with hot reload (`tsx watch`). |
| `npm run build`           | TypeScript build → `dist/`. |
| `npm start`               | Run the compiled server from `dist/`. |
| `npm run ingest <agency>` | One-shot ingest — `all` or one of `SEC` · `CFPB` · `FTC` · `FINRA` · `FINCEN` · `OCC`. |
| `npm run ingest:prod`     | Same, against the built output. |
| `npm run db:migrate`      | Run DB migrations (compiled). |
| `npm run db:migrate:dev`  | Run DB migrations directly from TypeScript (`tsx`). |
| `npm run typecheck`       | `tsc --noEmit`. |
| `npm test`                | Node test runner over `tests/**/*.test.ts`. |

---

## Environment Variables

See [`.env.example`](.env.example) for the full annotated list.

| Variable | Default | Notes |
|:---------|:--------|:------|
| `PORT` | `3000` | HTTP port. |
| `CONTEXT_AUTH_ENABLED` | `false` locally | **Must be `true` in production** — paid tool calls are JWT-verified. |
| `DATABASE_URL` | _required_ | Any Postgres connection string. Railway managed Postgres works out of the box. |
| `DATABASE_SSL_STRICT` | `false` | Set to `true` only if your provider mandates strict cert verification. |
| `REDIS_URL` | _(empty)_ | Optional. In-memory LRU is used when empty or unset. |
| `REDIS_TTL_SECONDS` | `900` | 15-minute hot cache. |
| `INGEST_ON_STARTUP` | `false` | Set to `true` for first-boot bootstrapping only. |
| `INGEST_CRON` | `7 */6 * * *` | Every 6 hours at minute 7. |
| `INGEST_DISABLE` | _(empty)_ | CSV list of agency codes to skip (e.g. `FTC` if you lack a Data.gov key). |
| `DATA_GOV_API_KEY` | _(empty)_ | Required for the FTC API. Adapter falls back to the press-release RSS feed if empty. |
| `OCC_API_KEY` | _(empty)_ | OCC enforcement-actions feed key. |
| `ADMIN_INGEST_TOKEN` | _(empty)_ | Shared secret for `POST /admin/ingest`. Empty disables the endpoint. |
| `LOG_LEVEL` | `info` | `debug` · `info` · `warn` · `error`. |

---

## Deploy to Railway

1. Create a Railway project and attach **PostgreSQL** (optionally **Redis**) services.
2. Link this repo — Railway auto-detects the [`Dockerfile`](Dockerfile), falling back to Nixpacks via [`railway.json`](railway.json).
3. Set the production environment — at minimum:
   - `CONTEXT_AUTH_ENABLED=true`
   - `INGEST_ON_STARTUP=true` **(first deploy only — toggle off afterwards)**
   - `DATA_GOV_API_KEY` — free signup at [api.data.gov](https://api.data.gov/signup/)
   - `OCC_API_KEY`
   - `ADMIN_INGEST_TOKEN` — random 32+ character string
4. Deploy. The `start` command runs `db:migrate && server`, so the schema is idempotently applied on every boot.
5. Grab the Railway public URL — your MCP endpoint is `https://<railway-url>/mcp`.
6. Register at [ctxprotocol.com/contribute](https://ctxprotocol.com/contribute), stake USDC, then follow Steps 5–6 of the grants doc (optimization skill + review email to `grants@ctxprotocol.com`).

See [`docs/deployment.md`](docs/deployment.md) for the full runbook.

---

## MCP Tools

### Query mode — `$0.10` per response

| Tool | Purpose |
|:-----|:--------|
| `search_enforcement_by_entity` | Full cross-agency enforcement history for a company or individual, with risk summary and entity-match confidence. |
| `get_enforcement_risk_profile` | Standalone regulatory-risk profile — total penalty exposure, agency breadth, time-series timeline, plain-English narrative. |
| `search_enforcement_by_topic`  | Enforcement actions matching a topic/violation keyword (e.g. "AML KYC", "mortgage servicing") with peer-enforcement stats. |

### Execute mode — `$0.001` per call

| Method | Purpose |
|:-------|:--------|
| `list_enforcement_actions` | Paginated, filterable typed primitive — agency, entity, dates, action-type, status, free-text. |
| `get_enforcement_action`   | Fetch one canonical action by `actionId`. |
| `resolve_entity`           | Fuzzy-match an entity string; returns canonical respondents with confidence buckets. |
| `get_agency_coverage`      | Coverage metadata — earliest/latest action, counts, last ingestion — per agency. |

Every tool returns:

- `outputSchema`-validated `structuredContent`
- Freshness metadata — `generatedAt`, `sourceUpdatedAt`, `dataFreshness`
- Per-response `entityMatchConfidence` so agents know when a match is fuzzy

---

## Architecture

```
 ┌───────────────────────────────────────────────────────────┐
 │ Agency Adapters (SEC, CFPB, FTC, FINRA, FinCEN, OCC)      │
 │ RSS · Atom · Socrata · Data.gov · HTML (Cheerio)          │
 └──────────────────────────┬────────────────────────────────┘
                            │  NormalizedAction[]
            ┌───────────────▼─────────────────┐
            │ Normalization engine            │
            │  • action-type taxonomy         │
            │  • entity resolution (fuzzy)    │
            │  • penalty parsing / USD        │
            │  • date + status canonicalize   │
            │  • field-provenance tagging     │
            └───────────────┬─────────────────┘
                            │  upsertAction()
            ┌───────────────▼─────────────────┐
            │ PostgreSQL (pg_trgm)            │
            │  + generated tsvector           │
            │  + agency / entity / date idx   │
            └───────────────┬─────────────────┘
                            │
            ┌───────────────▼─────────────────┐
            │ Redis (15-min TTL) — optional   │
            └───────────────┬─────────────────┘
                            │
            ┌───────────────▼─────────────────┐
            │ MCP tools (Query + Execute)     │
            │ Express · StreamableHTTP · JWT  │
            └─────────────────────────────────┘
```

### Stack

| Layer | Tech |
|:------|:-----|
| Runtime | **Node 20** · **TypeScript 5** (strict) |
| HTTP | **Express 4** — `/mcp` · `/health` · `/admin/ingest` |
| MCP | [`@modelcontextprotocol/sdk`](https://www.npmjs.com/package/@modelcontextprotocol/sdk) server + StreamableHTTP transport |
| Auth | [`@ctxprotocol/sdk`](https://www.npmjs.com/package/@ctxprotocol/sdk) — `createContextMiddleware` JWT verification |
| Storage | **PostgreSQL 14+** with `pg_trgm` trigram similarity + full-text search |
| Cache | **Redis** (optional) — falls back to in-memory LRU |
| Scheduler | `node-cron` |
| Parsing | `cheerio` · `fast-xml-parser` · Socrata / Data.gov REST |
| Logging | `pino` — structured, JSON |

---

## Ingestion Pipeline

```bash
npm run ingest all         # all six agencies
npm run ingest SEC         # a single agency
```

Scheduled ingestion runs on `INGEST_CRON` (default: every 6 hours at minute 7). Each run:

1. Fetches source bytes (RSS / Atom / JSON / HTML) with a short timeout.
2. Normalizes every action through the shared engine.
3. Upserts by `action_id`; only changed rows are written (`ON CONFLICT DO UPDATE WHERE`).
4. Writes a row to `ingestion_runs` — `duration`, `actionsNew`, `actionsUpdated`, `status`.

### Manual trigger

For when a source redesigns its page and you need to re-pull immediately:

```bash
curl -X POST "$URL/admin/ingest" \
  -H "x-admin-token: $ADMIN_INGEST_TOKEN" \
  -H 'content-type: application/json' \
  -d '{"agency":"FINRA"}'
```

### HTML-scraper adapters — `FINRA` · `OCC` · `FinCEN`

These three are the highest-risk adapters because they are pure HTML scrapers. Each documents its selectors inline with `// SELECTOR:` comments and degrades gracefully:

- Missing fields are tagged `fieldOrigin: "unknown"` rather than crashing.
- Each run records `actionsNew`, `actionsUpdated`, `errors[]`; a **zero-result run emits a WARN log** so operators can alert on page redesigns.
- `/admin/ingest` re-runs a single agency after a selector patch without waiting for cron.

---

## Entity Resolution

Matching blends three signals:

- **Postgres `pg_trgm` similarity** on the `entity_key` column — a punctuation-stripped, suffix-stripped normal form of the respondent name.
- **Jaccard token similarity** layered with **Levenshtein distance** on top candidates.
- Optional exact-key early-out for high-confidence matches.

| Score | Bucket | Meaning |
|:-----:|:-------|:--------|
| **≥ 0.9** | `high`   | Near-exact match. |
| **≥ 0.7** | `medium` | Likely same entity, minor name variation. |
| **< 0.7** | `low`    | Fuzzy — display but flag in the UI. |

---

## Product Contract

| | |
|:---|:---|
| **Target buyer** | AML/BSA compliance officers (banks + fintechs), fintech regulatory lawyers, VC/PE/M&A due-diligence teams, compliance consultants, bank examiners. |
| **Premium feature** | Unified cross-agency enforcement history for any company or individual in a single call. |
| **Painful substitute** | 6 agency websites + 10+ RSS feeds + 20+ spreadsheet tabs (per r/AMLCompliance). |
| **Paid substitute** | Bloomberg Law ($5k+/yr) · Intelligize ($10k+/yr) · Thomson Reuters RI ($8k+/yr). |
| **Freshness target** | Sub-3s cached · sub-10s uncached · sources ingested every 6h. |
| **Latency target** | Under 60s (hard Context requirement) — typically **<500ms**. |

### Must-win prompts

See [`docs/must-win-prompts.md`](docs/must-win-prompts.md).

1. *"Does Wise have any regulatory enforcement history across US financial regulators?"*
2. *"Give me a regulatory risk profile on Robinhood across the SEC, CFPB, and FINRA."*
3. *"Which banks have been penalized by the OCC or CFPB for AML/KYC failures in the last three years?"*
4. *"List every CFPB consent order against payments companies since 2022."*
5. *"Compare BSA/AML enforcement activity at FinCEN vs the OCC in the past 5 years."*

### Evidence on every record

- `actionId` · `agency` · `actionType` (canonical) · `rawActionType` (source phrasing)
- `status` (canonical) · `rawStatus`
- `actionDate` (ISO 8601) · `respondent` · `respondents[]`
- `penaltyAmount` (USD) · `penaltyBreakdown[]` — `civil_penalty` / `disgorgement` / `restitution` / `prejudgment_interest`
- `title` · `summary` · `allegations`
- `documentUrl` · `entityMatchConfidence` · `entityMatchScore`
- `provenance` = `{ source, fetchedAt, fieldOrigin }` — every field tagged `observed` / `normalized` / `inferred` / `unknown`

---

## Project Layout

```
src/
├── adapters/           # one module per agency
│   ├── sec.ts
│   ├── cfpb.ts
│   ├── ftc.ts
│   ├── finra.ts
│   ├── fincen.ts
│   └── occ.ts
├── cache/
│   └── redis.ts        # Redis or in-memory LRU backend
├── db/
│   ├── client.ts
│   ├── migrate.ts
│   └── repository.ts
├── ingest/
│   ├── runner.ts
│   ├── scheduler.ts
│   └── cli.ts
├── mcp/
│   ├── tools.ts        # inputSchema + outputSchema + _meta
│   └── handlers.ts     # business logic + dispatchTool()
├── normalization/
│   ├── actionTypes.ts
│   ├── dates.ts
│   ├── entities.ts
│   ├── penalties.ts
│   └── status.ts
├── config.ts
├── logger.ts
├── types.ts
└── server.ts           # Express + MCP transport + /health + /admin

db/schema.sql           # canonical schema (applied by db/migrate.ts)
docs/                   # deployment, grants email, marketplace copy, prompts
scripts/                # smoke tests (e.g. smoke-adapters.mjs)
```

---

## Error Contract

Every tool handler returns one of:

```ts
// success
{ ok: true,  data: <structuredContent> }
// wrapped as → { content: [text], structuredContent: data }

// failure
{ ok: false, error: { code, message, field? } }
// wrapped as → { content: [...], isError: true }
```

Error codes: `INVALID_INPUT` · `NOT_FOUND` · `UPSTREAM_UNAVAILABLE` · `INTERNAL`.

No crashes — every adapter failure is caught, logged, and surfaced as a structured error.

---

## License

MIT — see [`LICENSE`](LICENSE).

<div align="center">

— Built for compliance teams who are tired of scraping RSS feeds at 2 AM. —

</div>

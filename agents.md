# AGENT.md — Hormuz Transit Monitor

This file is the primary context document for any AI agent working on this project.
Read it fully before making any changes.

---

## What This Project Is

**Hormuz Transit Monitor** is a public real-time intelligence dashboard that tracks:
- Vessel movements through the Strait of Hormuz via AIS data
- Daily transit counts and throughput estimates (oil + LNG)
- Brent crude, WTI, and natural gas prices (FRED API)
- Major carrier disruption statuses
- Maritime incidents (attacks, seizures, GPS jamming)
- AIS feed health and data confidence indicators

**Audience:** Crisis monitors, energy-market analysts, investors, media.

**This is NOT a navigation tool.** Every decision about labeling, wording, and UI must reflect this.

---

## Repository

- **GitHub:** `kenyob/hormuz-shipping-tracker` (private)
- **Development branch:** `claude/hormuz-shipping-dashboard-mOgLl`
- **Default branch:** `main`
- **Working directory:** repo root (all code lives directly here, no nested monorepo wrapper)

---

## Non-Negotiable Safety Rules

These constraints are hardcoded into the product specification and must never be violated:

1. **Never label any corridor, route, or zone as "safe."**
   Use instead: `"Lower reported incident corridor"`, `"High traffic / low reported incident corridor"`, `"Recent incident proximity"`, `"AIS coverage degraded"`, `"Data insufficient"`

2. **Never claim AIS data is complete.** ~50% of actual Hormuz traffic is invisible due to deliberate blackouts, jamming, and spoofing. The UI must surface this prominently everywhere AIS data is shown.

3. **Never label a vessel as military** unless the classification is source-verified from official metadata.

4. **Never use the word "stranded" with certainty.** Always say "possibly stranded."

5. **Never present throughput estimates as official figures.** Always label: *"Estimated — methodology at /methodology"*

6. **Mandatory disclaimer on every page:**
   > This dashboard is for research and situational awareness only. It is not a navigation or safety system. Data may be incomplete, delayed, inaccurate, or misclassified.

7. **Commodity prices:** Always label *"Daily close values. Not financial advice."*

If you are ever uncertain whether a label or UI element violates these rules, err on the side of the more cautious label.

---

## Full Plan

The full v1 implementation plan (26 sections, architecture, schema, 30-step build order) is at:

```
docs/plans/HORMUZ_TRANSIT_MONITOR_v1.md
```

Read it before implementing anything. It is the source of truth for all design decisions.

---

## Repository Structure

```
/
├── apps/
│   ├── web/                     # Next.js 15 App Router (TypeScript)
│   │   ├── app/                 # Pages + API routes
│   │   │   └── api/             # REST endpoints + SSE streams
│   │   ├── components/          # React components by domain
│   │   ├── lib/                 # prisma.ts, redis.ts, auth.ts, sse.ts
│   │   └── hooks/               # useSSE.ts, useVesselStream.ts
│   │
│   └── worker/                  # Long-running Node.js worker (never imports Next.js)
│       └── src/
│           ├── ais/             # AIS providers, normalizer, mock generator
│           ├── ingest/          # BullMQ queues, dedup, vessel upsert
│           ├── analytics/       # transitDetector, delayDetector, throughputEstimator, corridorScorer
│           ├── commodity/       # fredIngestor, portWatchIngestor
│           ├── alerts/          # alertEvaluator
│           └── jobs/            # BullMQ repeatable job definitions
│
├── packages/
│   └── shared/                  # Types, Zod schemas, config defaults, geo utils
│       ├── types/               # vessel.ts, transit.ts, alert.ts, incident.ts, ais.ts
│       ├── schemas/             # Zod schemas matching DB models
│       ├── config/defaults.ts   # Baselines: 60 ships/day, 20M bbl/day, alert thresholds
│       └── utils/geo.ts         # Turf.js helpers
│
├── prisma/
│   ├── schema.prisma            # All DB models (12 models)
│   ├── seed.ts                  # System settings, geofences, seed incidents, carrier statuses
│   └── migrations/0001_init/   # CREATE EXTENSION postgis + geometry columns
│
├── config/
│   └── geofences/               # GeoJSON files: corridor, boundaries, anchorages, advisory zones
│
├── scripts/                     # backfill-brent.ts, backfill-portwatch.ts, backfill-ais.ts, generate-mock-ais.ts
├── tests/
│   ├── unit/                    # Vitest unit tests
│   ├── integration/             # Vitest integration tests
│   └── e2e/                     # Playwright smoke tests
│
├── docs/
│   └── plans/                   # Implementation plans (start here)
│
├── docker-compose.yml           # postgres + redis + web + worker
├── docker-compose.override.yml  # Dev: volume mounts + hot reload
├── Dockerfile.web
├── Dockerfile.worker
├── Caddyfile                    # Optional LAN reverse proxy for Mac mini staging
├── .env.example                 # All required env vars documented
├── AGENT.md                     # This file
└── package.json                 # npm workspaces root
```

---

## Technology Stack

| Layer | Choice | Key Detail |
|---|---|---|
| Web framework | Next.js 15 App Router | TypeScript, `app/` directory |
| Styling | Tailwind CSS v4 | |
| Map | MapLibre GL JS | Use `GeoJSONSourceDiff` for vessel updates — 6× faster than full replace |
| Charts | Recharts | Transit history, Brent price, throughput |
| Database | PostgreSQL 16 + PostGIS 3.4 | Docker image: `postgis/postgis:16-3.4` |
| ORM | Prisma 5 | PostGIS geometry via `Unsupported()` + `$queryRaw` / `$executeRaw` |
| Queue | BullMQ | Redis-backed; all analytics jobs go through here |
| Cache / Pub-Sub | Redis 7 | `redis:7-alpine`; also used for AIS dedup and alert cooldown |
| Real-time | SSE (Server-Sent Events) | Not WebSocket — works through proxies, auto-reconnects |
| Testing | Vitest + Playwright | Unit/integration + E2E smoke |
| Container | Docker Compose | arm64 images; targets Mac mini M1 staging |

---

## Data Sources

### AIS (Vessel Positions)
- **Primary:** AISStream WebSocket (`wss://stream.aisstream.io/v0/stream`)
- **Bounding box:** `[[25.5, 55.5], [27.0, 59.0]]` (lat/lon, covers Hormuz)
- **Mock fallback:** When `AISSTREAM_API_KEY` is blank or `AIS_MOCK_MODE=true`, `apps/worker/src/ais/mock.ts` generates ~60 realistic vessels. The app must work fully in mock mode.
- **Coverage warning:** ~50% of traffic may be invisible. Always show this.

### Commodity Prices (FRED)
| Symbol | FRED Series | Unit |
|---|---|---|
| `brent` | `DCOILBRENTEU` | USD/bbl |
| `wti` | `DCOILWTICO` | USD/bbl |
| `henry_hub` | `DHHNGSP` | USD/MMBtu |
| `lng_spot` | manual/future | USD/MMBtu |
| `ttf_gas` | manual/future | EUR/MWh |

All stored in `CommodityPrice` table with generic `commodity` string — no schema change needed to add a new source.

### Historical Context
- **Layer 1:** FRED prices — full history available immediately via `backfill:brent` script
- **Layer 2:** IMF PortWatch — free aggregate weekly/monthly transit counts; `backfill:portwatch` script
- **Layer 3:** Simulated baseline — hatched chart band labeled *"Estimated baseline (simulated)"*, dismissible
- **Layer 4:** Live from launch — all AIS positions stored permanently from day one
- **Layer 5:** Paid AIS historical — pluggable via `PaidAisHistoricalProvider` interface (stub only in MVP)

---

## Key Analytics Logic

### AIS Normalizer (`apps/worker/src/ais/normalizer.ts`)
Drop these before any DB write:
- Speed = 102.3 kn → drop (AIS sentinel)
- Speed > 40 kn → drop (decoder error)
- Position (0, 0) → drop (invalid GPS)
- MMSI duplicate within 30s → drop (Redis `SETEX` dedup)
- Position jump > 50 kn implied velocity → store with `anomalyFlag = true` (spoofing indicator)

### Transit Detection (`apps/worker/src/analytics/transitDetector.ts`)
- Vessel enters Hormuz corridor polygon (`ST_Within`)
- Exits from opposite side within 48h → confirmed transit
- `lon < 56.5` entry = westbound; `lon > 56.5` entry = eastbound
- Dedup: one transit per MMSI per 4h window

### Vessel Status (`apps/worker/src/analytics/delayDetector.ts`)
| Status | Speed | Duration |
|---|---|---|
| `delayed` | < 1 kn | > 2h in corridor/anchorage |
| `stationary` | < 0.5 kn | > 6h anywhere |
| `possibly_stranded` | < 0.5 kn | > 24h, no port/anchorage |

### Throughput (`apps/worker/src/analytics/throughputEstimator.ts`)
```
crude_tankers × 700,000 bbl + product_tankers × 250,000 bbl + lng_vessels × 65,000 BOE
```
All multipliers live in `SystemSetting` — change via `/settings` admin page, not code.

### Alert Rules (`apps/worker/src/alerts/alertEvaluator.ts`)
Runs every 5 min. All thresholds come from `SystemSetting`, not hardcoded values.
| Rule | Default Threshold |
|---|---|
| `transit_change_20pct` | Day-over-day ≥ 20% |
| `stranded_threshold` | delayed + stationary + possibly_stranded > 150 |
| `throughput_drop` | Est throughput < 80% of 20 M bbl/day |
| `brent_change` | Configurable % in 24h |
| `ais_stale` | No messages > 15 min |
| `ais_outage` | No messages > 30 min |

Dedup: `AlertEvent.dedupKey` + Redis `SETEX` cooldown (default 4h per type).

---

## Database

Prisma 5 with PostgreSQL 16 + PostGIS 3.4.

**Critical Prisma/PostGIS pattern:**
```prisma
// In schema.prisma — geometry column declared as:
position Unsupported("geometry(Point,4326)")?
```
```typescript
// Write: use $executeRaw
await prisma.$executeRaw`
  UPDATE "Vessel"
  SET position = ST_SetSRID(ST_MakePoint(${lon}, ${lat}), 4326)
  WHERE mmsi = ${mmsi}
`
// Read: use ST_AsGeoJSON() or ST_X() / ST_Y()
const rows = await prisma.$queryRaw`
  SELECT mmsi, ST_X(position) as lon, ST_Y(position) as lat FROM "Vessel"
`
```

**Never use Prisma `findMany` to query geometry columns** — use `$queryRaw` instead.

All configurable baselines and thresholds live in the `SystemSetting` table (key-value). Default values are seeded by `prisma/seed.ts`. Never hardcode a threshold in application logic.

---

## Environment Variables

```bash
DATABASE_URL=postgresql://hormuz:password@localhost:5432/hormuz_monitor
REDIS_URL=redis://localhost:6379
AISSTREAM_API_KEY=          # leave blank → mock mode
AIS_MOCK_MODE=true
FRED_API_KEY=               # free at fred.stlouisfed.org
ADMIN_TOKEN=change-me-in-production
NEXT_PUBLIC_DEFAULT_MAP_CENTER=56.25,26.5
NEXT_PUBLIC_DEFAULT_MAP_ZOOM=8
NEXT_PUBLIC_APP_URL=http://localhost:3000
NODE_ENV=development
```

---

## Development Setup

```bash
# 1. Copy env and fill in API keys
cp .env.example .env

# 2. Start all services
docker compose up -d

# 3. Run migrations
docker compose exec web npx prisma migrate deploy

# 4. Seed the database
docker compose exec web npx tsx prisma/seed.ts

# 5. Dashboard available at http://localhost:3000
```

For mock AIS mode (no API key required), ensure `AIS_MOCK_MODE=true` in `.env`.

### Hot Reload (Dev Override)
`docker-compose.override.yml` mounts source directories and enables hot reload for both `web` and `worker`. Start the same way — Docker Compose applies the override automatically in dev.

### Mac Mini M1 Staging
All Docker images use `platform: linux/arm64`. Deploy identically:
```bash
docker compose up --build -d
docker compose exec web npx prisma migrate deploy
docker compose exec web npx tsx prisma/seed.ts
```

---

## Running Tests

```bash
# Unit + integration tests
npm run test

# E2E smoke tests (requires running app)
npm run test:e2e

# Type check
npm run typecheck
```

Minimum test coverage requirement before considering any feature complete:
- Unit test for every analytics engine function
- Integration test for the full AIS → DB pipeline
- Playwright smoke: dashboard loads, map renders, disclaimer visible, no "safe" label

---

## Admin API Authentication

MVP uses a bearer token from `ADMIN_TOKEN` env var.

```bash
# Example: create an incident
curl -X POST http://localhost:3000/api/incidents \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "incidentType": "gps_interference", "title": "...", "lat": 26.5, "lon": 56.3, "occurredAt": "2026-05-01T00:00:00Z" }'
```

All `POST`, `PUT`, `DELETE` routes are admin-only. All `GET` routes are public.

---

## Real-Time Architecture

```
Worker → Redis Pub/Sub ("vessel-updates" | "alert-events")
       → Next.js SSE route (/api/vessels/stream | /api/alerts/stream)
       → Browser EventSource
       → MapLibre GeoJSONSourceDiff (vessels)
       → React state (alerts, KPIs)
```

**Do not use WebSocket** for the Next.js layer. SSE only — it works through proxies and reconnects automatically.

---

## Pages

| Route | Access | Notes |
|---|---|---|
| `/` | Public | Executive dashboard, all 8 KPI cards |
| `/map` | Public | MapLibre GL JS, all layers |
| `/vessels` | Public | Searchable/filterable vessel table |
| `/alerts` | Public | Active + historical alerts |
| `/carrier-status` | View: public / Edit: admin | |
| `/incidents` | View: public / Edit: admin | |
| `/settings` | Admin only | Baselines, thresholds |
| `/methodology` | Public | Data sources, limitations |
| `/methodology/safe-passage` | Public | Why corridors are NOT navigation guidance |

---

## Conventions

- **All timestamps:** UTC in DB. Display layer converts to user local time + Hormuz time (UTC+4).
- **Confidence badges:** Every data-derived value that involves estimation shows a confidence badge: `high` / `medium` / `low`. Never omit it.
- **Source attribution:** Every piece of data shows where it came from (FRED, AISStream, IMF PortWatch, manual entry, simulated).
- **No comments in code** unless the WHY is non-obvious. No docstrings. No task references ("added for issue #123").
- **No hardcoded thresholds.** Use `SystemSetting` table. Read via `getSystemSetting(key)` helper.
- **Vessel type labels** (stored in `Vessel.shipTypeLabel`): `tanker` | `lng_carrier` | `container` | `bulk` | `general_cargo` | `passenger` | `military` | `tug` | `unknown`
- **Alert severity scale:** `info` → `watch` → `warning` → `critical`
- **`DailyAggregate.source`:** `live` | `imf_portwatch` | `simulated` — never mix in UI without labeling

---

## What Is and Isn't Built Yet

**As of 2026-05-02:**
- [x] v1 implementation plan (`docs/plans/HORMUZ_TRANSIT_MONITOR_v1.md`)
- [x] Directory scaffold (folder structure with `.gitkeep` files)
- [x] This `AGENT.md`
- [ ] Everything else — implementation starts at Step 1 of the 30-step plan

The 30-step build order in the plan is the correct sequence. Do not skip steps or reorder without good reason.

---

## Who Owns This Project

Brian Kenyon / HelloDigital.co — `@kenyob` on GitHub.

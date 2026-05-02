# Hormuz Transit Monitor — Version 1 Build Plan

> **Status:** In progress — Step 1 next  
> **Repo:** `kenyob/hormuz-shipping-tracker` (private)  
> **Branch:** `claude/hormuz-shipping-dashboard-mOgLl`  
> **Root path:** repo root (no nesting)  
> **Staging target:** Mac mini M1 (Apple Silicon, Docker)

---

## Table of Contents

1. [Project Purpose](#1-project-purpose)
2. [Safety and Ethics Constraints](#2-safety-and-ethics-constraints)
3. [Architecture Overview](#3-architecture-overview)
4. [Technology Stack](#4-technology-stack)
5. [Repository Structure](#5-repository-structure)
6. [Data Sources](#6-data-sources)
7. [Historical Data Strategy](#7-historical-data-strategy)
8. [Database Schema](#8-database-schema)
9. [AIS Data Quality Filtering](#9-ais-data-quality-filtering)
10. [AIS Provider Interface](#10-ais-provider-interface)
11. [Analytics Engines](#11-analytics-engines)
12. [Commodity Tracking](#12-commodity-tracking)
13. [Dashboard KPIs](#13-dashboard-kpis)
14. [Alert System](#14-alert-system)
15. [Map Layers](#15-map-layers)
16. [Pages and Routes](#16-pages-and-routes)
17. [API Endpoints](#17-api-endpoints)
18. [Real-Time Architecture (SSE)](#18-real-time-architecture-sse)
19. [Admin Authentication](#19-admin-authentication)
20. [Docker Compose / Mac Mini Deployment](#20-docker-compose--mac-mini-deployment)
21. [Environment Variables](#21-environment-variables)
22. [Implementation Order (30 Steps)](#22-implementation-order-30-steps)
23. [Testing Plan](#23-testing-plan)
24. [Verification Checklist](#24-verification-checklist)
25. [Known Limitations](#25-known-limitations)
26. [Future Roadmap](#26-future-roadmap)

---

## 1. Project Purpose

**Hormuz Transit Monitor** is a public real-time intelligence dashboard tracking:

- Strait of Hormuz vessel movements via AIS data
- Daily transit counts vs. historical baseline (60 ships/day)
- Day-over-day change alerts (threshold: >20%)
- Delayed, stationary, and possibly stranded vessels (alert threshold: >150)
- Estimated oil and LNG throughput vs. baseline (20 M bbl/day)
- Brent crude, WTI, and natural gas prices
- Major carrier disruption statuses (Maersk, MSC, etc.)
- Maritime incidents (mine attacks, seizures, GPS jamming, etc.)
- AIS feed health and data confidence indicators

**Audience:** Crisis monitors, energy-market analysts, investors, and media.

**This is not a navigation tool.** All data is for research and situational awareness only.

---

## 2. Safety and Ethics Constraints

### Mandatory Disclaimer (shown on every page)

> This dashboard is for research and situational awareness only. It is not a navigation or safety system. Data may be incomplete, delayed, inaccurate, or misclassified.

### Hard Prohibitions

The application must never:

- Provide turn-by-turn or operational navigation guidance
- Label any corridor or route as "safe"
- Suggest routes that avoid military forces
- Imply complete AIS data coverage
- Present throughput estimates as official figures
- Label a vessel as military unless the classification is source-verified

### Required Labels (use these instead)

| Situation | Label to Use |
|---|---|
| Low-incident transit corridor | "Lower reported incident corridor" |
| High traffic, low incidents | "High traffic / low reported incident corridor" |
| Missing data | "Data insufficient" |
| Nearby incident | "Recent incident proximity" |
| AIS gaps | "AIS coverage degraded" |
| Vessel type unknown | "Unknown — insufficient metadata" |

---

## 3. Architecture Overview

```
AISStream WebSocket  ─┐
Mock AIS Generator    ├──► AIS Ingestion Worker (Node.js long-running process)
                      │         │
                      │    Redis Queue (BullMQ)
                      │         │
                      │    Normalizer + Deduplicator
                      │         │
                      │    PostgreSQL + PostGIS
                      │         │
                      │    ┌────┴──────────────────────────────┐
                      │    │ Transit Detection Engine           │
                      │    │ Delay / Stationary Engine          │
                      │    │ Throughput Estimator (oil + LNG)   │
                      │    │ Alert Evaluator (every 5 min)      │
                      │    │ Corridor Scorer (every 30 min)     │
                      │    └────────────────────────────────────┘
                      │         │
                      │    Redis Pub/Sub
                      │    (vessel-updates, alert-events)
                      │         │
FRED + IMF PortWatch  ├──► Commodity + Historical Workers
                      │         │
                      └──► Next.js API Layer (SSE + REST)
                                │
                                ▼
                      Dashboard UI / Map / Tables / Alerts
```

### Separation of Concerns

| Layer | Responsibility |
|---|---|
| `apps/worker` | AIS ingestion, analytics, scheduled jobs. Never imports Next.js. |
| `apps/web` | Next.js App Router, SSE streams, REST API, all UI. |
| `packages/shared` | Shared TypeScript types, Zod schemas, geo utilities, config defaults. |

---

## 4. Technology Stack

| Concern | Choice | Notes |
|---|---|---|
| Framework | Next.js 15 (App Router) | TypeScript throughout |
| Styling | Tailwind CSS v4 | |
| Map | MapLibre GL JS | `GeoJSONSourceDiff` for 6× perf on 60+ vessels |
| Charts | Recharts | Transit history, Brent price, throughput |
| Database | PostgreSQL 16 + PostGIS 3.4 | `postgis/postgis:16-3.4` Docker image |
| ORM | Prisma 5 | `Unsupported()` geometry + `$queryRaw` for PostGIS |
| Queue | BullMQ | Redis-backed job queue |
| Cache / Pub-Sub | Redis 7 | `redis:7-alpine` |
| Real-time | Server-Sent Events (SSE) | Next.js API routes → Redis Pub/Sub → browser |
| Testing | Vitest + Playwright | Unit/integration + E2E smoke |
| Container | Docker Compose | arm64-compatible images for Mac mini M1 |

---

## 5. Repository Structure

```
/hellodigital/app/hormuz-strait-path/
│
├── apps/
│   ├── web/                          # Next.js App Router application
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx              # / — Executive dashboard
│   │   │   ├── map/page.tsx
│   │   │   ├── vessels/page.tsx
│   │   │   ├── alerts/page.tsx
│   │   │   ├── carrier-status/page.tsx
│   │   │   ├── incidents/page.tsx
│   │   │   ├── settings/page.tsx
│   │   │   ├── methodology/page.tsx
│   │   │   ├── methodology/
│   │   │   │   └── safe-passage/page.tsx
│   │   │   └── api/
│   │   │       ├── kpis/route.ts
│   │   │       ├── vessels/
│   │   │       │   ├── current/route.ts
│   │   │       │   ├── [id]/route.ts
│   │   │       │   └── stream/route.ts     # SSE: live vessel positions
│   │   │       ├── transits/daily/route.ts
│   │   │       ├── alerts/
│   │   │       │   ├── route.ts
│   │   │       │   └── stream/route.ts     # SSE: live alerts
│   │   │       ├── incidents/route.ts
│   │   │       ├── carrier-status/route.ts
│   │   │       ├── commodity/brent/route.ts
│   │   │       ├── map/layers/route.ts
│   │   │       └── data-health/route.ts
│   │   │
│   │   ├── components/
│   │   │   ├── dashboard/            # KPI cards, charts, alert banner
│   │   │   ├── map/                  # MapLibre GL JS wrapper + layers
│   │   │   ├── vessels/              # Vessel table + filters
│   │   │   ├── alerts/               # Alert list, severity badges
│   │   │   ├── incidents/            # Incident form, map markers
│   │   │   ├── carrier-status/       # Status table, admin editor
│   │   │   └── ui/                   # Badge, Card, Tooltip, ConfidenceBadge, Disclaimer
│   │   │
│   │   ├── lib/
│   │   │   ├── prisma.ts             # Singleton Prisma client
│   │   │   ├── redis.ts              # IORedis client
│   │   │   ├── auth.ts               # Admin token middleware
│   │   │   └── sse.ts                # SSE stream helpers
│   │   │
│   │   └── hooks/
│   │       ├── useSSE.ts
│   │       └── useVesselStream.ts
│   │
│   └── worker/                       # Long-running Node.js worker process
│       └── src/
│           ├── index.ts              # Starts all workers + cron jobs
│           ├── ais/
│           │   ├── provider.ts       # AisProvider interface
│           │   ├── aisstream.ts      # AISStream WebSocket adapter
│           │   ├── mock.ts           # Mock AIS generator (Hormuz patterns)
│           │   ├── normalizer.ts     # Raw AIS → NormalizedPosition + quality filter
│           │   └── historicalProvider.ts  # PaidAisHistoricalProvider interface (stub)
│           ├── ingest/
│           │   ├── queue.ts          # BullMQ queue definitions
│           │   ├── dedup.ts          # Redis dedup (MMSI + 30s window)
│           │   └── vesselState.ts    # Upsert Vessel + AisPosition to DB
│           ├── analytics/
│           │   ├── transitDetector.ts
│           │   ├── delayDetector.ts
│           │   ├── throughputEstimator.ts
│           │   └── corridorScorer.ts
│           ├── commodity/
│           │   ├── fredIngestor.ts   # Generic FRED series ingestor
│           │   └── portWatchIngestor.ts  # IMF PortWatch aggregate counts
│           ├── alerts/
│           │   └── alertEvaluator.ts
│           └── jobs/
│               ├── alertEvalJob.ts       # Every 5 min
│               ├── commodityJob.ts       # Daily (Brent, WTI, HenryHub)
│               ├── dailyRollupJob.ts     # Midnight UTC
│               └── corridorScoreJob.ts   # Every 30 min
│
├── packages/
│   └── shared/
│       ├── types/
│       │   ├── vessel.ts
│       │   ├── transit.ts
│       │   ├── alert.ts
│       │   ├── incident.ts
│       │   └── ais.ts
│       ├── schemas/                  # Zod schemas matching DB models
│       ├── config/
│       │   └── defaults.ts           # Baselines: 60/day, 20M bbl/day, thresholds
│       └── utils/
│           └── geo.ts                # Turf.js lightweight helpers
│
├── prisma/
│   ├── schema.prisma
│   ├── seed.ts                       # System settings, geofences, seed incidents
│   └── migrations/
│       └── 0001_init/
│           └── migration.sql         # CREATE EXTENSION postgis + geometry columns
│
├── config/
│   └── geofences/
│       ├── hormuz-corridor.geojson
│       ├── east-boundary.geojson
│       ├── west-boundary.geojson
│       ├── anchorages.geojson
│       └── advisory-zones.geojson
│
├── scripts/
│   ├── generate-mock-ais.ts
│   ├── backfill-brent.ts             # FRED full history fetch
│   ├── backfill-portwatch.ts         # IMF PortWatch history fetch
│   └── seed-incidents.ts
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── docker-compose.yml
├── docker-compose.override.yml       # Dev: volume mounts + hot reload
├── Dockerfile.web
├── Dockerfile.worker
├── Caddyfile                         # Optional LAN reverse proxy for Mac mini
├── .env.example
├── package.json                      # npm workspaces root
└── README.md
```

---

## 6. Data Sources

### AIS (Vessel Positions)

**Primary:** AISStream WebSocket (`wss://stream.aisstream.io/v0/stream`)

```json
{
  "APIKey": "<AISSTREAM_API_KEY>",
  "BoundingBoxes": [[[25.5, 55.5], [27.0, 59.0]]],
  "FilterMessageTypes": ["PositionReport", "ShipStaticData"]
}
```

- Hormuz bounding box: lat 25.5–27.0, lon 55.5–59.0
- Free tier, API key required (register free at aisstream.io)
- **Critical caveat: ~50% of actual traffic may be invisible** due to AIS blackouts, deliberate jamming, and spoofing in this region. This must be surfaced prominently in the UI.

**Fallback:** Mock AIS generator — when `AISSTREAM_API_KEY` is blank or `AIS_MOCK_MODE=true`, the worker generates ~60 vessels with realistic Hormuz transit patterns, speeds (12–16 kn), and occasional stationary vessels.

### Commodity Prices

| Symbol | Source | Series | Frequency |
|---|---|---|---|
| `brent` | FRED | `DCOILBRENTEU` | Daily |
| `wti` | FRED | `DCOILWTICO` | Daily |
| `henry_hub` | FRED | `DHHNGSP` | Daily |
| `lng_spot` | Manual / future | JKM (Platts) | Daily |
| `ttf_gas` | Manual / future | TTF | Daily |

FRED API endpoint:
```
https://api.stlouisfed.org/fred/series/observations
  ?series_id=DCOILBRENTEU
  &api_key=<FRED_API_KEY>
  &file_type=json
```

### Carrier Status

Manual entry with source URL, confidence score, and status type. Initial seed data covers:

- Maersk, MSC, CMA CGM, Hapag-Lloyd, COSCO, ONE, Evergreen
- Frontline, Euronav/CMB.TECH, QatarEnergy LNG

### Maritime Incidents

Manual entry with admin token. Seed data includes sample historical incidents (mine attacks, seizures, GPS jamming events). Future: ingest from UKMTO, MSCHOA, and news sources.

### IMF PortWatch (Historical Aggregate Counts)

Free aggregate transit statistics for Hormuz. Fetched via `backfill:portwatch` and stored in `DailyAggregate` with `source = "imf_portwatch"`. Provides real trend context for historical charts. No individual vessel data — counts only.

---

## 7. Historical Data Strategy

### Layer 1 — Commodity Prices (Real, Free, Full History)
FRED provides Brent, WTI, and Henry Hub history going back decades. The `backfill:brent` script fetches all available history on first run. Historical price charts are available immediately.

### Layer 2 — IMF PortWatch Aggregates (Real, Free)
Weekly/monthly aggregate transit counts verified by IMF economists. Stored in `DailyAggregate` as `source = "imf_portwatch"`. Displayed in charts with clear source attribution.

### Layer 3 — Simulated Baseline (Synthetic, Clearly Labeled)
For date gaps, charts show a synthetic baseline band: 60 ships/day ± realistic variance (±8/day with mild seasonal pattern). This is:
- **Never mixed silently with real data**
- Shown with hatched fill and label: **"Estimated baseline (simulated)"**
- Dismissible by the user

### Layer 4 — Live from Launch (Real, Vessel-Level)
From the moment the worker starts, all AIS positions are stored permanently. Real vessel-level data accumulates from day one.

### Layer 5 — Paid Historical Provider (Future, Pluggable)
`PaidAisHistoricalProvider` interface scaffolded in `apps/worker/src/ais/historicalProvider.ts`. The `backfill:ais` script calls this interface and fails gracefully with a clear logged message when unconfigured. When revenue allows ($100–500/month for MarineTraffic or Datalastic), the adapter is implemented and historical AIS positions are backfilled without duplicates.

---

## 8. Database Schema

### PostGIS Note
Prisma 5 does not natively render PostGIS `geometry` types. Geometry columns are added via raw SQL migration using `Unsupported("geometry(Point,4326)")` in the schema and `$queryRaw` / `$executeRaw` for spatial operations.

### Migration: 0001_init/migration.sql (excerpt)
```sql
CREATE EXTENSION IF NOT EXISTS postgis;

ALTER TABLE "Vessel"
  ADD COLUMN IF NOT EXISTS "position" geometry(Point, 4326),
  ADD COLUMN IF NOT EXISTS "positionUpdatedAt" timestamptz;

CREATE INDEX IF NOT EXISTS vessel_position_idx
  ON "Vessel" USING GIST (position);
```

### Core Models

```prisma
model Vessel {
  id                 String    @id @default(cuid())
  mmsi               String    @unique
  imo                String?
  name               String?
  callSign           String?
  shipType           Int?
  shipTypeLabel      String?   // tanker|lng_carrier|container|bulk|general_cargo|passenger|military|tug|unknown
  classificationConf Float     @default(0)
  flag               String?
  length             Float?
  width              Float?
  lastLat            Float?
  lastLon            Float?
  lastHeading        Float?
  lastSog            Float?
  lastCog            Float?
  navStatus          Int?
  destination        String?
  lastSeen           DateTime?
  firstSeen          DateTime?
  isInsideCorridor   Boolean   @default(false)
  vesselStatus       String    @default("active") // active|delayed|stationary|possibly_stranded
  statusSince        DateTime?
  anomalyFlag        Boolean   @default(false)
  createdAt          DateTime  @default(now())
  updatedAt          DateTime  @updatedAt
  positions          AisPosition[]
  transits           TransitEvent[]
}

model AisPosition {
  id          String   @id @default(cuid())
  mmsi        String
  lat         Float
  lon         Float
  heading     Float?
  sog         Float?
  cog         Float?
  navStatus   Int?
  timestamp   DateTime
  source      String   @default("aisstream") // aisstream|mock|manual
  anomalyFlag Boolean  @default(false)
  rawMessage  Json?
  vessel      Vessel?  @relation(fields: [mmsi], references: [mmsi])
  @@index([mmsi, timestamp])
  @@index([timestamp])
}

model TransitEvent {
  id              String    @id @default(cuid())
  mmsi            String
  imo             String?
  direction       String    // eastbound|westbound|unknown
  enteredAt       DateTime?
  exitedAt        DateTime?
  durationMinutes Int?
  entryBoundary   String?
  exitBoundary    String?
  confidenceScore Float     @default(0.5)
  sourceProvider  String
  vessel          Vessel    @relation(fields: [mmsi], references: [mmsi])
  createdAt       DateTime  @default(now())
  @@index([enteredAt])
  @@index([mmsi])
}

model MaritimeIncident {
  id                  String   @id @default(cuid())
  incidentType        String   // mine_explosion|drone_attack|missile_attack|seizure|harassment|gps_interference|ais_spoofing|collision|other
  title               String
  description         String?
  lat                 Float
  lon                 Float
  occurredAt          DateTime
  sourceUrl           String?
  sourceName          String?
  confidenceScore     Float    @default(0.5)
  affectedVesselName  String?
  affectedVesselImo   String?
  affectedVesselMmsi  String?
  severity            String   @default("watch") // info|watch|warning|critical
  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt
  @@index([occurredAt])
}

model CarrierStatus {
  id              String    @id @default(cuid())
  carrierName     String
  status          String    @default("unknown") // normal|caution|rerouting|suspended|resumed|unknown
  affectedRegion  String?
  sourceUrl       String?
  sourceTitle     String?
  publishedAt     DateTime?
  capturedAt      DateTime  @default(now())
  confidenceScore Float     @default(0.5)
  notes           String?
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  @@index([carrierName])
}

model CommodityPrice {
  id          String   @id @default(cuid())
  commodity   String   // brent|wti|henry_hub|ttf_gas|lng_spot
  price       Float
  currency    String   @default("USD")
  unit        String   @default("per_barrel") // per_barrel|per_mmbtu|per_mwh
  date        DateTime
  sourceUrl   String?
  sourceName  String   @default("FRED")
  createdAt   DateTime @default(now())
  @@unique([commodity, date])
  @@index([commodity, date])
}

model AlertEvent {
  id          String    @id @default(cuid())
  type        String    // transit_change_20pct|stranded_threshold|throughput_drop|brent_change|carrier_status_change|ais_stale|ais_outage|incident_proximity|corridor_degraded
  severity    String    @default("info") // info|watch|warning|critical
  title       String
  message     String
  triggeredAt DateTime  @default(now())
  resolvedAt  DateTime?
  sourceData  Json?
  confidence  Float     @default(0.5)
  dedupKey    String?   @unique
  @@index([triggeredAt])
  @@index([severity])
}

model SystemSetting {
  key         String   @id
  value       String
  description String?
  updatedAt   DateTime @updatedAt
}

model IngestionHealth {
  id              String    @id @default(cuid())
  provider        String    @unique
  status          String    // healthy|degraded|outage
  lastMessageAt   DateTime?
  messagesPerMin  Float     @default(0)
  vesselsSeen     Int       @default(0)
  duplicateRate   Float     @default(0)
  outageStartedAt DateTime?
  updatedAt       DateTime  @updatedAt
}

model Geofence {
  id        String   @id @default(cuid())
  name      String
  type      String   // corridor|east_boundary|west_boundary|anchorage|advisory|delay_zone
  geoJson   Json
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model DailyAggregate {
  id                Int      @id @default(autoincrement())
  date              DateTime @unique
  source            String   @default("live") // live|imf_portwatch|simulated
  totalTransits     Int      @default(0)
  eastboundTransits Int      @default(0)
  westboundTransits Int      @default(0)
  stationaryCount   Int      @default(0)
  delayedCount      Int      @default(0)
  possiblyStranded  Int      @default(0)
  tankerCount       Int      @default(0)
  lngCount          Int      @default(0)
  unknownCount      Int      @default(0)
  estThroughputBbl  Float?
  throughputConf    String   @default("low")
  estLngMmcfd       Float?
  lngConf           String   @default("low")
  createdAt         DateTime @default(now())
  @@index([date])
}
```

### Default SystemSettings (seeded)

| Key | Default Value |
|---|---|
| `baseline_transits_per_day` | `60` |
| `baseline_throughput_mbbld` | `20` |
| `alert_transit_change_pct` | `20` |
| `alert_stranded_threshold` | `150` |
| `alert_throughput_drop_pct` | `20` |
| `alert_brent_change_pct` | `5` |
| `throughput_crude_bbl_per_tanker` | `700000` |
| `throughput_product_bbl_per_tanker` | `250000` |
| `throughput_lng_boe_per_vessel` | `65000` |
| `ais_stale_threshold_min` | `15` |
| `ais_outage_threshold_min` | `30` |
| `delay_speed_knots_threshold` | `1.0` |
| `stationary_speed_knots_threshold` | `0.5` |
| `delay_duration_hours` | `2` |
| `stationary_duration_hours` | `6` |
| `possibly_stranded_duration_hours` | `24` |

---

## 9. AIS Data Quality Filtering

Raw AIS messages include ~17% anomalous data (research-validated from production deployments). The `normalizer.ts` applies these filters **before** writing to the database:

| Condition | Action |
|---|---|
| Speed = 102.3 kn | Drop — AIS protocol sentinel value |
| Speed > 40 kn | Drop — decoder error |
| Position = (0, 0) | Drop — invalid GPS |
| Position jump > 50 kn implied velocity | Store with `anomalyFlag = true` — spoofing indicator, visible to analysts |
| MMSI duplicate within 30s | Drop — deduplication via Redis `SETEX` |

Anomaly-flagged vessels are not shown as live vessel positions but are accessible in the analyst view.

---

## 10. AIS Provider Interface

```typescript
// packages/shared/types/ais.ts

export interface AisProvider {
  name: string;
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  onMessage(handler: (msg: NormalizedPosition) => void): void;
  getHealth(): Promise<AisProviderHealth>;
}

export interface NormalizedPosition {
  mmsi: string;
  lat: number;
  lon: number;
  sog: number;
  cog: number;
  heading: number;
  navStatus: number;
  timestamp: Date;
  source: string;
}

export interface AisProviderHealth {
  provider: string;
  status: 'healthy' | 'degraded' | 'outage';
  lastMessageAt: Date | null;
  messagesPerMin: number;
  vesselsSeen: number;
  duplicateRate: number;
}

// Future paid historical provider:
export interface PaidAisHistoricalProvider {
  name: string;
  fetchPositions(params: {
    bbox: [number, number, number, number];
    from: Date;
    to: Date;
  }): AsyncGenerator<NormalizedPosition>;
}
```

**WebSocket reconnection strategy:** Exponential backoff starting at 1s, capped at 30s. Connection is considered stable if it survives >30s before reconnect counter resets.

---

## 11. Analytics Engines

### Transit Detection (`transitDetector.ts`)

A transit is confirmed when a vessel:
1. Enters the Hormuz corridor polygon (`ST_Within`)
2. Exits from the opposite boundary side within 48h

```
Direction logic:
  - Entered west boundary (lon < 56.5) → eastbound
  - Entered east boundary (lon > 56.5) → westbound
  - Cannot determine → unknown

Deduplication: One transit per MMSI per 4h rolling window
```

Transit event fields: `mmsi`, `direction`, `enteredAt`, `exitedAt`, `durationMinutes`, `entryBoundary`, `exitBoundary`, `confidenceScore`, `sourceProvider`

### Delay / Stationary / Possibly Stranded Detection (`delayDetector.ts`)

| State | Speed | Duration | Location |
|---|---|---|---|
| `delayed` | < 1 kn | > 2h | Inside corridor or anchorage |
| `stationary` | < 0.5 kn | > 6h | Anywhere in bounding box |
| `possibly_stranded` | < 0.5 kn | > 24h | No port/anchorage explanation |

- Status stored on `Vessel.vesselStatus`
- Never call a vessel "stranded" with certainty — always "possibly stranded"
- Alert fires when (delayed + stationary + possibly_stranded) count > 150

### Throughput Estimator (`throughputEstimator.ts`)

```
estimated crude throughput =
  crude_tanker_count × 700,000 bbl

estimated product throughput =
  product_tanker_count × 250,000 bbl

estimated LNG (BOE equivalent) =
  lng_vessel_count × 65,000 BOE

estimated LNG (MMcf/day) =
  lng_vessel_count × 3.4 Bcf per cargo / transit_days

All multipliers stored in SystemSetting — configurable without code change.
Confidence = high if ≥80% of vessels have verified type, low if <50%.
```

Label: **"Estimated — methodology in /methodology"**

### Corridor Scorer (`corridorScorer.ts`)

Runs every 30 minutes. Scores each corridor segment based on:
- Incident proximity (distance from known incidents)
- Incident recency (weighted by age)
- AIS coverage confidence
- Unknown vessel density
- Active advisory zones

Output: risk score (0–100) + label (see §2). Never outputs navigation guidance.

---

## 12. Commodity Tracking

### Tracked Commodities

| Symbol | Source | Unit | MVP |
|---|---|---|---|
| `brent` | FRED `DCOILBRENTEU` | USD/bbl | ✅ |
| `wti` | FRED `DCOILWTICO` | USD/bbl | ✅ |
| `henry_hub` | FRED `DHHNGSP` | USD/MMBtu | ✅ |
| `lng_spot` | Manual / future | USD/MMBtu | Scaffold |
| `ttf_gas` | Manual / future | EUR/MWh | Scaffold |

All ingested into the same `CommodityPrice` table. Adding a new commodity requires only a new job calling `fredIngestor(seriesId, symbol)` — no schema change.

### Dashboard Display

- **Primary KPI card:** Brent crude price + 24h change %
- **Energy Prices panel:** All active commodities in compact table
- Label: *"Daily close values. Not financial advice."*

### Throughput Display (Oil vs LNG)

- Estimated crude oil: X Mbbl/day
- Estimated LNG: Y MMcf/day
- Combined BOE: Z Mbbl/day

---

## 13. Dashboard KPIs

| # | KPI Card | Data | Alert Condition |
|---|---|---|---|
| 1 | Today's Transits | Count + % vs 60/day baseline | >20% day-over-day change |
| 2 | Delayed / Possibly Stranded | Count | Count > 150 |
| 3 | Est. Oil Throughput | Mbbl/day + % of 20M baseline + confidence badge | Drop > 20% |
| 4 | Brent Crude | Price + 24h change % + last updated | Change > configurable % |
| 5 | Carrier Disruptions | Count of non-normal statuses (clickable) | Any new suspension |
| 6 | AIS Feed Health | healthy / degraded / outage + last message time | Stale >15 min or outage >30 min |
| 7 | Active Alerts | Count by severity | Always visible |
| 8 | Data Freshness | Last ingestion timestamp + user local time + Hormuz time (UTC+4) | — |

---

## 14. Alert System

**Delivery:** In-app only for MVP (in-app banner + `/alerts` page).

### Alert Types

| Type | Trigger | Default Severity |
|---|---|---|
| `transit_change_20pct` | Today vs yesterday ≥ 20% delta | warning / critical |
| `stranded_threshold` | Delayed + stationary + possibly stranded > 150 | warning |
| `throughput_drop` | Est throughput < 80% of 20M baseline | warning |
| `brent_change` | Brent price moved > N% in 24h (configurable) | watch |
| `carrier_status_change` | Any carrier status record changed | info / watch |
| `ais_stale` | No AIS messages > 15 min | watch |
| `ais_outage` | No AIS messages > 30 min | critical |
| `incident_proximity` | New incident within corridor buffer | watch / warning |
| `corridor_degraded` | Corridor confidence score drops > 20% | watch |

### Deduplication

Each alert type has a `dedupKey` (e.g., `transit_change_20pct:2026-05-01`) stored in the `AlertEvent` table with `@unique`. Redis `SETEX` handles cooldown windows (default: 4h per alert type).

### Severity Scale

`info` → `watch` → `warning` → `critical`

---

## 15. Map Layers

Built with MapLibre GL JS. All layers toggle-able.

| Layer | Type | Notes |
|---|---|---|
| Live vessel positions | Symbol | Icon rotates by heading; color by vessel type |
| Vessel trails (last 2h) | Line | Fades by age |
| Transit path density | Heatmap | All recorded transit paths |
| Hormuz corridor polygon | Fill + outline | Semi-transparent |
| East / west boundary lines | Line | Transit detection boundaries |
| Anchorage zones | Fill | Labeled |
| Advisory zones | Fill | Labeled with caution color |
| Incident markers | Symbol + popup | Color by severity |
| Delayed / stationary vessels | Symbol | Distinct icon, pulsing ring |
| Unknown type vessels | Symbol | Grey with "?" icon |
| Military (verified only) | Symbol | Only shown when source-verified |
| AIS coverage quality overlay | Raster/fill | Shows degraded zones when applicable |
| Lower-incident corridors | Line + label | Hatched, labeled correctly (never "safe") |

---

## 16. Pages and Routes

| Route | Description | Access |
|---|---|---|
| `/` | Executive dashboard — all KPIs, charts, alert banner | Public |
| `/map` | Full geospatial map with all layers | Public |
| `/vessels` | Searchable vessel table with filters | Public |
| `/alerts` | Active + historical alerts, severity filters | Public |
| `/carrier-status` | Carrier status table + admin editor | View: public / Edit: admin |
| `/incidents` | Incident list + map + admin editor | View: public / Edit: admin |
| `/settings` | System configuration (baselines, thresholds) | Admin |
| `/methodology` | Data sources, limitations, calculation methodology | Public |
| `/methodology/safe-passage` | Corridor model explanation — why it is NOT navigation guidance | Public |

---

## 17. API Endpoints

### Public (GET)

```
GET  /api/kpis                   All KPI card values
GET  /api/vessels/current        Active vessel list (GeoJSON)
GET  /api/vessels/:id            Single vessel detail
GET  /api/vessels/stream         SSE: real-time vessel position updates
GET  /api/transits/daily         Daily transit counts (history)
GET  /api/alerts                 Alert list (active + history)
GET  /api/alerts/stream          SSE: real-time alert events
GET  /api/incidents              Incident list
GET  /api/carrier-status         Carrier status list
GET  /api/commodity/brent        Brent + all commodity prices
GET  /api/map/layers             GeoJSON for all map layers
GET  /api/data-health            AIS feed health + ingestion stats
```

### Admin (POST / PUT / DELETE — requires `Authorization: Bearer <ADMIN_TOKEN>`)

```
POST   /api/incidents            Create incident
PUT    /api/incidents/:id        Update incident
DELETE /api/incidents/:id        Delete incident
POST   /api/carrier-status       Create carrier status
PUT    /api/carrier-status/:id   Update carrier status
POST   /api/settings             Update system setting
POST   /api/geofences            Create/update geofence
```

---

## 18. Real-Time Architecture (SSE)

```
Worker upserts vessel position
  → publishes JSON to Redis channel "vessel-updates"

Next.js /api/vessels/stream SSE route
  → subscribes to Redis "vessel-updates"
  → streams `data: {...}\n\n` to browser

Browser useVesselStream() hook
  → receives events via EventSource
  → calls map.getSource('vessels').updateData(diff)  [GeoJSONSourceDiff]
  → updates React state for vessel table
```

SSE chosen over WebSocket because:
- Works through proxies and firewalls (pure HTTP)
- Automatic browser reconnection built-in
- Simpler to implement in Next.js App Router
- One-way server → client is all we need for dashboard updates

---

## 19. Admin Authentication

**MVP:** `ADMIN_TOKEN` environment variable. All `POST`/`PUT`/`DELETE` API routes check `Authorization: Bearer <ADMIN_TOKEN>`. Public `GET` routes are unrestricted.

**Upgrade path:** NextAuth.js / Auth.js with email magic link or GitHub OAuth — no schema changes needed.

---

## 20. Docker Compose / Mac Mini Deployment

### arm64 Compatibility

All images are multi-arch and support Apple Silicon (arm64):
- `postgis/postgis:16-3.4` — official, multi-arch
- `redis:7-alpine` — official, multi-arch
- `node:20-alpine` — official, multi-arch

Add `platform: linux/arm64` to each service definition for explicit declaration.

### Services

```yaml
services:
  postgres:
    image: postgis/postgis:16-3.4
    platform: linux/arm64
    environment:
      POSTGRES_USER: hormuz
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: hormuz_monitor
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "hormuz"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7-alpine
    platform: linux/arm64
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3

  web:
    build:
      context: .
      dockerfile: Dockerfile.web
    platform: linux/arm64
    ports:
      - "3000:3000"
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }

  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    platform: linux/arm64
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }

volumes:
  postgres_data:
  redis_data:
```

### Deploying to Mac Mini Staging

```bash
# On Mac mini:
git clone <repo> && cd hellodigital/app/hormuz-strait-path
cp .env.example .env          # fill in API keys
docker compose up --build -d  # build + start all services
docker compose exec web npx prisma migrate deploy
docker compose exec web npx tsx prisma/seed.ts
# Access at http://<mac-mini-ip>:3000
```

### Optional: Caddy LAN Reverse Proxy

`Caddyfile` included for hostname access with self-signed cert:
```
staging.hormuz.local {
  reverse_proxy web:3000
}
```

---

## 21. Environment Variables

```bash
# Database
DATABASE_URL=postgresql://hormuz:password@localhost:5432/hormuz_monitor
DB_PASSWORD=change-me

# Redis
REDIS_URL=redis://localhost:6379

# AIS — leave blank to use mock mode
AISSTREAM_API_KEY=
AIS_MOCK_MODE=true

# Commodities — both free at fred.stlouisfed.org
FRED_API_KEY=

# Admin
ADMIN_TOKEN=change-me-in-production

# Map defaults
NEXT_PUBLIC_DEFAULT_MAP_CENTER=56.25,26.5
NEXT_PUBLIC_DEFAULT_MAP_ZOOM=8

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
NODE_ENV=development
```

---

## 22. Implementation Order (30 Steps)

**Legend:** ✅ Done &nbsp; 🔜 Next &nbsp; ⬜ Pending

### Pre-Build (Complete)

| # | Step | Status |
|---|---|---|
| — | Create repo `kenyob/hormuz-shipping-tracker`, branch `claude/hormuz-shipping-dashboard-mOgLl` | ✅ |
| — | Full directory skeleton (`apps/`, `packages/`, `config/`, `prisma/`, `scripts/`, `tests/`, `docs/`) | ✅ |
| — | `agents.md` — project guide for all agents | ✅ |
| — | `docs/plans/HORMUZ_TRANSIT_MONITOR_v1.md` — this plan | ✅ |

### Build Steps

| # | Step | Files | Status |
|---|---|---|---|
| 1 | Init npm workspaces root; `package.json` for `apps/web`, `apps/worker`, `packages/shared` | `package.json` × 4 | 🔜 |
| 2 | Docker Compose + Dockerfiles + Caddyfile | `docker-compose.yml`, `docker-compose.override.yml`, `Dockerfile.web`, `Dockerfile.worker`, `Caddyfile` | ⬜ |
| 3 | Prisma schema + PostGIS migration + `.env.example` | `prisma/schema.prisma`, `prisma/migrations/0001_init/migration.sql`, `.env.example` | ⬜ |
| 4 | Shared types, Zod schemas, config defaults, geo utils | `packages/shared/` | ⬜ |
| 5 | GeoJSON geofence files | `config/geofences/` | ⬜ |
| 6 | Seed: SystemSettings defaults, geofences, 3 incidents, 5 carrier statuses | `prisma/seed.ts` | ⬜ |
| 7 | Mock AIS generator (Hormuz traffic patterns, ~60 vessels) | `apps/worker/src/ais/mock.ts`, `scripts/generate-mock-ais.ts` | ⬜ |
| 8 | AisProvider interface + normalizer (with quality filters) | `apps/worker/src/ais/provider.ts`, `normalizer.ts` | ⬜ |
| 9 | AISStream WebSocket adapter (exponential backoff reconnection) | `apps/worker/src/ais/aisstream.ts` | ⬜ |
| 10 | BullMQ queue setup + Redis dedup + vesselState upsert worker | `apps/worker/src/ingest/` | ⬜ |
| 11 | IngestionHealth heartbeat (stale/outage alert triggers) | `apps/worker/src/ingest/vesselState.ts` | ⬜ |
| 12 | Transit detection (PostGIS corridor crossing) | `apps/worker/src/analytics/transitDetector.ts` | ⬜ |
| 13 | Delay/stationary detector (updates `Vessel.vesselStatus`) | `apps/worker/src/analytics/delayDetector.ts` | ⬜ |
| 14 | Throughput estimator (oil + LNG) | `apps/worker/src/analytics/throughputEstimator.ts` | ⬜ |
| 15 | FRED commodity ingestor (Brent, WTI, HenryHub) + `backfill:brent` script | `apps/worker/src/commodity/fredIngestor.ts`, `scripts/backfill-brent.ts` | ⬜ |
| 16 | IMF PortWatch ingestor + `backfill:portwatch` script | `apps/worker/src/commodity/portWatchIngestor.ts`, `scripts/backfill-portwatch.ts` | ⬜ |
| 17 | Paid AIS historical provider stub + `backfill:ais` graceful-fail script | `apps/worker/src/ais/historicalProvider.ts`, `scripts/backfill-ais.ts` | ⬜ |
| 18 | Alert evaluator (all rule types, Redis cooldown dedup) | `apps/worker/src/alerts/alertEvaluator.ts` | ⬜ |
| 19 | Corridor scorer (risk scoring, every 30 min) | `apps/worker/src/analytics/corridorScorer.ts` | ⬜ |
| 20 | Worker entry: starts all processes + BullMQ jobs | `apps/worker/src/index.ts` | ⬜ |
| 21 | Daily aggregate rollup job (midnight UTC) | `apps/worker/src/jobs/dailyRollupJob.ts` | ⬜ |
| 22 | Next.js API routes: GET (kpis, vessels, transits, incidents, commodity, data-health, SSE streams) | `apps/web/app/api/` | ⬜ |
| 23 | Next.js API routes: POST/admin (incidents, carrier-status, settings, geofences) | `apps/web/app/api/` | ⬜ |
| 24 | Dashboard page `/` — KPIs, Recharts charts, alert banner, disclaimer | `apps/web/app/page.tsx` + dashboard components | ⬜ |
| 25 | Map page `/map` — MapLibre GL JS, all layers, layer toggles | `apps/web/app/map/page.tsx`, `components/map/` | ⬜ |
| 26 | Vessels page `/vessels` — searchable/filterable table, confidence badges | `apps/web/app/vessels/page.tsx` | ⬜ |
| 27 | Alerts page + carrier status + incidents pages | `apps/web/app/alerts/`, `carrier-status/`, `incidents/` | ⬜ |
| 28 | Settings admin page | `apps/web/app/settings/page.tsx` | ⬜ |
| 29 | Methodology pages + safe-passage explainer | `apps/web/app/methodology/` | ⬜ |
| 30 | Unit tests (Vitest) + Playwright smoke tests + README | `tests/`, `README.md` | ⬜ |

---

## 23. Testing Plan

### Unit Tests (Vitest)

| Test | File |
|---|---|
| AIS message normalization + quality filter | `tests/unit/normalizer.test.ts` |
| MMSI deduplication (Redis window) | `tests/unit/dedup.test.ts` |
| Geofence corridor crossing detection | `tests/unit/transitDetector.test.ts` |
| Transit counting + deduplication | `tests/unit/transitCounter.test.ts` |
| Delay/stationary/stranded classification | `tests/unit/delayDetector.test.ts` |
| Throughput estimation (oil + LNG) | `tests/unit/throughputEstimator.test.ts` |
| Alert threshold logic (20% rule, 150 vessels) | `tests/unit/alertEvaluator.test.ts` |
| Corridor risk scoring | `tests/unit/corridorScorer.test.ts` |

### Integration Tests (Vitest)

| Test | Description |
|---|---|
| Mock AIS → DB pipeline | End-to-end from mock message to DB upsert |
| Worker queue processing | BullMQ job completion |
| API `/kpis` response shape | Validates all KPI fields present |
| Alert generation from threshold breach | Simulates >150 delayed vessels |
| Daily rollup job | Aggregate computed correctly |

### E2E Smoke Tests (Playwright)

| Test | Check |
|---|---|
| Dashboard loads | All 8 KPI cards visible |
| Map renders | MapLibre canvas present, corridor polygon visible |
| Alerts page loads | Alert list renders |
| Disclaimer visible | Text "for research and situational awareness only" present on `/` and `/map` |
| No "safe" label | Assert text "safe route" never appears anywhere |

---

## 24. Verification Checklist

Before considering the build complete, all of the following must pass:

- [ ] `docker compose up --build` — all 4 services reach healthy state
- [ ] `docker compose exec web npx prisma migrate deploy` — migrations apply cleanly
- [ ] `docker compose exec web npx tsx prisma/seed.ts` — seed succeeds
- [ ] Dashboard at `http://localhost:3000` shows all 8 KPI cards with data
- [ ] Map at `/map` renders Hormuz corridor polygon and vessel positions
- [ ] Mock AIS generates ~60 vessels; at least one transit event recorded within 10 min
- [ ] A 20% day-over-day change triggers a visible alert within 5 min
- [ ] Alert at `/alerts` shows with correct severity badge
- [ ] Admin `POST /api/incidents` with bearer token → incident appears on map
- [ ] No API route labeled as "safe"; disclaimer visible on every page
- [ ] `AISSTREAM_API_KEY` removed → app runs in mock mode without error
- [ ] `FRED_API_KEY` removed → Brent widget shows "unavailable" without crash
- [ ] `npm run test` (Vitest) → all unit + integration tests pass
- [ ] `npm run test:e2e` (Playwright) → smoke tests pass
- [ ] `docker compose up` on Mac mini M1 → all services start correctly on arm64

---

## 25. Known Limitations

| Limitation | Notes |
|---|---|
| ~50% AIS coverage gap | Deliberate blackouts, jamming, and spoofing in Hormuz mean roughly half of actual traffic is invisible to AIS. Surfaced prominently in UI. |
| No historical raw AIS (pre-launch) | Vessel-level AIS data only accumulates from launch. Historical context uses FRED prices + IMF PortWatch aggregates + simulated baseline band. |
| FRED prices are daily, not intraday | Brent/WTI updates once per business day. Labeled clearly in UI. |
| LNG spot and TTF prices | Not yet connected to a live source in MVP. Shown as "unavailable" until manually entered or a provider is integrated. |
| Throughput is estimated, not measured | Based on vessel counts × cargo assumptions. Never presented as official flow data. |
| Vessel type classification | Relies on AIS ship type codes — frequently incomplete or wrong for Gulf traffic. Confidence badge shown. |
| AISStream free tier limits | Exact rate limits not published. Monitor `IngestionHealth` for degradation. |

---

## 26. Future Roadmap

| Feature | Priority | Dependency |
|---|---|---|
| Paid historical AIS backfill (MarineTraffic / Datalastic) | High | Revenue / budget |
| LNG spot price live feed | Medium | Platts or EIA API access |
| Email / Slack / webhook alerts | Medium | User auth system |
| NextAuth.js user authentication | Medium | Post-MVP |
| Satellite AIS overlay (Spire, exactEarth) | High | Paid provider |
| News/OSINT incident ingestion (UKMTO, MSCHOA) | Medium | Web scraping or RSS |
| Mobile-responsive map view | Medium | UI work |
| Multi-strait expansion (Bab-el-Mandeb, Malacca) | Low | Geofence config only |
| TimescaleDB for position history | Low | Performance at scale |
| Public API / data export | Low | Post-launch |

---

*Plan version: 1.0 — Generated 2026-05-01*  
*Branch: `claude/hormuz-shipping-dashboard-mOgLl`*  
*Working path: `/hellodigital/app/hormuz-strait-path/`*

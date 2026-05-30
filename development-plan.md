# HVAC Service Management — Phased Development Plan

> Project: 241-hvac-service-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan describes how to build an open-source, AI-native HVAC field service platform that competes with ServiceTitan, FieldEdge, Housecall Pro, BuildOps, and Commusoft. It assumes the research in `research.md`, `features.md`, `standards.md`, and `data-model-suggestion-3.md` (Hybrid Relational + JSONB) as inputs. Where the data model spans concerns covered better by suggestion-1 (e.g. partitioned telemetry, refrigerant transactions, ASHRAE 180 PM templates), those patterns are pulled in.

The plan is written to be executable by a developer who understands modern web architecture but does **not** need HVAC domain expertise to follow it — domain concepts are defined where they appear.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary backend language | **TypeScript (Node.js 22 LTS)** | The platform is API-heavy with a mobile-first technician app, web dispatch board, customer portal, and external integrations (QuickBooks, payment processors, IoT brokers). TypeScript gives end-to-end typing across backend and the React/React Native clients. Node's event-loop model fits the high-throughput webhook/telemetry ingestion path. |
| API framework | **Fastify 5** with **@fastify/swagger** for auto-generated **OpenAPI 3.1** | Fastify outperforms Express on routing and schema validation, has first-class JSON Schema (Draft 2020-12) support — directly aligned with the `standards.md` requirement to publish a complete OpenAPI spec. |
| Secondary IoT/AI language | **Python 3.12** (separate microservice — `telemetry-ingest`) | BACnet/Modbus libraries (`bacpypes3`, `pymodbus`) are the most mature in Python. Fault-detection ML models (scikit-learn, PyTorch) and the LLM orchestration layer (LangChain, Anthropic SDK) belong here. |
| Database | **PostgreSQL 16** with **TimescaleDB extension** | Suggestion 3 (Hybrid Relational + JSONB) is the chosen data model — relational backbone for compliance integrity, JSONB for per-equipment-type variability. TimescaleDB hypertables provide the partitioned-by-time storage that `telemetry_reading` and `energy_reading` need (36-month ASHRAE 90.1 retention). The `pg_trgm` and GIN extensions support full-text search across notes and equipment history. |
| Schema migrations | **Drizzle ORM + Drizzle Kit** | Type-safe schema in TypeScript that compiles to SQL, with first-class JSONB column support and Postgres-specific features (Row Level Security, partitioned tables). Avoids the ORM-vs-SQL split of Prisma/TypeORM. |
| Vector store for AI queries | **pgvector** (Postgres extension) | Lets us keep one database. Used for natural-language history queries — embed equipment notes, work-order resolutions, and PM task results; retrieve by cosine similarity. |
| Cache and queues | **Redis 7** with **BullMQ** | BullMQ provides typed job queues for: PM schedule materialisation, webhook fan-out, notification dispatch, telemetry batch processing, anomaly detection. Redis also caches dispatch-board reads and customer-portal session state. |
| Message broker for IoT | **MQTT 5 (Eclipse Mosquitto)** with **Sparkplug B** decoding | The IoT path needs publish/subscribe, not request/response. MQTT is the emerging-standard mentioned in `standards.md`. BACnet/Modbus gateways translate to MQTT topics; the Python `telemetry-ingest` service subscribes and writes to TimescaleDB. |
| LLM provider | **Anthropic Claude (Sonnet 4.7 default, Opus 4.7 for complex diagnosis)** via **AI SDK (Vercel)** | The AI-native capabilities (natural-language history query, voice-to-structured-notes, compliance-doc auto-population, upsell recommendations) all live on Claude. AI SDK abstracts the provider so we can fallback to GPT or local models. |
| Web frontend | **Next.js 16 (App Router) + React 19 + TypeScript + Tailwind CSS + shadcn/ui** | The dispatch board, admin console, and customer portal are all browser apps. App Router supports the streaming chat UI needed for the natural-language history query. shadcn/ui gives accessible components without lock-in. |
| Mobile technician app | **React Native (Expo SDK 53)** with **WatermelonDB** for offline sync | Offline support is a table-stakes MVP requirement. WatermelonDB has a battle-tested sync protocol against a Postgres backend and handles 10k+ records on-device — sufficient for a year's worth of one technician's jobs and assets. |
| Authentication | **OpenID Connect** via **Clerk** (managed) for the SaaS deployment, with a self-hosted **Keycloak** option | OAuth 2.0 + OIDC is the standard required by `standards.md`. Clerk gives MFA, OTP, magic links, and multi-tenant orgs out of the box; Keycloak supports the self-hosted/open-source promise. |
| Payments | **Stripe** primary, with adapter pattern so QuickBooks Payments / Square plug in later | Stripe Connect supports the multi-tenant SaaS billing model and provides payment-collection UX for the customer portal. |
| Accounting sync | **QuickBooks Online API v3** first (largest share of HVAC SMB market), then **Xero** and **Sage** | QuickBooks is mandated by every competitor analysis in `features.md`. Built behind an `AccountingProvider` interface so additional providers are additive. |
| Object storage | **S3-compatible** (AWS S3 in cloud, MinIO for self-hosted) | For technician photos, signed documents, scanned compliance certificates, voice memos awaiting transcription. |
| Containerisation | **Docker** + **docker-compose** for dev, **Helm chart** for Kubernetes self-host | The project promises a self-hostable open-source alternative; this needs a one-command local stack and a production-grade chart. |
| Testing — backend | **Vitest** (unit + integration), **Pactum** (HTTP integration), **Playwright** (E2E for web apps) | Vitest is faster than Jest and integrates cleanly with Drizzle/TypeScript. Pactum provides expressive HTTP assertions. Playwright drives both the web dispatch board and the customer portal. |
| Testing — mobile | **Jest + React Native Testing Library** + **Detox** for E2E on simulators | Standard React Native testing toolchain. |
| Testing — Python services | **pytest** + **pytest-asyncio** + **respx** for HTTP mocking | Standard Python testing stack with async support for the MQTT/BACnet code. |
| Code quality | **Biome** (lint + format for TS/JS), **Ruff** (lint + format for Python), **TypeScript strict mode**, **mypy strict** | Biome is faster than ESLint+Prettier and replaces both. Ruff is the de-facto Python standard. |
| Package manager | **pnpm** (Node monorepo via workspaces), **uv** (Python services) | pnpm has the best monorepo story (workspace protocol, shared lockfile). uv is dramatically faster than pip/poetry. |
| Monorepo tooling | **Turborepo** | Caches builds/tests across the TS workspace; integrates with pnpm and CI. |
| Observability | **OpenTelemetry** → **Grafana Tempo (traces) + Loki (logs) + Prometheus (metrics)**; **Sentry** for app errors | OTel is the vendor-neutral default; Grafana stack is open-source and self-hostable to match deployment story. |
| Secrets | **Doppler** for cloud SaaS, **.env.local** + **SOPS** for self-hosted | Avoids hardcoded creds; SOPS encrypts at rest in Git for the OSS deployment path. |
| Deployment targets | **Vercel** for the Next.js apps, **Fly.io** / **Railway** for the API and worker services, **Helm** for self-host | Vercel for the web tier (we use Next.js, AI SDK, and Cache Components), separate platform for the long-running API/worker because edge/serverless does not suit the WebSocket + queue worker workloads. |
| CI/CD | **GitHub Actions** with reusable workflows; **Changesets** for versioning | Standard for open-source projects. Changesets handles per-package versioning in the monorepo. |
| License | **AGPL v3** for the platform, **MIT** for SDKs/clients | AGPL prevents proprietary SaaS forks of the server; MIT for libraries others embed. (To be confirmed; README marks license TBD.) |
| API style | **REST/JSON (primary)** + **GraphQL gateway (later phase)** + **MCP server (later phase)** | REST/OpenAPI 3.1 because every competitor integration expects it. GraphQL added in a later phase for the mobile app's selective field needs (mirrors Jobber). The MCP server (per `standards.md`) exposes work orders, equipment, and history to AI assistants. |

### Project Structure

```
hvac-service-management/
├── apps/
│   ├── api/                          # Fastify backend (REST + GraphQL + Webhooks)
│   │   ├── src/
│   │   │   ├── server.ts             # Fastify entry, plugin registration
│   │   │   ├── plugins/              # Fastify plugins (auth, db, redis, OTel)
│   │   │   ├── routes/
│   │   │   │   ├── v1/
│   │   │   │   │   ├── auth/         # OAuth2 / OIDC token endpoints
│   │   │   │   │   ├── tenants/
│   │   │   │   │   ├── users/
│   │   │   │   │   ├── customers/
│   │   │   │   │   ├── service-locations/
│   │   │   │   │   ├── equipment/
│   │   │   │   │   ├── service-agreements/
│   │   │   │   │   ├── pm-schedules/
│   │   │   │   │   ├── work-orders/
│   │   │   │   │   ├── service-appointments/
│   │   │   │   │   ├── technicians/
│   │   │   │   │   ├── invoices/
│   │   │   │   │   ├── payments/
│   │   │   │   │   ├── pricebook/
│   │   │   │   │   ├── refrigerant/
│   │   │   │   │   ├── compliance/
│   │   │   │   │   ├── telemetry/
│   │   │   │   │   ├── ai/           # /ai/query, /ai/upsell, /ai/transcribe
│   │   │   │   │   └── webhooks/     # /webhooks/quickbooks, /stripe etc.
│   │   │   │   └── public/           # customer portal endpoints
│   │   │   ├── modules/              # Business logic per domain
│   │   │   │   ├── crm/
│   │   │   │   ├── equipment/
│   │   │   │   ├── work-orders/
│   │   │   │   ├── scheduling/
│   │   │   │   ├── pm/
│   │   │   │   ├── invoicing/
│   │   │   │   ├── refrigerant/
│   │   │   │   ├── energy/
│   │   │   │   ├── ai/
│   │   │   │   ├── integrations/
│   │   │   │   │   ├── quickbooks/
│   │   │   │   │   ├── stripe/
│   │   │   │   │   └── notifications/
│   │   │   │   └── tenancy/
│   │   │   ├── shared/
│   │   │   │   ├── errors/
│   │   │   │   ├── auth/             # JWT/OIDC verification, RBAC
│   │   │   │   ├── rls/              # Row Level Security helpers
│   │   │   │   ├── audit/
│   │   │   │   └── schemas/          # Zod schemas shared with clients
│   │   │   └── jobs/                 # BullMQ queue producers and processors
│   │   ├── test/
│   │   │   ├── unit/
│   │   │   ├── integration/
│   │   │   └── fixtures/
│   │   ├── drizzle.config.ts
│   │   └── package.json
│   ├── web/                          # Next.js — dispatch board, admin, customer portal
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── (admin)/          # tenant admin pages
│   │   │   │   ├── (dispatch)/       # dispatch board, schedule view
│   │   │   │   ├── (portal)/         # customer portal
│   │   │   │   ├── (ai)/             # NL-query chat UI
│   │   │   │   └── api/              # Next.js API routes (BFF, OAuth callback)
│   │   │   ├── components/
│   │   │   ├── lib/                  # API client (generated from OpenAPI)
│   │   │   └── styles/
│   │   └── package.json
│   ├── mobile/                       # React Native (Expo) — technician app
│   │   ├── src/
│   │   │   ├── screens/
│   │   │   ├── db/                   # WatermelonDB models + sync
│   │   │   ├── components/
│   │   │   └── services/
│   │   ├── app.json
│   │   └── package.json
│   ├── telemetry-ingest/             # Python: BACnet/Modbus → MQTT → TimescaleDB
│   │   ├── src/hvac_ingest/
│   │   │   ├── bacnet_client.py
│   │   │   ├── modbus_client.py
│   │   │   ├── mqtt_subscriber.py
│   │   │   ├── normaliser.py
│   │   │   └── writer.py
│   │   ├── tests/
│   │   └── pyproject.toml
│   ├── ai-services/                  # Python: ML/AI workloads (fault detection, embeddings)
│   │   ├── src/hvac_ai/
│   │   │   ├── fault_detection/
│   │   │   ├── embeddings/
│   │   │   ├── transcription/        # Whisper for voice notes
│   │   │   └── server.py             # FastAPI exposing /predict, /embed, /transcribe
│   │   └── pyproject.toml
│   └── mcp-server/                   # Model Context Protocol server
│       └── src/
├── packages/
│   ├── db/                           # Drizzle schema + migrations + seed
│   │   └── src/
│   │       ├── schema/               # one file per domain
│   │       ├── seed/                 # ASHRAE 180 tasks, refrigerant types, etc.
│   │       └── client.ts
│   ├── shared-types/                 # Zod schemas + inferred TS types
│   ├── api-client/                   # OpenAPI-generated TypeScript client
│   ├── ui/                           # Shared shadcn components used by web + portal
│   ├── prompts/                      # LLM prompt templates (versioned)
│   ├── compliance-rules/             # ASHRAE 180 task library + EPA thresholds
│   └── eslint-config/                # Biome config sharing (placeholder)
├── infra/
│   ├── docker/
│   │   ├── docker-compose.dev.yml    # Postgres+Timescale+Redis+Mosquitto+MinIO+API+web
│   │   └── docker-compose.prod.yml
│   ├── helm/
│   │   └── hvac-service/             # Helm chart for K8s self-hosting
│   ├── github-actions/
│   └── otel-collector/
├── docs/
│   ├── architecture.md
│   ├── api/                          # auto-generated from OpenAPI spec
│   ├── compliance/                   # EPA, ASHRAE references
│   └── self-hosting.md
├── scripts/
│   ├── dev.sh                        # one-command local startup
│   └── seed-demo-data.ts
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
├── README.md
└── CONTRIBUTING.md
```

---

## Phase 1: Foundation — Monorepo, Database, Auth, Multi-Tenancy

### Purpose
Lay the groundwork on which every feature is built: a working monorepo with TypeScript across all apps, PostgreSQL with TimescaleDB and pgvector extensions, Drizzle migrations, OpenID Connect authentication, multi-tenant row-level security, a Fastify API skeleton serving an empty OpenAPI 3.1 spec, and a deployable docker-compose stack. After this phase, contributors can clone the repo, run `pnpm dev`, and hit a `/health` endpoint with a valid JWT.

### Tasks

#### 1.1 — Monorepo Bootstrap

**What**: Initialise the Turborepo + pnpm workspace with the `apps/` and `packages/` directory structure shown above. Stub each application with a `package.json` and a placeholder `README.md`.

**Design**:
- `pnpm-workspace.yaml` declares `apps/*` and `packages/*`.
- `turbo.json` defines pipelines: `build`, `lint`, `test`, `dev` (with `cache: false` for `dev`).
- Root `package.json` scripts: `dev`, `build`, `lint`, `test`, `typecheck`.
- Biome config in repo root: `biome.json` with strict rules (no implicit any, sorted imports, organize imports on save).
- `.nvmrc` pinned to Node 22 LTS.
- `tsconfig.base.json` enables `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`. Each app extends this.
- Conventional Commits enforced via `commitlint` + Husky pre-commit hook (lint-staged on changed files).

**Testing**:
- Unit: `pnpm install` from a clean clone resolves without warnings.
- Unit: `pnpm typecheck` runs across all packages and exits 0.
- Unit: `pnpm lint` runs Biome and exits 0.
- E2E: `turbo run build` produces output in every app/package's `dist/`.

#### 1.2 — Database & Drizzle Schema Bootstrap

**What**: Provision PostgreSQL 16 with TimescaleDB, pgvector, pg_trgm extensions; write the Drizzle schema for the foundation tables; create the first migration; seed the system roles, permissions, and reference data (equipment categories, refrigerant types, BACnet object types).

**Design**:
- `packages/db/src/schema/tenant.ts`:
  ```ts
  export const tenant = pgTable('tenant', {
    id: uuid('id').primaryKey().defaultRandom(),
    name: varchar('name', { length: 255 }).notNull(),
    slug: varchar('slug', { length: 100 }).notNull().unique(),
    subscriptionTier: varchar('subscription_tier', { length: 50 }).notNull().default('standard'),
    timezone: varchar('timezone', { length: 50 }).notNull().default('America/New_York'),
    countryCode: char('country_code', { length: 2 }).notNull().default('US'),
    settings: jsonb('settings').$type<TenantSettings>().notNull().default({}),
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  });

  export type TenantSettings = {
    workOrderPrefix?: string;
    invoiceNumberFormat?: string;
    flatRatePricingEnabled?: boolean;
    epaCertNumber?: string;
    epaLicenseExpiry?: string;  // ISO date
    aimActComplianceMode?: 'standard' | 'strict';
  };
  ```
- `packages/db/src/schema/user.ts`, `role.ts`, `permission.ts`, `userRole.ts`, `rolePermission.ts` per suggestion-1 with tenant scoping.
- Row-Level Security helper:
  ```ts
  // packages/db/src/rls.ts
  export async function setTenantContext(tx: PgTransaction, tenantId: string, userId: string) {
    await tx.execute(sql`SET LOCAL app.current_tenant = ${tenantId}`);
    await tx.execute(sql`SET LOCAL app.current_user = ${userId}`);
  }
  ```
  Every business table gets a policy: `USING (tenant_id = current_setting('app.current_tenant')::uuid)`.
- Seed data:
  - System roles: `super_admin`, `tenant_admin`, `dispatcher`, `technician`, `customer`, `api_consumer`.
  - Permissions: `<entity>.<action>` form (e.g. `work_order.create`, `equipment.read`, `refrigerant.write`).
  - Equipment categories (per suggestion-1): `ahu`, `chiller`, `boiler`, `rtu`, `vav`, `split_system`, `cooling_tower`, `vfd`, `thermostat`, each linked to its ASHRAE 180 section.
  - Refrigerant types (per suggestion-1): `R-410A`, `R-32`, `R-454B`, `R-22`, `R-134a`, `R-407C`, `R-1234yf` with GWP and AIM Act compliance flags.
  - BACnet object types: `analog_input`, `analog_output`, `binary_input`, `binary_output`, `multistate_value`, `schedule`, `trend_log`.

**Testing**:
- Unit: Drizzle schema compiles to expected SQL — snapshot test against `pnpm db:generate` output.
- Integration: `pnpm db:migrate` against an ephemeral Postgres container runs cleanly twice (idempotent).
- Integration: `pnpm db:seed` populates the expected number of system roles (6), refrigerant types (7), categories (9).
- Integration: Inserting a row without a `tenant_id` triggers RLS deny when context is unset.
- Integration: Two tenants cannot see each other's `customer` rows when `app.current_tenant` is correctly set per session.

#### 1.3 — Fastify API Skeleton with OpenAPI 3.1

**What**: Stand up a Fastify server with structured logging, OpenTelemetry instrumentation, error handling middleware, Zod-to-JSON-Schema route validation, auto-generated OpenAPI 3.1 at `/openapi.json`, and a `/health` endpoint.

**Design**:
- `apps/api/src/server.ts`:
  ```ts
  export async function buildServer(opts?: BuildServerOpts): Promise<FastifyInstance> {
    const app = Fastify({
      logger: pinoLoggerConfig(),
      ajv: { customOptions: { keywords: ['kind', 'modifier'] } },
    });
    await app.register(cors, corsConfig);
    await app.register(sensible);
    await app.register(swagger, { openapi: openapiBase });
    await app.register(swaggerUi, { routePrefix: '/docs' });
    await app.register(authPlugin);
    await app.register(dbPlugin);
    await app.register(redisPlugin);
    await app.register(otelPlugin);
    await app.register(v1Routes, { prefix: '/v1' });
    app.setErrorHandler(errorHandler);
    return app;
  }
  ```
- OpenAPI base info (per `standards.md`):
  ```ts
  const openapiBase = {
    openapi: '3.1.0',
    info: { title: 'HVAC Service Management API', version: '0.1.0', license: { name: 'AGPL-3.0' } },
    servers: [{ url: 'https://api.example.com/v1' }],
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
      },
    },
    security: [{ bearerAuth: [] }],
  };
  ```
- Error response shape (RFC 7807 Problem Details):
  ```ts
  type ProblemDetails = {
    type: string;          // URI reference identifying the error type
    title: string;
    status: number;
    detail?: string;
    instance?: string;     // URI of the specific occurrence
    correlationId: string;
  };
  ```
- Health endpoint returns `{ status: 'ok', db: 'ok', redis: 'ok', uptime: <s> }`.
- Configuration loaded from environment via Zod schema in `apps/api/src/config.ts`. Required env vars: `DATABASE_URL`, `REDIS_URL`, `OIDC_ISSUER_URL`, `OIDC_AUDIENCE`, `S3_ENDPOINT`, `S3_BUCKET`.

**Testing**:
- Unit: Config Zod schema rejects missing required env vars with a clear message naming the field.
- Unit: Error handler converts a thrown `BadRequestError` into a 400 Problem Details response.
- Integration (Pactum): `GET /health` returns 200 with status fields.
- Integration: `GET /openapi.json` returns a valid OpenAPI 3.1 document (validated against the spec's JSON Schema).
- Integration: A request without a bearer token to a protected route returns 401 Problem Details.

#### 1.4 — Authentication & Tenant Resolution

**What**: Implement OpenID Connect bearer token verification via Clerk or Keycloak (selectable by env), tenant resolution from JWT claims, and RBAC permission checks via Fastify decorators.

**Design**:
- JWT claims expected (Clerk and Keycloak both emit these via custom claims):
  - `sub` — user ID
  - `org_id` — tenant ID (Clerk org) OR `tenant_id` for Keycloak
  - `roles` — array of role codes
  - `email`, `name`
- `apps/api/src/plugins/auth.ts`:
  ```ts
  app.decorate('authenticate', async (req: FastifyRequest) => {
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) throw new UnauthorizedError('Missing bearer token');
    const claims = await verifyJwt(token, oidcConfig);
    req.user = { id: claims.sub, tenantId: claims.org_id, roles: claims.roles };
    await db.transaction(async (tx) => setTenantContext(tx, req.user.tenantId, req.user.id));
  });

  app.decorate('requirePermission', (perm: string) => async (req) => {
    if (!await userHasPermission(req.user, perm)) throw new ForbiddenError(perm);
  });
  ```
- Routes opt in: `app.get('/customers', { preHandler: [app.authenticate, app.requirePermission('customer.read')] }, handler)`.
- API-key auth (for tenant-to-tenant integrations) implemented as a parallel strategy: rows in `api_key` table with hashed secrets, scoped to a service-account user.

**Testing**:
- Unit: `verifyJwt` rejects an expired token; rejects a token with wrong audience; accepts a valid token.
- Unit: `userHasPermission` returns true when user has role with the permission via `role_permission` table.
- Integration (mocked OIDC): Request with valid JWT to a protected endpoint succeeds and `req.user` is populated.
- Integration: Request with valid JWT but missing permission returns 403 with the permission code in the detail.
- Integration: Two requests with tokens from different tenants do not see each other's data — RLS verified end-to-end.

#### 1.5 — Docker Compose Dev Stack

**What**: One-command local environment: `docker compose up -d` brings up Postgres (with TimescaleDB + pgvector), Redis, Mosquitto, MinIO, the API, the web app, and the telemetry-ingest service.

**Design**:
- `infra/docker/docker-compose.dev.yml` services:
  - `db`: `timescale/timescaledb-ha:pg16` image; init script enables `pgvector` and `pg_trgm`.
  - `redis`: `redis:7-alpine`.
  - `mqtt`: `eclipse-mosquitto:2`.
  - `minio`: `minio/minio:latest` with one bucket auto-created.
  - `api`: built from `apps/api/Dockerfile`, hot-reloads via mounted volumes.
  - `web`: built from `apps/web/Dockerfile.dev`.
  - `telemetry-ingest`: built from `apps/telemetry-ingest/Dockerfile`.
- Healthchecks for `db`, `redis`, `mqtt`. API waits on `db` and `redis`.
- `scripts/dev.sh` wraps: `docker compose up -d db redis mqtt minio && pnpm db:migrate && pnpm db:seed && pnpm dev`.

**Testing**:
- Integration (in CI): `docker compose -f infra/docker/docker-compose.dev.yml up -d`, wait for healthy, `curl /health` returns 200.
- Integration: After tear-down and re-up, no migration errors (idempotency).

---

## Phase 2: Core Domain — Customers, Locations, Equipment, Service History

### Purpose
Build the CRM and asset spine — every later feature depends on customers, service locations, and equipment records. By the end of this phase, a tenant admin can create customers via the API, attach service locations, register equipment with serial numbers and refrigerant types, and view the per-equipment service history (initially empty). This phase implements the relational core from data-model-suggestion-3 and the equipment/refrigerant tables from suggestion-1.

### Tasks

#### 2.1 — Customer & Contact CRUD

**What**: REST endpoints for managing customers (residential, commercial, government, property_manager) and their contacts.

**Design**:
- Table: `customer` (per suggestion-1 §"Customer and Location Management"), with `settings` JSONB column for tenant-defined custom fields.
- Zod request schema:
  ```ts
  export const createCustomerInput = z.object({
    name: z.string().min(1).max(255),
    customerType: z.enum(['residential', 'commercial', 'government', 'property_manager']),
    billingEmail: z.string().email().optional(),
    billingPhone: z.string().optional(),
    billingAddress: addressSchema.optional(),
    taxExempt: z.boolean().default(false),
    customFields: z.record(z.string(), z.unknown()).optional(),
    notes: z.string().optional(),
  });
  ```
- Endpoints:
  - `POST /v1/customers` → 201 + `Customer`
  - `GET /v1/customers/:id` → 200 + `Customer` (with embedded counts: `_links.locations`, `_links.equipment`)
  - `GET /v1/customers?search=...&type=...&page=...&limit=...` → 200 + paginated list. Pagination follows RFC 5988 Link header.
  - `PATCH /v1/customers/:id` → 200 + `Customer`
  - `DELETE /v1/customers/:id` → 204 (soft delete: sets `is_active = false`)
- Contacts as a nested resource: `/v1/customers/:id/contacts` with `POST`, `GET`, `DELETE`. First contact created becomes `is_primary`.

**Testing**:
- Unit: Zod schema rejects invalid customer_type with field-level error.
- Integration: `POST /v1/customers` creates a row with the correct tenant_id from the JWT.
- Integration: A customer in tenant A is invisible to tenant B (RLS enforced).
- Integration: `GET /v1/customers?search=acme` returns customers whose name matches via `pg_trgm` similarity.
- Integration: Soft-deleting a customer cascades to setting their contacts inactive but does not delete equipment rows.
- E2E: Create a customer via API; the dispatch-board web UI lists it within 1 second.

#### 2.2 — Service Locations

**What**: Each customer can have many service locations (job sites). Locations carry the building metadata used by AI features later (building type, sqft, occupancy schedule).

**Design**:
- Table: `service_location` per suggestion-1 §"Customer and Location Management", extended with `properties` JSONB:
  ```ts
  properties: jsonb('properties').$type<LocationProperties>().notNull().default({});
  // LocationProperties:
  // {
  //   building_type: 'office'|'retail'|'warehouse'|'residential'|'hospital'|'school';
  //   year_built?: number;
  //   floor_count?: number;
  //   occupancy_schedule?: { weekday_start: string; weekday_end: string; weekend?: boolean };
  //   access_codes?: string;            // encrypted at app layer
  //   site_contact_id?: string;         // FK to contact, validated app-side
  //   notes?: string;
  // }
  ```
- Endpoints under `/v1/customers/:customerId/locations` and a flat `/v1/service-locations/:id` for lookup.
- Geocoding: when address is set/changed, enqueue a `geocode-location` job that calls an injectable provider (defaults to Nominatim for self-host, Google Maps for SaaS). The job sets `latitude`, `longitude`.

**Testing**:
- Unit: `LocationProperties` Zod schema accepts a minimal `{}` and a fully-populated example.
- Integration: Creating a location enqueues exactly one `geocode-location` job.
- Integration: The geocode worker, with a mocked Nominatim, sets coordinates on the row.
- Integration: GeoJSON `?bbox=...` query returns locations whose lat/lng fall in the box.

#### 2.3 — Equipment Registry

**What**: Equipment records per service location, including manufacturer/model/serial, refrigerant type, warranty dates, and type-specific attributes via JSONB.

**Design**:
- Table: `equipment` (suggestion-1 §"Equipment and Asset Management") with the JSONB addition from suggestion-3:
  ```ts
  typeAttributes: jsonb('type_attributes').$type<EquipmentTypeAttributes>().notNull().default({});
  // Discriminated by equipment.categoryCode, validated against per-category JSON Schemas:
  // - chiller:    { tonnage: number; refrigerant_circuit_count: number; chiller_type: 'centrifugal'|'screw'|'scroll'|'reciprocating'; condenser_type: 'air'|'water'|'evaporative'; ... }
  // - rtu:        { cooling_tons: number; heating_btu: number; stages: number; economizer: boolean; ... }
  // - boiler:     { input_btu: number; output_btu: number; fuel: 'natural_gas'|'propane'|'oil'|'electric'; combustion_efficiency_percent: number; ... }
  // - vav:        { min_cfm: number; max_cfm: number; reheat_coil_type?: 'electric'|'hot_water'|'none'; ... }
  // - ahu:        { supply_cfm: number; outdoor_air_cfm: number; cooling_coil_type: string; heating_coil_type: string; ... }
  ```
- JSON Schemas live in `packages/compliance-rules/src/equipment-attributes/`. Validation is enforced in the API handler before insert/update.
- Equipment-component sub-resource (compressors, coils, filters) per suggestion-1.
- Endpoints:
  - `POST /v1/service-locations/:locId/equipment`
  - `GET /v1/equipment/:id` returns the equipment plus `_embedded.history` (last 10 work orders) and `_embedded.refrigerant_balance` (calculated from `refrigerant_transaction` view).
  - `GET /v1/equipment?location_id=...&category=...&search=...`
  - `POST /v1/equipment/:id/components`
  - `POST /v1/equipment/:id/decommission` → sets `decommission_date` and a reason.

**Testing**:
- Unit: Type-attribute Zod-via-JSONSchema validator rejects a chiller without `tonnage`.
- Integration: Creating equipment for a different tenant's location is forbidden (404, not 403, to avoid existence leakage).
- Integration: Soft-deleting (decommissioning) equipment preserves its work-order history.
- Integration: Search by serial number is exact; search by name uses `pg_trgm` similarity.
- E2E: Tech mobile app scans an equipment QR code → loads equipment detail in offline mode if previously synced.

#### 2.4 — Audit Log Infrastructure

**What**: Capture before/after diffs of every state-changing API call to the `audit_log` table. Required for EPA and ASHRAE compliance posture; surfaces in admin views.

**Design**:
- Drizzle middleware that wraps `db.transaction` callbacks; mutations within a transaction collect diffs and write one `audit_log` row per affected entity.
- Table: `audit_log` per suggestion-1 §"Notifications and Audit Log".
- Diff format:
  ```json
  {
    "fields": {
      "status": { "old": "new", "new": "dispatched" },
      "scheduled_start": { "old": null, "new": "2026-06-01T09:00:00Z" }
    },
    "actor": { "id": "...", "type": "user" },
    "request": { "ip": "1.2.3.4", "user_agent": "...", "correlation_id": "..." }
  }
  ```
- Background worker prunes rows older than 7 years (configurable per tenant).

**Testing**:
- Unit: Diff generator produces only changed fields.
- Integration: Updating a customer name writes exactly one audit row with the old/new name.
- Integration: A failed transaction (rolled back) writes no audit row.
- Integration: Querying `/v1/audit?entity_type=customer&entity_id=...` returns reverse-chronological audit rows.

#### 2.5 — OpenAPI-Generated Client & Web Skeleton

**What**: Generate the TypeScript client from the OpenAPI spec, scaffold the Next.js app with auth wiring, and build the customer/equipment list pages.

**Design**:
- `packages/api-client/` generated via `openapi-typescript` + `openapi-fetch`. Regenerated on `pnpm build` of `packages/db` or `apps/api`.
- Next.js app uses App Router. Auth via Clerk middleware (or Keycloak adapter).
- Pages:
  - `/customers` — table view, search box, "New customer" button.
  - `/customers/[id]` — detail with tabs: Locations, Equipment, Service History, Invoices.
  - `/equipment/[id]` — detail with tabs: Specs, Components, Service History, Telemetry (placeholder).
- React Query for data fetching, Tailwind + shadcn/ui for components, server components for the initial render.

**Testing**:
- Unit: API client wraps the spec; calling `client.GET('/v1/customers/{id}')` is typed (compile fails if you pass wrong path).
- Component (Vitest + RTL): `<CustomersTable>` renders the right count when given mock data.
- E2E (Playwright): Sign in → create customer → it appears in the table → click into detail → URL contains the ID.

---

## Phase 3: Work Orders, Scheduling, and Dispatch

### Purpose
The operational heart of the platform: dispatchers and CSRs can create work orders, the system assigns them to technicians, and the dispatch board shows the day's schedule with drag-and-drop reassignment. By the end of this phase, the platform can run a real HVAC service business for break-fix work.

### Tasks

#### 3.1 — Work Order Lifecycle

**What**: Create, transition, and complete work orders. Enforce the state machine: `new → dispatched → en_route → in_progress → on_hold? → completed → invoiced` (with `cancelled` allowed from any state).

**Design**:
- Tables from suggestion-1 §"Work Orders and Scheduling": `work_order_type` (seeded with the codes shown), `work_order`, `work_order_assignment`, `work_order_task`, `work_order_note`.
- `work_order.workOrderNumber` format: `<tenant.settings.workOrderPrefix><YYYYMMDD>-<sequence>` (e.g. `WO20260601-0042`). Sequence is per-tenant per-day.
- State machine in `apps/api/src/modules/work-orders/state-machine.ts`:
  ```ts
  const allowedTransitions: Record<WorkOrderStatus, WorkOrderStatus[]> = {
    new: ['dispatched', 'cancelled'],
    dispatched: ['en_route', 'on_hold', 'cancelled'],
    en_route: ['in_progress', 'on_hold', 'cancelled'],
    in_progress: ['on_hold', 'completed', 'cancelled'],
    on_hold: ['dispatched', 'in_progress', 'cancelled'],
    completed: ['invoiced'],
    cancelled: [],
    invoiced: [],
  };
  ```
- Transition guards: cannot complete a WO with incomplete required tasks; cannot invoice without `actual_end`.
- Endpoints:
  - `POST /v1/work-orders`
  - `GET /v1/work-orders/:id` (with `_embedded.assignments`, `_embedded.tasks`, `_embedded.notes`)
  - `GET /v1/work-orders?status=...&assigned_to=...&from=...&to=...&customer_id=...`
  - `POST /v1/work-orders/:id/transitions` body `{ to: 'in_progress', reason?: string }`
  - `POST /v1/work-orders/:id/notes`
  - `POST /v1/work-orders/:id/tasks` and `PATCH /v1/work-orders/:id/tasks/:taskId`
- Webhook emission on every state change (Phase 6 wires the dispatcher; here just enqueue).

**Testing**:
- Unit: Transition from `new` to `completed` rejected with a clear error citing the allowed targets.
- Unit: Transition from `in_progress` to `completed` rejected when a task is `is_completed = false` AND `is_regulatory = true`.
- Integration: Creating a WO assigns the next sequential number per tenant per day.
- Integration: Two parallel `POST /v1/work-orders` requests in the same tenant get distinct numbers (no race).
- Integration: Transitioning a WO writes an audit log row of type `status_change`.
- E2E: Create a WO via API, view it in the dispatch UI, transition through to completion.

#### 3.2 — Technicians, Skills, and Certifications

**What**: Technician profiles with skills (e.g. `chiller_repair`, `refrigerant_handling`) and tracked certifications (EPA Section 608, NATE, OSHA-30) with expiry alerts.

**Design**:
- Tables from suggestion-1 §"Technician and Skill Management": `technician`, `skill`, `technician_skill`, `certification_type`, `technician_certification`.
- Seed `certification_type` with: `epa_608_type_i`, `epa_608_type_ii`, `epa_608_type_iii`, `epa_608_universal`, `nate_ac`, `nate_hp`, `osha_10`, `osha_30`.
- Background job runs daily: technicians with certifications expiring in <60 days get a notification; expired certifications mark the technician's `can_work_refrigerant` derived flag false (computed in a view).
- Endpoints:
  - `POST /v1/technicians` (creates a `user` row too with role `technician`)
  - `GET /v1/technicians/:id` (with skills, certifications, current location)
  - `POST /v1/technicians/:id/skills`
  - `POST /v1/technicians/:id/certifications`

**Testing**:
- Unit: A certification with `expiry_date` 30 days out flags as `expiring_soon` in the response.
- Integration: Assigning a non-EPA-certified technician to a refrigerant-handling task is blocked with a 409 Conflict.
- Integration: Daily expiry job, with frozen clock, generates the expected notifications.

#### 3.3 — Service Appointments & Dispatch Assignment

**What**: A work order generates one or more `service_appointment` rows (multi-visit jobs are common — install today, commission tomorrow). The dispatcher assigns appointments to technicians.

**Design**:
- Table: `service_appointment` per suggestion-1.
- Skill-matched auto-assignment helper (basic for MVP):
  ```ts
  async function suggestAssignees(workOrderId: string): Promise<TechnicianSuggestion[]> {
    // 1. Pull required skill from work_order_type or pm_task_template
    // 2. Find technicians with that skill in the tenant
    // 3. Filter to those available in the requested time window
    // 4. Rank by: distance from job, current job count today, skill proficiency
    // 5. Return top 5 with score breakdown
  }
  ```
- Constraint: a technician cannot have overlapping `service_appointment` rows (Postgres `EXCLUDE USING gist` constraint with `tstzrange`).
- Endpoints:
  - `POST /v1/work-orders/:id/appointments`
  - `PATCH /v1/appointments/:id` (reschedule or reassign)
  - `POST /v1/appointments/:id/check-in` (sets `actual_arrival`, transitions WO to `in_progress`)
  - `POST /v1/appointments/:id/check-out` (sets `actual_departure`, optionally `completed`)
  - `GET /v1/work-orders/:id/suggest-assignees` (returns the helper output above)

**Testing**:
- Unit: Overlap constraint prevents creating a second appointment in the same window for the same technician.
- Unit: `suggestAssignees` ranks closer technicians higher when distances differ.
- Integration: Check-in transitions the parent WO to `in_progress` exactly once even on duplicate calls.
- E2E: Drag-and-drop in the dispatch board issues `PATCH /v1/appointments/:id` and updates the calendar.

#### 3.4 — Dispatch Board (Web)

**What**: A calendar / Gantt-style view showing technicians as rows and times as columns, with draggable appointment blocks. Live-updates via Server-Sent Events.

**Design**:
- Use **FullCalendar Resource Timeline** (MIT license) or build with **dnd-kit** on a custom grid.
- Real-time updates: API publishes `appointment.changed`, `wo.changed` to a Redis pub/sub channel per tenant; the web app subscribes via an SSE endpoint `/v1/realtime/dispatch`.
- View modes: Day, Week, Map (technician locations + jobs on a Mapbox/MapLibre tile layer).
- Filters: technician, work-order type, status, customer.

**Testing**:
- Component: Dragging an appointment block fires the expected mutation call.
- E2E: Open dispatch board in two browsers; reassign in one; second board updates within 2 seconds.

#### 3.5 — Customer & Technician Notifications (Stub)

**What**: When a work order is dispatched, completed, or scheduled, enqueue notifications. Concrete channels (email, SMS, push) plug in later.

**Design**:
- Table: `notification` per suggestion-1.
- Channel adapters interface:
  ```ts
  interface NotificationChannel {
    send(notification: PendingNotification): Promise<DeliveryResult>;
  }
  ```
- Adapters in this phase: `LogChannel` (writes to logs), `WebhookChannel` (POSTs to a tenant-configured URL). Real email/SMS in Phase 6.
- Template registry in `packages/prompts/src/notifications/` with handlebars-style templates per `template_code`.

**Testing**:
- Unit: Template `appointment_reminder` renders correctly given a sample appointment payload.
- Integration: Dispatching a WO enqueues exactly one `appointment_confirmation` notification per assigned technician and one per customer contact with `notify_appointments: true`.

---

## Phase 4: Mobile Technician App & Offline Sync

### Purpose
Field technicians work in basements, on rooftops, and in parking garages — connectivity is unreliable. The mobile app must work fully offline, sync when network is available, and give the technician everything needed for a job: customer history, equipment specs, prior service notes, the checklist, and payment collection. By the end of this phase, a technician can complete an entire job offline and have it sync cleanly.

### Tasks

#### 4.1 — React Native + Expo Bootstrap

**What**: Initialise the Expo app, configure auth via Clerk (or Keycloak), set up navigation, and wire it to the API for online operations.

**Design**:
- Expo SDK 53, expo-router for navigation.
- Auth: `@clerk/clerk-expo` (or `react-native-app-auth` for Keycloak).
- Navigation tree:
  - `(auth)/login`
  - `(tabs)/jobs` — today's appointments
  - `(tabs)/schedule` — week view
  - `(tabs)/customers` — search
  - `(modal)/job/[id]` — job detail with subscreens
- Theming: Tailwind via `nativewind`.

**Testing**:
- Unit: Auth provider sets the token in the API client.
- E2E (Detox): Launch app, log in with a test user, see today's jobs.

#### 4.2 — Offline Database with WatermelonDB

**What**: Local SQLite via WatermelonDB mirroring the subset of server tables the technician needs: customers, locations, equipment, my appointments, work orders, work order tasks, pricebook items, notes.

**Design**:
- Sync protocol: pull-based with `?updated_since=<lastPulledAt>`; conflict resolution is last-write-wins on the server with optimistic local writes.
- Sync endpoint: `POST /v1/sync` body `{ lastPulledAt: string; changes: { <table>: { created: [], updated: [], deleted: [] } } }` returns `{ changes: ..., timestamp: ... }` matching the WatermelonDB sync protocol.
- Pull scope per technician:
  - All customers and equipment for service locations they have an upcoming appointment for in the next 7 days.
  - Their own appointments and work orders for the past 30 days and next 30 days.
  - Tenant's full pricebook (typically <5k items).
- Photos and signatures saved to device filesystem; uploaded to MinIO/S3 via a background sync queue.

**Testing**:
- Unit: Sync protocol handles the empty-state pull (no `lastPulledAt`).
- Unit: Conflict — local update of WO notes + server update of WO status both survive.
- Integration: Take device offline → modify WO → complete tasks → take photo → reconnect → server reflects all changes.
- E2E: Tech checks in, completes 4 tasks, captures signature, marks complete, all offline. Reconnect → server WO shows the captures.

#### 4.3 — Job Detail & Task Checklist

**What**: The technician-facing view of a work order: customer info, equipment specs, prior service summary, task list with check-off, notes, photos.

**Design**:
- Screen sections (collapsible cards):
  - Customer & site (with click-to-call, click-to-map)
  - Equipment (with prior 5 work orders inline)
  - Tasks (each with: description, expected duration, ASHRAE 180 reference, completion checkbox, measurement field if applicable)
  - Notes (technician-only and customer-visible tabs)
  - Photos (camera + gallery, EXIF-tagged with location)
  - Signature (canvas component)
  - Refrigerant log (added/recovered amounts; cylinder serial)
- Task measurement fields driven by `pm_task_template.measurement_schema` (JSONB):
  ```json
  {
    "type": "object",
    "properties": {
      "supply_air_temp_f": { "type": "number", "minimum": 40, "maximum": 80 },
      "return_air_temp_f": { "type": "number" },
      "filter_pressure_drop_in_wc": { "type": "number" }
    },
    "required": ["supply_air_temp_f", "return_air_temp_f"]
  }
  ```

**Testing**:
- Component: Checking a task with required measurements not filled in shows a validation error.
- E2E: Complete a checklist → all measurements persist to `work_order_task.result` as JSON.

#### 4.4 — In-Field Payment Collection

**What**: Technician collects payment on site via Stripe Terminal (in-person card present) or Stripe Payment Intents for keyed entry / card-on-file.

**Design**:
- `@stripe/stripe-react-native` SDK.
- New endpoint: `POST /v1/payments/intent` body `{ workOrderId, amount, currency, method }` returns `{ clientSecret, intentId }`. Server-side this creates the Stripe PaymentIntent on the tenant's connected Stripe account.
- On success, app POSTs `/v1/work-orders/:id/payments` with the intent ID; server verifies via Stripe, writes `payment` and `invoice` rows (auto-creates invoice from WO line items if not yet generated).

**Testing**:
- Integration (Stripe test mode): Create payment intent → confirm with a test card → invoice row created.
- Integration: Webhook from Stripe (failed payment) updates the invoice to `partial` or back to `sent`.
- E2E (manual + checklist): Take payment on a physical Stripe Terminal in Stripe test mode.

---

## Phase 5: Service Agreements, Preventive Maintenance & Refrigerant Compliance

### Purpose
Move beyond break-fix into recurring revenue and compliance — the highest-margin part of an HVAC business and the area where competitors (Commusoft, ServiceTitan) build the deepest moats. This phase implements service agreement management, automated PM scheduling derived from ASHRAE 180 task templates, and EPA Section 608 / AIM Act refrigerant tracking with automatic threshold alerts.

### Tasks

#### 5.1 — Service Agreement Management

**What**: Sell, renew, suspend, and track service agreements that cover specific equipment with defined SLAs and billing schedules.

**Design**:
- Tables: `service_agreement`, `agreement_equipment` per suggestion-1.
- Agreement creation flow:
  1. POST `/v1/service-agreements` with start/end, billing schedule, coverage type.
  2. POST `/v1/service-agreements/:id/equipment` to attach equipment items.
  3. POST `/v1/service-agreements/:id/activate` transitions to `active` and triggers PM schedule materialisation (task 5.2).
- Recurring billing: a daily worker scans agreements where `next_invoice_date <= today AND status = 'active'` and creates draft invoices.
- Endpoint `POST /v1/service-agreements/:id/renew` clones the agreement with new dates.

**Testing**:
- Unit: Agreement status transitions follow allowed paths.
- Integration: Activating an agreement creates PM schedule rows for every attached equipment × applicable PM task template.
- Integration: Recurring billing worker creates the right number of invoices for a quarterly agreement over 12 months.

#### 5.2 — PM Task Templates & Schedule Materialisation

**What**: Seed the ASHRAE 180-2018 PM task library and materialise per-equipment PM schedules from service agreements.

**Design**:
- Table: `pm_task_template` per suggestion-1, extended with `measurement_schema: jsonb` (see 4.3).
- Seed data lives in `packages/compliance-rules/src/ashrae-180/`. Example entry:
  ```ts
  {
    equipmentCategoryCode: 'rtu',
    code: 'rtu-filter-inspect',
    name: 'Inspect and Replace Air Filter',
    ashrae180Ref: 'Table 7-1, Task 3',
    frequency: 'quarterly',
    estimatedDurationMinutes: 15,
    isRegulatory: false,
    requiresSkillCode: null,
    measurementSchema: {
      type: 'object',
      properties: {
        filter_condition: { enum: ['clean', 'dirty', 'replaced'] },
        pressure_drop_in_wc: { type: 'number' }
      },
      required: ['filter_condition']
    }
  }
  ```
- Initial seed covers AHU, chiller, boiler, RTU, VAV, cooling tower, split system tasks per ASHRAE 180 Tables 7-1 through 7-9.
- `pm_schedule` table per suggestion-1.
- Daily PM materialisation job: for every `pm_schedule` with `next_due_date <= today + lookAheadDays (default 14)` and `is_active = true`, create a work order of type `preventive`, set `source = 'pm_schedule'`, link `pm_schedule_id`, create one `work_order_task` per applicable `pm_task_template`.
- After WO completion, update `pm_schedule.last_completed_date` and recompute `next_due_date` from the frequency.

**Testing**:
- Unit: Seed loader imports the expected count of ASHRAE 180 tasks (snapshot test).
- Integration: Activating an agreement covering 3 RTUs creates the expected number of `pm_schedule` rows.
- Integration: Materialisation worker creates work orders with the expected tasks and skips already-materialised schedules.
- Integration: Completing a quarterly PM WO advances `next_due_date` by 3 months.

#### 5.3 — Refrigerant Transactions & EPA Compliance

**What**: Capture every refrigerant addition or recovery against equipment, with technician certification, cylinder details, and AIM Act 15 lb threshold alerts.

**Design**:
- Tables: `refrigerant_transaction`, `refrigerant_leak_inspection`, `equipment_refrigerant_balance` (view) per suggestion-1.
- Endpoint: `POST /v1/equipment/:id/refrigerant-transactions` body:
  ```json
  {
    "workOrderId": "...",
    "transactionType": "add" | "recover" | "reclaim" | "recycle" | "dispose",
    "refrigerantTypeId": "...",
    "quantityOz": 24.0,
    "cylinderSerialNumber": "TANK-12345",
    "recoveryMachineSerial": "RM-67890",
    "leakRatePercent": 4.2,
    "notes": "..."
  }
  ```
- Server-side checks:
  - Technician (from JWT) has an active EPA cert appropriate for the refrigerant (Type II for high-pressure, Type III for low-pressure).
  - Cylinder serial validated against per-tenant cylinder registry.
  - On AIM Act-restricted refrigerants (R-410A in new equipment after Jan 2025), warn and require manager override.
- Post-insert trigger evaluates the rolling 365-day net charge per equipment; if `net_charge_lb >= 15` or `annual_leak_rate_percent >= 35%` (commercial AIM Act threshold), insert into `energy_anomaly`-style `compliance_alert` table and create a `refrigerant_leak_inspection` task on the next service visit.
- Endpoint: `GET /v1/equipment/:id/refrigerant-history` returns transactions + leak inspections in chronological order.
- Endpoint: `GET /v1/compliance/refrigerant-report?period=YYYY-Q[1-4]&format=csv|pdf` produces an EPA-aligned summary report.

**Testing**:
- Unit: Refrigerant transaction by non-certified technician rejected with 403.
- Unit: AIM Act post-2025 add of R-410A to a new install flagged.
- Integration: Three additions totalling >15 lb on a single piece of equipment trigger a follow-up leak inspection task.
- Integration: CSV refrigerant report matches the expected shape (one row per add/recover, with cert number, quantity, equipment).

#### 5.4 — Compliance Documentation & E-Signatures

**What**: Auto-generate and store compliance documents (refrigerant logs, leak inspection reports, PM checklists) with technician signatures and customer acknowledgement.

**Design**:
- Tables: `compliance_document`, `jurisdiction`, `compliance_rule` per suggestion-1.
- PDF generation: on WO completion, generate per-document type PDFs via `@react-pdf/renderer` (server-side). Store in S3/MinIO; record reference in `compliance_document`.
- Document templates:
  - PM service report (per ASHRAE 180 task list completed)
  - Refrigerant handling certificate (per EPA Section 608)
  - Leak inspection report
  - Customer invoice receipt
- Signing: technicians sign in the mobile app at job completion; customer signs via the mobile app or via a signed URL sent to email post-completion.
- Document retention: default 7 years, configurable per tenant (some jurisdictions require longer).

**Testing**:
- Unit: PDF renderer produces a deterministic byte stream for a fixed payload (snapshot/hash).
- Integration: Completing a refrigerant transaction work order produces both a PM service report and a refrigerant handling certificate stored in S3 with the right metadata.
- E2E: Customer receives an email, clicks the link, signs in the browser; the document is updated with the signature.

---

## Phase 6: Notifications, Customer Portal, and Accounting Integration

### Purpose
Close the customer-facing loop: real notifications (email, SMS, push), a self-service customer portal mirroring the depth Commusoft offers (job history, reports, invoices, online payment), and the QuickBooks Online sync that every HVAC business mandates. By the end of this phase, the platform is genuinely SMB-deployable.

### Tasks

#### 6.1 — Real Notification Channels

**What**: Pluggable channel implementations for email (Resend or SES), SMS (Twilio), and mobile push (Expo Push). Tenants configure their providers; the queue routes notifications accordingly.

**Design**:
- Channel registry:
  ```ts
  interface NotificationChannel {
    readonly id: 'email' | 'sms' | 'push' | 'webhook';
    canHandle(notification: PendingNotification): boolean;
    send(notification: PendingNotification): Promise<DeliveryResult>;
  }
  ```
- Providers configurable per tenant via `tenant.settings.notifications` JSONB.
- Templates stored in `packages/prompts/src/notifications/` per `template_code` × `channel` × `locale`. Default templates included for `appointment_confirmation`, `appointment_reminder` (24h, 1h), `technician_en_route`, `wo_completed`, `invoice_sent`, `payment_received`, `pm_due`, `cert_expiring`.
- Retry policy: exponential backoff, max 5 attempts; final failure flagged.

**Testing**:
- Unit: Each channel correctly rejects notifications it cannot handle.
- Integration (mocked Twilio): `appointment_reminder` SMS sent contains the expected merge fields.
- Integration: Push notification with invalid Expo token marks notification as `failed` after one attempt.

#### 6.2 — Customer Self-Service Portal

**What**: A separate Next.js area where customers log in to view their equipment, service history, request work, view and pay invoices, and download reports.

**Design**:
- Auth: customer users have role `customer`, scoped to a single customer record via a `customer_user` table:
  ```ts
  customerUser = pgTable('customer_user', {
    userId: uuid('user_id').references(() => user.id),
    customerId: uuid('customer_id').references(() => customer.id),
    canRequestService: boolean('can_request_service').default(true),
    canViewInvoices: boolean('can_view_invoices').default(true),
    canPayInvoices: boolean('can_pay_invoices').default(true),
  });
  ```
- Public-facing API endpoints under `/v1/public/...` enforce that customer users only see their own customer's records.
- Pages:
  - `/portal/dashboard` — upcoming appointments, open invoices, PM schedule
  - `/portal/equipment` — list of their equipment with service history per item
  - `/portal/work-requests/new` — submit a new service request; creates a `work_order` in `new` status
  - `/portal/invoices` — list with view/pay
  - `/portal/documents` — compliance documents (PDFs)

**Testing**:
- Integration: Customer A user cannot read customer B's invoices (403).
- E2E: Customer logs in, requests service, dispatcher sees the new WO in the dispatch board.
- E2E: Customer pays an invoice via Stripe → status flips to `paid`.

#### 6.3 — QuickBooks Online Integration

**What**: Two-way sync of customers, invoices, and payments with QuickBooks Online.

**Design**:
- OAuth 2.0 connection flow per tenant; tokens stored encrypted in `integration_connection` table.
- Outbound sync (HVAC platform → QBO):
  - On `customer.created/updated` → QBO Customer upsert
  - On `invoice.sent` → QBO Invoice create
  - On `payment.received` → QBO Payment create
  - Field mappings live in `apps/api/src/modules/integrations/quickbooks/mappers/`.
- Inbound sync (QBO → HVAC platform):
  - QBO webhook → `payment.received_from_qbo` event → mark invoice as paid (idempotent on the external ID).
- Reconciliation report endpoint: `GET /v1/integrations/quickbooks/reconciliation?from=...&to=...` returns unmatched invoices/payments.
- `AccountingProvider` interface keeps the door open for Xero, Sage, etc.

**Testing**:
- Unit: Customer mapper produces a QBO Customer payload with the expected fields.
- Integration (mocked QBO API via Pactum): Creating an invoice sends the expected request body.
- Integration: QBO webhook signature verification rejects forged payloads.
- Integration: Idempotency — receiving the same QBO webhook twice does not double-mark the invoice.

#### 6.4 — Pricebook & Flat-Rate Pricing

**What**: A categorised pricebook with parts, labour rates, and flat-rate services (the "Good/Better/Best" pattern common in HVAC). Technicians select items on the mobile app; line items flow to invoices.

**Design**:
- Tables: `pricebook_category`, `pricebook_item`, `work_order_line_item` per suggestion-1.
- Bulk import endpoint: `POST /v1/pricebook/import` accepts CSV (industry-standard FieldEdge/ServiceTitan-style columns).
- Flat-rate variants: a pricebook item can have child variants (good/better/best) presented to the customer side-by-side.
- Tax engine: tax rate determined by service location's jurisdiction (table from suggestion-1); tenant can override per item.

**Testing**:
- Unit: CSV import parses 10k rows in <5 seconds (benchmark test).
- Integration: Adding a line item to a WO updates the WO total; deleting it reverses the change.
- E2E: Tech selects "Compressor replacement — Better" on mobile; invoice reflects the chosen variant.

---

## Phase 7: IoT Telemetry Ingestion (BACnet, Modbus, MQTT)

### Purpose
Begin the AI-native differentiator: connect to real HVAC equipment. The Python `telemetry-ingest` service speaks BACnet/IP, Modbus TCP, and MQTT (Sparkplug B); normalises readings into a common shape; writes them to a TimescaleDB hypertable; and pushes critical alerts to the API. By the end of this phase, equipment telemetry is flowing into the platform — the foundation for fault diagnosis and energy monitoring.

### Tasks

#### 7.1 — BACnet/IP Client

**What**: Python service that discovers BACnet devices on a configured network, reads `Analog Input`, `Binary Input`, and `Multi-state Value` objects on a polling schedule, and publishes the readings to MQTT.

**Design**:
- Library: `bacpypes3` (async-native fork of bacpypes).
- Configuration (`telemetry-ingest.config.toml`):
  ```toml
  [bacnet]
  local_address = "192.168.1.50/24"
  bbmd_address = "10.0.0.1:47808"   # optional BBMD for cross-subnet

  [[bacnet.sites]]
  tenant_id = "..."
  site_id   = "..."
  poll_interval_seconds = 60
  device_filter = [12345, 12346]   # optional; otherwise auto-discover
  ```
- Discovery flow: emit `Who-Is`, collect `I-Am` responses, enumerate object list per device, persist device + point inventory via API call to `/v1/telemetry-points/bulk-upsert`.
- Polling: each device's points read in batches via `ReadPropertyMultiple`; results published as MQTT messages on `hvac/<tenant_id>/<site_id>/<equipment_id>/<point_name>`.

**Testing**:
- Unit: Message encoder produces a valid BACnet APDU for `ReadPropertyMultiple`.
- Integration: Against a Docker BACnet simulator (`open-bacnet-stack-test`), discovers the expected devices and points.
- Integration: Polling continues after a single device timeout (does not crash the loop).

#### 7.2 — Modbus TCP Client

**What**: Same as 7.1 but for Modbus TCP devices (chillers, VFDs, energy meters).

**Design**:
- Library: `pymodbus` (async).
- Per-equipment register map config stored in `equipment.telemetry_config` JSONB (see suggestion-3):
  ```json
  {
    "protocol": "modbus_tcp",
    "host": "10.0.0.45",
    "port": 502,
    "unit_id": 1,
    "registers": [
      { "name": "supply_air_temp_f", "address": 40001, "type": "float32", "scale": 1.0 },
      { "name": "compressor_amps", "address": 40005, "type": "uint16", "scale": 0.1 }
    ],
    "poll_interval_seconds": 30
  }
  ```

**Testing**:
- Unit: Type conversion for float32 / int16 / scaled-integer.
- Integration: Against `pymodbus`'s test server, polls all configured registers and publishes to MQTT.

#### 7.3 — MQTT Subscriber and Telemetry Writer

**What**: Subscribe to the `hvac/#` topic tree, normalise messages, batch-insert into the `telemetry_reading` TimescaleDB hypertable.

**Design**:
- Telemetry hypertable (replaces partitioning from suggestion-1 with TimescaleDB):
  ```sql
  CREATE TABLE telemetry_reading (
    telemetry_point_id UUID NOT NULL,
    equipment_id       UUID NOT NULL,
    tenant_id          UUID NOT NULL,
    reading_value      DOUBLE PRECISION NOT NULL,
    quality            VARCHAR(20) DEFAULT 'good',
    recorded_at        TIMESTAMPTZ NOT NULL
  );
  SELECT create_hypertable('telemetry_reading', 'recorded_at', chunk_time_interval => INTERVAL '7 days');
  SELECT add_retention_policy('telemetry_reading', INTERVAL '36 months');
  ```
- Continuous aggregate for the 15-minute roll-up required by ASHRAE 90.1 §8:
  ```sql
  CREATE MATERIALIZED VIEW telemetry_15min
  WITH (timescaledb.continuous) AS
  SELECT
    telemetry_point_id,
    equipment_id,
    tenant_id,
    time_bucket('15 minutes', recorded_at) AS bucket,
    AVG(reading_value) AS avg_value,
    MIN(reading_value) AS min_value,
    MAX(reading_value) AS max_value,
    COUNT(*) AS sample_count
  FROM telemetry_reading
  GROUP BY telemetry_point_id, equipment_id, tenant_id, bucket;
  ```
- Writer batches 500 readings or 1 second of arrivals, whichever first.
- Out-of-range readings (versus `equipment_telemetry_point.alert_min/max`) emit an `alert.raised` message on Redis pub/sub that the API forwards as a real-time event.

**Testing**:
- Unit: Normaliser handles missing `equipment_id` (drops with metric).
- Integration: 10k synthetic messages published → all rows present within 5 seconds.
- Integration: TimescaleDB continuous aggregate produces correct 15-minute buckets.
- Load test: 5k readings/sec sustained for 60 seconds — p95 ingest latency under 2 seconds.

#### 7.4 — Telemetry API & Dashboard

**What**: REST endpoints to query telemetry, plus a per-equipment telemetry chart in the web app.

**Design**:
- Endpoints:
  - `GET /v1/equipment/:id/telemetry?point=...&from=...&to=...&resolution=raw|15min|hourly|daily`
  - `GET /v1/equipment/:id/telemetry/latest`
  - `GET /v1/equipment/:id/telemetry/points` (configured points and their current values)
- Web charts: ECharts or Recharts time-series with zoom; live updates via SSE.

**Testing**:
- Integration: Hourly resolution query returns one row per hour from the continuous aggregate.
- E2E: Equipment detail page shows a live-updating temperature chart.

---

## Phase 8: AI-Native Features — Fault Diagnosis, NL History Query, Energy Monitoring

### Purpose
Deliver the differentiating capabilities: AI-assisted fault diagnosis from telemetry, natural-language history querying, energy-efficiency baseline monitoring with anomaly alerting, voice-to-structured field notes, and AI upsell recommendations. This is where the project moves from "another FSM" to "AI-native FSM".

### Tasks

#### 8.1 — Energy Baselines & Anomaly Detection

**What**: For every piece of telemetered equipment, learn an energy-consumption baseline; flag deviations as anomalies; if confirmed, auto-generate a work order.

**Design**:
- Tables: `energy_baseline`, `energy_anomaly` per suggestion-1.
- Baseline algorithm (initial):
  - Daily kWh from a power-monitoring point; aggregate via continuous aggregate.
  - Build a regression model: `kWh ~ outdoor_temp + day_of_week + occupied_hours` per equipment.
  - Recompute weekly.
  - Anomaly = actual vs predicted residual > 2σ for 3 consecutive days.
- Anomaly types (per suggestion-1):
  - `efficiency_degradation`
  - `consumption_spike`
  - `runtime_excessive`
  - `refrigerant_leak_suspected` (combined evaporator-temp + suction-pressure pattern)
- Auto-WO generation: severity `critical` anomalies create a `work_order` with type `corrective` and source `iot_alert`, linking back to the anomaly.

**Testing**:
- Unit: Baseline regression fitted on a known synthetic dataset matches expected coefficients within tolerance.
- Integration: Three consecutive days of synthetic high readings generate exactly one anomaly (not three).
- Integration: A `critical` anomaly creates a WO; downgrading to `warning` does not.

#### 8.2 — Embeddings Pipeline & Vector Index

**What**: Embed equipment notes, work-order resolutions, technician diagnoses, and compliance documents into pgvector; expose semantic search.

**Design**:
- New table:
  ```sql
  CREATE TABLE knowledge_embedding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_type     VARCHAR(50) NOT NULL,  -- 'work_order_resolution', 'equipment_note', 'pm_task_result', 'compliance_doc'
    source_id       UUID NOT NULL,
    content         TEXT NOT NULL,
    embedding       VECTOR(1536),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
  );
  CREATE INDEX ON knowledge_embedding USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
  ```
- Background job re-embeds new/updated content via OpenAI `text-embedding-3-small` (1536 dims) — or Cohere/local equivalent through AI SDK.
- Search endpoint: `POST /v1/ai/search` body `{ query: string, sourceTypes?: string[], equipmentId?: string, k: number }` returns matched chunks with similarity scores and source citations.

**Testing**:
- Unit: Embedding chunk size enforced (≤8k tokens).
- Integration: After ingesting 100 sample notes, a semantic query returns the expected top-3 with predictable order.

#### 8.3 — Natural-Language History Query (Streaming Chat)

**What**: A chat interface — for technicians and managers — that answers questions like "which RTUs at the Acme site failed twice this summer?" by combining structured SQL queries with embedding retrieval.

**Design**:
- Architecture: agent loop with **Anthropic Claude Sonnet 4.7** (default) via **Vercel AI SDK**, plus tool calls:
  - `sql_query(question)` → constrained SQL executor against read-only views per tenant.
  - `vector_search(query, k, filters)` → calls 8.2's search endpoint.
  - `equipment_lookup(id_or_serial)` → API call.
  - `work_order_lookup(filters)` → API call.
- System prompt template lives in `packages/prompts/src/ai/nl-query.system.md`. Key constraints:
  - Always cite source IDs in `[wo:WO-...]`, `[eq:...]` format that the UI renders as links.
  - Never invent equipment, customers, or service events not returned by tools.
  - Use the user's role to scope answers (technicians see their own history; managers see all).
- Endpoint: `POST /v1/ai/query` returns Server-Sent Events streaming the response.
- Web UI: `/ai` page with chat thread, message bubbles, citation pills.
- Prompt caching enabled via Anthropic SDK to amortise the system prompt and tool definitions.

**Testing**:
- Unit: SQL tool rejects DDL/DML; allows only SELECT against `*_view` tables.
- Unit: Citation parser extracts `[wo:...]` references for UI rendering.
- Integration (with deterministic mock LLM): Query "show me chiller failures in Q1" produces a SQL call with the expected predicates.
- E2E (against real LLM, optional in CI): Sample 20 questions; ≥80% return answers grounded in tool results (no hallucinated equipment).

#### 8.4 — Voice-to-Structured Field Notes

**What**: Technician speaks into the mobile app; transcription + structured-extraction yields a populated work-order task result, diagnosis, and recommended pricebook items.

**Design**:
- Mobile records audio (Expo AV), uploads to `/v1/ai/transcribe`.
- API enqueues a job to the Python `ai-services` FastAPI:
  - Whisper (or AssemblyAI cloud) transcribes audio.
  - Claude with a strict structured output schema extracts:
    ```json
    {
      "summary": "...",
      "diagnosis": "...",
      "measurements": { "supply_air_temp_f": 58.0, ... },
      "parts_used": [ { "name": "20x25x1 filter", "quantity": 2 } ],
      "follow_up_required": true,
      "follow_up_reason": "Compressor making unusual noise"
    }
    ```
- API merges into the work order (technician reviews before save).

**Testing**:
- Unit: Schema rejects malformed extracted JSON; falls back to raw transcription in notes.
- Integration (mocked Whisper + Claude): Sample audio file produces the expected structured output.
- E2E: Technician records voice memo, taps "Apply" — task fields populate.

#### 8.5 — Upsell Recommendations

**What**: At job-completion time on the mobile app, suggest 0-3 upsell pricebook items based on equipment age, failure history, and current visit context.

**Design**:
- Endpoint: `POST /v1/ai/upsell` body `{ workOrderId, equipmentId }`.
- Server pulls: equipment age, last 12 months of work orders, current refrigerant balance, applicable PM schedules near due, current pricebook flat-rate items with category mapping.
- Claude prompt (cached): given context, output ranked suggestions with rationale and pricebook item IDs.
- Mobile displays as a card list; tech can tap to add to invoice.

**Testing**:
- Integration: Equipment older than 12 years + recent refrigerant adds → suggests "Refrigerant leak diagnostic" service.
- Integration: New equipment with no failures → returns 0 suggestions.

---

## Phase 9: Public APIs, Webhooks, GraphQL, MCP, and Marketplace

### Purpose
Make the platform an integration hub: a clean public REST API (already implemented), versioned webhooks for external systems, a GraphQL gateway for partners and the mobile app's selective fetches, an MCP server exposing read access to AI assistants, and a marketplace listing for early integrations (Zapier, Make, n8n).

### Tasks

#### 9.1 — Versioned Webhook Delivery

**What**: Tenants subscribe to events (`work_order.created`, `wo.completed`, `invoice.paid`, `pm.due`, `anomaly.raised`) and receive HMAC-signed JSON POSTs with at-least-once delivery and retry.

**Design**:
- Tables:
  ```ts
  webhookEndpoint = pgTable('webhook_endpoint', {
    id, tenantId, url, secret /* hashed */, events /* text[] */,
    isActive, lastDeliveryAt, failureCount, createdAt, updatedAt
  });
  webhookDelivery = pgTable('webhook_delivery', {
    id, endpointId, eventType, payload, status, attempts,
    responseStatus, responseBody, nextRetryAt, createdAt, deliveredAt
  });
  ```
- Signature: `X-HVAC-Signature: t=<ts>,v1=<hmac_sha256(secret, t + '.' + payload)>` per the Stripe-style convention.
- Retry: 30s, 5m, 30m, 3h, 1d, 3d, 7d (then dead-letter).
- Replay endpoint: `POST /v1/webhooks/deliveries/:id/replay`.

**Testing**:
- Unit: Signature verifier accepts a valid signature, rejects clock skew >5 minutes.
- Integration: Failing target URL retried per schedule; eventual success marks delivery `succeeded`.

#### 9.2 — GraphQL Gateway

**What**: A read-focused GraphQL layer (mirroring Jobber's approach) on top of the REST/DB layer for selective-field queries the mobile app uses.

**Design**:
- Library: `mercurius` (Fastify-native GraphQL).
- Schema generated from Drizzle via a thin code generator; relations exposed as GraphQL relations.
- Auth integrates with the existing JWT middleware.
- N+1 prevention via dataloader.

**Testing**:
- Integration: Query nested `customer { locations { equipment { workOrders } } }` issues no more than 4 DB round trips on a sample dataset.

#### 9.3 — Model Context Protocol Server

**What**: An MCP server exposing read-only tools so AI assistants can query work orders, equipment, customers, and service history via the standard MCP transport.

**Design**:
- Implements MCP spec resources and tools:
  - Tool: `search_work_orders(filters)`
  - Tool: `get_equipment(id)` with embedded history
  - Tool: `get_compliance_status(equipment_id)`
  - Tool: `ask(question)` — proxies to the NL query agent from 8.3
- Auth via OAuth 2.0 token from the platform; scoped per tenant.
- Distributed as both a hosted endpoint and an installable CLI (`npx @hvac/mcp-server`).

**Testing**:
- Integration: MCP client lists the expected tools and invokes each successfully against a seeded tenant.

#### 9.4 — Zapier / Make / n8n Connectors

**What**: Pre-built triggers (job completed, invoice paid) and actions (create job, send notification) for the major automation platforms.

**Design**:
- Built atop the webhook + REST API.
- Distributed via the respective platforms' app directories.

**Testing**:
- Manual checklist + smoke tests per platform.

---

## Phase 10: Hardening — Security, Observability, Performance, Compliance Posture

### Purpose
Take the platform from "feature complete" to "production-grade open-source SaaS". This phase has no new user-facing functionality; it is about being safe to deploy.

### Tasks

#### 10.1 — Security Hardening per OWASP API Security Top 10

**What**: Audit and remediate against each of the 10 risks; publish a security disclosure policy.

**Design**:
- API1 BOLA: every handler verifies the requested resource's `tenant_id` matches the request's — enforced via a Fastify pre-handler decorator `app.requireResourceTenant(resourceType)`.
- API2 Broken Authentication: token expiry ≤1 hour for user tokens; refresh tokens rotated.
- API3 Excessive Data Exposure: response schemas reviewed; PII fields gated behind explicit serialiser.
- API4 Resource Consumption: per-tenant rate limits (`@fastify/rate-limit`); request body size cap 10MB.
- API5 BFLA: `requirePermission` middleware applied to every mutating route.
- API6 Unrestricted Resource Consumption / mass assignment: Zod strict mode rejects unknown fields.
- API7 SSRF: webhook URL set requires DNS resolution check that disallows RFC 1918 / link-local addresses (configurable for self-host).
- API8 Security Misconfiguration: Helmet defaults; CORS allowlist per tenant.
- API9 Improper Inventory Management: OpenAPI spec served, deprecated routes documented with `x-deprecated`.
- API10 SSRF / unsafe consumption: every outbound API call (LLM, QBO, Stripe, geocoder) has explicit timeouts and circuit breakers.
- Static analysis: `semgrep --config=p/owasp-top-10` and Dependabot in CI.

**Testing**:
- Security tests: cross-tenant access attempts (403); excessive payload (413); SQL injection attempts in search fields (rejected at Zod layer).
- Integration: `npm audit --production` clean.
- Manual: dependency review per release.

#### 10.2 — Observability & SLOs

**What**: Trace every request end-to-end across API → worker → Python services; meaningful dashboards; on-call alerts.

**Design**:
- OpenTelemetry SDK in all services. Trace context propagated to BullMQ jobs and MQTT messages (via custom headers).
- Dashboards in Grafana:
  - API: request rate, error rate, p50/p95/p99 latency per route.
  - DB: connection pool saturation, slow query log.
  - Queues: depth, processing time, failures per queue.
  - Telemetry: ingestion lag, drop rate.
  - LLM: cost per tenant per day, token usage, cache hit rate.
- SLOs (initial):
  - API availability 99.9% monthly
  - Dispatch board p95 latency <500ms
  - Telemetry p95 ingest lag <30s
  - Webhook successful delivery >99% within 1 hour
- Sentry for error reporting in API + web + mobile.

**Testing**:
- Integration: A request emits a trace span with the expected attributes (tenant_id, user_id sanitised, route).
- Manual: dashboards populated against seed traffic.

#### 10.3 — Backups, Disaster Recovery, and Data Export

**What**: Daily encrypted backups; tested restore; full per-tenant data export for GDPR / customer ownership.

**Design**:
- pgBackRest or barman for Postgres; daily full + 5-minute WAL retention; backups stored in versioned S3 bucket.
- Quarterly restore drill checklist published in `docs/runbooks/`.
- Tenant export: `POST /v1/tenant/export` produces a signed download URL for a ZIP containing JSON dumps of every entity and PDFs of compliance documents.
- Tenant deletion: 30-day soft-delete window then irreversible purge.

**Testing**:
- Integration: Restore from yesterday's backup into a scratch environment; row counts match.
- Integration: Tenant export ZIP unpacks cleanly and validates against published JSON schemas.

#### 10.4 — Performance & Cost Optimisation

**What**: Identify and fix N+1 queries, add Cache Components for read-heavy web views, batch LLM calls, and tune TimescaleDB chunk sizes for the production telemetry pattern.

**Design**:
- Add `pg_stat_statements`; weekly slow-query report.
- Cache Components on `web` dashboard pages (Next.js 16 `use cache` directive) for tenant-level aggregates.
- LLM call batching: batch retry-safe upsell/transcription calls; use Anthropic message batches API for non-realtime workloads.
- Telemetry: tune `chunk_time_interval` based on actual ingest rate (start at 7 days, may shrink to 1 day for high-volume tenants).

**Testing**:
- Load test: 50 concurrent dispatchers, 200 concurrent customer portal users, 1000 readings/sec telemetry — system stable for 30 minutes.

#### 10.5 — Documentation, Self-Hosting Guide, and Helm Chart

**What**: Publish complete docs: getting started, self-hosting on K8s, configuring providers (auth, payments, LLM), compliance posture, API reference.

**Design**:
- Docusaurus site under `docs/` (separate Next.js app may be added).
- API reference auto-generated from the OpenAPI spec via Redocly.
- Helm chart under `infra/helm/hvac-service/` with values for Postgres (managed or in-cluster), Redis, S3, OIDC, payment provider, LLM provider.
- Sample tenant seed via `helm install --set demoData=true`.

**Testing**:
- Integration: `helm install` against `k3d` cluster brings up a working stack; `kubectl port-forward` and the web app loads.
- Manual: a fresh contributor follows `docs/self-hosting.md` end-to-end without assistance.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (Monorepo, DB, Auth, RLS, Docker)           ─── required by everything
    │
Phase 2: Core Domain (Customers, Locations, Equipment, Audit)   ─── requires P1
    │
Phase 3: Work Orders, Scheduling, Dispatch                      ─── requires P2
    │
    ├── Phase 4: Mobile Technician App & Offline Sync          ─── requires P3
    │
    ├── Phase 5: Service Agreements, PM, Refrigerant Compliance ─── requires P3
    │       │
    │       └─ depends on technician/skills from P3.2
    │
    └── Phase 6: Notifications, Customer Portal, Accounting     ─── requires P3 (+ P5 for PM-due notifications)
              │
Phase 7: IoT Telemetry Ingestion (BACnet, Modbus, MQTT)         ─── requires P2 (equipment) + P5 (refrigerant tie-ins)
    │
Phase 8: AI-Native Features (Fault Diagnosis, NL Query, Energy) ─── requires P7 (telemetry) + P5 (history)
    │
Phase 9: Public APIs, Webhooks, GraphQL, MCP, Marketplace       ─── requires P6 + P8
    │
Phase 10: Hardening (Security, Observability, Backups, Docs)    ─── runs in parallel from P6 onward; finalised after P9
```

Phases that can be developed in parallel after their gates clear:
- **P4 (Mobile) ∥ P5 (Agreements/PM) ∥ P6 (Portal/QBO)** — all unlock once P3 is in place.
- **P7 (Telemetry) ∥ P6 (Customer Portal completion)** — independent track once P5 (refrigerant equipment hooks) is done.
- **P9 sub-tasks (webhooks, GraphQL, MCP, marketplace)** are independent and can each be tackled by separate contributors.
- **P10** is a continuous track running alongside P6 onwards; final hardening pass and Helm chart land at the end.

Suggested ordering by criticality for an MVP launch: P1 → P2 → P3 → P5 → P4 → P6 → P10 (basic hardening). Then P7 → P8 → P9 → P10 (full hardening) for the "1.1" release that ships the AI-native differentiators.

---

## Definition of Done (per phase)

A phase is complete only when **every** item below is satisfied:

1. All tasks in the phase implemented and merged to `main`.
2. All unit, integration, and E2E tests pass in CI (no skipped tests without a tracked issue).
3. Biome lint and Ruff lint pass with zero warnings.
4. TypeScript `strict` and mypy `strict` pass with zero errors.
5. Drizzle migrations created, reversible, and applied cleanly against a fresh database via `pnpm db:migrate`.
6. Seed data updated and demo data refreshed (`scripts/seed-demo-data.ts`).
7. Docker dev stack starts cleanly via `docker compose up -d` and the new feature works against it.
8. Helm chart values updated for any new env vars, secrets, or services.
9. New API endpoints appear in `/openapi.json` with descriptions, request/response schemas, and example payloads; the OpenAPI document validates against the spec's JSON Schema.
10. Generated `packages/api-client/` rebuilt and committed; downstream apps compile.
11. New configuration options documented in `docs/configuration.md` with defaults.
12. Audit log writes verified for all new state-changing operations.
13. RLS verified for any new tenant-scoped tables (cross-tenant access denied in tests).
14. Telemetry instrumentation present on new spans/metrics; dashboard tile or alert rule added if appropriate.
15. CHANGELOG entry added via Changesets with affected packages and a user-facing summary.
16. Manual smoke test performed against a hosted preview environment and screenshots attached to the release notes.

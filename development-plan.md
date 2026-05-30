# Co-Working Space Management — Phased Development Plan

> Project: 470-co-working-space-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` proposals into a concrete, phased build. The data layer is anchored on **Data Model Suggestion 1** (normalized PostgreSQL with GiST temporal-exclusion constraints), because its database-level `no_overlapping_bookings` constraint is the single most direct solution to the project's core requirement — eliminating double-bookings without race conditions. JSONB escape hatches (from Suggestion 3) are used for operator-custom fields and provider config, and TimescaleDB (from Suggestion 4) is deferred to a scaling phase for the high-write `access_events` stream. Event-sourcing (Suggestion 2) is rejected for the MVP as over-engineered.

---

## Core Requirements Summary

- **What it does**: A unified, self-hostable platform for co-working/flex-space operators covering memberships, real-time desk/room booking, automated Stripe billing, smart-lock access control, community engagement, and analytics — replacing fragmented closed-source SaaS (Nexudus, Coworks, Archie, Spacebring, Cobot, OfficeRnD Flex).
- **Primary personas**: (1) Operator admins / location managers (configure plans, resources, pricing, view analytics), (2) Front-desk staff (check-ins, walk-in day passes, visitor management), (3) Members (book, pay, unlock doors, participate in community), (4) Corporate account admins (manage seats, consolidated invoices).
- **Key differentiators**: Open-source + self-hostable (no per-location SaaS lock-in), DB-level booking integrity, vendor-neutral access-control gateway, AI-native layer (occupancy forecasting, dynamic pricing, community moderation, sentiment/churn signals) that incumbents lack.
- **Deployment model**: Self-hosted, cloud, or hybrid via Docker Compose; multi-tenant (organization → location hierarchy) from day one.
- **Integration surface**: Stripe (billing/webhooks), smart locks (Kisi, Salto, Brivo, Hakuna) behind an abstraction gateway, Google Calendar / Microsoft Graph / iCal (RFC 5545), email + SMS, Slack/Teams, QuickBooks/Xero (later), LLM provider for AI features.
- **Standards compliance**: OpenAPI 3.1 for all REST endpoints, OAuth 2.0 (RFC 6749) + OIDC for SSO and third-party access, JWT (RFC 7519) for stateless auth, RFC 5545 iCalendar for calendar interop, Stripe webhook idempotency, OWASP Top 10 (2021) mitigations, GDPR/CCPA controls (consent, audit, deletion), PCI scope minimised by delegating card data to Stripe.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript (Node.js 22 LTS) | Project is API/integration/frontend-heavy, not ML-heavy. One language across API, web portal, and admin reduces context-switching. AI features are LLM-API calls, not local model training, so Python is unnecessary. Strong Stripe, Google Calendar, MS Graph, and smart-lock SDK support in TS. |
| API framework | NestJS 11 | Modular DI architecture maps cleanly to the bounded contexts (membership, booking, billing, access, community). First-class OpenAPI 3.1 generation via `@nestjs/swagger`, guards for RBAC, interceptors for audit logging, and a built-in queue abstraction. |
| Database | PostgreSQL 16 | Required by the chosen data model: `btree_gist` exclusion constraints make double-booking structurally impossible; `TSTZRANGE`, JSONB, generated columns, partial indexes, and materialized views are all used directly. Mature, no lock-in. |
| ORM / query layer | Drizzle ORM | TypeScript-native, SQL-first (does not hide the GiST/exclusion/range features Prisma struggles to express), generates migrations, and supports raw SQL escape hatches for the exclusion constraint and materialized views. |
| Migrations | drizzle-kit + raw SQL migrations | Version-controlled. Raw SQL files handle features Drizzle's schema DSL cannot express (exclusion constraints, materialized views, partitioning). |
| Task queue / async | BullMQ on Redis | Billing dunning, webhook processing, access-permission sync, notification fan-out, and materialized-view refresh are async. BullMQ gives retries, backoff, repeatable (cron) jobs, and idempotency keys. |
| Cache / locks / rate limit | Redis 7 | Availability snapshots, session/refresh-token store, rate limiting, BullMQ backend, and short-lived booking holds. |
| Frontend (web) | Next.js 15 (App Router) + React 19 + TypeScript | Two surfaces: member portal and operator admin dashboard, both server-rendered for SEO/whitelabel and fast first paint. shadcn/ui + Tailwind for the component system. Shares TS types with the backend via a generated OpenAPI client. |
| Mobile | React Native (Expo) — v1.1 phase | Reuses TS skill set and shared API client. BLE local credential caching for offline door unlock (research §4) is feasible via Expo modules. |
| Auth | Self-issued JWT (access + refresh) + Passport strategies; OAuth2/OIDC for SSO | RFC 7519 access tokens (15 min) + rotating refresh tokens in Redis. OIDC (RFC 6749 / OpenID Connect) for corporate SSO and Google/Microsoft login. |
| Payments | Stripe (SDK + webhooks) behind a `PaymentGateway` interface | Near-universal in the category; abstraction interface keeps the door open for PayPal/Square (backlog). Idempotency keys on all writes. |
| Access control | Internal `LockProvider` gateway interface with per-vendor adapters | Hardware market is fragmented with no common API (standards.md §lock-in). Gateway pattern (à la Seam) avoids vendor lock-in. |
| AI layer | Provider-agnostic LLM client (default OpenAI-compatible / Anthropic) | Moderation, sentiment, forecasting, dynamic pricing. Forecasting uses lightweight statistical models in TS (no Python service needed for MVP-grade forecasts). |
| Email / SMS | Provider interface: SMTP/SendGrid (email), Twilio (SMS) | Pluggable so self-hosters can use their own SMTP. |
| Containerisation | Docker + docker-compose | Self-hosted is a primary deployment mode; compose bundles API, web, Postgres, Redis. |
| Testing | Vitest (unit), Supertest (HTTP integration), Testcontainers (real Postgres/Redis), Playwright (web e2e) | Fast unit runner; Testcontainers exercises the real exclusion constraint, which a mock cannot validate. |
| Quality tools | ESLint + Prettier + `tsc --noEmit` (strict) | Standard TS toolchain. |
| Package manager / monorepo | pnpm workspaces + Turborepo | Monorepo holds `apps/api`, `apps/web`, `apps/mobile`, and shared `packages/*` (types, OpenAPI client, config). |
| API docs | Swagger UI + Redoc from generated OpenAPI 3.1 spec | standards.md recommended alignment. |
| Observability | pino structured logs, OpenTelemetry traces, Prometheus metrics | Audit logging doubles as GDPR/ISO 27001 evidence. |

### Project Structure

```
co-working-space-management/
├── package.json                 # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml           # api + web + postgres + redis
├── docker-compose.prod.yml
├── .env.example
├── README.md
├── apps/
│   ├── api/                      # NestJS backend
│   │   ├── Dockerfile
│   │   ├── nest-cli.json
│   │   ├── drizzle.config.ts
│   │   ├── src/
│   │   │   ├── main.ts           # bootstrap, OpenAPI, global pipes/filters
│   │   │   ├── app.module.ts
│   │   │   ├── config/           # env schema (zod), config service
│   │   │   ├── common/           # guards, interceptors, decorators, errors
│   │   │   │   ├── guards/        # JwtAuthGuard, RolesGuard, OrgScopeGuard
│   │   │   │   ├── interceptors/  # audit-log, transform, idempotency
│   │   │   │   ├── filters/       # http-exception, db-constraint filter
│   │   │   │   └── decorators/    # @CurrentUser, @Roles, @OrgScoped
│   │   │   ├── db/
│   │   │   │   ├── schema/        # Drizzle table definitions (per domain)
│   │   │   │   ├── migrations/    # generated + raw SQL (exclusion, MVs)
│   │   │   │   └── client.ts
│   │   │   ├── modules/
│   │   │   │   ├── auth/
│   │   │   │   ├── organizations/
│   │   │   │   ├── locations/
│   │   │   │   ├── users/
│   │   │   │   ├── plans/
│   │   │   │   ├── memberships/
│   │   │   │   ├── resources/
│   │   │   │   ├── bookings/
│   │   │   │   ├── day-passes/
│   │   │   │   ├── billing/       # invoices, payments, credits, dunning
│   │   │   │   ├── access/        # gateway, adapters, permissions, events
│   │   │   │   ├── community/     # posts, comments, DMs, directory
│   │   │   │   ├── events/        # event mgmt, RSVP, ticketing
│   │   │   │   ├── visitors/
│   │   │   │   ├── notifications/
│   │   │   │   ├── calendar/      # iCal, Google, MS Graph sync
│   │   │   │   ├── analytics/
│   │   │   │   ├── ai/            # moderation, forecasting, pricing, sentiment
│   │   │   │   ├── branding/      # white-label config
│   │   │   │   └── webhooks/      # stripe, lock-provider inbound
│   │   │   ├── integrations/
│   │   │   │   ├── stripe/
│   │   │   │   ├── locks/         # kisi, salto, brivo, hakuna adapters
│   │   │   │   ├── email/
│   │   │   │   ├── sms/
│   │   │   │   └── llm/
│   │   │   └── jobs/              # BullMQ processors + schedulers
│   │   └── test/
│   │       ├── unit/
│   │       ├── integration/      # Supertest + Testcontainers
│   │       └── fixtures/
│   ├── web/                      # Next.js member portal + admin dashboard
│   │   ├── Dockerfile
│   │   ├── src/app/
│   │   │   ├── (member)/
│   │   │   ├── (admin)/
│   │   │   ├── (auth)/
│   │   │   └── (kiosk)/
│   │   ├── src/components/       # shadcn/ui based
│   │   ├── src/lib/api/          # generated OpenAPI client
│   │   └── tests/                # Playwright e2e
│   └── mobile/                   # React Native (Expo) — v1.1
├── packages/
│   ├── shared-types/             # DTOs, enums shared api <-> web
│   ├── api-client/               # generated typed client from OpenAPI
│   ├── config/                   # shared eslint/tsconfig/tailwind presets
│   └── domain/                   # pure business logic (pricing, proration, RRULE) — framework-free, 100% unit-tested
└── docs/
    ├── openapi.json              # generated spec artifact
    └── adr/                      # architecture decision records
```

The structure is grouped by bounded context, not by phase — each phase adds modules/tables without restructuring. `packages/domain` holds pure functions (proration maths, credit accounting, RRULE expansion, availability calculation) so the trickiest logic is testable in isolation without a database.

---

## Phase 1: Foundation, Auth & Multi-Tenancy

### Purpose
Establish the monorepo, containerised dev environment, database with migrations, configuration, structured logging, and the authentication/authorization backbone. Multi-tenancy (organization → location) and RBAC are foundational because every subsequent entity is org-scoped. After this phase a developer can register an organization, create users with roles, log in, and call a protected, org-scoped endpoint.

### Tasks

#### 1.1 — Monorepo & containerised dev environment

**What**: pnpm + Turborepo workspace with `apps/api`, `apps/web`, shared packages, and a `docker-compose.yml` that brings up API, web, Postgres 16, and Redis 7.

**Design**:
- `docker-compose.yml` services: `postgres` (image `postgres:16`, `POSTGRES_DB`, volume), `redis` (`redis:7`), `api` (build `apps/api`, depends_on postgres+redis, healthcheck on `/healthz`), `web` (build `apps/web`).
- Env loaded via a zod schema in `apps/api/src/config/env.schema.ts`:
```ts
export const envSchema = z.object({
  NODE_ENV: z.enum(['development','test','production']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_ACCESS_SECRET: z.string().min(32),
  JWT_REFRESH_SECRET: z.string().min(32),
  ACCESS_TOKEN_TTL: z.string().default('15m'),
  REFRESH_TOKEN_TTL_DAYS: z.coerce.number().default(30),
  STRIPE_SECRET_KEY: z.string().optional(),
  STRIPE_WEBHOOK_SECRET: z.string().optional(),
  LLM_API_KEY: z.string().optional(),
});
export type Env = z.infer<typeof envSchema>;
```
- `main.ts` boots Nest, applies a global `ValidationPipe({ whitelist: true, transform: true })`, mounts Swagger at `/docs`, registers the global exception filter and audit interceptor.
- `/healthz` returns `{ status, db, redis }` with dependency checks.

**Testing**:
- `Unit: env.schema with valid vars → parsed Env object with defaults applied`
- `Unit: env.schema missing JWT_ACCESS_SECRET → ZodError naming the field`
- `Integration (real): docker compose up → GET /healthz returns 200 with db:ok, redis:ok`
- `Integration: GET /docs serves Swagger UI (200, HTML contains "openapi")`

#### 1.2 — Database schema bootstrap & enums

**What**: Drizzle schema for the core identity tables plus all enum types, and the migration pipeline.

**Design**:
- Create all PostgreSQL enums from Data Model Suggestion 1 (`membership_status`, `plan_type`, `billing_cycle`, `booking_status`, `resource_type`, `invoice_status`, `payment_status`, `payment_method_type`, `access_credential_type`, `access_event_type`, `post_type`, `moderation_status`, `event_rsvp_status`, `visitor_status`, `notification_channel`, `user_role`).
- Enable extensions in the first raw migration: `uuid-ossp`, `btree_gist`, `pgcrypto`.
- Implement tables `organizations`, `locations`, `users`, `user_org_roles`, `corporate_accounts`, `corporate_members` exactly as specified in Suggestion 1 (UUID PKs, soft-delete `deleted_at`, partial indexes).
- `drizzle.config.ts` points at `src/db/schema` and emits to `src/db/migrations`. A `pnpm db:migrate` script applies generated migrations then any hand-written raw SQL migrations in order (filename-sorted).
- Each org-scoped table carries `organization_id`; an `OrgScopeGuard` (task 1.4) enforces tenant isolation at the query layer.

**Testing**:
- `Integration (Testcontainers): run all migrations against a fresh Postgres → all tables and enums exist (query information_schema)`
- `Integration: migrations are idempotent re-run → no error (drizzle tracks applied)`
- `Unit: inserting a user with duplicate email → unique violation surfaced as 409 by the constraint filter`

#### 1.3 — Authentication (register, login, refresh, logout)

**What**: Email/password auth issuing short-lived JWT access tokens and rotating refresh tokens, plus password reset.

**Design**:
- Passwords hashed with `argon2id`. Endpoints:
  - `POST /auth/register` → `{ email, password, firstName, lastName, organizationName? }` → creates user; if `organizationName` present, creates an org and assigns `org_admin`, else user is unattached. Returns `{ user, accessToken, refreshToken }`.
  - `POST /auth/login` → `{ email, password }` → `{ accessToken, refreshToken }` or `401`.
  - `POST /auth/refresh` → `{ refreshToken }` → rotates: old token invalidated in Redis, new pair returned. Reuse of a rotated token → `401` + revoke entire token family (theft detection).
  - `POST /auth/logout` → invalidates the refresh token.
  - `POST /auth/forgot-password` / `POST /auth/reset-password` → time-boxed token (Redis, 1h TTL).
- Access token claims: `{ sub: userId, email, roles: [{orgId, role}], type: 'access' }`. Refresh tokens are opaque random IDs stored in Redis keyed `refresh:{userId}:{tokenId}` with TTL.
- `JwtAuthGuard` validates the access token; `@CurrentUser()` decorator injects the principal.

**Testing**:
- `Integration: register with new email → 201, user persisted, valid JWT returned`
- `Integration: register with existing email → 409`
- `Integration: login wrong password → 401, no token`
- `Integration: refresh with valid token → new pair, old token rejected on reuse (401 + family revoked)`
- `Unit: argon2 hash then verify → true; verify wrong password → false`
- `Integration: protected route without Authorization header → 401`

#### 1.4 — RBAC & organization scoping

**What**: Role-based guards and a tenant-isolation mechanism so every request is scoped to the caller's organization(s).

**Design**:
- Roles from `user_role` enum. `@Roles('org_admin','location_manager')` decorator + `RolesGuard` check the principal's role for the resolved org.
- Org resolution: org id comes from the route (`/orgs/:orgId/...`) or a `X-Org-Id` header; `OrgScopeGuard` verifies the principal has a non-revoked `user_org_roles` row for that org, else `403`.
- A request-scoped `TenantContext` carries `{ userId, orgId, role }`; repository helpers automatically append `WHERE organization_id = :orgId` on org-scoped reads.

**Testing**:
- `Integration: member calls an org_admin-only endpoint → 403`
- `Integration: org_admin of Org A requests Org B resource → 403 (OrgScopeGuard)`
- `Integration: location_manager reads own org locations → 200`
- `Unit: RolesGuard with super_admin → bypasses org role check (200)`

### Definition of Done
Phase-wide DoD checklist (below) plus: a member can register, log in, refresh, and reach an org-scoped, role-gated endpoint; cross-tenant access is provably blocked by integration tests against a real Postgres.

---

## Phase 2: Membership Plans, Resources & Locations (Operator Configuration)

### Purpose
Give operators the configuration surface that everything else depends on: locations, floors, bookable resources with pricing/amenities, and membership plan definitions (the templates members later subscribe to). No bookings or billing yet — this phase produces the catalogue.

### Tasks

#### 2.1 — Locations & floors CRUD

**What**: CRUD for `locations` and `floors` under an organization.

**Design**:
- `POST/GET/PATCH/DELETE /orgs/:orgId/locations`, nested `/locations/:id/floors`. DTOs validate address, `country_code` (ISO 3166-1 alpha-2), `timezone` (IANA), and `operating_hours` JSONB shape:
```ts
type OperatingHours = Partial<Record<'mon'|'tue'|'wed'|'thu'|'fri'|'sat'|'sun',
  { open: string; close: string } | null>>; // "HH:mm", null = closed
```
- Soft delete sets `deleted_at`; lists exclude soft-deleted. Only `org_admin`/`location_manager` may write.

**Testing**:
- `Unit: operating_hours with "25:00" → ValidationError`
- `Integration: create location → 201 with generated slug unique per org`
- `Integration: duplicate slug within org → 409`
- `Integration: delete location → soft-deleted, excluded from list`

#### 2.2 — Resources, amenities & resource types

**What**: CRUD for `resources` (desks, meeting rooms, phone booths, etc.), `amenities`, and `resource_amenities`.

**Design**:
- Implement `resources` table per Suggestion 1 (capacity, hourly/half-day/daily rates, `credit_cost`, physical attribute booleans, `min/max_booking_minutes`, `buffer_minutes`, `advance_booking_days`, `access_point_id`). Add a JSONB `custom_fields` column (Suggestion 3 hybrid) for operator-defined attributes to avoid migration-per-field friction.
- `floors` and `resources` are location-scoped; `OrgScopeGuard` resolves org via the location.
- Amenity tagging via the `resource_amenities` join table; endpoint to attach/detach.

**Testing**:
- `Unit: resource with max_booking_minutes < min_booking_minutes → ValidationError`
- `Integration: create meeting room with amenities → resource + join rows persisted`
- `Integration: list resources filtered by resource_type and is_active`
- `Integration: custom_fields arbitrary JSON round-trips intact`

#### 2.3 — Membership plan definitions

**What**: CRUD for `membership_plans` with resource inclusions, credit rules, access rules, and location scope.

**Design**:
- Implement `membership_plans` + `plan_locations` per Suggestion 1: `plan_type`, `billing_cycle`, `base_price`, included desk hours/room minutes/guest passes/credits, `credit_rollover` + `max_rollover_credits`, access window (`access_24_7`, `access_start_time`, `access_end_time`, `access_days[]`), `all_locations` flag, community permissions, trial/commitment/cancellation-notice fields.
- When `all_locations = false`, require ≥1 `plan_locations` row. Stripe product/price IDs left null here; populated in Phase 4.
- Validation: `base_price >= 0`; if `credit_rollover` then `max_rollover_credits` required; `access_days` ⊆ {1..7}.

**Testing**:
- `Unit: plan with credit_rollover=true and null max_rollover_credits → ValidationError`
- `Unit: access_days=[1,2,8] → ValidationError (8 invalid)`
- `Integration: create non-all-locations plan with no plan_locations → 400`
- `Integration: list publicly-listed active plans for a location`

### Definition of Done
Operators can fully configure a space's catalogue (locations, floors, resources+amenities, plans) via the API; pure validation rules covered by unit tests; CRUD covered by integration tests.

---

## Phase 3: Booking Engine (Core Value Proposition)

### Purpose
The heart of the product: real-time availability and conflict-free booking of resources. This phase delivers the DB-level exclusion constraint, availability calculation, the booking lifecycle state machine, recurrence, and check-in/out — the capability that makes the platform usable as a booking system even before billing is wired up.

### Tasks

#### 3.1 — Bookings table, exclusion constraint & lifecycle

**What**: The `bookings` table with the GiST `no_overlapping_bookings` exclusion constraint and the booking state machine.

**Design**:
- Implement `bookings` per Suggestion 1: `start_time`/`end_time`, generated `time_range TSTZRANGE`, `total_price`, `credits_used`, `recurrence_rule`, `parent_booking_id`, check-in/out timestamps, `CHECK (end_time > start_time)`.
- Raw SQL migration adds the exclusion constraint:
```sql
ALTER TABLE bookings ADD CONSTRAINT no_overlapping_bookings
  EXCLUDE USING gist (resource_id WITH =, time_range WITH &&)
  WHERE (status NOT IN ('cancelled','no_show'));
```
- State machine: `pending → confirmed → checked_in → completed`; any non-terminal → `cancelled`; `confirmed → no_show`. Encode allowed transitions in `packages/domain/bookingStateMachine.ts`; reject illegal transitions with `409`.
- On `INSERT` that violates the exclusion constraint, the DB raises error `23P01`; the `DbConstraintFilter` maps it to `409 Conflict` `{ code: 'BOOKING_CONFLICT' }`.

**Testing**:
- `Integration (Testcontainers, REAL): two overlapping confirmed bookings same resource → second insert → 409 BOOKING_CONFLICT` (must hit a real Postgres — the exclusion constraint cannot be mocked)
- `Integration: adjacent non-overlapping bookings (10:00–11:00, 11:00–12:00) → both succeed`
- `Integration: overlapping where one is cancelled → second succeeds (WHERE clause excludes cancelled)`
- `Unit: state machine confirmed→checked_in allowed; completed→pending rejected`

#### 3.2 — Availability calculation & booking rules

**What**: Compute free/busy slots for a resource over a window, honouring operating hours, buffers, min/max duration, and advance-booking limits.

**Design**:
- Pure function in `packages/domain`:
```ts
function computeAvailability(input: {
  resource: ResourceRules;        // min/max minutes, buffer, advance days
  locationHours: OperatingHours;  // per weekday windows in location tz
  existingBookings: { start: Date; end: Date }[];
  window: { from: Date; to: Date };
  tz: string;                     // IANA
}): { start: Date; end: Date }[]  // bookable gaps
```
- Algorithm: clip window to advance-booking horizon → intersect with operating hours per day → subtract existing bookings expanded by `buffer_minutes` on each side → drop gaps shorter than `min_booking_minutes`.
- Endpoint `GET /resources/:id/availability?from&to` returns gaps. A 30s Redis cache per `(resourceId, day)` is invalidated on any booking write to the resource.
- Booking creation re-validates rules server-side (duration within min/max, within advance horizon, within operating hours) before insert; the exclusion constraint is the final backstop.

**Testing**:
- `Unit: resource with 60-min buffer, existing 10:00–11:00 → 11:00–11:30 not offered (buffer), 11:30+ offered`
- `Unit: window beyond advance_booking_days → clipped to horizon`
- `Unit: booking outside operating hours → rejected by rule check (no DB call)`
- `Integration: availability reflects a just-created booking (cache invalidated)`

#### 3.3 — Booking CRUD, recurrence & check-in/out

**What**: Create/cancel/reschedule bookings, RFC 5545 recurrence expansion, attendees, and check-in/out.

**Design**:
- `POST /resources/:id/bookings` `{ startTime, endTime, title?, attendees?, recurrenceRule? }`. If `recurrenceRule` (RRULE) present, expand into N child bookings (cap N, e.g. 90 occurrences) sharing `parent_booking_id`; each child inserted in one transaction — if any conflicts, roll back the whole series and return `409` listing the conflicting occurrence.
- RRULE expansion uses the `rrule` library wrapped in `packages/domain/recurrence.ts` so it is unit-testable.
- `POST /bookings/:id/check-in` / `check-out` set timestamps and advance state; `DELETE /bookings/:id` → cancel (state-machine guarded).
- `booking_attendees` for meeting rooms (internal users by id, external by email).
- Webhook/event emitted (`booking.created`, `booking.cancelled`, `booking.checked_in`) onto a BullMQ topic for downstream consumers (access sync in Phase 6, notifications in Phase 7, calendar sync in Phase 8).

**Testing**:
- `Unit: weekly RRULE for 4 weeks → 4 occurrence datetimes`
- `Integration: create recurring series, one occurrence conflicts → whole series rolled back, 409 with conflicting date`
- `Integration: check-in a confirmed booking → status checked_in, checked_in_at set`
- `Integration: cancel completed booking → 409 (illegal transition)`
- `Integration: booking emits booking.created job onto queue (assert job enqueued)`

### Definition of Done
Members/staff can view real availability and create conflict-free single and recurring bookings; double-booking is impossible (proven against real Postgres); lifecycle transitions enforced; booking events published for downstream phases.

---

## Phase 4: Billing & Payments (Stripe)

### Purpose
Turn the catalogue and bookings into revenue: members subscribe to plans, get invoiced on a recurring cycle, pay via Stripe, and the platform reconciles via webhooks. Includes proration for mid-cycle changes, credit accounting, dunning, and discount codes — the billing edge cases called out in research §4.

### Tasks

#### 4.1 — Stripe gateway abstraction & customer/product sync

**What**: A `PaymentGateway` interface with a Stripe implementation; sync plans→Stripe products/prices and members→Stripe customers.

**Design**:
```ts
interface PaymentGateway {
  ensureCustomer(user: User): Promise<string>;            // returns external customer id
  ensureProduct(plan: MembershipPlan): Promise<{ productId: string; priceId: string }>;
  createSubscription(p: { customerId; priceId; trialDays?; couponId? }): Promise<SubResult>;
  updateSubscription(subId: string, p: { priceId?; prorationDate?: number }): Promise<SubResult>;
  cancelSubscription(subId: string, atPeriodEnd: boolean): Promise<void>;
  chargeOneOff(p: { customerId; amount; currency; idempotencyKey; description }): Promise<Charge>;
  refund(p: { chargeId; amount?; reason? }): Promise<Refund>;
}
```
- All write calls pass an idempotency key (Stripe requirement, standards.md). On plan create/update (Phase 2 endpoints) trigger `ensureProduct` to backfill `stripe_product_id`/`stripe_price_id`.
- Stripe is optional in dev: if `STRIPE_SECRET_KEY` unset, a `NullPaymentGateway` logs and returns deterministic fake ids so the rest of the system runs.

**Testing**:
- `Integration (mocked Stripe via nock): ensureCustomer twice for same user → one Stripe call, cached external id`
- `Unit: NullPaymentGateway returns stable fake ids when Stripe disabled`
- `Unit: chargeOneOff always includes idempotencyKey`

#### 4.2 — Memberships subscribe / change / cancel with proration

**What**: Subscribe a user to a plan, change plans mid-cycle with proration, and cancel with notice rules.

**Design**:
- Implement `memberships`, `membership_status_changes`, `credit_transactions` per Suggestion 1.
- `POST /orgs/:orgId/memberships` `{ userId, planId, locationId?, startDate, couponCode? }` → creates Stripe subscription, persists membership (`status=active` or `pending` if trial), seeds `credit_balance` from plan, grants access (Phase 6 consumes the event).
- Proration maths is a pure function in `packages/domain/proration.ts`:
```ts
function prorate(p: { oldPrice; newPrice; periodStart; periodEnd; changeDate }): {
  credit: number; charge: number; netAmount: number;
}
```
- `PATCH /memberships/:id` `{ newPlanId }` → compute proration, call `updateSubscription`, write a credit/charge line, log a `membership_status_changes` row.
- `DELETE /memberships/:id` → honour `cancellation_notice_days`; set `cancellation_effective_date`, `status` stays active until then; Stripe cancel-at-period-end.

**Testing**:
- `Unit: prorate half-way through a 30-day month, $100→$200 → credit ≈ $50, charge ≈ $100, net ≈ $50`
- `Unit: prorate same price → net 0`
- `Integration (mocked Stripe): subscribe → membership active, credit_balance seeded, status-change logged`
- `Integration: cancel with 30-day notice → effective_date = now+30d, status still active`

#### 4.3 — Invoices, line items, payments & credit ledger

**What**: Invoice generation, line items, payment records, and the credit-transaction ledger.

**Design**:
- Implement `invoices`, `invoice_line_items`, `payments`, `discount_codes` per Suggestion 1 (generated `amount_due`, dunning fields). Invoice numbers from a per-org sequence (`INV-{org}-{0000001}`).
- Credit ledger is append-only: every debit/credit writes a `credit_transactions` row with `balance_after`; the membership `credit_balance` is updated in the same transaction. A nightly job verifies `balance_after` of the latest row equals `credit_balance` (consistency check).
- Booking credit consumption (Phase 3) deducts via this ledger when a resource has `credit_cost`.

**Testing**:
- `Unit: invoice total = sum(line amounts) + tax - discount`
- `Integration: book a credit-costed resource → credit_transactions debit row, balance_after correct`
- `Integration: spend more credits than balance → 402/400, no debit row written`
- `Integration: invoice_number is unique and monotonic per org`

#### 4.4 — Stripe webhooks, reconciliation & dunning

**What**: Inbound Stripe webhook handler with signature verification and idempotent processing, plus the dunning workflow.

**Design**:
- `POST /webhooks/stripe` verifies the `Stripe-Signature` header against `STRIPE_WEBHOOK_SECRET`; unverified → `400`, no processing. Verified events enqueued to BullMQ keyed by Stripe `event.id` for idempotent, retried processing.
- Handle `invoice.paid` (mark invoice paid, payment row), `invoice.payment_failed` (start dunning), `customer.subscription.updated/deleted` (reconcile membership status), `charge.refunded`.
- Dunning: a repeatable job escalates `dunning_attempts` on schedule (e.g. day 1, 3, 7), sends notifications (Phase 7), and on final failure sets membership `suspended` and revokes access (Phase 6 event).

**Testing**:
- `Integration: webhook with invalid signature → 400, nothing enqueued`
- `Integration: webhook with valid signature → 200, job enqueued`
- `Integration: same event.id delivered twice → processed once (idempotent)`
- `Integration: payment_failed → invoice overdue, dunning job scheduled`
- `Integration: 3rd dunning failure → membership suspended, access-revoke event emitted`

### Definition of Done
A full subscribe→invoice→pay→reconcile loop works against Stripe test mode; proration and credit maths are unit-proven; webhooks are signature-verified and idempotent; dunning suspends delinquent members.

---

## Phase 5: Member Portal & Operator Admin Dashboard (Web)

### Purpose
Expose the API through usable web surfaces. Members self-serve (browse plans, book, view invoices); operators manage plans, resources, members, and see a revenue/occupancy overview. This is the first end-user-facing increment and validates the API shape. Can be developed in parallel with Phase 6 once Phases 1–4 are done.

### Tasks

#### 5.1 — Generated API client & auth flow (web)

**What**: A typed OpenAPI client in `packages/api-client`, plus login/refresh handling in Next.js.

**Design**:
- CI step generates `openapi.json` from the Nest app and runs `openapi-typescript` + a thin fetch wrapper into `packages/api-client`.
- Next.js stores the access token in memory and the refresh token in an httpOnly, Secure, SameSite=Lax cookie; a middleware silently refreshes on 401.
- Route groups: `(auth)` login/register, `(member)` portal, `(admin)` dashboard, `(kiosk)` front-desk. `(admin)` and `(kiosk)` gated by role from the token.

**Testing**:
- `E2E (Playwright): login with valid creds → redirected to portal, token cookie set`
- `E2E: expired access token → silent refresh → request succeeds`
- `E2E: member navigates to /admin → redirected/403`

#### 5.2 — Member portal: booking, plans, billing history

**What**: Member-facing pages for browsing plans, booking resources via an availability calendar, and viewing invoices/payment history.

**Design**:
- Pages: `/portal/book` (resource picker + availability calendar drawing on `GET /resources/:id/availability`, optimistic UI with conflict toast on `409 BOOKING_CONFLICT`), `/portal/bookings` (upcoming/past, cancel), `/portal/plans` (browse + subscribe via Stripe Checkout/Elements), `/portal/billing` (invoices, download receipt).
- Calendar component built on shadcn/ui + a date library; renders gaps from the availability endpoint; submitting a slot calls the booking endpoint.

**Testing**:
- `E2E: member books an available slot → confirmation shown, booking appears in list`
- `E2E: member books a slot that was just taken → conflict toast, no booking created`
- `E2E: member views invoice list → matches API data`

#### 5.3 — Operator admin dashboard

**What**: Admin pages for plans, resources/locations, member management, and an overview with revenue + occupancy summaries.

**Design**:
- Pages under `(admin)`: locations/resources/plans CRUD forms (reusing Phase 2 endpoints), members table (search, view, suspend, change plan), and an overview consuming the analytics endpoints (Phase 9 placeholder returns counts until then: active members, MRR estimate, today's bookings, occupancy %).
- Kiosk surface `(kiosk)`: walk-in day-pass sale and booking check-in (front_desk role).

**Testing**:
- `E2E (admin): create a plan via form → appears in member /portal/plans`
- `E2E (admin): suspend a member → member status reflects suspended`
- `E2E (kiosk): sell a day pass at front desk → day_pass + payment recorded`

### Definition of Done
Members can self-serve booking/billing and operators can run day-to-day configuration and member management entirely through the web UI; core flows covered by Playwright e2e against a running stack.

---

## Phase 6: Access Control Integration

### Purpose
Connect bookings and memberships to physical access: automatically grant/revoke door and WiFi permissions, sync them to fragmented smart-lock vendors behind an abstraction gateway, and record an audit trail of access events. This is a key differentiator (vendor-neutral gateway) and depends on Phases 3–4 for the membership/booking events that drive permissions.

### Tasks

#### 6.1 — Lock provider gateway & adapters

**What**: A `LockProvider` interface with adapters for Kisi and Salto (Brivo/Hakuna stubbed to the same interface).

**Design**:
```ts
interface LockProvider {
  readonly name: 'kisi'|'salto'|'brivo'|'hakuna';
  grantAccess(p: { externalUserId; accessPointExternalId; validFrom; validUntil?; schedule? }): Promise<{ providerGrantId: string }>;
  revokeAccess(p: { providerGrantId: string }): Promise<void>;
  ensureUser(user: User): Promise<{ externalUserId: string }>;
  unlock(accessPointExternalId: string): Promise<void>;   // remote unlock
}
```
- Adapters wrap each vendor REST API (Kisi `docs.kisi.io`, Salto KS Connect) with auth, retry/backoff, and error normalization. Provider chosen per `access_points.provider`; config in `provider_config` JSONB.
- A `LockProviderRegistry` resolves the adapter by access point.

**Testing**:
- `Integration (mocked vendor API via nock): Kisi grantAccess → maps to vendor payload, returns providerGrantId`
- `Integration (mocked): vendor 5xx → retried with backoff, then surfaced as a sync failure`
- `Unit: registry resolves correct adapter for provider='salto'`

#### 6.2 — Permission derivation & provider sync

**What**: Derive `access_permissions` from active memberships and confirmed bookings, then sync them to the lock provider asynchronously.

**Design**:
- Implement `access_points`, `resource_access_points`, `access_credentials`, `access_permissions` per Suggestion 1.
- A consumer of the booking/membership event topic (Phase 3/4) computes permissions: membership grants standing access to its location's general access points within the plan's access window (`access_days`, start/end time); a confirmed booking grants access to the resource's linked access points for `[start-buffer, end+buffer]`.
- Each permission write enqueues a `access.sync` job that calls the provider and sets `synced_to_provider`/`last_synced_at`. Revocation on cancel/suspend/expiry enqueues a revoke job.
- Suspended membership (from dunning) → all derived permissions revoked.

**Testing**:
- `Integration: confirm booking → access_permission created for resource's access points within booking window + buffer`
- `Integration: cancel booking → permission revoked, revoke job enqueued`
- `Integration: suspend membership → standing permissions revoked`
- `Integration: plan access window mon–fri 08:00–20:00 → permission schedule fields set accordingly`

#### 6.3 — Access events ingestion & audit log

**What**: Inbound webhook from lock providers recording entry/exit/denied events into `access_events`, plus a remote-unlock endpoint.

**Design**:
- `POST /webhooks/locks/:provider` verifies the provider signature, normalizes the payload, and inserts an `access_events` row (`event_type`, `success`, `raw_event` JSONB).
- `POST /access-points/:id/unlock` (front_desk/admin or member with a valid current permission) calls `LockProvider.unlock`.
- `access_events` created with a monthly partition strategy (raw SQL: declarative `PARTITION BY RANGE (occurred_at)` with a job creating next-month partitions) per Suggestion 1's high-write warning.

**Testing**:
- `Integration: lock webhook valid signature → access_events row inserted`
- `Integration: lock webhook invalid signature → 400, no row`
- `Integration: member without current permission calls unlock → 403`
- `Integration: events route into the correct monthly partition`

### Definition of Done
Memberships and bookings automatically provision/deprovision door access through a vendor-neutral gateway with retry-safe async sync; all access activity is auditably logged; remote unlock works for authorized users.

---

## Phase 7: Notifications

### Purpose
Reliable multi-channel (email, SMS, in-app) notifications for booking confirmations/reminders, billing/dunning, visitor arrivals, and community activity — wired to the events emitted by earlier phases. Respects per-user preferences (GDPR-friendly opt-outs).

### Tasks

#### 7.1 — Notification engine, channels & preferences

**What**: A channel-abstracted notification service consuming domain events, honouring `notification_preferences`, persisting to `notifications`.

**Design**:
- Implement `notification_preferences`, `notifications` per Suggestion 1.
- `NotificationChannel` interface with `EmailChannel` (SMTP/SendGrid), `SmsChannel` (Twilio), `InAppChannel` (DB row + websocket push). A `NotificationService.notify(userId, category, payload)` resolves enabled channels from preferences, renders the template, enqueues a per-channel send job (retry/backoff), and records `delivery_status`.
- Templates per category support white-label branding (Phase 5/branding config): from-name/address and header/footer HTML.

**Testing**:
- `Unit: user with email-only prefs for billing_alerts → SMS channel skipped`
- `Integration (mocked SMTP): notify booking_confirmation → email send job enqueued, notification row created`
- `Integration: send failure → retried, delivery_status reflects attempts`
- `Integration: in-app notification → row created, unread index populated`

#### 7.2 — Event-driven triggers

**What**: Subscribe the notification engine to booking, billing, visitor, and community events.

**Design**:
- Map events to categories: `booking.created→booking_confirmations`, scheduled `booking.reminder` (repeatable job firing N hours before `start_time`), `invoice.payment_failed→billing_alerts`, dunning escalations, `visitor.checked_in→visitor_arrivals` (notify host), `community.mention→community_mentions`.
- Reminders scheduled at booking time and cancelled if the booking is cancelled.

**Testing**:
- `Integration: create booking 2h out with 1h reminder pref → reminder job scheduled at start-1h`
- `Integration: cancel booking → scheduled reminder removed`
- `Integration: payment_failed event → billing_alert notification sent (respecting prefs)`

### Definition of Done
Domain events produce preference-respecting, branded, retry-safe notifications across email/SMS/in-app; reminders schedule and cancel correctly.

---

## Phase 8: Calendar Interop & Day-Pass / Visitor Management

### Purpose
Round out front-desk and member-experience essentials: iCal/Google/Outlook calendar sync (RFC 5545), standalone day-pass sales, and visitor pre-registration with temporary access. These are largely independent of each other and can be parallelised. Day passes and visitors complete the walk-in and guest flows from the MVP/v1.1 scope.

### Tasks

#### 8.1 — iCalendar export & two-way calendar sync

**What**: RFC 5545 feed of a member's bookings plus OAuth-based push to Google Calendar and Microsoft Graph.

**Design**:
- `GET /calendar/:userToken.ics` returns a signed, read-only iCal feed (VEVENTs from the user's bookings) consumable by any calendar client (RFC 5545).
- OAuth 2.0 connect flow stores per-user Google/Microsoft tokens (encrypted); on `booking.created/updated/cancelled` events, a sync job upserts/deletes the corresponding calendar event via the provider API. Conflicts resolved last-write-wins with the platform as source of truth.

**Testing**:
- `Unit: booking → VEVENT with correct DTSTART/DTEND/UID/RRULE`
- `Integration (mocked Google API): booking.created → calendar insert called with mapped fields`
- `Integration: booking.cancelled → calendar event deleted`
- `Integration: .ics feed validates against an iCal parser`

#### 8.2 — Day-pass & drop-in sales

**What**: Online and walk-in purchase of single-day passes, tied to a one-off Stripe charge.

**Design**:
- Implement `day_passes` per Suggestion 1 (anonymous walk-ins allowed via `guest_name/email`). `POST /locations/:id/day-passes` `{ passDate, userId? | guest{name,email}, paymentMethod }` → `chargeOneOff` (or cash at kiosk) → day-pass row + payment row → grants access for that day (Phase 6 permission with the day's window).
- Front-desk kiosk surface (Phase 5) calls this for in-person cash/card sales.

**Testing**:
- `Integration: online day-pass purchase → Stripe charge (mocked), day_pass + payment rows, access permission for pass_date`
- `Integration: walk-in cash day pass → recorded with payment_method=cash, no Stripe call`
- `Integration: check-in/out timestamps recorded`

#### 8.3 — Visitor management & temporary access

**What**: Visitor pre-registration, host notification, check-in/out, and temporary credential issuance.

**Design**:
- Implement `visitor_passes` per Suggestion 1. `POST /locations/:id/visitors` (host) `{ visitorName, email?, expectedArrival, expectedDeparture?, accessPointIds? }` → notify host on creation, optionally pre-issue a temporary PIN/QR credential, and create scoped `access_permissions` (FK `visitor_pass_id`) valid for the visit window.
- Kiosk check-in marks `checked_in`, notifies host (Phase 7), activates the temporary credential; check-out revokes it.

**Testing**:
- `Integration: pre-register visitor → visitor_pass + host notification + temp access permission scoped to window`
- `Integration: visitor check-in → status checked_in, host notified, credential active`
- `Integration: visitor check-out → credential revoked`

### Definition of Done
Members' bookings sync to external calendars and export as valid iCal; day passes (online + walk-in) sell and grant access; visitors pre-register, get temporary access, and trigger host notifications.

---

## Phase 9: Community, Events & Analytics

### Purpose
Deliver the retention-driving differentiators incumbents under-serve: a moderated community feed, member directory and DMs, event management with RSVP/ticketing, and the operator analytics (occupancy, utilisation, revenue) via materialized views. Community/events and analytics are independent and can be parallelised.

### Tasks

#### 9.1 — Community feed, comments, likes & directory

**What**: Posts, threaded comments, likes, member directory, and direct messages.

**Design**:
- Implement `community_posts`, `post_likes`, `post_comments`, `member_profiles`, `direct_message_threads/participants/messages` per Suggestion 1. Denormalized `like_count`/`comment_count` maintained by DB triggers (raw SQL migration) to avoid update anomalies (Suggestion 1 con).
- Posting gated by the plan's `community_post_allowed`. Feed `GET /orgs/:orgId/feed` paginated, org- or location-scoped, excludes soft-deleted and non-visible (moderation) posts. Full-text search via Postgres `tsvector`.

**Testing**:
- `Integration: member on a plan with community_post_allowed=false → 403 on post`
- `Integration: like a post → like_count incremented by trigger; unlike → decremented`
- `Integration: threaded comment reply → parent_comment_id set, appears nested`
- `Integration: DM between two members → thread + message rows, unread tracking`

#### 9.2 — Event management, RSVP & ticketing

**What**: Create events (optionally linked to a room booking), RSVP, and paid ticketing.

**Design**:
- Implement `events`, `event_rsvps` per Suggestion 1. Event creation gated by `event_creation_allowed`. Optional `booking_id` links a room reservation (created via Phase 3, conflict-checked). Paid events: RSVP triggers `chargeOneOff`; `event_rsvps.payment_id` links it. `max_attendees` enforced with waitlisting (`waitlisted` status); denorm `rsvp_count`/`attendee_count` via triggers.

**Testing**:
- `Integration: RSVP to free event → status attending, count incremented`
- `Integration: RSVP to full event → status waitlisted`
- `Integration: paid event RSVP → Stripe charge (mocked), payment linked`
- `Integration: event linked to room booking that conflicts → 409`

#### 9.3 — Analytics & materialized views

**What**: Occupancy/utilisation and revenue analytics powering the admin overview, backed by materialized views.

**Design**:
- Create `mv_daily_occupancy` and `mv_monthly_revenue` per Suggestion 1; refresh via a repeatable BullMQ job (e.g. every 15 min) `REFRESH MATERIALIZED VIEW CONCURRENTLY`.
- Endpoints: `GET /orgs/:orgId/analytics/occupancy?from&to&locationId?`, `/analytics/revenue?from&to`, `/analytics/membership` (active/churn/growth, plan-mix). Utilisation % = booked-hours / (available-hours × resources). These replace the Phase 5 placeholder overview numbers.

**Testing**:
- `Integration: seed bookings → refresh MV → occupancy endpoint returns correct total_hours_booked and utilisation %`
- `Integration: seed invoices → mv_monthly_revenue → revenue endpoint returns collected vs outstanding`
- `Integration: CONCURRENTLY refresh requires the unique index (present) — refresh succeeds`

### Definition of Done
Members engage via a moderated feed, directory, DMs, and events with RSVP/ticketing; operators see accurate occupancy, utilisation, and revenue analytics from materialized views feeding the admin overview.

---

## Phase 10: AI-Native Layer

### Purpose
Implement the differentiating AI capabilities absent from incumbents (research §AI-Augmentation): automated community moderation, member sentiment/churn signals, occupancy forecasting, and dynamic pricing recommendations. Built last because it consumes data and events produced by all prior phases. Every AI output is advisory/auditable, with human override.

### Tasks

#### 10.1 — LLM client & automated community moderation

**What**: A provider-agnostic LLM client and an AI moderation pass on new community posts/comments.

**Design**:
```ts
interface LlmClient {
  classify(p: { text: string; labels: string[]; instructions: string }): Promise<{ label: string; confidence: number; reason: string }>;
  summarize(p: { text: string; maxTokens?: number }): Promise<string>;
  score(p: { text: string; rubric: string }): Promise<{ score: number; rationale: string }>; // e.g. sentiment -1..1
}
```
- On `community.post.created`, a moderation job calls `classify` with labels `['ok','spam','harassment','inappropriate']`. Result sets `moderation_status`: `auto_approved` (ok, high confidence) or `flagged` (otherwise) for human review in the admin queue. Operators can configure thresholds. System prompt instructs the model on the community norms; the model never hard-deletes — it only flags.

**Testing**:
- `Unit: classify result 'spam' high confidence → moderation_status flagged`
- `Integration (mocked LLM): benign post → auto_approved, visible in feed`
- `Integration (mocked LLM): flagged post → hidden from feed, appears in admin moderation queue`
- `Integration: LLM unavailable → post defaults to flagged (fail-safe), not auto-approved`

#### 10.2 — Member sentiment & churn signals

**What**: Analyse member feedback/community activity and engagement to surface churn risk.

**Design**:
- A scheduled job aggregates per-member signals (booking frequency trend, last login, support/feedback sentiment via `LlmClient.score`, community participation) into a `member_risk_scores` table (`user_id`, `org_id`, `risk_score 0..1`, `top_factors JSONB`, `computed_at`). Surfaced on the admin member view with explanations; never auto-acts.

**Testing**:
- `Unit: declining booking trend + negative sentiment → higher risk score`
- `Integration (mocked LLM): compute scores → member_risk_scores rows with top_factors`
- `Integration: admin member view shows risk score + factors`

#### 10.3 — Occupancy forecasting & dynamic pricing recommendations

**What**: Forecast resource demand per location/time and recommend off-peak pricing adjustments.

**Design**:
- Forecasting uses a lightweight statistical model in TS over historical `mv_daily_occupancy`/bookings (e.g. seasonal-naïve + moving average by weekday/hour) — no Python service for MVP. Output stored in `occupancy_forecasts` (`location_id`, `resource_type`, `forecast_date`, `predicted_utilisation`, `confidence`).
- Dynamic pricing job reads forecasts and recommends rate deltas for low-predicted-utilisation windows, written to `pricing_recommendations` (advisory; operator applies via the resource/plan endpoints). Never auto-applies.

**Testing**:
- `Unit: seasonal-naïve forecast on synthetic weekly pattern → predicts next-week matching weekday`
- `Integration: low forecast utilisation window → discount recommendation generated`
- `Integration: recommendation is advisory — resource rate unchanged until operator applies it`

### Definition of Done
AI moderation, churn signals, forecasting, and pricing recommendations run as scheduled/event-driven advisory features behind a swappable LLM client, fail safe when the provider is unavailable, and never take destructive or financial action without operator confirmation.

---

## Phase 11: White-Label, Security Hardening & Compliance

### Purpose
Make the platform deployable as the operator's own brand and production-ready: white-label branding, OAuth2/OIDC for third-party access and SSO, GDPR/CCPA data-subject tooling, audit completeness, and OWASP hardening. This gates a 1.0 release.

### Tasks

#### 11.1 — White-label branding & custom domains

**What**: Per-org branding config driving portal/app appearance, email templates, and custom domains.

**Design**:
- Implement `branding_configs` per Suggestion 1. The web app resolves branding by host (custom domain → org) and themes colours/logo/CSS at render; emails (Phase 7) use the org's from-name/address and header/footer HTML. Custom-domain TLS via the deployment proxy (documented in compose/prod notes).

**Testing**:
- `E2E: request via custom domain → org's logo and primary colour applied`
- `Integration: email rendered with org's branded header/footer`

#### 11.2 — OAuth2/OIDC provider, SSO & public API tokens

**What**: OAuth 2.0 (RFC 6749) authorization for third-party integrations, OIDC SSO login, and scoped API tokens for the public API.

**Design**:
- OIDC login (Google/Microsoft/corporate IdP) maps to `users.oauth_provider/oauth_uid`. An authorization-code OAuth2 server issues scoped tokens (`bookings:read`, `members:read`, `billing:read`, etc., RFC 6750 bearer) for third-party apps. Webhook subscriptions (booking/payment/access events) let external systems integrate (standards.md recommendation).

**Testing**:
- `Integration: OIDC callback with valid code → user provisioned/linked, session issued`
- `Integration: API call with insufficient scope → 403`
- `Integration: webhook subscription → external endpoint receives signed event on booking.created`

#### 11.3 — GDPR/CCPA tooling, audit log & OWASP hardening

**What**: Data-subject access/export/erasure, comprehensive audit logging, and OWASP Top 10 mitigations.

**Design**:
- `GET /me/data-export` (machine-readable export of a member's data) and `DELETE /me` (right-to-erasure: anonymise PII while preserving financial/audit records as required by law). Consent flags stored on the user.
- The `AuditInterceptor` (Phase 1) records all writes to an `audit_log` (actor, action, entity, before/after, ip, timestamp) — doubles as ISO 27001/access-accountability evidence.
- OWASP pass: parameterised queries (Drizzle, A03), RBAC tests (A01), rate limiting on auth + webhooks (Redis), secrets only via env, security headers (helmet), dependency audit in CI (A06), structured request logging (A09).

**Testing**:
- `Integration: data-export returns all of a user's records in JSON`
- `Integration: erasure anonymises PII, retains invoices with member reference nulled/pseudonymised`
- `Integration: every write produces an audit_log entry (actor + before/after)`
- `Integration: brute-force login attempts → rate limited (429)`
- `Security: dependency audit and SAST pass in CI with no high findings`

### Definition of Done
The platform runs under an operator's own brand and domain, supports SSO and a scoped public API, satisfies GDPR/CCPA data-subject requests, logs all mutations, and passes an OWASP-aligned security review — ready for a 1.0 release.

---

## Phase 12 (v1.1): Mobile App & Scaling

### Purpose
Native mobile experience and the infrastructure to scale beyond ~50 locations. Sequenced last; not required for the web-first 1.0.

### Tasks

#### 12.1 — React Native (Expo) member app with offline BLE unlock

**What**: iOS/Android app reusing the API client, with BLE local credential caching for offline door unlock (research §4).

**Design**: Login, browse/book, billing, community, push notifications, and a wallet that caches a short-lived BLE credential locally so doors unlock even if connectivity briefly drops; the cached credential syncs/expires per the access permission window.

**Testing**:
- `E2E (Detox): book a resource from the app → appears in web portal`
- `Integration: cached BLE credential unlocks within validity window offline; rejected when expired`

#### 12.2 — Scaling: read replicas, partitioning, TimescaleDB option

**What**: Apply Suggestion 1's scaling plan and Suggestion 4's time-series option for high-write data.

**Design**: Move analytics queries to a read replica; partition `bookings` by org+month for large operators; optionally migrate `access_events` and occupancy sensor data to a TimescaleDB hypertable (Suggestion 4) for IoT-scale ingestion and continuous aggregates feeding forecasting (Phase 10).

**Testing**:
- `Integration: analytics queries hit the read replica (connection routing)`
- `Load: 10k access events/min sustained ingestion within latency budget`

### Definition of Done
Members have native iOS/Android apps with offline unlock; the system sustains 500+-location-scale read and access-event write loads.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Auth & Multi-Tenancy        ─── required by everything
    │
Phase 2: Operator Config (locations/resources/plans) ─── requires 1
    │
Phase 3: Booking Engine (core)                   ─── requires 2
    │
Phase 4: Billing & Payments (Stripe)             ─── requires 2,3
    │
    ├── Phase 5: Web Portal & Admin    ─── requires 1–4 · can parallel with 6
    ├── Phase 6: Access Control        ─── requires 3,4 · can parallel with 5
    │
Phase 7: Notifications                ─── requires 3,4 (events) [+6,8 triggers]
    │
Phase 8: Calendar / Day-Pass / Visitors ─── requires 3,4,6,7 · 8.1/8.2/8.3 parallelisable
    │
Phase 9: Community / Events / Analytics ─── requires 3,4,5 · community∥analytics
    │
Phase 10: AI-Native Layer             ─── requires 9 (data) + 3,4 (history)
    │
Phase 11: White-Label / Security / Compliance ─── requires 5,7; gates 1.0
    │
Phase 12 (v1.1): Mobile & Scaling     ─── requires a stable 1.0 (1–11)
```

Parallelism opportunities:
- After Phase 4: **Phase 5 (web)** and **Phase 6 (access control)** proceed concurrently.
- Within Phase 8: **8.1 calendar**, **8.2 day-pass**, **8.3 visitors** are independent.
- Within Phase 9: **community/events (9.1, 9.2)** and **analytics (9.3)** are independent.

Scope estimate: **Large** (12 phases, 36 tasks).

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pnpm test`); booking/billing/access tests that require real Postgres/Redis run via Testcontainers in CI.
3. Linting and formatting pass (`pnpm lint`, Prettier check).
4. Type checking passes (`pnpm typecheck` — `tsc --noEmit`, strict mode, no `any` escapes in new code).
5. Docker build succeeds for affected apps; `docker compose up` brings up a working stack.
6. The phase's feature works end-to-end (verified by an integration or Playwright e2e test, not only unit tests).
7. New configuration options added to `.env.example` and documented in `README.md`.
8. New/changed API endpoints appear in the generated OpenAPI 3.1 spec (`docs/openapi.json`) and render in Swagger UI/Redoc.
9. Database changes ship as version-controlled migrations (Drizzle-generated and/or raw SQL for exclusion constraints, materialized views, triggers, partitions); migrations are idempotent and tested against a fresh database.
10. All write endpoints produce audit-log entries; new PII fields are covered by the GDPR export/erasure logic (from Phase 11 onward).
11. An ADR is recorded in `docs/adr/` for any decision that diverges from this plan.
```

# Backend — Architecture & Work Plan

**Team:** Backend · **Owner:** (backend lead)
**Companion to** [ARCHITECTURE.md](ARCHITECTURE.md) · [TEAM_PLAYBOOKS.md](TEAM_PLAYBOOKS.md)

> **Your mission:** be the **single source of truth** — devices, tenants,
> telemetry, audio, and commands — as a **multi-tenant** service. You ingest what
> devices report, enforce that each brand only ever sees its own data, push audio
> over the air, and expose a clean API the dashboard builds on. **MQTT is the one
> unfamiliar piece — retire it first with a spike.**

---

## 1. What you must achieve

| # | Outcome | Proof |
|---|---|---|
| 1 | Devices register & report; online/offline is live | A device shows `online`, events persist |
| 2 | **Tenant isolation** — a brand can't read another's data | RLS blocks cross-tenant queries in tests |
| 3 | Audio upload → stored → pushed to a device with ack | End-to-end `audio_update` works |
| 4 | Analytics queries return per-tenant rollups fast | Dashboard endpoints < 200 ms on seed data |
| 5 | **OpenAPI published + seed data** so Frontend isn't blocked | Frontend builds against the mock |

---

## 2. Stack & packages

| Concern | Choice | Package(s) |
|---|---|---|
| Framework | NestJS | `@nestjs/core` `@nestjs/common` `@nestjs/platform-express` |
| Auth & tenancy | **BetterAuth** (organization plugin) | `better-auth` |
| ORM | Prisma | `prisma` `@prisma/client` |
| DB | PostgreSQL + TimescaleDB | (Timescale extension via SQL migration) |
| MQTT ingest | EMQX + consumer | `@nestjs/microservices` (MQTT transport) or `mqtt` (mqtt.js) |
| Queue / jobs / presence | Redis + BullMQ | `bullmq` `ioredis` |
| Object storage | MinIO / S3 | `@aws-sdk/client-s3` `@aws-sdk/s3-request-presigner` |
| Realtime to UI | WebSocket | `@nestjs/websockets` `@nestjs/platform-socket.io` `socket.io` |
| Validation / DTO | zod or class-validator | `zod` (+ `nestjs-zod`) or `class-validator` `class-transformer` |
| API docs | OpenAPI/Swagger | `@nestjs/swagger` |
| Testing | Jest + supertest | `jest` `supertest` `@nestjs/testing` |
| Deploy | Coolify + Docker | `Dockerfile`, `docker-compose.yml` |

---

## 3. Module structure (modular monolith)

```
backend/src/
├─ common/  guards(roles) · interceptors(tenant-context) · decorators · prisma
├─ modules/
│  ├─ auth/         # BetterAuth wiring, session guard, role guard
│  ├─ tenants/      # brands (= BetterAuth orgs), tiers — platform only
│  ├─ users/        # invite/manage users within an org
│  ├─ devices/      # registry, provisioning, assignment, lifecycle
│  ├─ products/     # tenant catalogue (SKUs)
│  ├─ locations/    # tenant store map
│  ├─ audio/        # upload, versioning, checksum, assign, push
│  ├─ commands/     # command channel + OTA orchestration
│  ├─ analytics/    # telemetry read models + rollups
│  └─ realtime/     # WS gateway (device status/events to UI)
├─ workers/
│  ├─ ingestion/    # MQTT consumer → DB + presence
│  └─ jobs/         # BullMQ: audio push fan-out, nightly rollups, offline sweep
└─ config/
```

---

## 4. Data model (Prisma) & multi-tenancy

**Tenant = BetterAuth `organizationId`.** Every domain row carries `tenantId`
(= organizationId). Isolation is enforced two ways: the app sets a request-scoped
tenant, and **Postgres RLS** blocks anything that slips through.

Key models: `Tenant` (mirrors org), `User`, `Product`, `Location`, `Device`,
`AudioClip`, `DeviceAudio`, `DeviceEvent` (**Timescale hypertable**), `Command`.
(Full ER diagram in [ARCHITECTURE.md §4](ARCHITECTURE.md).)

```typescript
// common/interceptors/tenant-context.interceptor.ts
const session = req.session;                          // BetterAuth
const isPlatform = session.user.role?.startsWith('PLATFORM_');
const tenantId = isPlatform
  ? (req.headers['x-tenant-id'] ?? null)              // platform impersonation
  : session.activeOrganizationId;                     // org = tenant
await prisma.$executeRawUnsafe(
  `SET app.tenant_id='${tenantId ?? ''}'; SET app.is_platform='${isPlatform}';`);
```

```sql
-- migration: RLS on every tenant-scoped table
ALTER TABLE device_event ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON device_event USING (
  current_setting('app.is_platform', true) = 'true'
  OR tenant_id = current_setting('app.tenant_id', true)::uuid);
```

**Timescale:** make `device_event` a hypertable; add **continuous aggregates**
for per-device/day counts so dashboards read rollups, not raw rows.

---

## 5. Auth (BetterAuth)
- Email/password + **organization plugin**: orgs = tenants, with members, roles, invitations built in.
- Roles: `PLATFORM_OWNER`, `PLATFORM_OPERATOR` (Baliyo) and `BRAND_ADMIN`, `BRAND_STAFF`, `BRAND_VIEWER` (tenant).
- `RolesGuard` reads the BetterAuth session; `TenantContextInterceptor` sets the RLS session vars.
- Platform users can **impersonate** a tenant (`x-tenant-id`) for onboarding/support.

---

## 6. MQTT ingestion (contract ① — you ↔ Electronics)

**Start with a spike (day 1–2):** EMQX in Docker → poke with [MQTTX](https://mqttx.app)
as a fake device → wire a NestJS consumer. *This is the only new tech; kill the risk early.*

- **Topics:** `t/{tenant}/d/{device}/{status|telemetry|event|cmd|ack}` (schemas in [ELECTRONICS_ARCHITECTURE §4](ELECTRONICS_ARCHITECTURE.md)).
- **Device auth + ACL in EMQX:** authenticate device by serial + token (EMQX built-in DB, or an **HTTP auth hook** to your `/mqtt/auth` endpoint); ACL restricts each device to **its own** `t/{tenant}/d/{device}/#`.
- **Ingestion worker:** subscribe → validate JSON → insert `DeviceEvent`, update `device.lastSeen` + presence in Redis → forward to the realtime gateway via Redis pub/sub.
- **Presence:** `status` retained + LWT; a Redis TTL key per device + an **offline-sweep job** flips stale devices to offline.

---

## 7. Commands, audio push & OTA (contract ③)
- **Upload:** `POST /audio` → stream to MinIO/S3 → create `AudioClip` (version, checksum, duration).
- **Push:** enqueue a BullMQ job → publish `cmd {audio_update, signedUrl, checksum, version}` to the device(s) → device downloads over HTTPS, verifies, swaps, `ack`s → mark `Command` acked. Track per-device ack state for the UI.
- **Firmware OTA:** same command pattern with a firmware artifact + A/B rollout.
- **Audio format** is a shared contract — agree WAV (PCM mono 16 kHz) vs MP3 + max size with Electronics.

---

## 8. API surface (contract ② — you → Frontend)

Publish an **OpenAPI spec** early + **seed data** so Frontend mocks against it.
Representative endpoints (all tenant-scoped except platform ones):

```
POST   /auth/*                        # BetterAuth (login, org switch, invites)
GET    /tenants            (platform) # list/create/suspend brands, set tier
GET    /devices                       # list (scoped); filters: status, location
POST   /devices/provision (platform)  # register serial → token
POST   /devices/:id/assign(platform)  # → tenant + location + product
GET    /devices/:id                   # detail + health
GET    /products · POST /products
GET    /locations · POST /locations
POST   /audio           (upload)      # → storage
POST   /audio/:id/push                # → command fan-out
GET    /analytics/overview            # KPIs (today's detections, plays, uptime)
GET    /analytics/devices/:id         # per-device series
GET    /analytics/dwell               # dwell distribution, per-store
WS     /realtime                      # device.status, device.event
```

---

## 9. Deployment (Coolify + Docker)
Resources in one Coolify project: `api` (NestJS), `worker` (same image, worker
entrypoint), `emqx`, `postgres` (Timescale image), `redis`, `minio`. Traefik
handles TLS. Secrets via Coolify env vars. Run Prisma migrations on deploy
(including the RLS + Timescale SQL). Point BetterAuth base URL at the `api` domain.

---

## 10. Tasks

### Sprint-01
1. **MQTT spike** (EMQX + MQTTX + NestJS consumer).
2. Prisma schema + **RLS migration** + Timescale hypertable.
3. BetterAuth org auth + `RolesGuard` + `TenantContextInterceptor`.
4. Devices + Tenants modules; ingestion worker → device shows `online`, events persist.
5. Audio upload → storage → `audio_update` push → track ack.
6. **Publish OpenAPI + seed data** for Frontend.

### Backlog
- Continuous aggregates + analytics endpoints; offline-sweep job.
- EMQX HTTP auth hook + per-device ACL hardening; TLS.
- Billing/tier enforcement; firmware OTA orchestration; audit logging.

---

## 11. How you coordinate with other teams

| With | On | You give | You need |
|---|---|---|---|
| **Electronics** | Contract ① MQTT, ③ audio, ④ provisioning | Broker URL, device credentials + ACL, command payloads, signed audio URLs, final audio format | Real events in-schema; fw version reporting; ack behaviour |
| **Frontend** | Contract ② API/WS | **OpenAPI spec + seed data early**, WS events, stable types | Their data needs per screen (shape the endpoints to them) |
| **Mechanical** | — | (none direct) | (none) |
| **Lead (Manav)** | Tenancy/roles, hosting | Working multi-tenant API | Coolify/VPS access, domains, decisions on tiers |

**Golden rule:** publish the OpenAPI contract on day 2–3 even if endpoints return
seed data — Frontend must never be blocked waiting for real devices.

---

## 12. Testing & security
- **Unit** (services) + **e2e** (`supertest`) per module.
- **Tenant-isolation tests** are mandatory: assert brand A cannot read brand B (both at the app layer and by trying to defeat RLS).
- **Load-test** ingestion (simulate N devices publishing) before the 100-unit deploy.
- **Security:** per-device credentials, signed time-limited audio URLs, no cross-tenant leakage, secrets in env, rate-limit public endpoints.

## 13. Definition of done
Devices register & report; telemetry stored per-tenant with **RLS enforced**;
audio push works end-to-end with ack; analytics endpoints return per-tenant
rollups; **OpenAPI + seed data published**; deployed on Coolify.

## 14. Risks
- **MQTT unfamiliarity** → spike first, thin consumer, lean on `@nestjs/microservices`.
- **Cross-tenant leakage** → RLS + explicit isolation tests (defense in depth).
- **Telemetry volume** → Timescale hypertable + aggregates from day one.
- **Frontend blocked** → OpenAPI + seed data early, non-negotiable.

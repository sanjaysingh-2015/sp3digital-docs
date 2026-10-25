# SP3Digital — Architecture

## High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL CLIENTS                                │
│  ┌─────────────────────┐         ┌─────────────────────────────────────┐   │
│  │  identity-admin-ui  │         │   organization-admin-ui             │   │
│  │    (Angular 20)     │         │      (Angular 20)                   │   │
│  └──────────┬──────────┘         └──────────────┬──────────────────────┘   │
└─────────────┼────────────────────────────────────┼──────────────────────────┘
              │ HTTPS (Bearer JWT)                 │ HTTPS (Bearer JWT)
              ▼                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY / LOAD BALANCER                          │
│                    (Not in repo — infra concern)                             │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│  Identity     │         │  Organization │         │   Patient     │
│  Admin Svc    │         │  Admin Svc    │         │  Admin Svc    │
│   :3000       │         │   :3100       │         │   :3400       │
└───────┬───────┘         └───────┬───────┘         └───────┬───────┘
        │                         │                         │
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│ sp3digital_   │         │ sp3digital_   │         │ sp3digital_   │
│ identity      │         │ organization  │         │ patients      │
└───────────────┘         └───────────────┘         └───────────────┘

        ┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│   Provider    │         │ Appointment   │         │ Notification  │
│   Admin Svc   │         │  Admin Svc    │         │  Admin Svc    │
│    :3300      │         │    :3200      │         │    :3500      │
└───────┬───────┘         └───────┬───────┘         └───────┬───────┘
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│ sp3digital_   │         │ sp3digital_   │         │ sp3digital_   │
│ providers     │         │ appointments  │         │ notifications │
└───────────────┘         └───────────────┘         └───────────────┘
```

## Request Lifecycle (Typical Authenticated Request)

```
1. Client → HTTPS request with `Authorization: Bearer <JWT>`
2. Service middleware/authentication.js:
   a. Extract token from header
   b. If token matches SERVICE_INTERNAL_TOKEN → validate X-Tenant-Uuid header, set req.auth.isInternalService=true
   c. Else → verify JWT via JWKS (RS256) or shared secret (HS256) against identity-admin-service's keys
   d. Extract tenantUuid from token claims (tenant_uuid / tenantUuid / tid)
   e. Extract scopes/permissions from token claims
   f. Attach req.auth = { claims, tenantUuid, scopes, userId?, isInternalService }
3. Service middleware/validate.js (if route has Joi schema):
   a. Validate req.body/query/params against schema
   b. Throw 400 VALIDATION_ERROR on failure
4. Route-specific authorize middleware (identity service) OR token-scope check (other services):
   a. Identity: Query DB for user's active roles → permissions → check both token scopes AND DB permissions
   b. Others: Check token scopes only (no DB role lookup)
5. Controller:
   a. Read req.auth.tenantUuid for tenant scoping
   b. Call service layer with validated input + tenantUuid
6. Service:
   a. Business logic, Sequelize queries (implicitly scoped by tenantUuid)
   b. May call other services via src/clients/*.js (Axios with forwarded Bearer token)
7. Controller → JSON response (standard envelope)
8. Error handler middleware:
   a. Joi errors → 400 VALIDATION_ERROR
   b. Sequelize FK errors → 400 INVALID_REFERENCE (appointment service)
   c. Auth errors → 401/403 with code
   d. Unexpected → 500 INTERNAL_ERROR (logged)
```

## Authentication & Authorization Architecture

### Token Issuance (Identity Service Only)
- `authService.issueTokens()` creates:
  - **Access Token** (JWT, short-lived, ~15 min): carries `userId`, `userUuid`, `tenantUuid`, `permissions[]`, `audience`
  - **ID Token** (JWT): carries user profile claims + `mfaVerified`
  - **Refresh Token** (opaque, stored hashed in `refresh_tokens`, rotatable, single-use)
- Session tracking in `user_sessions` (linked to refresh token hash)

### Token Verification (All Services)
- Shared logic in each service's `middleware/authentication.js`:
  - HS256 fallback via `ADMIN_JWT_SECRET` (dev)
  - RS256 via JWKS at `ADMIN_JWKS_URL` (prod) — cached 10 min
  - Validates `iss` (ADMIN_JWT_ISSUER), `aud` (ADMIN_JWT_AUDIENCE)
  - Extracts `tenantUuid` → **required** on every token

### Service-to-Service Auth
- Each service defines `*_SERVICE_INTERNAL_TOKEN` env var
- Caller sends this token as Bearer + `X-Tenant-Uuid` header
- `isInternalServiceToken()` does constant-time comparison
- Grants `ALL_PERMISSIONS` scope, bypasses user lookup

### Authorization Models

| Service | AuthZ Approach |
|---------|----------------|
| **identity-admin** | **Dual-gate**: Token scopes **AND** DB role-permissions (defense-in-depth). `SUPERADMIN` role bypasses tenant scoping. |
| **organization-admin** | Token scopes only (no DB access to identity tables). `authorize()` exported but **not mounted** on routes yet — waits for identity to issue `ORGANIZATION-ADMIN:*` scopes. |
| **patient-admin** | Token scopes only. `hasPermission()` matches `patient-admin:<resource>:<action>` or `patient-admin:*`. |
| **provider-admin** | Token scopes only. `hasPermission()` matches `provider-admin:<resource>:<action>` or `provider-admin:*`. |
| **appointment-admin** | Token scopes only. **Active** on slot-config routes (maker-checker: TENANT_USER create/read, TENANT_ADMIN update/approve). |
| **notification-admin** | Token scopes only. `hasPermission()` matches `notification-admin:<resource>:<action>` or `notification-admin:*`. |

### Permission Code Format
- **Flat** (legacy): `identity-admin:read`, `identity-admin:write`
- **Fine-grained** (current): `IDENTITY-ADMIN:TENANT_USERS:READ`, `ORGANIZATION-ADMIN:FACILITIES:CREATE`, etc.
- **Wildcards**: `identity-admin:*`, `IDENTITY-ADMIN:TENANT_USERS:*`, `ALL_PERMISSIONS`

## Database Architecture

### Per-Service Schema Ownership
Each service owns its tables; **no cross-service foreign keys** in DB. Referential integrity across services enforced at application layer via API calls.

### Shared Tenant Identifier
- `tenant_uuid` (CHAR(36)) column on most tables
- `identity_tenants` is the source of truth for tenant metadata
- Other services treat `tenantUuid` as opaque string from token/header

### Sequelize Conventions
- Model files: `src/models/*.model.js` exporting `define()` factory
- `src/models/index.js` loads all `*.model.js`, then `require('./associations')(db)`
- `timestamps: false`, `freezeTableName: true` — manual `created_on`/`modified_on` columns
- Primary keys: `BIGINT UNSIGNED AUTO_INCREMENT` (internal) + `CHAR(36) UUID` (external)
- Status column: `VARCHAR(30)` with values `ACTIVE`/`DELETED`/`INACTIVE`/`PENDING`/`REVOKED`
- Soft delete via `status = 'DELETED'` (not hard delete)

## Inter-Service Communication Patterns

### Synchronous (Request-Response)
- Axios clients in `src/clients/` (e.g., `patientDirectory.js`, `organizationDirectory.js`)
- Forward `Authorization: Bearer <token>` from incoming request
- Timeout: 5s default (JWKS fetch), configurable per client
- **No circuit breaker / retry** in clients currently

### Asynchronous (Event-Driven)
- **Notification service** is the primary consumer:
  - Other services call `POST /api/v1/notification/notifications` with `eventCode` + `variables`
  - Notification service resolves template, queues in `notifications` table (`status: QUEUED`)
  - Background worker (`worker/dispatcher.js`) polls, renders, delivers via provider (email/SMS/WhatsApp), records attempts
- **No message broker** (RabbitMQ/Kafka) — uses DB polling (simpler, but polling overhead)

## Frontend Architecture

### identity-admin-ui
- **Core**: `auth.service.ts` (token storage, login, refresh, logout), `auth.interceptor.ts` (attaches Bearer), `auth.guard.ts` (route protection), `auth-error.interceptor.ts` (401 → logout)
- **Features**: One folder per domain (users, roles, tenants, idp, oauth, mfa, audit, api-clients, service-accounts, sessions, security-policy, auth-config, permissions, dashboard, login, register-organization)
- **Permissions**: `user.service.ts` holds current user permissions; components check via `userService.hasPermission()`

### organization-admin-ui
- **Core**: Same auth core + `org-directory.service.ts`, `patient-api.service.ts`, `provider-api.service.ts`, `appointment-api.service.ts`, `notification-api.service.ts`, `geography.service.ts`
- **Features**: Broader — organizations, facilities, departments, services, patients, providers, appointments, slot-configs, facility-closures, notification-channels/templates/events, onboarding, users, dashboard
- **Permissions**: Per-feature `*_permissions.ts` (e.g., `appointment-permissions.ts`, `patient-permissions.ts`)

### Shared UI Patterns
- Standalone components, signals for reactive state
- AG Grid for all tabular data (server-side pagination/sort/filter)
- Reactive forms with custom validators
- Confirmation modals, notification toasts via shared components

## Security Boundaries

| Boundary | Mechanism |
|----------|-----------|
| **Network** | Services not directly exposed; API gateway terminates TLS |
| **Transport** | HTTPS enforced; `helmet()` CSP/HSTS/X-Frame-Options |
| **CORS** | `ALLOWED_ORIGINS` env var (comma-separated); dev fallback allows all |
| **Auth** | JWT verification (RS256/JWKS) on every request; no session cookies |
| **Tenant Isolation** | `tenantUuid` from token used in **every** query; SUPERADMIN only bypass |
| **Secrets** | RSA keys for JWT signing (identity only); client secrets encrypted in DB (`secretProtector.js`); internal service tokens in env |
| **Rate Limiting** | Global per-service + stricter on `/auth/login` (IP-based) |
| **Input Validation** | Joi on every mutating endpoint |
| **Audit** | Write operations in identity service logged to `audit_logs` (actor, action, changes, IP) |

## Deployment & Operations

### Configuration (per service)
- `.env` (not committed) with:
  - `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`
  - `PORT`
  - `ADMIN_JWT_ISSUER`, `ADMIN_JWT_AUDIENCE`, `ADMIN_JWKS_URL` (or `ADMIN_JWT_SECRET`)
  - `*_SERVICE_INTERNAL_TOKEN` (for inbound service calls)
  - `ALLOWED_ORIGINS`
  - Service-specific: `NOTIFICATION_WORKER`, SMTP/Twilio creds, etc.

### Startup Sequence
1. `dotenv.config()`
2. `sequelize.authenticate()` → verify DB connectivity
3. `sequelize.sync({ alter: false })` → create/update tables (dev only; prod uses manual migration)
4. Notification service: `templateService.ensureDefaults()` + `eventDefinitionService.ensureDefaults()`
5. `app.listen(PORT)`
6. Notification service: `startWorker()` (unless `NOTIFICATION_WORKER=false`)

### Background Processes
| Service | Process | Trigger |
|---------|---------|---------|
| notification | `worker/dispatcher.js` | Polls `notifications` where `status=QUEUED` AND `scheduled_for <= NOW` every 5s |
| notification | `worker/purge.js` | `npm run purge` — deletes old notifications/attempts per retention policy |
| patient | `jobs/anonymizeExpired.js` | `npm run anonymize` — scheduled externally (cron) |
| appointment | `scripts/generate-slots.js` | `npm run generate-slots` — scheduled externally (cron) |
| organization | `scripts/seed-india-geo.js` | `npm run seed:india-geo` — one-time data load |

### Testing
- **Framework**: Node native test runner (`node --test`)
- **Location**: `tests/*.test.js` in each service
- **Coverage**: Unit tests for services, middleware, utils; no integration/e2e tests in repo

## Design Patterns in Use

| Pattern | Location |
|---------|----------|
| **Layered Architecture** | Controllers → Services → Models (all services) |
| **Middleware Pipeline** | Express: helmet → cors → json → rate-limit → auth → validate → routes → error handler |
| **Factory Functions** | Sequelize models: `module.exports = (sequelize, DataTypes) => sequelize.define(...)` |
| **Dynamic Model Loading** | `models/index.js` reads `*.model.js` files at runtime |
| **Centralized Associations** | `models/associations.js` receives loaded models, wires relationships |
| **Token-Based Auth (Stateless)** | JWT access + refresh rotation; no server-side session store beyond revocation lists |
| **Scope-Based Authorization** | Fine-grained `SERVICE:RESOURCE:ACTION` codes in token claims |
| **Service Client Wrappers** | `src/clients/*.js` encapsulate Axios + auth forwarding |
| **Outbox Pattern** | Notification service: write to `notifications` table → worker delivers → updates status |
| **Maker-Checker** | Appointment slot configs: TENANT_USER creates, TENANT_ADMIN approves |
| **Soft Delete** | `status` column + default scope `where: { status: 'ACTIVE' }` in queries |
| **Optimistic Concurrency** | Not implemented (no `version` column) — potential gap |
| **Idempotency** | Not implemented at API layer — potential gap |

## Scaling Considerations
- **Stateless services** — horizontal scaling via replicas (shared DB, no sticky sessions)
- **JWKS caching** (10 min) reduces identity service load
- **DB connection pooling** (max 10) per service instance
- **Notification worker** — single-process polling; for scale, run separate worker processes with `NOTIFICATION_WORKER=false` on API instances
- **Geography data** — ~150K cities in `cities` table; indexed for lookups
- **Rate limits** — tuned per service (patient: 300/15min, notification: 3000/15min for bursty reminder traffic)
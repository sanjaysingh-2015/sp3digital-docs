# SP3Digital — Project Context (Entry Point)

## Purpose
Multi-tenant healthcare administration platform: 7 Node.js/Express 5 services + 2 Angular 20 UIs, shared MySQL 8 (per-service schema).

## Service Inventory

| Service | Port | Domain | Entry |
|---------|------|--------|-------|
| `sp3digital-identity-admin-service` | 3000 | Auth, RBAC, tenants, OAuth/OIDC, MFA, API clients, service accounts | `src/app.js` |
| `sp3digital-organization-admin-service` | 3100 | Orgs, facilities, departments, geography (India postal hierarchy), services/categories | `src/app.js` |
| `sp3digital-patient-admin-service` | 3400 | Patient registry, per-facility MRNs, guardians, duplicate detection/merge, anonymization | `src/app.js` |
| `sp3digital-provider-admin-service` | 3300 | Provider profiles, qualifications, registrations, facility affiliations (Physical/Remote/Other) | `src/app.js` |
| `sp3digital-appointment-admin-service` | 3200 | Slot config (RRULE, maker-checker), closures, bookings, reminders, token counters | `src/app.js` |
| `sp3digital-notification-admin-service` | 3500 | Templates (email/SMS/WhatsApp), event-driven outbox + retries, suppression, delivery log, worker | `src/app.js` + `scripts/run-worker.js` |
| `sp3digital-identity-admin-ui` | 4200 | Angular 20: users, roles, tenants, IDPs, OAuth, MFA, audit, API clients | `ng serve` |
| `sp3digital-organization-admin-ui` | 4201 | Angular 20: org/facility/patient/provider/appointment/notification admin | `ng serve` |

## Tech Stack
- **Backend**: Node.js (CommonJS), Express 5, Sequelize 6, MySQL 8, JWT (RS256/JWKS), Joi, Swagger, pino
- **Frontend**: Angular 20 (standalone, signals), AG Grid, FullCalendar, RxJS 7.8

## Architecture Overview
- **Auth**: Identity service issues JWTs (access + refresh rotation); all services verify via shared `middleware/authentication.js`
- **Authorization**: Identity = dual-gate (token scopes + DB roles); downstream = token scopes only
- **Multi-tenancy**: `tenantUuid` from JWT claim or `X-Tenant-Uuid` header; all queries implicitly scoped
- **Inter-service**: Axios clients in `src/clients/` with forwarded Bearer token + service-to-service secrets
- **Database**: No cross-service FKs; referential integrity at application layer

## Critical Business Rules
1. **Tenant isolation**: No query crosses `tenantUuid` boundary (SUPERADMIN bypass only in identity)
2. **Soft delete**: `status` column (`ACTIVE`/`DELETED`/`MERGED`/`ANONYMIZED`/`REVOKED`/`SUPPRESSED`)
3. **Dual-key**: `*_id` (BIGINT PK) + `*_uuid` (CHAR(36) UK) on all entities
4. **Maker-checker**: Slot configs require TENANT_ADMIN approval (`status: PENDING_APPROVAL → ACTIVE`)
5. **Refresh rotation**: Single-use; reuse revokes entire session
6. **Patient MRN**: Sequential per (tenant, facility), never reused

## Security & Data Integrity
- JWT verification via JWKS (10-min cache) or HS256 secret (dev)
- Service-to-service: constant-time token compare + `X-Tenant-Uuid` header
- CORS: `ALLOWED_ORIGINS` env var (comma-separated)
- Rate limits: identity 10/15min login, others 300-3000/15min
- Encrypted secrets: `secretProtector.js` (AES-256-GCM with key versioning)

## Known Limitations
- **Org-admin authorize() not mounted** — token scopes only, no DB role check
- **Duplicated auth middleware** across 5 services (~90% identical)
- **No migrations** — `sequelize.sync({ alter: false })` on startup
- **No circuit breakers** on inter-service calls
- **Force password change unauthenticated** (identity `POST /auth/change-password` with `forceChange=true`)
- **MFA challenge secret falls back to JWT_PRIVATE_KEY** (cryptographically wrong)

## Documentation Index

| Topic | Document | Key Contents |
|-------|----------|--------------|
| Architecture & flows | `ARCHITECTURE.md` | Component diagram, request lifecycle, authZ models, patterns |
| Business workflows | `FUNCTIONAL_FLOWS.md` | 10 core flows, user roles, state transitions, error codes |
| Database schema | `DATABASE.md` | 6 DBs, 80+ tables, FKs, indexes, cross-service refs |
| API contracts | `API_AND_DEPENDENCIES.md` | All endpoints, inter-service contracts, dependency graph |
| Module index | `MODULE_INDEX.md` | Key files by responsibility, symbol index, test coverage |
| Issues & risks | `KNOWN_ISSUES.md` | 15 verified issues (P0-P3), inferred gaps, unknowns |

## Task-to-Context Index

| Task Type | Read First | Then Inspect | Key Files | Tests | Regression Risks |
|-----------|------------|--------------|-----------|-------|------------------|
| **Auth changes** (login, MFA, tokens) | `PROJECT_CONTEXT.md` → `ARCHITECTURE.md` | `identity-admin-service/src/services/authService.js` | `authService.js`, `tokenService.js`, `mfaService.js`, `credentialService.js`, `middleware/authentication.js` | `authentication.test.js`, `login-rate-limit.test.js` | All services' `verifyToken()`, frontend `auth.service.ts` |
| **RBAC / permissions** | `PROJECT_CONTEXT.md` → `FUNCTIONAL_FLOWS.md` | `identity-admin-service/src/services/authorizationService.js` | `authorizationService.js`, `middleware/authentication.js` (authorize), `models/permissions.model.js` | `authorization-service.test.js` | All services' `hasPermission()`, frontend `*_permissions.ts` |
| **Tenant / user management** | `PROJECT_CONTEXT.md` → `FUNCTIONAL_FLOWS.md` | `identity-admin-service/src/services/{tenantService,userService}.js` | `tenantService.js`, `userService.js`, `registrationService.js` | — | Org service internal org creation, frontend user/role UI |
| **Organization / facility / geography** | `PROJECT_CONTEXT.md` → `DATABASE.md` | `organization-admin-service/src/services/{organization,facility,geography}Service.js` | `organizationService.js`, `facilityService.js`, `geographyService.js`, `models/*.model.js` | — | Patient, provider, appointment service clients |
| **Patient registry / merge** | `PROJECT_CONTEXT.md` → `FUNCTIONAL_FLOWS.md` | `patient-admin-service/src/services/patientService.js` | `patientService.js`, `models/patient.model.js`, `clients/organizationDirectory.js` | — | Appointment booking (patient validation) |
| **Provider / affiliations** | `PROJECT_CONTEXT.md` → `FUNCTIONAL_FLOWS.md` | `provider-admin-service/src/services/{provider,affiliation}Service.js` | `providerService.js`, `affiliationService.js`, `clients/organizationDirectory.js` | — | Appointment slot config (provider validation) |
| **Slot config / maker-checker** | `PROJECT_CONTEXT.md` → `FUNCTIONAL_FLOWS.md` | `appointment-admin-service/src/services/appointmentSlotConfigService.js` | `appointmentSlotConfigService.js`, `slotGenerationService.js`, `models/appointmentSlotConfig.model.js` | — | Notification reminders, booking |
| **Booking / appointments** | `PROJECT_CONTEXT.md` → `FUNCTIONAL_FLOWS.md` | `appointment-admin-service/src/services/bookingService.js` | `bookingService.js`, `appointmentService.js`, `clients/{patient,provider,notification}Directory.js` | — | Notification service, token counters |
| **Notifications / templates** | `PROJECT_CONTEXT.md` → `FUNCTIONAL_FLOWS.md` | `notification-admin-service/src/services/{notification,template}Service.js` | `notificationService.js`, `templateService.js`, `worker/dispatcher.js`, `providers/*.js` | — | Appointment reminders, future event triggers |
| **Frontend feature** | `PROJECT_CONTEXT.md` → `MODULE_INDEX.md` | `sp3digital-*-ui/src/app/features/<domain>/` | `*.component.ts`, `core/*.service.ts`, `core/*_permissions.ts` | — | Backend API contract, permission codes |
| **Database change** | `PROJECT_CONTEXT.md` → `DATABASE.md` | `*/src/models/*.model.js`, `*/src/models/associations.js` | Model files, `associations.js`, SQL dumps in `database/` | `model-associations.test.js` | All services using that entity, cross-service refs |

## File Location Quick Reference

```
C:\MyWorkspace\SP3Digital\
├── sp3digital-identity-admin-service\src\
│   ├── app.js                          # Express setup, route mounting
│   ├── middleware/authentication.js    # JWT verify, dual-gate authorize
│   ├── services/authService.js         # Login, MFA, tokens, refresh, passwords
│   ├── services/authorizationService.js # Roles, permissions, resolution
│   ├── services/registrationService.js # Self-service tenant + admin user
│   ├── models/*.model.js               # 27 model definitions
│   └── models/associations.js          # All Sequelize relationships
├── sp3digital-organization-admin-service\src\
│   ├── services/{organization,facility,geography}Service.js
│   └── models/{organization,facility,department,service,geography}.model.js
├── sp3digital-patient-admin-service\src\
│   ├── services/patientService.js
│   ├── clients/{organization,geography}Directory.js
│   └── models/patient*.model.js
├── sp3digital-provider-admin-service\src\
│   ├── services/{provider,affiliation}Service.js
│   └── models/provider*.model.js
├── sp3digital-appointment-admin-service\src\
│   ├── services/{appointmentSlotConfig,booking,slotGeneration}Service.js
│   ├── clients/{organization,patient,provider,notification}Directory.js
│   └── models/appointment*.model.js
├── sp3digital-notification-admin-service\src\
│   ├── services/{notification,template,eventDefinition}Service.js
│   ├── worker/dispatcher.js            # Polling delivery worker
│   ├── providers/{email,sms,whatsapp}Provider.js
│   └── models/notification*.model.js
├── sp3digital-identity-admin-ui\src\app\
│   ├── core/{auth,api,user}.service.ts
│   └── features/{users,roles,tenants,idp,oauth,mfa,...}/
└── sp3digital-organization-admin-ui-\src\app\
    ├── core/{org-directory,patient-api,provider-api,appointment-api,notification-api}.service.ts
    └── features/{organizations,facilities,patients,providers,appointments,notifications,...}/
```

## Loading Strategy for Future Tasks
1. **Read `PROJECT_CONTEXT.md` first** — understand scope and constraints
2. **Load detailed doc only for affected feature** (see Task-to-Context Index)
3. **Inspect actual implementation** before editing — verify against source
4. **Expand investigation only when dependencies require it** (use index)
5. **Avoid rereading unrelated modules** — use symbol/file paths in index
6. **Update documentation after meaningful changes** — keep index current

## Validation Checklist
- [x] All links point to existing files in `docs/ai/`
- [x] Referenced source paths exist in repository
- [x] Symbol names match actual exports (verified during analysis)
- [x] Service ports and base paths consistent across docs
- [x] Security rules and business invariants preserved
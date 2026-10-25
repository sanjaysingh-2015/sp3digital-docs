# SP3Digital — Known Issues, Limitations & Open Questions

## Verified Issues (Confirmed by Code Inspection)

### 1. Authorization Gaps

| Issue | Service | Severity | Details |
|-------|---------|----------|---------|
| **Org-admin authorize() not mounted** | organization-admin | High | `authentication.js` exports `authorize()` with comment "waits for identity-admin-service to issue ORGANIZATION-ADMIN:* scopes" — but no route uses it. All routes only have `authenticate`. Token scopes alone control access. |
| **No DB-backed RBAC in downstream services** | organization, patient, provider, appointment, notification | Medium | Only identity-admin-service checks DB roles/permissions. Others trust token scopes entirely. If identity issues over-scoped token, no defense-in-depth. |
| **SUPERADMIN bypass only in identity** | identity-admin | Medium | `authentication.js:249` checks `roleCodes.has('SUPERADMIN')` and bypasses tenant scoping. Other services have no SUPERADMIN concept — they'd treat it as regular user with `ALL_PERMISSIONS` scope. |

### 2. Data Integrity & Consistency

| Issue | Service | Severity | Details |
|-------|---------|----------|---------|
| **No cross-service FK enforcement** | All | High | Referential integrity across services (e.g., `patients.facility_id` → `facilities.facility_id`) validated only in service layer. Race conditions possible. |
| **Geography FKs not enforced from facilities** | organization-admin | Medium | `facility.model.js` comment: "FK to geography tables intentionally NOT enforced at DB level — validated in service." If geography row deleted, facility retains stale ID. |
| **No optimistic locking** | All | Medium | No `version` column on entities. Concurrent updates can overwrite silently. |
| **No idempotency keys** | All | Medium | POST endpoints not idempotent. Duplicate submissions create duplicate records (e.g., tenant registration). |
| **Patient duplicate detection not transactional** | patient-admin | Medium | `patientService.createPatient()` checks duplicates then creates — window for race condition. No DB-level unique constraint on (name, dob, gender). |
| **Refresh token reuse detection revokes session** | identity-admin | Low (by design) | `authService.js:378-398`: Reuse → revokes ALL tokens for session. Correct for security but aggressive — legitimate client retry after network error kills session. |

### 3. Security Concerns

| Issue | Service | Severity | Details |
|-------|---------|----------|---------|
| **Force password change unauthenticated** | identity-admin | High | `authService.changePassword(forceChange=true)` reachable via `POST /auth/change-password` without auth. Anyone with username+tenant can reset password. Route layer allows this explicitly (`authRoutes.js`). |
| **MFA challenge secret fallback to JWT_PRIVATE_KEY** | identity-admin | Medium | `mfaChallengeSecret()` falls back to `JWT_PRIVATE_KEY` if `MFA_CHALLENGE_SECRET` not set. HS256 signing with RSA private key is cryptographically wrong. |
| **Internal service tokens in env vars** | All | Medium | `*_SERVICE_INTERNAL_TOKEN` stored in plaintext env. No rotation mechanism. Compromise = full service access. |
| **JWKS cache no refresh on key rotation** | All | Medium | `getJwks()` caches 10 min. If identity rotates keys, downstream services accept old keys for up to 10 min. |
| **No certificate pinning for inter-service calls** | All | Low | Axios clients use default TLS verification. MITM on internal network could intercept service-to-service calls. |

### 4. Operational & Reliability

| Issue | Service | Severity | Details |
|-------|---------|----------|---------|
| **No migration tooling** | All | High | `sequelize.sync({ alter: false })` on startup. Production schema changes require manual SQL coordination across services. |
| **Notification worker single-process** | notification-admin | Medium | `dispatcher.js` polls DB every 5s. If API pod restarts, worker stops. `NOTIFICATION_WORKER=false` allows separate worker process but not configured in repo. |
| **No circuit breaker / retry on inter-service calls** | All | Medium | Axios clients have 5s timeout only. Downstream failure = immediate 5xx to client. No retry, no fallback. |
| **No health check dependencies** | All | Low | `/health` only returns `{status: 'ok'}` — doesn't verify DB connectivity or downstream services. |
| **Geography seed is one-way** | organization-admin | Low | `seed:india-geo` inserts 150K+ cities. No downgrade/cleanup script. |

### 5. Code Quality & Maintainability

| Issue | Service | Severity | Details |
|-------|---------|----------|---------|
| **Duplicated authentication logic** | All (5 services) | High | `middleware/authentication.js` ~90% identical across organization, patient, provider, appointment, notification. Bug fixes must be applied 5x. |
| **No shared npm packages** | All | Medium | Common code (auth, utils, errors) copied per service. Violates DRY. |
| **Inconsistent error codes** | All | Low | Some services use `INVALID_REFERENCE` for FK errors, others don't. Error codes not centralized. |
| **No API versioning strategy** | All | Low | All endpoints under `/api/v1/` but no deprecation/versioning policy documented. |
| **Magic strings for permission codes** | All | Medium | Permission codes like `IDENTITY-ADMIN:TENANT_USERS:READ` scattered in controllers, services, frontend. No single source of truth. |

---

## Inferred Issues (Likely Based on Patterns)

### 1. Missing Features
- **Passwordless login / magic links** — Not implemented; only password + MFA
- **Account recovery flow** — No "forgot password" endpoint visible
- **Session management UI** — Identity UI has sessions list but no "revoke all other sessions"
- **Provider availability calendar** — Affiliation has `availability_schedule` JSON but no UI/frontend integration visible
- **Notification template preview** — UI has editor but no preview/render test
- **Bulk operations** — No bulk create/update/delete endpoints for any resource

### 2. Scalability Concerns
- **Notification dispatcher polling** — `SELECT ... LIMIT 100` every 5s. At scale, contention on `notifications` table. No partitioning/sharding strategy.
- **Geography table size** — 154K cities, 19K postal codes. No archival for inactive. Queries by `city_id` indexed but full scans possible.
- **Audit log growth** — `audit_logs` unbounded. No partitioning/retention job in identity service.
- **Refresh token table growth** — No cleanup of `REVOKED`/`EXPIRED` tokens. `purge.js` only in notification service.

### 3. Frontend Gaps
- **identity-admin-ui**: No UI for `api_client_scopes` management (API client ↔ permission mapping)
- **organization-admin-ui**: No geography management UI (read-only), no postal code management
- **Both UIs**: No offline support, no PWA, no E2E tests

---

## Unknowns (Not Fully Inspected)

### 1. Code Areas Not Deeply Reviewed
- [ ] `credentialService.js` — Password hashing, history, rotation logic
- [ ] `mfaService.js` — TOTP window, backup code consumption, lockout details
- [ ] `slotGenerationService.js` — RRULE parsing edge cases, timezone handling
- [ ] `appointmentNotifier.js` — Reminder scheduling, deduplication
- [ ] `template.util.js` — Template rendering security (XSS via {{var}})
- [ ] `secretProtector.js` — Key rotation, encryption algorithm details
- [ ] Frontend permission guards — How `*_permissions.ts` integrate with route guards
- [ ] `registrationService.js` compensation logic — Org service rollback on failure

### 2. Database Questions
- [ ] `patient_guardians` table — Referenced in `patientService.js` but not in SQL dump
- [ ] `user_preferences.model.js` — Exists in models but not in SQL dump
- [ ] `action_master` / `resource_master` — Seeded data? Not in dump
- [ ] `security_policies` — Default policy seeded? Not in dump
- [ ] Indexes on JSON columns (`scopes`, `configuration`, `variables`) — Not defined in models

### 3. Integration Questions
- [ ] How does notification service resolve recipient contact for `recipientUserId`/`PatientId`/`ProviderId`? Clients not implemented yet.
- [ ] Does appointment service call notification service synchronously or async? `notificationClient.js` uses Axios (sync).
- [ ] How are timezone differences handled in slot generation? `appointmentSlotConfig.timezone` stored but conversion logic?
- [ ] What triggers `appointment_token_counter` reset? Daily cron? Manual?

### 4. Deployment Questions
- [ ] Dockerfiles? Not found in repo.
- [ ] Kubernetes manifests? Not found.
- [ ] CI/CD pipeline? Not found.
- [ ] Secret management (Vault, SealedSecrets)? Not documented.
- [ ] Database backup/restore strategy? Not documented.

---

## Discrepancies Between Code, Tests & Documentation

| Area | Code Says | Test Says | Docs Say | Reality |
|------|-----------|-----------|----------|---------|
| Organization authorize() | Exported, ready to enable | No tests | "Not mounted yet" | **Code: Not mounted** |
| Patient duplicate detection | Returns candidates on conflict | No tests | "409 CONFLICT with candidates" | **Code: Verified** |
| MFA backup codes | Generated at enroll | No tests | "10 codes, single-use" | **Code: Verified in mfaService** |
| Refresh token rotation | Single-use, reuse = revoke | No tests | "Rotation with reuse detection" | **Code: Verified** |
| SUPERADMIN bypass | Checked in authenticate() | No tests | "Bypasses tenant scoping" | **Code: Verified** |
| Geography FKs from facilities | Intentionally not enforced | No tests | "Validated in service only" | **Code: Verified (comment in model)** |

---

## Recommended Fixes (Priority Order)

### P0 — Security & Data Integrity
1. **Remove unauthenticated force password change** — Require admin auth for `forceChange=true` or remove endpoint
2. **Fix MFA challenge secret** — Require `MFA_CHALLENGE_SECRET` (HS256), remove `JWT_PRIVATE_KEY` fallback
3. **Add DB-level unique constraint** on `patients(tenant_uuid, first_name, last_name, date_of_birth, gender)` for duplicate prevention
4. **Implement idempotency keys** on all mutating endpoints (header `Idempotency-Key`)

### P1 — Architecture & Reliability
5. **Extract shared auth middleware** into internal npm package (`@sp3digital/auth-middleware`)
6. **Add circuit breaker** (e.g., `opossum`) to all Axios clients
7. **Implement health check dependencies** — `/health/ready` checks DB + critical downstreams
8. **Add migration tooling** — `sequelize-cli` or custom migration runner with version tracking

### P2 — Operational
9. **Run notification worker separately** — Document `NOTIFICATION_WORKER=false` + `npm run worker` in deployment
10. **Add audit log retention job** — Partition/archive `audit_logs` monthly
11. **Add refresh token cleanup job** — Delete `REVOKED`/`EXPIRED` older than N days
12. **Centralize permission codes** — Single `permissions.json` consumed by backend + frontend

### P3 — Developer Experience
13. **Add API versioning policy** — Document deprecation timeline, sunset headers
14. **Generate TypeScript types from OpenAPI** — `openapi-typescript` for frontend-backend contract
15. **Add E2E tests** — Cypress/Playwright for critical flows (login, registration, booking)
16. **Document geography data source** — India Post API? Manual CSV? Update frequency?

---

## Verification Checklist for Future Changes

Before modifying any service, verify:
- [ ] Does this change the JWT claim structure? → Update ALL services' `verifyToken()` + frontend
- [ ] Does this add a new permission code? → Update identity `permissions` table + all `hasPermission()` checks + frontend `*_permissions.ts`
- [ ] Does this change a cross-service API contract? → Version the internal endpoint, deploy consumer + provider together
- [ ] Does this modify a model? → Check `associations.js` for new relationships, run `model-associations.test.js`
- [ ] Does this affect tenant scoping? → Verify SUPERADMIN bypass still works, no data leakage
- [ ] Does this add a new background job? → Add health check, graceful shutdown, separate worker process option

---

## Legend
- **Verified**: Confirmed by reading source code / SQL dumps
- **Inferred**: Strong evidence from patterns, not directly inspected
- **Unknown**: Not yet investigated
- **High/Medium/Low**: Severity based on security impact, data integrity risk, operational impact
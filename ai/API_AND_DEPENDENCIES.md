# SP3Digital — API Endpoints & Dependencies

## API Base Paths

| Service | Base Path | Swagger UI |
|---------|-----------|------------|
| identity-admin | `/api/v1/identity-admin` | `/docs` |
| organization-admin | `/api/v1/organization-admin` | `/docs` |
| patient-admin | `/api/v1/patient-admin` | `/docs` |
| provider-admin | `/api/v1/provider-admin` | `/docs` |
| appointment-admin | `/api/v1/appointment-admin` | `/docs` |
| notification-admin | `/api/v1/notification` | `/docs` |

## Identity Admin Service — Key Endpoints

### Public (Unauthenticated)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/public/register-organization` | Self-service tenant + admin user registration |
| GET | `/public/tenants/search` | Search tenants by code/name (for login UI) |
| GET | `/.well-known/jwks.json` | JWKS endpoint for token verification |
| GET | `/.well-known/openid-configuration` | OIDC discovery |

### Auth (Login/MFA/Token)
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/login` | None | Username/password → tokens or MFA challenge |
| POST | `/auth/mfa/verify` | None | MFA challenge + TOTP → tokens |
| POST | `/auth/refresh` | None | Refresh token rotation → new tokens |
| POST | `/auth/logout` | None | Revoke refresh token + session |
| POST | `/auth/change-password` | None* | Self-service or forced password change |

> *`forceChange=true` bypasses old password check — reachable unauthenticated by design (see `authService.js:489`)

### Tenants
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/tenants` | `identity-admin:read` | List tenants (SUPERADMIN only) |
| POST | `/tenants` | `identity-admin:write` | Create tenant (SUPERADMIN) |
| GET | `/tenants/:tenantUuid` | `identity-admin:read` | Get tenant |
| PATCH | `/tenants/:tenantUuid` | `identity-admin:write` | Update tenant |
| PATCH | `/tenants/:tenantUuid/status` | `identity-admin:write` | Activate/deactivate tenant |

### Users
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/users` | `identity-admin:read` | Paginated list (tenant-scoped) |
| POST | `/users` | `identity-admin:write` | Create user |
| GET | `/users/:userId` | `identity-admin:read` | Get user |
| PATCH | `/users/:userId` | `identity-admin:write` | Update user |
| PATCH | `/users/:userId/status` | `identity-admin:write` | Activate/deactivate user |
| POST | `/users/:userId/assign-roles` | `identity-admin:write` | Assign roles to user |
| DELETE | `/users/:userId/roles/:roleId` | `identity-admin:write` | Remove role from user |

### Authorization (Roles/Permissions)
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/authorization/roles` | `identity-admin:read` | List roles |
| POST | `/authorization/roles` | `identity-admin:write` | Create role |
| GET | `/authorization/roles/:roleId` | `identity-admin:read` | Get role |
| PATCH | `/authorization/roles/:roleId` | `identity-admin:write` | Update role |
| PATCH | `/authorization/roles/:roleId/status` | `identity-admin:write` | Activate/deactivate role |
| POST | `/authorization/roles/:roleId/permissions` | `identity-admin:write` | Assign permissions to role |
| DELETE | `/authorization/roles/:roleId/permissions/:permissionId` | `identity-admin:write` | Remove permission from role |
| GET | `/authorization/permissions` | `identity-admin:read` | List permissions (catalog) |

### Identity Providers (OIDC/SAML)
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/identity-providers` | `identity-admin:read` | List IdPs |
| POST | `/identity-providers` | `identity-admin:write` | Create IdP |
| GET | `/identity-providers/:id` | `identity-admin:read` | Get IdP |
| PATCH | `/identity-providers/:id` | `identity-admin:write` | Update IdP |
| PATCH | `/identity-providers/:id/status` | `identity-admin:write` | Activate/deactivate IdP |
| GET | `/identity-providers/:id/metadata` | None | SAML/OIDC metadata XML/JSON |

### OAuth Clients
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/oauth/clients` | `identity-admin:read` | List OAuth clients |
| POST | `/oauth/clients` | `identity-admin:write` | Create OAuth client |
| GET | `/oauth/clients/:id` | `identity-admin:read` | Get client |
| PATCH | `/oauth/clients/:id` | `identity-admin:write` | Update client |
| PATCH | `/oauth/clients/:id/status` | `identity-admin:write` | Activate/deactivate client |
| POST | `/oauth/clients/:id/rotate-secret` | `identity-admin:write` | Rotate client secret |

### API Clients
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/api-clients` | `identity-admin:read` | List API clients |
| POST | `/api-clients` | `identity-admin:write` | Create API client |
| GET | `/api-clients/:id` | `identity-admin:read` | Get client |
| PATCH | `/api-clients/:id` | `identity-admin:write` | Update client |
| PATCH | `/api-clients/:id/status` | `identity-admin:write` | Activate/deactivate client |
| POST | `/api-clients/:id/rotate-secret` | `identity-admin:write` | Rotate client secret |

### MFA
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/users/:userId/mfa` | `identity-admin:read` | List user's MFA methods |
| POST | `/users/:userId/mfa` | `identity-admin:write` | Enroll MFA (TOTP) |
| POST | `/users/:userId/mfa/verify` | `identity-admin:write` | Verify + activate TOTP |
| DELETE | `/users/:userId/mfa/:methodId` | `identity-admin:write` | Remove MFA method |
| POST | `/users/:userId/mfa/:methodId/backup-codes` | `identity-admin:write` | Regenerate backup codes |

### Service Accounts
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/service-accounts` | `identity-admin:read` | List service accounts |
| POST | `/service-accounts` | `identity-admin:write` | Create service account |
| GET | `/service-accounts/:id` | `identity-admin:read` | Get account |
| PATCH | `/service-accounts/:id` | `identity-admin:write` | Update account |
| PATCH | `/service-accounts/:id/status` | `identity-admin:write` | Activate/deactivate |
| POST | `/service-accounts/:id/rotate-secret` | `identity-admin:write` | Rotate secret |

### Security Policies & Auth Config
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/security-policy` | `identity-admin:read` | Get tenant policy |
| PATCH | `/security-policy` | `identity-admin:write` | Update policy |
| GET | `/auth-configs` | `identity-admin:read` | Get auth config |
| PATCH | `/auth-configs` | `identity-admin:write` | Update auth config |

### Audit & Sessions
| Method | Path | Permission | Description |
|--------|------|------------|-------------|
| GET | `/audit-logs` | `identity-admin:read` | Paginated audit log |
| GET | `/sessions` | `identity-admin:read` | List user sessions |
| DELETE | `/sessions/:sessionId` | `identity-admin:write` | Revoke session |
| GET | `/resource-actions` | `identity-admin:read` | List resource-action catalog |

---

## Organization Admin Service — Key Endpoints

All endpoints require valid JWT (no public routes). Base: `/api/v1/organization-admin`

| Resource | Endpoints | Notes |
|----------|-----------|-------|
| Organizations | `GET/POST /organizations`, `GET/PATCH /organizations/:id`, `PATCH /organizations/:id/status` | Hierarchical, tenant-scoped |
| Facilities | `GET/POST /facilities`, `GET/PATCH /facilities/:id`, `PATCH /facilities/:id/status` | Belong to organization |
| Departments | `GET/POST /departments`, `GET/PATCH /departments/:id`, `PATCH /departments/:id/status` | Belong to facility |
| Services | `GET/POST /services`, `GET/PATCH /services/:id` | Global catalog |
| Service Categories | `GET/POST /service-categories`, `GET/PATCH /service-categories/:id` | Global catalog |
| Facility Services | `GET/POST /facility-services`, `GET/PATCH /facility-services/:id` | Facility↔Service↔Category↔Dept |
| Geography | `GET /geography/countries`, `GET /geography/states`, `GET /geography/districts`, `GET /geography/sub-districts`, `GET /geography/cities`, `GET /geography/postal-codes` | Read-only hierarchy |
| Internal | `POST /internal/organizations` | **Service-to-service only** (`requireInternalService`), called by identity registration |

> **Note**: `authorize()` middleware exported but **not mounted** on routes — token scopes only, no DB role check (see `authentication.js:180-183`)

---

## Patient Admin Service — Key Endpoints

Base: `/api/v1/patient-admin`

| Resource | Endpoints | Permission |
|----------|-----------|------------|
| Patients | `GET/POST /patients`, `GET/PATCH /patients/:id`, `PATCH /patients/:id/status`, `GET /patients/:id/access-log` | `patient-admin:patients:READ/CREATE/UPDATE` |
| Patient Search | `GET /patients/search` | `patient-admin:patients:SEARCH` |
| Duplicate Check | `POST /patients/duplicate-check` | `patient-admin:patients:SEARCH` |
| Merge | `POST /patients/:id/merge` | `patient-admin:patients:UPDATE` |
| Facility Numbers | `GET /patients/:id/facility-numbers`, `POST /patients/:id/facility-numbers` | `patient-admin:patients:READ/CREATE` |
| Settings | `GET/PATCH /settings` | `patient-admin:settings:READ/UPDATE` |

---

## Provider Admin Service — Key Endpoints

Base: `/api/v1/provider-admin`

| Resource | Endpoints | Permission |
|----------|-----------|------------|
| Providers | `GET/POST /providers`, `GET/PATCH /providers/:id`, `PATCH /providers/:id/status` | `provider-admin:providers:READ/CREATE/UPDATE` |
| Qualifications | `GET/POST /providers/:id/qualifications`, `PATCH/DELETE /qualifications/:qid` | `provider-admin:qualifications:*` |
| Registrations | `GET/POST /providers/:id/registrations`, `PATCH/DELETE /registrations/:rid` | `provider-admin:registrations:*` |
| Specialties | `GET/POST /providers/:id/specialties`, `PATCH/DELETE /specialties/:sid` | `provider-admin:specialties:*` |
| Languages | `GET/POST /providers/:id/languages`, `PATCH/DELETE /languages/:lid` | `provider-admin:languages:*` |
| Experience | `GET/POST /providers/:id/experience`, `PATCH/DELETE /experience/:eid` | `provider-admin:experience:*` |
| Documents | `GET/POST /providers/:id/documents`, `DELETE /documents/:did` | `provider-admin:documents:*` |
| Affiliations | `GET/POST /affiliations`, `GET/PATCH /affiliations/:id`, `PATCH /affiliations/:id/status` | `provider-admin:affiliations:*` |
| Affiliation Services | `GET/POST /affiliations/:id/services`, `DELETE /affiliations/:id/services/:svcId` | `provider-admin:affiliation_services:*` |

---

## Appointment Admin Service — Key Endpoints

Base: `/api/v1/appointment-admin`

| Resource | Endpoints | Permission | Notes |
|----------|-----------|------------|-------|
| Slot Configs | `GET/POST /slot-configs`, `GET/PATCH /slot-configs/:id`, `PATCH /slot-configs/:id/approve`, `PATCH /slot-configs/:id/reject` | `appointment-admin:slot_configs:READ/CREATE/UPDATE` + `APPROVE` (TENANT_ADMIN) | Maker-checker |
| Slots | `GET /slots`, `GET /slots/availability` | `appointment-admin:slots:READ` | Generated from configs |
| Appointments | `GET/POST /appointments`, `GET/PATCH /appointments/:id`, `PATCH /appointments/:id/cancel`, `PATCH /appointments/:id/reschedule` | `appointment-admin:appointments:*` | Booking workflow |
| Facility Closures | `GET/POST /facility-closures`, `GET/PATCH /facility-closures/:id`, `PATCH /facility-closures/:id/status` | `appointment-admin:facility_closures:*` | Blocks slot generation |
| Cancellation Reasons | `GET/POST /cancellation-reasons`, `PATCH /cancellation-reasons/:id` | `appointment-admin:cancellation_reasons:*` | Catalog |
| Reminder Settings | `GET/PATCH /reminder-settings` | `appointment-admin:reminder_settings:*` | Lead times + channels |

---

## Notification Admin Service — Key Endpoints

Base: `/api/v1/notification`

| Resource | Endpoints | Permission |
|----------|-----------|------------|
| Notifications | `POST /notifications`, `GET /notifications`, `GET /notifications/:id`, `GET /notifications/:id/attempts` | `notification-admin:notifications:*` |
| Templates | `GET/POST /templates`, `GET/PATCH /templates/:id`, `PATCH /templates/:id/status` | `notification-admin:templates:*` |
| Event Definitions | `GET/POST /events`, `GET/PATCH /events/:id`, `PATCH /events/:id/status` | `notification-admin:events:*` |
| Suppressions | `GET/POST /suppressions`, `GET/PATCH /suppressions/:id`, `PATCH /suppressions/:id/status` | `notification-admin:suppressions:*` |
| Channel Settings | `GET/POST /channels`, `GET/PATCH /channels/:id`, `PATCH /channels/:id/status` | `notification-admin:channels:*` |

---

## Inter-Service API Contracts (Client → Server)

### identity-admin-service → organization-admin-service
```
POST /api/v1/organization-admin/internal/organizations
Headers: Authorization: Bearer <ORGANIZATION_SERVICE_INTERNAL_TOKEN>, X-Tenant-Uuid: <uuid>
Body: { tenantUuid, organizationCode, organizationName, organizationType, address... }
Response: { organizationId, organizationUuid }
```
Called from: `registrationService.js` during tenant registration

### identity-admin-service → organization-admin-service (compensation)
```
DELETE /api/v1/organization-admin/internal/organizations/:organizationId
Headers: Authorization: Bearer <ORGANIZATION_SERVICE_INTERNAL_TOKEN>, X-Tenant-Uuid: <uuid>
```
Called on registration rollback

### patient-admin-service → organization-admin-service
```
GET /api/v1/organization-admin/internal/facilities/:facilityId
GET /api/v1/organization-admin/internal/geography/cities/:cityId
```
Client: `src/clients/organizationDirectory.js`, `geographyDirectory.js`
Used by: `patientService.createPatient()`, address validation

### provider-admin-service → organization-admin-service
```
GET /api/v1/organization-admin/internal/organizations/:id
GET /api/v1/organization-admin/internal/facilities/:id
GET /api/v1/organization-admin/internal/departments/:id
GET /api/v1/organization-admin/internal/geography/*
```
Client: `src/clients/organizationDirectory.js`, `geographyDirectory.js`
Used by: `affiliationService.create()`, `providerService.create()`

### appointment-admin-service → organization-admin-service
```
GET /api/v1/organization-admin/internal/facilities/:id
GET /api/v1/organization-admin/internal/departments/:id
GET /api/v1/organization-admin/internal/facility-resources/:id
```
Client: `src/clients/organizationDirectory.js`
Used by: `appointmentSlotConfigService.create()`, `bookingService.book()`

### appointment-admin-service → patient-admin-service
```
GET /api/v1/patient-admin/internal/patients/:patientId
```
Client: `src/clients/patientDirectory.js`
Used by: `bookingService.book()` — validate patient exists

### appointment-admin-service → provider-admin-service
```
GET /api/v1/provider-admin/internal/providers/:providerId
```
Client: `src/clients/providerDirectory.js`
Used by: `appointmentSlotConfigService.create()` — validate provider exists

### appointment-admin-service → notification-admin-service
```
POST /api/v1/notification/notifications
Headers: Authorization: Bearer <APPOINTMENT_SERVICE_INTERNAL_TOKEN>, X-Tenant-Uuid: <uuid>
Body: { eventCode, recipientPatientId, variables, channel, priority }
```
Client: `src/clients/notificationClient.js`
Used by: `bookingService.book()`, `appointmentNotifier.js` (reminders)

### notification-admin-service → identity-admin-service
```
GET /api/v1/identity-admin/internal/users/:userId
```
Client: Not yet implemented — currently resolves recipient contact from token claims only

### notification-admin-service → patient-admin-service
```
GET /api/v1/patient-admin/internal/patients/:patientId
```
Client: Not yet implemented

### notification-admin-service → provider-admin-service
```
GET /api/v1/provider-admin/internal/providers/:providerId
```
Client: Not yet implemented

---

## Frontend → Backend API Mapping

### identity-admin-ui → identity-admin-service
| UI Feature | Service Endpoints |
|------------|-------------------|
| Login | `POST /auth/login`, `POST /auth/mfa/verify` |
| Dashboard | `GET /users`, `GET /tenants`, `GET /audit-logs` |
| Users | `GET/POST/PATCH /users`, `POST /users/:id/assign-roles` |
| Roles | `GET/POST/PATCH /authorization/roles`, `POST /roles/:id/permissions` |
| Tenants | `GET/POST/PATCH /tenants` |
| IDPs | `GET/POST/PATCH /identity-providers`, `GET /:id/metadata` |
| OAuth Clients | `GET/POST/PATCH /oauth/clients`, `POST /:id/rotate-secret` |
| API Clients | `GET/POST/PATCH /api-clients`, `POST /:id/rotate-secret` |
| MFA | `GET/POST/DELETE /users/:id/mfa` |
| Service Accounts | `GET/POST/PATCH /service-accounts`, `POST /:id/rotate-secret` |
| Security Policy | `GET/PATCH /security-policy` |
| Auth Config | `GET/PATCH /auth-configs` |
| Sessions | `GET/DELETE /sessions` |
| Audit Logs | `GET /audit-logs` |

### organization-admin-ui → Multiple Services
| UI Feature | Primary Service | Endpoints |
|------------|-----------------|-----------|
| Organizations | organization-admin | `/organizations` |
| Facilities | organization-admin | `/facilities` |
| Departments | organization-admin | `/departments` |
| Services | organization-admin | `/services`, `/service-categories`, `/facility-services` |
| Geography | organization-admin | `/geography/*` |
| Patients | patient-admin | `/patients`, `/settings` |
| Providers | provider-admin | `/providers`, `/affiliations` |
| Appointments | appointment-admin | `/appointments`, `/slots`, `/slot-configs`, `/facility-closures` |
| Notifications | notification-admin | `/templates`, `/events`, `/channels`, `/suppressions` |
| Users | identity-admin | `/users` (read-only for assignment) |
| Onboarding | identity + organization | `/public/register-organization`, `/internal/organizations` |

---

## Dependency Graph (Services → Services)

```
identity-admin-service
  ├─→ organization-admin-service (internal/org create + compensation)
  └─→ (issues JWTs consumed by ALL services)

organization-admin-service
  ├─← identity-admin-service (internal/org create)
  ├─→ patient-admin-service (facility/geography validation)
  ├─→ provider-admin-service (org/facility/dept/geography validation)
  └─→ appointment-admin-service (facility/dept/resource validation)

patient-admin-service
  ├─← organization-admin-service (facility/geography)
  └─← identity-admin-service (user audit stamping)

provider-admin-service
  ├─← organization-admin-service (org/facility/dept/geography)
  └─← identity-admin-service (user audit stamping)

appointment-admin-service
  ├─← organization-admin-service (facility/dept/resource)
  ├─← patient-admin-service (patient validation)
  ├─← provider-admin-service (provider validation)
  └─→ notification-admin-service (booking/reminder notifications)

notification-admin-service
  ├─← appointment-admin-service (notifications)
  ├─← identity-admin-service (user contact resolution - TODO)
  ├─← patient-admin-service (patient contact resolution - TODO)
  └─← provider-admin-service (provider contact resolution - TODO)
```

---

## Shared Environment Variables (Cross-Service)

| Variable | Services | Purpose |
|----------|----------|---------|
| `ADMIN_JWT_ISSUER` | All | JWT `iss` claim validation |
| `ADMIN_JWT_AUDIENCE` | All | JWT `aud` claim validation |
| `ADMIN_JWKS_URL` | All (prod) | JWKS endpoint for RS256 verification |
| `ADMIN_JWT_SECRET` | All (dev) | HS256 shared secret fallback |
| `ORGANIZATION_SERVICE_INTERNAL_TOKEN` | identity, organization | Service-to-service auth |
| `PATIENT_SERVICE_INTERNAL_TOKEN` | identity, patient | Service-to-service auth |
| `PROVIDER_SERVICE_INTERNAL_TOKEN` | identity, provider | Service-to-service auth |
| `APPOINTMENT_SERVICE_INTERNAL_TOKEN` | identity, appointment | Service-to-service auth |
| `NOTIFICATION_SERVICE_INTERNAL_TOKEN` | identity, notification | Service-to-service auth |
| `ALLOWED_ORIGINS` | All | CORS origin allowlist |
| `DB_HOST/PORT/NAME/USER/PASSWORD` | All (per service) | Database connection |

---

## Impact Analysis: Changing a Service's API

| Change | Affected Consumers | Mitigation |
|--------|-------------------|------------|
| Identity: JWT claims structure | ALL services (verifyToken) | Version claims; maintain backward compat |
| Identity: JWKS URL/keys | ALL services | Rotate keys via JWKS; cache 10 min |
| Organization: Facility/Dept API | Patient, Provider, Appointment | Version internal endpoints; deploy together |
| Patient: Patient lookup API | Appointment | Version internal endpoint |
| Provider: Provider lookup API | Appointment | Version internal endpoint |
| Notification: Send notification API | Appointment (and future) | Stable eventCode contract; add fields only |

---

## Rate Limits (Per Service)

| Service | Global Limit | Auth Endpoint Limit |
|---------|--------------|---------------------|
| identity-admin | 600/15min | 10/15min (`/auth/login`) |
| organization-admin | 600/15min | N/A (no auth endpoint) |
| patient-admin | 300/15min | N/A |
| provider-admin | 600/15min | N/A |
| appointment-admin | 600/15min | N/A |
| notification-admin | 3000/15min | N/A (bursty reminder traffic) |

---

## Error Response Envelope (All Services)

```json
{
  "error": {
    "code": "VALIDATION_ERROR|INVALID_TOKEN|INSUFFICIENT_PERMISSION|NOT_FOUND|CONFLICT|INTERNAL_ERROR|RATE_LIMITED|INVALID_REFERENCE",
    "message": "Human-readable message",
    "details": ["field-specific messages"]  // only for VALIDATION_ERROR
  }
}
```

Success envelope varies by endpoint but typically:
```json
{ "data": { ... }, "meta": { "page", "limit", "total" } }  // for lists
{ "data": { ... } }  // for single resources
```
# SP3Digital — Functional Flows

## Business Purpose
SP3Digital is a **multi-tenant healthcare administration platform** enabling organizations to manage:
- **Identity & Access**: Users, roles, permissions, tenants, SSO (OIDC/SAML), MFA, OAuth/OIDC clients
- **Organization Master Data**: Hierarchical organizations, facilities, departments, geography (India postal hierarchy), services & categories
- **Patient Registry**: Per-tenant patients with per-facility MRNs, guardians, duplicate detection/merge, access audit, retention/anonymization
- **Provider Registry**: Doctor profiles, qualifications, registrations, facility affiliations (Physical/Remote/Other)
- **Appointment Scheduling**: Recurring slot rules (maker-checker), facility closures, bookings, reminders, token counters
- **Notifications**: Template-driven multi-channel (email/SMS/WhatsApp) delivery with retries, suppression, event triggers

---

## User Types & Roles

| Role | Service | Scope | Capabilities |
|------|---------|-------|--------------|
| **SUPERADMIN** | identity | Cross-tenant | Full platform access; tenant creation; bypasses tenant scoping |
| **TENANT_ADMIN** | identity | Single tenant | User/role/permission management; IDP/OAuth config; security policies; MFA enforcement |
| **TENANT_USER** | identity | Single tenant | Self-service (password, MFA, sessions); read-only on admin resources |
| **ORG_ADMIN** | organization | Tenant-scoped | CRUD organizations, facilities, departments, services, geography |
| **PATIENT_ADMIN** | patient | Tenant-scoped | CRUD patients, merge, anonymize, access logs, facility number sequences |
| **PROVIDER_ADMIN** | provider | Tenant-scoped | CRUD providers, affiliations, qualifications, documents |
| **APPOINTMENT_ADMIN** | appointment | Tenant-scoped | Slot config (create/read all; update/approve TENANT_ADMIN only), closures, bookings, reminders |
| **NOTIFICATION_ADMIN** | notification | Tenant-scoped | Templates, event definitions, channel settings, suppression, delivery logs |

---

## Core Business Workflows

### 1. Self-Service Tenant Registration
**Entry**: `POST /api/v1/identity-admin/public/register-organization` (unauthenticated, IP rate-limited)

```
1. Client submits: { tenantCode, tenantName, adminUser: { username, email, password, firstName, lastName } }
2. identity-admin-service.registrationService.registerOrganization():
   a. Validate tenantCode unique (identity_tenants)
   b. Create tenant row (identity_tenants) → tenantUuid
   c. Create default auth_configuration (password login enabled, MFA optional)
   d. Create default security_policy (password rules, session timeout)
   e. Create TENANT_ADMIN role + ALL_PERMISSIONS for this tenant
   f. Create admin user (users) with tenantUuid, status=ACTIVE
   g. Assign user to TENANT_ADMIN role (user_roles)
   h. Call organization-admin-service POST /internal/organizations (service token + X-Tenant-Uuid)
      → Creates root organization row linked to tenantUuid
   i. Return { tenantUuid, adminUserId, credentials }
3. On org-service failure: Compensation — delete tenant, user, roles (best-effort)
```

**Business Rules**:
- `tenantCode` globally unique, alphanumeric + underscore
- Admin user `username`/`email` unique within tenant
- Password must meet tenant's security_policy (default: 8 chars, 1 upper, 1 lower, 1 digit, 1 special)
- Idempotency: Not implemented — duplicate submissions create multiple tenants

---

### 2. User Login (Password + Optional MFA)
**Entry**: `POST /api/v1/identity-admin/auth/login`

```
1. Client submits: { usernameOrEmail, password, tenantUuid, audience? }
2. authService.login():
   a. Find user by username/email + tenantUuid + status=ACTIVE
   b. credentialService.verifyPassword() → bcrypt compare + failed_login_count/lockout logic
   c. Load tenant's security_policy (mfaRequired)
   d. Check user has active MFA method (mfaService.hasActiveMfa)
   e. If mfaRequired && !hasMfa → 403 MFA_ENROLLMENT_REQUIRED
   f. If hasMfa → issue short-lived MFA challenge token (HS256, 5 min), return { mfaRequired: true, mfaToken }
   g. Else → issueTokens() → accessToken, idToken, refreshToken
3. issueTokens() (in transaction):
   a. resolvePermissions(userId) → query user_roles → roles → role_permissions → permissions
   b. tokenService.signAccessToken(permissions) → JWT with permissions[] claim
   c. tokenService.signIdToken(...) → JWT with profile + mfaVerified
   d. Generate refresh token (opaque), store hash in refresh_tokens + session_token_hash in user_sessions
   e. Update user.last_login_on, reset failed_login_count
4. Client receives tokens → stores in memory (Angular auth.service.ts)
```

**Business Rules**:
- Failed login increments `failed_login_count`; lock after 5 attempts for 15 min (configurable in security_policy)
- MFA challenge token is single-use, audience=`mfa_challenge`
- Access token TTL: `ACCESS_TOKEN_TTL_SECONDS` (default 900s)
- Refresh token TTL: `REFRESH_TOKEN_TTL_DAYS` (default 30 days)
- Session expires with refresh token; `last_activity_on` updated on refresh

---

### 3. MFA Enrollment & Verification
**Entry**: `POST /api/v1/identity-admin/users/:userId/mfa` (enroll), `POST /api/v1/identity-admin/auth/mfa/verify` (verify)

```
Enroll (TOTP):
1. User calls enroll → mfaService.generateTotpSecret(userId) → returns { secret, otpAuthUrl, backupCodes }
2. User scans QR / enters code → mfaService.verifyAndActivateTotp(userId, code)
3. On success: mfa_methods row created (method_type=TOTP, secret_encrypted, is_verified=true, is_primary=true)

Verify (Login Step 2):
1. Client submits { mfaToken, code, audience }
2. authService.completeMfaLogin():
   a. Verify mfaToken (HS256, mfa_challenge audience)
   b. mfaService.verifyLoginCode(userId, code) → TOTP verify + lockout on 5 failures
   c. issueTokens(mfaVerified=true) → tokens with mfaVerified claim in idToken
```

**Business Rules**:
- One primary MFA method per user (enforced by `is_primary` unique partial index)
- Backup codes: 10 generated at enroll, single-use, stored hashed
- TOTP window: ±1 step (30s)
- MFA lockout: 5 failures → 15 min lockout

---

### 4. Role & Permission Management (Identity Service)
**Entry**: `POST/PATCH/GET /api/v1/identity-admin/authorization/roles`, `.../permissions`, `.../role-permissions`

```
Create Role:
1. TENANT_ADMIN submits { roleName, roleType: TENANT|SYSTEM, description, permissionCodes[] }
2. authorizationService.createRole():
   a. Generate role_code = sanitize(roleName) + unique suffix
   b. Create role row (roles)
   c. For each permissionCode → link via role_permissions (validates permission exists + ACTIVE)
3. Role assigned to users via user_roles (user_id, role_id, status=ACTIVE)

Permission Resolution (at login/refresh):
1. UserRoles.findAll(where: { user_id, status: ACTIVE }, include: Role → RolePermissions → Permission)
2. Collect permission_code from all active role-permission mappings
3. Return Set<permission_code> → embedded in access token
```

**Business Rules**:
- `role_type`: `TENANT` (tenant-scoped) vs `SYSTEM` (platform-wide, SUPERADMIN only)
- `SUPERADMIN` role code is hardcoded bypass in `authentication.js:249`
- Permissions reference `resource_master.resource_code` + `action_master.action_code` (FK enforced)
- Deleting a role → soft delete (status=DELETED); existing user_roles remain but filtered by role.status

---

### 5. Organization & Facility Hierarchy
**Entry**: `POST /api/v1/organization-admin/organizations`, `.../facilities`, `.../departments`

```
Create Organization:
1. ORG_ADMIN submits { organizationCode, organizationName, organizationType, parentOrganizationId?, address..., tenantUuid(from token) }
2. organizationService.create():
   a. Validate organizationCode unique within tenant
   b. If parentOrganizationId → verify exists + same tenant
   c. Create organization row (tenantUuid required)
   d. Return organization

Create Facility:
1. Submits { facilityCode, facilityName, facilityType, organizationId, address..., cityId?, districtId?, stateId?, countryId?, postalCodeId? }
2. facilityService.create():
   a. Validate facilityCode unique globally (not per-tenant)
   b. Verify organizationId exists + same tenant (via organization.tenantUuid === req.auth.tenantUuid)
   c. Validate geography IDs exist (FK to cities/districts/states/countries/postal_codes — NOT enforced at DB level)
   d. Create facility row

Create Department:
1. Submits { departmentCode, departmentName, departmentType, facilityId }
2. departmentService.create():
   a. Validate departmentCode unique within facility
   b. Verify facilityId exists + same tenant
   c. Create department row
```

**Business Rules**:
- Organization hierarchy: `parent_organization_id` self-ref FK (no cycles checked)
- Facility → Organization (many-to-one, required)
- Department → Facility (many-to-one, required)
- Geography: Country → State → District → SubDistrict → City (postal hierarchy, India-specific)
- **No DB FKs** from facilities to geography tables (intentional — see `facility.model.js` comment); validated in service only

---

### 6. Patient Registration & Facility Numbers
**Entry**: `POST /api/v1/patient-admin/patients`

```
1. PATIENT_ADMIN submits patient demographics + optional { facilityId, guardians[] }
2. patientService.createPatient():
   a. Duplicate check: exact match on (firstName, lastName, dob, gender) + fuzzy (phone, email) → returns candidates
   b. If candidates exist → return 409 CONFLICT with masked candidates (caller decides merge vs new)
   c. Generate patientNumber per facility: patientNumberSequence.next(facilityId) → zero-padded
   d. Create patient row (tenantUuid from token)
   e. Create patient_facility_number row (patientId, facilityId, patientNumber, isPrimary=true)
   f. Create patientAccessLog entry (action=CREATE, actorUserId)
   g. Return patient with facilityNumber
```

**Business Rules**:
- Patient `tenantUuid` from token (implicit scoping)
- Per-facility MRN: sequence per (tenant, facility), format configurable (default: 8-digit zero-padded)
- Guardians: separate `guardians` array linked via `patient_guardians` (not in current dump — verify model)
- Duplicate detection: exact (name+dob+gender) + fuzzy (phone/email) — threshold configurable
- Merge: `patient_merge` table records source→target, reason, actor; source patient status=MERGED
- Anonymization: `npm run anonymize` → finds patients where `retention_expires_on <= NOW()`, scrubs PII, sets status=ANONYMIZED

---

### 7. Provider Profile & Affiliations
**Entry**: `POST /api/v1/provider-admin/providers`, `.../affiliations`

```
Create Provider:
1. PROVIDER_ADMIN submits profile (name, gender, dob, contact, photo) + arrays: qualifications, registrations, specialties, languages, experience, documents
2. providerService.create():
   a. Create provider row (tenantUuid from token)
   b. Bulk insert related tables (provider_qualifications, etc.) with providerId
   c. Return provider with all relations

Create Affiliation:
1. Submits { providerId, facilityId, organizationId, departmentId, affiliationType: PHYSICAL|REMOTE|OTHER, availabilitySchedule?, startDate, endDate? }
2. affiliationService.create():
   a. Verify provider, facility, organization, department all exist + same tenant
   b. Validate no overlapping PHYSICAL affiliation at same facility for same time
   c. Create provider_affiliation row
   d. If services provided → bulk insert provider_affiliation_services
```

**Business Rules**:
- Provider `tenantUuid` from token
- Affiliation types: `PHYSICAL` (on-site), `REMOTE` (telehealth), `OTHER`
- One primary affiliation per provider (enforced by `is_primary` flag)
- Qualifications/registrations have `verificationStatus` (PENDING/VERIFIED/REJECTED/EXPIRED)
- Documents encrypted at rest via `secretProtector.js`

---

### 8. Appointment Slot Configuration (Maker-Checker)
**Entry**: `POST /api/v1/appointment-admin/slot-configs` (create), `PATCH .../:id/approve` (approve)

```
Create Slot Config (TENANT_USER or TENANT_ADMIN):
1. Submits { facilityId, departmentId, providerId?, resourceId?, recurrenceRule (RRULE), startTime, endTime, slotDuration, maxAppointmentsPerSlot, effectiveFrom, effectiveTo, timezone }
2. appointmentSlotConfigService.create():
   a. Validate all refs exist + same tenant
   b. Parse RRULE (rrule lib) → validate generates slots
   c. Create appointment_slot_config row (status=PENDING_APPROVAL)
   d. Return config

Approve Slot Config (TENANT_ADMIN only):
1. PATCH /:id/approve → appointmentSlotConfigService.approve(configId, actorUserId)
2. Transaction:
   a. Verify config.status = PENDING_APPROVAL
   b. Generate slots via slotGenerationService.generateForConfig(config) → bulk insert appointment_slots
   c. Update config.status = ACTIVE, approvedBy=actorUserId, approvedOn=NOW
   d. Return { config, slotsGenerated }

Slot Generation (background `npm run generate-slots`):
1. Finds all ACTIVE configs where effectiveFrom <= today <= effectiveTo
2. For each config → generates slots for next N days (configurable)
3. Skips dates with facility_closure
4. Upserts appointment_slots (ON DUPLICATE KEY UPDATE)
```

**Business Rules**:
- RRULE subset: FREQ=DAILY/WEEKLY, INTERVAL, BYDAY, COUNT/UNTIL
- Slot duration: 5-120 min, multiples of 5
- Max appointments per slot: 1-10 (group booking support)
- Maker (TENANT_USER) cannot approve own config; requires TENANT_ADMIN
- Facility closures block slot generation for that facility+date

---

### 9. Appointment Booking
**Entry**: `POST /api/v1/appointment-admin/appointments`

```
1. Caller (patient/front-desk) submits { patientId, slotId, bookingReason, notes }
2. bookingService.book():
   a. Verify slot exists, status=AVAILABLE, slot.startTime > NOW
   b. Verify patient exists + same tenant
   c. Verify provider/facility/department from slot match patient's tenant
   d. Check patient not double-booked at same time
   e. Create appointment row (status=BOOKED)
   f. Update slot.status = BOOKED, appointmentId
   g. Create appointmentStatusHistory (BOOKED)
   h. Increment appointmentTokenCounter for (facility, date) → tokenNumber
   i. Trigger notification: POST /api/v1/notification/notifications { eventCode: APPOINTMENT_BOOKED, recipientPatientId: patientId, variables: { tokenNumber, slotTime, facilityName, providerName } }
3. Return appointment with tokenNumber
```

**Business Rules**:
- Token counter: per facility per day, sequential (1, 2, 3...)
- Cancellation: `PATCH /appointments/:id/cancel` → reason required → slot=AVAILABLE, appointment=CANCELLED, notification APPOINTMENT_CANCELLED
- Reschedule: cancel + rebook (new token number)
- Reminders: `tenant_notification_settings` defines lead times (e.g., 24h, 2h); worker sends via notification service

---

### 10. Notification Delivery
**Entry**: `POST /api/v1/notification/notifications` (from other services) or event-triggered

```
Send Notification:
1. Caller submits { eventCode, tenantUuid, recipientUserId?, recipientPatientId?, recipientProviderId?, channel: EMAIL|SMS|WHATSAPP, variables{}, priority?, scheduledFor? }
2. notificationService.send():
   a. Resolve eventDefinition (notification_event_definitions) by eventCode + tenantUuid
   b. Resolve template (notification_templates) by eventCode + channel + tenantUuid + locale
   c. Render subject/body via template.util.js (Handlebars-like {{var}} replacement)
   d. Determine recipient contact (email/phone) from user/patient/provider via directory clients
   e. Create notification row (status=QUEUED, scheduledFor)
   f. Return notificationId

Worker (dispatcher.js, every 5s):
1. SELECT * FROM notifications WHERE status=QUEUED AND scheduledFor <= NOW() LIMIT 100
2. For each:
   a. Update status=SENDING
   b. Call provider (emailProvider/smsProvider/whatsappProvider)
   c. Create notificationAttempt (providerResponse, status=SENT/FAILED)
   d. On SENT → notification.status=SENT, sentOn=NOW
   e. On FAILED → increment attemptCount; if < maxRetries → status=QUEUED, scheduledFor=NOW+backoff; else status=FAILED
3. Backoff: 1m, 5m, 15m, 1h, 6h, 24h (configurable)

Suppression:
- notification_suppressions: (tenantUuid, contactValue, channel, reason, expiresOn?)
- Checked before queueing; suppressed → status=SUPPRESSED, no attempt
```

**Business Rules**:
- Default templates shipped via `templateService.ensureDefaults()` + `eventDefinitionService.ensureDefaults()`
- Channels: Email (Nodemailer/SMTP), SMS (Twilio), WhatsApp (Twilio)
- Template variables: `{{patientName}}`, `{{tokenNumber}}`, `{{appointmentTime}}`, `{{facilityName}}`, `{{providerName}}`, custom
- Retention: `purge.js` deletes notifications/attempts older than `NOTIFICATION_RETENTION_DAYS` (default 365)

---

## Cross-Service Data Dependencies

| Consumer | Provider | Data Accessed | Purpose |
|----------|----------|---------------|---------|
| patient-admin | organization-admin | Facilities, geography | Validate facilityId, address hierarchy |
| patient-admin | identity-admin | Users (for createdBy) | Audit stamping |
| provider-admin | organization-admin | Facilities, organizations, departments, geography | Validate affiliation refs |
| provider-admin | identity-admin | Users | Audit stamping |
| appointment-admin | organization-admin | Facilities, departments, resources | Validate slot config refs |
| appointment-admin | patient-admin | Patients | Validate patientId on booking |
| appointment-admin | provider-admin | Providers | Validate providerId on slot config |
| appointment-admin | notification-admin | Send notifications | Booking confirmations, reminders |
| notification-admin | identity-admin | Users | Resolve recipient email/phone |
| notification-admin | patient-admin | Patients | Resolve recipient contact |
| notification-admin | provider-admin | Providers | Resolve recipient contact |
| organization-admin | identity-admin | (inbound) | Service-to-service token validation |

---

## Error Scenarios & Expected Outcomes

| Scenario | Service | HTTP | Error Code | Client Action |
|----------|---------|------|------------|---------------|
| Invalid/expired JWT | All | 401 | INVALID_TOKEN | Redirect to login |
| Missing tenantUuid claim | All | 403 | TENANT_CLAIM_REQUIRED | Re-authenticate |
| Tenant mismatch (param vs token) | identity | 403 | TENANT_MISMATCH | Check URL params |
| Insufficient permission | All | 403 | INSUFFICIENT_PERMISSION | Show access denied |
| Validation failure (Joi) | All | 400 | VALIDATION_ERROR | Show field errors |
| Duplicate unique key | All | 400/409 | DUPLICATE_ENTRY / CONFLICT | Handle per field |
| FK reference missing | appointment | 400 | INVALID_REFERENCE | Validate refs client-side |
| Patient duplicate detected | patient | 409 | CONFLICT | Show candidates, allow merge |
| MFA required but not enrolled | identity | 403 | MFA_ENROLLMENT_REQUIRED | Redirect to MFA enroll |
| MFA code invalid/expired | identity | 401 | INVALID_MFA_CODE / MFA_CHALLENGE_EXPIRED | Re-enter code |
| Refresh token reused | identity | 401 | REFRESH_TOKEN_REUSED | Force full login |
| Rate limit exceeded | All | 429 | RATE_LIMITED | Retry after window |

---

## State Transitions (Key Entities)

### User (identity)
```
PENDING_VERIFICATION → ACTIVE → (LOCKED) → ACTIVE → DELETED
                            ↑_______|
```
- Locked via failed login or admin action; unlock via admin or time expiry

### Patient (patient)
```
ACTIVE → MERGED (source) / ACTIVE (target)
ACTIVE → ANONYMIZED (retention expiry)
ACTIVE → DELETED (admin action, soft)
```

### Appointment Slot Config (appointment)
```
PENDING_APPROVAL → ACTIVE → EXPIRED (effectiveTo passed)
PENDING_APPROVAL → REJECTED (admin action)
ACTIVE → SUPERSEDED (new config overlaps)
```

### Appointment (appointment)
```
BOOKED → CHECKED_IN → COMPLETED
BOOKED → CANCELLED (patient/provider/admin)
BOOKED → NO_SHOW (post slot time)
BOOKED → RESCHEDULED (→ new appointment BOOKED)
```

### Notification (notification)
```
QUEUED → SENDING → SENT
QUEUED → SENDING → FAILED → QUEUED (retry) → ... → FAILED (max retries)
QUEUED → SUPPRESSED
```

---

## External Integrations

| Integration | Service | Protocol | Purpose |
|-------------|---------|----------|---------|
| **OIDC/SAML IdPs** | identity | OIDC/SAML | Federated login (Google, Azure AD, Okta, custom) |
| **SMTP** | notification | SMTP | Email delivery (Nodemailer) |
| **Twilio** | notification | REST API | SMS + WhatsApp delivery |
| **Geography CSV** | organization | File import | India postal hierarchy seed (`seed:india-geo`) |

---

## Business Rule Summary (Invariants)

1. **Tenant Isolation**: No query crosses `tenantUuid` boundary (except SUPERADMIN)
2. **Unique Codes**: `*_code` fields unique within scope (tenant, facility, organization)
3. **Soft Delete**: `status=DELETED` hides from default queries; no hard deletes
4. **Audit Trail**: All write operations in identity service logged; patient access logged; notification attempts logged
5. **Maker-Checker**: Slot configs require TENANT_ADMIN approval
6. **Refresh Token Rotation**: Single-use; reuse revokes entire session
7. **MFA Enforcement**: Tenant policy can require MFA; unenrolled users blocked at login
8. **Password Policy**: Per-tenant via `security_policies`; validated on set/change
9. **Facility MRN**: Sequential per (tenant, facility); never reused
10. **Token Audience**: Every access token carries `audience`; services validate their own
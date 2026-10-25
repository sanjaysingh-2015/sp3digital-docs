# SP3Digital — Module Index

## Backend Services — Key Files by Responsibility

### sp3digital-identity-admin-service

#### Entry & Config
| File | Purpose |
|------|---------|
| `src/app.js` | Express app setup, middleware pipeline, route mounting, error handling |
| `src/server.js` | Legacy entry (unused — kept for reference) |
| `src/config/db.js` | Sequelize instance (MySQL connection pool) |
| `src/config/swagger.js` | OpenAPI spec generation |

#### Middleware
| File | Purpose |
|------|---------|
| `src/middleware/authentication.js` | JWT verification (HS256/RS256), tenant extraction, scope parsing, `authenticate()`, `authorize()` (dual-gate), `tenantMatchesPath()` |
| `src/middleware/validate.js` | Joi validation wrapper |
| `src/middleware/audit.js` | `auditWrites` — logs mutating requests to `audit_logs` |

#### Models (`src/models/*.model.js`)
| File | Entity |
|------|--------|
| `users.model.js` | User accounts |
| `roles.model.js` | Role definitions |
| `permissions.model.js` | Permission catalog |
| `user_roles.model.js` | User↔Role assignment |
| `role_permissions.model.js` | Role↔Permission mapping |
| `resource_master.model.js` | Resource catalog |
| `action_master.model.js` | Action catalog |
| `identity_tenants.model.js` | Tenant registry |
| `user_sessions.model.js` | Login sessions |
| `refresh_tokens.model.js` | Refresh token store |
| `access_tokens.model.js` | Access token store |
| `mfa_methods.model.js` | Enrolled MFA factors |
| `user_devices.model.js` | Trusted devices |
| `user_credentials.model.js` | Current password hash |
| `user_credential_history.model.js` | Password history |
| `identity_providers.model.js` | OIDC/SAML IdP config |
| `external_identities.model.js` | User↔IdP linkage |
| `oauth_clients.model.js` | OAuth 2.0 clients |
| `api_clients.model.js` | API client credentials |
| `api_client_scopes.model.js` | API client↔Permission |
| `service_accounts.model.js` | Machine-to-machine accounts |
| `security_policies.model.js` | Per-tenant password/session/MFA policy |
| `auth_configurations.model.js` | Per-tenant login method config |
| `audit_logs.model.js` | Write-operation audit trail |
| `organizations_users.model.js` | User↔Organization membership (ref) |
| `facilities_users.model.js` | User↔Facility membership (ref) |

#### Associations
| File | Purpose |
|------|---------|
| `src/models/associations.js` | All Sequelize relationships (hasMany/belongsTo) — wired in `models/index.js` |

#### Controllers (`src/controllers/*.js`)
| File | Responsibility |
|------|----------------|
| `authController.js` | Login, MFA verify, refresh, logout, change-password |
| `userController.js` | User CRUD, role assignment |
| `tenantController.js` | Tenant CRUD (SUPERADMIN) |
| `authorizationController.js` | Roles, permissions, role-permissions |
| `idpController.js` | Identity providers (OIDC/SAML) |
| `oAuthClientController.js` | OAuth clients |
| `apiClientController.js` | API clients |
| `mfaController.js` | MFA enroll/verify/remove |
| `serviceAccountController.js` | Service accounts |
| `securityPolicyController.js` | Security policies |
| `authConfigController.js` | Auth configurations |
| `auditController.js` | Audit log queries |
| `sessionController.js` | Session queries/revoke |
| `resourceActionController.js` | Resource-action catalog |
| `accessControlController.js` | Access control checks |
| `credentialController.js` | Password operations |
| `publicController.js` | Public registration + tenant search |

#### Services (`src/services/*.js`)
| File | Responsibility |
|------|----------------|
| `authService.js` | Login flow, MFA challenge, token issuance, refresh rotation, logout, password change |
| `userService.js` | User CRUD, tenant scoping, SUPERADMIN bypass |
| `tenantService.js` | Tenant CRUD, SUPERADMIN bypass |
| `authorizationService.js` | Roles, permissions, role-permissions, permission resolution |
| `idpService.js` | IdP CRUD, metadata generation |
| `oAuthClientService.js` | OAuth client CRUD, secret rotation |
| `apiClientService.js` | API client CRUD, secret rotation |
| `mfaService.js` | TOTP generate/verify, backup codes, lockout |
| `serviceAccountService.js` | Service account CRUD, secret rotation |
| `securityPolicyService.js` | Policy CRUD, defaults |
| `credentialService.js` | Password verify/set/history, bcrypt |
| `registrationService.js` | Self-service tenant + admin user + org creation (compensation) |
| `auditService.js` | Audit log write |
| `resourceActionService.js` | Resource-action catalog |
| `sessionService.js` | Session queries |

#### Security Utils
| File | Purpose |
|------|---------|
| `src/security/tokenService.js` | JWT signing (access/id), refresh token generation, hashing, TTL config |
| `src/security/secretProtector.js` | AES-256-GCM encryption/decryption with key versioning (client secrets, MFA secrets) |
| `src/security/totp.js` | TOTP generation/verification (otplib) |

#### Validations (`src/validations/*.validation.js`)
| File | Schemas |
|------|---------|
| `registration.validation.js` | Tenant registration payload |
| `tenant.validation.js` | Tenant CRUD |
| `identityProvider.validation.js` | IdP config |

#### Utils
| File | Purpose |
|------|---------|
| `src/utils/code.util.js` | Error code constants |
| `src/utils/pagination.js` | Pagination helper (page/limit → offset/limit) |
| `src/utils/secretCipher.util.js` | Legacy encryption (deprecated → secretProtector) |
| `src/utils/uuid.util.js` | UUID v4 generator |
| `src/utils/lifecycle.js` | App lifecycle hooks |

#### Tests (`tests/*.test.js`)
| File | Coverage |
|------|----------|
| `authentication.test.js` | JWT verify, scope parsing |
| `authorization-service.test.js` | Permission resolution |
| `audit-service.test.js` | Audit write |
| `credential-tenant-scope.test.js` | Credential tenant isolation |
| `login-rate-limit.test.js` | Login rate limiter |
| `model-associations.test.js` | Sequelize associations |
| `resource-action-service.test.js` | Resource-action catalog |
| `service-account-service.test.js` | Service account CRUD |

---

### sp3digital-organization-admin-service

#### Entry & Config
| File | Purpose |
|------|---------|
| `src/app.js` | Express app, middleware, route mounting |
| `src/config/db.js` | Sequelize instance |
| `src/config/swagger.js` | OpenAPI spec |

#### Middleware
| File | Purpose |
|------|---------|
| `src/middleware/authentication.js` | JWT verify (shared logic), `isInternalServiceToken()`, `authenticate()`, `authorize()` (token-only, not mounted), `requireInternalService()` |

#### Models (`src/models/*.model.js`)
| File | Entity |
|------|--------|
| `organization.model.js` | Organizations (hierarchical) |
| `facility.model.js` | Facilities |
| `department.model.js` | Departments |
| `service.model.js` | Services catalog |
| `serviceCategory.model.js` | Service categories |
| `facilityService.model.js` | Facility↔Service↔Category↔Dept |
| `country.model.js`, `state.model.js`, `district.model.js`, `subDistrict.model.js`, `city.model.js`, `postalCode.model.js` | Geography hierarchy |

#### Controllers
| File | Resource |
|------|----------|
| `organizationController.js` | Organizations |
| `facilityController.js` | Facilities |
| `departmentController.js` | Departments |
| `serviceController.js` | Services |
| `serviceCategoryController.js` | Service categories |
| `facilityServiceCatalogController.js` | Facility services |
| `geographyController.js` | Geography read-only |

#### Services
| File | Responsibility |
|------|----------------|
| `organizationService.js` | Org CRUD, hierarchy, tenant scoping |
| `facilityService.js` | Facility CRUD, org validation |
| `departmentService.js` | Department CRUD |
| `serviceService.js` | Service catalog |
| `serviceCategoryService.js` | Category catalog |
| `facilityServiceCatalogService.js` | Facility service mappings |
| `geographyService.js` | Geography queries |

#### Clients
| File | Target Service |
|------|----------------|
| (none — this service is a provider, not a consumer) | — |

#### Scripts
| File | Purpose |
|------|---------|
| `scripts/seed-india-geo.js` | Load India postal hierarchy (countries→states→districts→sub-districts→cities→postal_codes) |

#### Validations
| File | Schemas |
|------|---------|
| `organization.validation.js` | Org CRUD |
| `facility.validation.js` | Facility CRUD |
| `department.validation.js` | Department CRUD |
| `service.validation.js` | Service CRUD |
| `serviceCategory.validation.js` | Category CRUD |
| `facilityService.validation.js` | Facility service mapping |
| `geography.validation.js` | Geography queries |

---

### sp3digital-patient-admin-service

#### Entry & Config
| File | Purpose |
|------|---------|
| `src/app.js` | Express app, middleware, route mounting |
| `src/config/db.js` | Sequelize instance |
| `src/config/swagger.js` | OpenAPI spec |

#### Middleware
| File | Purpose |
|------|---------|
| `src/middleware/authentication.js` | JWT verify, `isInternalServiceToken()`, `authenticate()`, `authorize()` (token scopes), `hasPermission()` |

#### Models
| File | Entity |
|------|--------|
| `patient.model.js` | Patients |
| `patientFacilityNumber.model.js` | Patient↔Facility MRN |
| `patientMerge.model.js` | Merge audit |
| `patientAccessLog.model.js` | Access audit |
| `patientNumberSequence.model.js` | MRN sequence per facility |
| `tenantPatientSetting.model.js` | Per-tenant config |

#### Controllers
| File | Resource |
|------|----------|
| `patientController.js` | Patients, search, duplicate check, merge, facility numbers, access log |

#### Services
| File | Responsibility |
|------|----------------|
| `patientService.js` | Patient CRUD, duplicate detection, merge, MRN generation, access logging |
| `settingsService.js` | Tenant patient settings |

#### Clients
| File | Target Service |
|------|----------------|
| `src/clients/organizationDirectory.js` | organization-admin (facilities, geography) |
| `src/clients/geographyDirectory.js` | organization-admin (geography) |

#### Jobs
| File | Purpose |
|------|---------|
| `src/jobs/anonymizeExpired.js` | Anonymize patients past retention (`npm run anonymize`) |

#### Validations
| File | Schemas |
|------|---------|
| `patient.validation.js` | Patient CRUD, search, merge |

#### Utils
| File | Purpose |
|------|---------|
| `src/utils/patient.util.js` | Patient helpers (name formatting, age calc) |
| `src/utils/permissions.js` | Permission code constants |
| `src/utils/errors.js` | Custom error classes |
| `src/utils/pagination.js` | Pagination helper |

---

### sp3digital-provider-admin-service

#### Entry & Config
| File | Purpose |
|------|---------|
| `src/app.js` | Express app, middleware, route mounting |
| `src/config/db.js` | Sequelize instance |
| `src/config/swagger.js` | OpenAPI spec |

#### Middleware
| File | Purpose |
|------|---------|
| `src/middleware/authentication.js` | JWT verify, `isInternalServiceToken()`, `authenticate()`, `authorize()`, `hasPermission()` |

#### Models
| File | Entity |
|------|--------|
| `provider.model.js` | Providers |
| `providerAffiliation.model.js` | Provider↔Facility/Org/Dept |
| `providerAffiliationService.model.js` | Affiliation↔Service |
| `providerDocument.model.js` | Documents (encrypted) |
| `providerExperience.model.js` | Work history |
| `providerLanguage.model.js` | Languages |
| `providerQualification.model.js` | Qualifications |
| `providerRegistration.model.js` | Council registrations |
| `providerSpecialty.model.js` | Specialties |

#### Controllers
| File | Resource |
|------|----------|
| `providerController.js` | Providers + nested qualifications/registrations/specialties/languages/experience/documents |
| `affiliationController.js` | Affiliations + services |

#### Services
| File | Responsibility |
|------|----------------|
| `providerService.js` | Provider CRUD + nested relations |
| `affiliationService.js` | Affiliation CRUD, overlap validation, services |

#### Clients
| File | Target Service |
|------|----------------|
| `src/clients/organizationDirectory.js` | organization-admin (orgs, facilities, depts, geography) |
| `src/clients/geographyDirectory.js` | organization-admin (geography) |

#### Validations
| File | Schemas |
|------|---------|
| `provider.validation.js` | Provider + nested |
| `affiliation.validation.js` | Affiliation + services |

---

### sp3digital-appointment-admin-service

#### Entry & Config
| File | Purpose |
|------|---------|
| `src/app.js` | Express app, middleware, route mounting |
| `src/config/db.js` | Sequelize instance |
| `src/config/swagger.js` | OpenAPI spec |

#### Middleware
| File | Purpose |
|------|---------|
| `src/middleware/authentication.js` | JWT verify, `isInternalServiceToken()`, `authenticate()`, `authorize()`, `hasPermission()` |

#### Models
| File | Entity |
|------|--------|
| `appointment.model.js` | Appointments |
| `appointmentSlot.model.js` | Generated slots |
| `appointmentSlotConfig.model.js` | Recurring slot rules |
| `appointmentCancellationReason.model.js` | Cancellation reasons |
| `appointmentReminder.model.js` | Reminder schedule |
| `appointmentStatusHistory.model.js` | Status audit |
| `appointmentTokenCounter.model.js` | Daily token sequence |
| `facilityClosure.model.js` | Closure dates |
| `facilityResource.model.js` | Bookable resources |
| `tenantNotificationSetting.model.js` | Reminder lead times + channels |

#### Controllers
| File | Resource |
|------|----------|
| `appointmentController.js` | Appointments CRUD, cancel, reschedule |
| `bookingController.js` | Booking workflow (token generation) |
| `appointmentSlotConfigController.js` | Slot configs (maker-checker) |
| `facilityClosureController.js` | Closures |

#### Services
| File | Responsibility |
|------|----------------|
| `appointmentService.js` | Appointment CRUD, status transitions |
| `bookingService.js` | Booking, token counter, notification trigger |
| `appointmentSlotConfigService.js` | Slot config CRUD, approve/reject, slot generation trigger |
| `slotGenerationService.js` | RRULE parsing, bulk slot generation |
| `slotService.js` | Slot queries, availability |
| `facilityClosureService.js` | Closure CRUD |
| `providerResourceService.js` | Resource validation |
| `reminderSettingsService.js` | Tenant reminder settings |
| `appointmentNotifier.js` | Reminder sending (calls notification service) |

#### Clients
| File | Target Service |
|------|----------------|
| `src/clients/organizationDirectory.js` | organization-admin (facilities, depts, resources) |
| `src/clients/patientDirectory.js` | patient-admin (patients) |
| `src/clients/providerDirectory.js` | provider-admin (providers) |
| `src/clients/notificationClient.js` | notification-admin (send notifications) |

#### Scripts
| File | Purpose |
|------|---------|
| `scripts/generate-slots.js` | Batch slot generation for active configs (`npm run generate-slots`) |

#### Validations
| File | Schemas |
|------|---------|
| `appointment.validation.js` | Appointment CRUD |
| `appointmentSlotConfig.validation.js` | Slot config CRUD, RRULE |
| `booking.validation.js` | Booking payload |
| `facilityClosure.validation.js` | Closure CRUD |

#### Utils
| File | Purpose |
|------|---------|
| `src/utils/clock.js` | Timezone-aware date/time helpers |
| `src/utils/permissions.js` | Permission code constants |
| `src/utils/errors.js` | Custom error classes |

---

### sp3digital-notification-admin-service

#### Entry & Config
| File | Purpose |
|------|---------|
| `src/app.js` | Express app, middleware, route mounting, worker startup |
| `src/config/db.js` | Sequelize instance |
| `src/config/swagger.js` | OpenAPI spec |

#### Middleware
| File | Purpose |
|------|---------|
| `src/middleware/authentication.js` | JWT verify, `isInternalServiceToken()`, `authenticate()`, `authorize()`, `hasPermission()` |

#### Models
| File | Entity |
|------|--------|
| `notification.model.js` | Outbox queue |
| `notificationAttempt.model.js` | Delivery attempts |
| `notificationTemplate.model.js` | Channel templates |
| `notificationEventDefinition.model.js` | Event catalog |
| `notificationSuppression.model.js` | Suppression list |
| `tenantChannelSetting.model.js` | Per-tenant channel config |

#### Controllers
| File | Resource |
|------|----------|
| `notificationController.js` | Notifications, templates, events, suppressions, channels |

#### Services
| File | Responsibility |
|------|----------------|
| `notificationService.js` | Send notification (resolve template, queue), retry logic |
| `templateService.js` | Template CRUD, `ensureDefaults()` (shipped templates) |
| `eventDefinitionService.js` | Event definition CRUD, `ensureDefaults()` |
| `settingsService.js` | Channel settings CRUD |
| `suppressionService.js` | Suppression CRUD |

#### Providers (`src/providers/`)
| File | Channel |
|------|---------|
| `emailProvider.js` | Nodemailer/SMTP |
| `smsProvider.js` | Twilio SMS |
| `whatsappProvider.js` | Twilio WhatsApp |

#### Worker (`src/worker/`)
| File | Purpose |
|------|---------|
| `dispatcher.js` | Polls `notifications` (QUEUED, scheduled_for <= NOW), delivers via providers, records attempts, handles retry/backoff |
| `purge.js` | Deletes old notifications/attempts per retention (`npm run purge`) |

#### Defaults (`src/defaults/`)
| File | Purpose |
|------|---------|
| `templates.js` | Shipped notification templates |
| `events.js` | Shipped event definitions |

#### Validations
| File | Schemas |
|------|---------|
| `notification.validation.js` | All notification resources |

#### Utils
| File | Purpose |
|------|---------|
| `src/utils/template.util.js` | Template rendering ({{var}} replacement) |
| `src/utils/crypto.util.js` | Crypto helpers |
| `src/utils/mask.js` | PII masking for logs |
| `src/utils/clock.js` | Time helpers |
| `src/utils/permissions.js` | Permission code constants |

---

## Frontend — Key Files

### sp3digital-identity-admin-ui

#### Core (`src/app/core/`)
| File | Purpose |
|------|---------|
| `auth.service.ts` | Token storage, login, refresh, logout, user profile |
| `auth.interceptor.ts` | Attaches Bearer token to requests |
| `auth-error.interceptor.ts` | 401 → logout redirect |
| `auth.guard.ts` | Route protection (authenticated only) |
| `user.service.ts` | Current user permissions, `hasPermission()` |
| `api.service.ts` | Base HTTP client |
| `appointment-api.service.ts` | Appointment service calls |
| `org-api.service.ts` | Organization service calls |
| `geo.service.ts` | Geography service calls |
| `ui.service.ts` | UI state (loading, toasts) |
| `config.ts` | Environment config |

#### Features (`src/app/features/<domain>/`)
| Domain | Key Components |
|--------|----------------|
| `users` | User list, form, role assignment |
| `roles` | Role list, form, permission matrix |
| `tenants` | Tenant list, form |
| `idp` | IdP list, form, metadata view |
| `oauth` | OAuth client list, form, secret rotation |
| `api-clients` | API client list, form, secret rotation |
| `mfa` | MFA enroll/verify |
| `service-accounts` | Service account list, form |
| `security-policy` | Policy form |
| `auth-config` | Auth config form |
| `audit` | Audit log viewer |
| `permissions` | Permission catalog |
| `sessions` | Session viewer |
| `dashboard` | Dashboard widgets |
| `login` | Login form, MFA challenge |
| `register-organization` | Public registration form |

#### Shared
| File | Purpose |
|------|---------|
| `src/app/shared/components/confirm-modal` | Confirmation dialog |
| `src/app/shared/components/notification-modal` | Toast notifications |
| `src/app/shared/components/assign-roles-modal` | Role assignment UI |

---

### sp3digital-organization-admin-ui

#### Core (`src/app/core/`)
| File | Purpose |
|------|---------|
| `auth.service.ts` | Token storage, login, refresh, logout, user profile |
| `auth.interceptor.ts` | Attaches Bearer token |
| `auth-error.interceptor.ts` | 401 handling |
| `auth.guard.ts` | Route protection |
| `user.service.ts` | Current user permissions |
| `api.service.ts` | Base HTTP client |
| `org-directory.service.ts` | Organization service client |
| `patient-api.service.ts` | Patient service client |
| `provider-api.service.ts` | Provider service client |
| `appointment-api.service.ts` | Appointment service client |
| `notification-api.service.ts` | Notification service client |
| `geography.service.ts` | Geography service client |
| `appointment-permissions.ts` | Appointment permission checks |
| `patient-permissions.ts` | Patient permission checks |
| `provider-permissions.ts` | Provider permission checks |
| `notification-permissions.ts` | Notification permission checks |
| `config.ts` | Environment config |
| `ui.service.ts` | UI state |

#### Features (`src/app/features/<domain>/`)
| Domain | Key Components |
|--------|----------------|
| `organizations` | Org tree, form, hierarchy |
| `facilities` | Facility list, form, map |
| `departments` | Department list, form |
| `services` | Service catalog, facility services |
| `service-categories` | Category list, form |
| `facility-services` | Facility↔Service mapping |
| `facility-closures` | Closure calendar, form |
| `appointment-slot-configs` | Slot config grid, maker-checker UI |
| `appointments` | Booking calendar (FullCalendar), book modal, quick-register |
| `patients` | Patient list, form, detail, picker |
| `providers` | Provider list, form, detail, affiliations |
| `notification-channels` | Channel config (SMTP, Twilio) |
| `notification-templates` | Template editor (Handlebars) |
| `notification-events` | Event definition list, form |
| `users` | User list, role assignment |
| `onboarding` | First-time setup wizard |
| `dashboard` | Dashboard widgets |
| `register-organization` | Public registration form |
| `login` | Login form |

#### Shared Components (`src/app/shared/components/`)
| Component | Purpose |
|-----------|---------|
| `address-picker` | Country→State→District→City→PostalCode cascade |
| `assign-roles-modal` | Role assignment |
| `confirm-modal` | Confirmation dialog |
| `notification-modal` | Toast notifications |

---

## Symbol Index (Key Exported Functions/Classes)

### Identity Service
| Symbol | File | Type |
|--------|------|------|
| `AuthService` | `src/services/authService.js` | Class (singleton export) |
| `UserService` | `src/services/userService.js` | Class |
| `TenantService` | `src/services/tenantService.js` | Class |
| `AuthorizationService` | `src/services/authorizationService.js` | Class |
| `RegistrationService` | `src/services/registrationService.js` | Class |
| `MfaService` | `src/services/mfaService.js` | Class |
| `CredentialService` | `src/services/credentialService.js` | Class |
| `SecurityPolicyService` | `src/services/securityPolicyService.js` | Class |
| `TokenService` | `src/security/tokenService.js` | Module (functions) |
| `SecretProtector` | `src/security/secretProtector.js` | Class |
| `authenticate` | `src/middleware/authentication.js` | Function (Express middleware) |
| `authorize` | `src/middleware/authentication.js` | Function (returns middleware) |
| `auditWrites` | `src/middleware/audit.js` | Function (Express middleware) |

### Organization Service
| Symbol | File | Type |
|--------|------|------|
| `OrganizationService` | `src/services/organizationService.js` | Class |
| `FacilityService` | `src/services/facilityService.js` | Class |
| `DepartmentService` | `src/services/departmentService.js` | Class |
| `GeographyService` | `src/services/geographyService.js` | Class |
| `authenticate` | `src/middleware/authentication.js` | Function |
| `authorize` | `src/middleware/authentication.js` | Function (not mounted) |
| `requireInternalService` | `src/middleware/authentication.js` | Function |

### Patient Service
| Symbol | File | Type |
|--------|------|------|
| `PatientService` | `src/services/patientService.js` | Class |
| `SettingsService` | `src/services/settingsService.js` | Class |
| `authenticate` | `src/middleware/authentication.js` | Function |
| `authorize` | `src/middleware/authentication.js` | Function |
| `hasPermission` | `src/middleware/authentication.js` | Function |

### Provider Service
| Symbol | File | Type |
|--------|------|------|
| `ProviderService` | `src/services/providerService.js` | Class |
| `AffiliationService` | `src/services/affiliationService.js` | Class |
| `authenticate` | `src/middleware/authentication.js` | Function |
| `authorize` | `src/middleware/authentication.js` | Function |
| `hasPermission` | `src/middleware/authentication.js` | Function |

### Appointment Service
| Symbol | File | Type |
|--------|------|------|
| `AppointmentService` | `src/services/appointmentService.js` | Class |
| `BookingService` | `src/services/bookingService.js` | Class |
| `AppointmentSlotConfigService` | `src/services/appointmentSlotConfigService.js` | Class |
| `SlotGenerationService` | `src/services/slotGenerationService.js` | Class |
| `AppointmentNotifier` | `src/services/appointmentNotifier.js` | Class |
| `authenticate` | `src/middleware/authentication.js` | Function |
| `authorize` | `src/middleware/authentication.js` | Function |
| `hasPermission` | `src/middleware/authentication.js` | Function |

### Notification Service
| Symbol | File | Type |
|--------|------|------|
| `NotificationService` | `src/services/notificationService.js` | Class |
| `TemplateService` | `src/services/templateService.js` | Class |
| `EventDefinitionService` | `src/services/eventDefinitionService.js` | Class |
| `SettingsService` | `src/services/settingsService.js` | Class |
| `SuppressionService` | `src/services/suppressionService.js` | Class |
| `EmailProvider` | `src/providers/emailProvider.js` | Class |
| `SmsProvider` | `src/providers/smsProvider.js` | Class |
| `WhatsAppProvider` | `src/providers/whatsappProvider.js` | Class |
| `startWorker` | `src/worker/dispatcher.js` | Function |
| `authenticate` | `src/middleware/authentication.js` | Function |
| `authorize` | `src/middleware/authentication.js` | Function |
| `hasPermission` | `src/middleware/authentication.js` | Function |

---

## Test Coverage Summary

| Service | Test Files | Focus Areas |
|---------|------------|-------------|
| identity-admin | 8 | Auth, authorization, audit, credentials, rate limit, models, resource-actions, service-accounts |
| organization-admin | 0 | — |
| patient-admin | 0 | — |
| provider-admin | 0 | — |
| appointment-admin | 0 | — |
| notification-admin | 0 | — |

> **Gap**: Only identity-admin-service has tests. Other services rely on manual/API testing.

---

## Configuration Files (Per Service)

| Service | Required `.env` Variables |
|---------|---------------------------|
| identity-admin | `DB_*`, `PORT`, `ADMIN_JWT_ISSUER`, `ADMIN_JWT_AUDIENCE`, `ADMIN_JWKS_URL` or `ADMIN_JWT_SECRET`, `ORGANIZATION_SERVICE_INTERNAL_TOKEN`, `PATIENT_SERVICE_INTERNAL_TOKEN`, `PROVIDER_SERVICE_INTERNAL_TOKEN`, `APPOINTMENT_SERVICE_INTERNAL_TOKEN`, `NOTIFICATION_SERVICE_INTERNAL_TOKEN`, `ALLOWED_ORIGINS`, `MFA_CHALLENGE_SECRET`, `ACCESS_TOKEN_TTL_SECONDS`, `REFRESH_TOKEN_TTL_DAYS` |
| organization-admin | `DB_*`, `PORT`, `ADMIN_JWT_*`, `ORGANIZATION_SERVICE_INTERNAL_TOKEN`, `ALLOWED_ORIGINS` |
| patient-admin | `DB_*`, `PORT`, `ADMIN_JWT_*`, `PATIENT_SERVICE_INTERNAL_TOKEN`, `ALLOWED_ORIGINS` |
| provider-admin | `DB_*`, `PORT`, `ADMIN_JWT_*`, `PROVIDER_SERVICE_INTERNAL_TOKEN`, `ALLOWED_ORIGINS` |
| appointment-admin | `DB_*`, `PORT`, `ADMIN_JWT_*`, `APPOINTMENT_SERVICE_INTERNAL_TOKEN`, `ALLOWED_ORIGINS` |
| notification-admin | `DB_*`, `PORT`, `ADMIN_JWT_*`, `NOTIFICATION_SERVICE_INTERNAL_TOKEN`, `ALLOWED_ORIGINS`, `NOTIFICATION_WORKER`, SMTP/Twilio creds |

---

## Database SQL Dumps (Reference)

| File | Database | Size | Tables |
|------|----------|------|--------|
| `database/dump-identity-20261016.sql` | `identity` (legacy) | 10 KB | 6 |
| `database/dump-sp3digital_identity-20261016.sql` | `sp3digital_identity` | 441 KB | ~30 |
| `database/dump-sp3digital_organization-20261016.sql` | `sp3digital_organization` | 1.4 MB | ~15 (+ geography) |
| `database/dump-sp3digital_patients-20261016.sql` | `sp3digital_patients` | 13 KB | 6 |
| `database/dump-sp3digital_providers-20261016.sql` | `sp3digital_providers` | 18 KB | 9 |
| `database/dump-sp3digital_appointments-20261016.sql` | `sp3digital_appointments` | 91 KB | 10 |
| `database/dump-sp3digital_notifications-20261016.sql` | `sp3digital_notifications` | 11 KB | 6 |
# SP3Digital — Database Schema Reference

## Overview
- **6 databases** (one per backend service), MySQL 8.0+
- **No cross-database foreign keys** — referential integrity enforced at application layer
- **Shared tenant identifier**: `tenant_uuid` (CHAR(36)) on most tables
- **Soft delete pattern**: `status VARCHAR(30)` with values `ACTIVE`/`DELETED`/`INACTIVE`/`PENDING`/`REVOKED`/`EXPIRED`/`MERGED`/`ANONYMIZED`/`SUPPRESSED`
- **Dual-key pattern**: `*_id` (BIGINT UNSIGNED PK, auto-increment) + `*_uuid` (CHAR(36) UK, external reference)
- **Audit columns**: `created_by`, `created_on`, `modified_by`, `modified_on` (manual, no Sequelize timestamps)
- **No migrations** — `sequelize.sync({ alter: false })` on startup; production requires manual `ALTER TABLE`

---

## 1. sp3digital_identity (Identity Admin Service)

### Core Tables

| Table | Purpose | Key Columns | Key Indexes/Constraints |
|-------|---------|-------------|-------------------------|
| `identity_tenants` | Tenant registry | `tenant_uuid` (PK), `tenant_code` (UK), `tenant_name`, `status` | `uq_identity_tenants_code` |
| `users` | User accounts | `user_id` (PK), `user_uuid` (UK), `tenant_uuid`, `username`, `email`, `password_hash`, `status`, `first_name`, `last_name`, `user_type`, `email_verified`, `phone_verified`, `failed_login_count`, `account_locked_until`, `last_login_on` | `uk_users_uuid`, `uq_users_tenant_username` (implied), `idx_users_tenant_status` |
| `roles` | Role definitions | `role_id` (PK), `role_uuid` (UK), `role_code` (UK), `role_name`, `role_type` (TENANT/SYSTEM), `status` | `uk_roles_uuid`, `uk_roles_code`, `idx_roles_type` |
| `permissions` | Permission catalog | `permission_id` (PK), `permission_uuid` (UK), `permission_code` (UK), `permission_name`, `resource`, `action`, `resource_category`, `action_category`, `status` | `uk_permissions_uuid`, `uk_permissions_code`, `idx_permissions_resource_action`, FK→`resource_master`, FK→`action_master` |
| `user_roles` | User↔Role assignment | `user_role_id` (PK), `user_id` (FK), `role_id` (FK), `status` | `uk_user_role` (`user_id`,`role_id`), FK→`users`, FK→`roles` |
| `role_permissions` | Role↔Permission mapping | `role_permission_id` (PK), `role_id` (FK), `permission_id` (FK), `status` | `uk_role_permission` (`role_id`,`permission_id`), FK→`roles`, FK→`permissions` |
| `resource_master` | Resource catalog | `resource_id` (PK), `resource_code` (UK), `resource_name`, `description`, `status` | `uk_resource_code` |
| `action_master` | Action catalog | `action_id` (PK), `action_code` (UK), `action_name`, `action_category`, `description`, `status` | `uk_action_code` |

### Auth & Token Tables

| Table | Purpose | Key Columns | Key Indexes/Constraints |
|-------|---------|-------------|-------------------------|
| `user_sessions` | Login sessions | `session_id` (PK), `session_uuid` (UK), `user_id` (FK), `session_token_hash`, `ip_address`, `user_agent`, `login_method`, `expires_on`, `last_activity_on`, `status` | `uk_session_uuid`, `idx_sessions_user_status`, FK→`users` |
| `refresh_tokens` | Refresh token store | `refresh_token_id` (PK), `token_uuid` (UK), `token_hash` (UK), `user_id` (FK), `oauth_client_id` (FK), `session_id` (FK), `parent_token_id` (FK self), `issued_on`, `expires_on`, `used_on`, `revoked_on`, `status` | `uk_refresh_token_uuid`, `uk_refresh_token_hash`, `idx_refresh_user_status`, `idx_refresh_expiry`, FK→`users`, FK→`user_sessions`, FK→`oauth_clients`, self-FK |
| `access_tokens` | Access token store | `access_token_id` (PK), `token_uuid` (UK), `token_hash` (UK), `user_id` (FK), `oauth_client_id` (FK), `api_client_id` (FK), `service_account_id` (FK), `session_id` (FK), `token_type`, `scopes` (JSON), `issued_on`, `expires_on`, `revoked_on`, `status` | `uk_access_token_uuid`, `uk_access_token_hash`, `idx_access_user_status`, `idx_access_expiry`, FK→`users`, FK→`user_sessions`, FK→`oauth_clients`, FK→`api_clients`, FK→`service_accounts` |

### MFA & Device Tables

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `mfa_methods` | Enrolled MFA factors | `mfa_method_id` (PK), `tenant_uuid`, `user_id` (FK), `method_type` (TOTP/SMS/EMAIL), `method_identifier`, `secret_encrypted`, `secret_key_version`, `failed_verification_count`, `locked_until`, `is_primary`, `is_verified`, `verified_on`, `last_used_on`, `status` |
| `user_devices` | Trusted devices | `device_id` (PK), `device_uuid` (UK), `user_id` (FK), `device_name`, `device_fingerprint`, `trusted_on`, `last_seen_on`, `status` |
| `user_credentials` | Password history | `credential_id` (PK), `user_id` (FK), `credential_type` (PASSWORD), `credential_hash`, `key_version`, `rotated_on`, `expires_on`, `force_rotation_on_next_login`, `status` |
| `user_credential_history` | Password history (audit) | `history_id` (PK), `user_id` (FK), `credential_hash`, `key_version`, `changed_on`, `changed_by`, `reason` |

### Federation & OAuth Tables

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `identity_providers` | OIDC/SAML IdP config | `identity_provider_id` (PK), `tenant_uuid`, `provider_uuid` (UK), `provider_code` (UK), `provider_name`, `provider_type` (OIDC/SAML), `issuer_url`, `authorization_url`, `token_url`, `jwks_url`, `client_id`, `client_secret_encrypted`, `secret_key_version`, `scopes` (JSON), `configuration` (JSON), `status` |
| `external_identities` | User↔IdP linkage | `external_identity_id` (PK), `user_id` (FK), `identity_provider_id` (FK), `external_subject`, `external_username`, `external_email`, `claims` (JSON), `last_login_on`, `status` |
| `oauth_clients` | OAuth 2.0 clients | `oauth_client_id` (PK), `tenant_uuid`, `client_uuid` (UK), `client_id` (UK), `client_name`, `client_type` (CONFIDENTIAL/PUBLIC), `client_secret_hash`, `redirect_uris` (JSON), `allowed_scopes` (JSON), `grant_types` (JSON), `token_endpoint_auth_method`, `organization_id` (FK), `status`, `expires_on` |
| `api_clients` | API client credentials | `api_client_id` (PK), `tenant_uuid`, `client_uuid` (UK), `client_code` (UK), `client_name`, `client_type`, `client_secret_hash`, `secret_rotated_on`, `deactivated_on`, `revoked_on`, `organization_id` (FK), `allowed_ips` (JSON), `allowed_origins` (JSON), `status`, `expires_on` |
| `api_client_scopes` | API client↔Permission | `api_client_scope_id` (PK), `api_client_id` (FK), `permission_id` (FK), `status` |
| `service_accounts` | Machine-to-machine accounts | `service_account_id` (PK), `tenant_uuid`, `account_uuid` (UK), `account_code` (UK), `account_name`, `description`, `status`, `expires_on` |

### Security & Audit Tables

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `security_policies` | Per-tenant password/session/MFA policy | `policy_id` (PK), `tenant_uuid` (UK), `min_password_length`, `require_uppercase`, `require_lowercase`, `require_digit`, `require_special`, `password_expiry_days`, `password_history_count`, `max_failed_logins`, `lockout_duration_minutes`, `max_session_duration_minutes`, `mfa_required`, `allowed_mfa_methods` (JSON), `status` |
| `auth_configurations` | Per-tenant login method config | `config_id` (PK), `tenant_uuid` (UK), `allow_password_login`, `allow_social_login`, `allow_mfa_enforcement`, `max_session_duration_minutes`, `status` |
| `audit_logs` | Write-operation audit trail | `audit_id` (PK), `tenant_uuid`, `actor_user_id` (FK→users), `action`, `target_resource`, `changes` (JSON), `ip_address`, `created_on` |

### User-Org/Facility Linkage (Cross-Service References)

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `organizations_users` | User↔Organization membership | `organization_user_id` (PK), `organization_id` (FK→org-service), `user_id` (FK), `membership_type`, `is_primary`, `effective_from`, `effective_to`, `status` |
| `facilities_users` | User↔Facility membership | `facility_user_id` (PK), `facility_id` (FK→org-service), `user_id` (FK), `membership_type`, `is_primary`, `effective_from`, `effective_to`, `status` |

> **Note**: `organization_id` and `facility_id` reference tables in `sp3digital_organization` database — **no DB-level FK**.

---

## 2. sp3digital_organization (Organization Admin Service)

### Master Data Tables

| Table | Purpose | Key Columns | Key Constraints |
|-------|---------|-------------|-----------------|
| `organizations` | Organization hierarchy | `organization_id` (PK), `organization_uuid` (UK), `tenant_uuid`, `organization_code` (UK), `organization_name`, `organization_type`, `parent_organization_id` (FK self), `address_line1/2`, `city_id`, `district_id`, `sub_district_id`, `state_id`, `country_id`, `postal_code_id`, `latitude`, `longitude`, `status` | `uk_organizations_uuid`, `uk_organizations_code`, `uq_org_tenant_code` (`tenant_uuid`,`organization_code`), FK→`organizations` (self), **no FK to geography** |
| `facilities` | Facilities (clinics/hospitals) | `facility_id` (PK), `facility_uuid` (UK), `tenant_uuid`, `organization_id` (FK), `facility_code` (UK), `facility_name`, `facility_type`, `address_line1/2`, `city_id`, `district_id`, `sub_district_id`, `state_id`, `country_id`, `postal_code_id`, `latitude`, `longitude`, `phone_number`, `email`, `status` | `uk_facilities_uuid`, `uk_facilities_code`, `uq_fac_tenant_code` (`tenant_uuid`,`facility_code`), FK→`organizations`, **no FK to geography** |
| `departments` | Departments within facilities | `department_id` (PK), `department_uuid` (UK), `tenant_uuid`, `facility_id` (FK), `department_code`, `department_name`, `department_type`, `status` | `uk_departments_uuid`, `uk_facility_department_code` (`facility_id`,`department_code`), FK→`facilities` |
| `services` | Clinical services | `service_id` (PK), `service_uuid` (UK), `service_code` (UK), `service_name`, `description`, `status` | `uk_services_uuid`, `uk_services_code` |
| `service_categories` | Service groupings | `service_category_id` (PK), `service_category_uuid` (UK), `service_category_code` (UK), `service_category_name`, `description`, `status` | `uk_service_categories_uuid`, `uk_service_categories_code` |
| `facility_services` | Facility↔Service↔Category↔Department | `facility_service_id` (PK), `tenant_uuid`, `facility_id` (FK), `facility_service_uuid` (UK), `department_id` (FK), `service_category_id` (FK), `service_id` (FK), `status` | `uk_facility_service_code` (`facility_id`,`service_id`), FK→`facilities`, FK→`departments`, FK→`service_categories`, FK→`services` |

### Geography Tables (India Postal Hierarchy)

| Table | Purpose | Key Columns | Row Count (Seed) |
|-------|---------|-------------|------------------|
| `countries` | Countries | `country_id` (PK), `name`, `iso_alpha2` (UK), `iso_alpha3` (UK), `status` | ~250 |
| `states` | States/Provinces | `state_id` (PK), `country_id` (FK), `name`, `iso_code`, `status` | ~36 (India) |
| `districts` | Districts | `district_id` (PK), `state_id` (FK), `name`, `status` | ~700 |
| `sub_districts` | Sub-districts (Taluks/Tehsils) | `sub_district_id` (PK), `district_id` (FK), `name`, `status` | ~6,000 |
| `cities` | Cities/Towns/Post offices | `city_id` (PK), `sub_district_id` (FK), `name`, `raw_office_name`, `office_type` (H.O./S.O./B.O.), `status` | ~154,000 |
| `postal_codes` | PIN codes | `postal_code_id` (PK), `city_id` (FK), `postal_code` (UK), `office_name`, `office_type`, `delivery_status`, `latitude`, `longitude`, `status` | ~19,000 |

> **Geography FKs**: Enforced at DB level within geography hierarchy. **NOT enforced** from `facilities`/`organizations` to geography (intentional — validated in service layer only).

---

## 3. sp3digital_patients (Patient Admin Service)

| Table | Purpose | Key Columns | Key Constraints |
|-------|---------|-------------|-----------------|
| `patients` | Patient registry | `patient_id` (PK), `patient_uuid` (UK), `tenant_uuid`, `facility_id` (FK→org), `patient_number` (per-facility MRN), `first_name`, `middle_name`, `last_name`, `gender`, `date_of_birth`, `blood_group`, `phone`, `email`, `address_line1/2`, `city_id`, `district_id`, `state_id`, `country_id`, `postal_code_id`, `status` (ACTIVE/MERGED/ANONYMIZED/DELETED), `created_by`, `created_on` | `uk_patients_uuid`, `uq_patient_facility_number` (`tenant_uuid`,`facility_id`,`patient_number`), **no FK to org/geography tables** |
| `patient_facility_number` | Patient↔Facility MRN | `patient_facility_number_id` (PK), `patient_id` (FK), `facility_id` (FK→org), `patient_number`, `is_primary`, `status` | `uk_patient_facility_number` (`patient_id`,`facility_id`), FK→`patients` |
| `patient_merge` | Merge audit | `merge_id` (PK), `source_patient_id` (FK), `target_patient_id` (FK), `merged_by` (FK→users), `merged_on`, `reason`, `status` | FK→`patients` (source), FK→`patients` (target) |
| `patient_access_log` | Access audit | `access_log_id` (PK), `patient_id` (FK), `actor_user_id` (FK→identity.users), `action` (VIEW/CREATE/UPDATE/SEARCH/EXPORT/MERGE), `ip_address`, `user_agent`, `accessed_on` | FK→`patients`, `idx_access_patient_actor` |
| `patient_number_sequence` | MRN sequence per facility | `sequence_id` (PK), `tenant_uuid`, `facility_id` (FK→org), `current_number`, `prefix`, `format`, `status` | `uk_tenant_facility_sequence` (`tenant_uuid`,`facility_id`) |
| `tenant_patient_setting` | Per-tenant patient config | `setting_id` (PK), `tenant_uuid` (UK), `duplicate_threshold`, `auto_merge_enabled`, `retention_years`, `anonymization_enabled`, `status` | `uk_tenant_patient_setting` |

---

## 4. sp3digital_providers (Provider Admin Service)

| Table | Purpose | Key Columns | Key Constraints |
|-------|---------|-------------|-----------------|
| `providers` | Provider (doctor) profiles | `provider_id` (PK), `provider_uuid` (UK), `tenant_uuid`, `first_name`, `middle_name`, `last_name`, `gender`, `date_of_birth`, `phone`, `email`, `photo_url`, `national_id`, `status`, `created_by`, `created_on` | `uk_providers_uuid`, `uq_provider_tenant_email` (`tenant_uuid`,`email`) |
| `provider_affiliations` | Provider↔Facility/Org/Dept | `affiliation_id` (PK), `provider_id` (FK), `tenant_uuid`, `organization_id` (FK→org), `facility_id` (FK→org), `department_id` (FK→org), `affiliation_type` (PHYSICAL/REMOTE/OTHER), `availability_schedule` (JSON), `start_date`, `end_date`, `is_primary`, `status` | `uk_provider_affiliation` (`provider_id`,`facility_id`,`department_id`), FK→`providers` |
| `provider_affiliation_services` | Services offered at affiliation | `affiliation_service_id` (PK), `affiliation_id` (FK), `service_id` (FK→org), `status` | FK→`provider_affiliations` |
| `provider_qualifications` | Degrees/diplomas | `qualification_id` (PK), `provider_id` (FK), `qualification_name`, `institution`, `year`, `verification_status` (PENDING/VERIFIED/REJECTED/EXPIRED), `verified_by`, `verified_on`, `status` | FK→`providers` |
| `provider_registrations` | Medical council registrations | `registration_id` (PK), `provider_id` (FK), `registration_number`, `council_name`, `state`, `valid_from`, `valid_to`, `verification_status`, `status` | FK→`providers` |
| `provider_specialties` | Medical specialties | `specialty_id` (PK), `provider_id` (FK), `specialty_name`, `is_primary`, `status` | FK→`providers` |
| `provider_languages` | Spoken languages | `language_id` (PK), `provider_id` (FK), `language`, `proficiency`, `status` | FK→`providers` |
| `provider_experience` | Work history | `experience_id` (PK), `provider_id` (FK), `organization_name`, `designation`, `from_date`, `to_date`, `is_current`, `status` | FK→`providers` |
| `provider_documents` | Certificates, licenses | `document_id` (PK), `provider_id` (FK), `document_type`, `document_name`, `file_url`, `encrypted_content` (BLOB), `key_version`, `status` | FK→`providers` |

---

## 5. sp3digital_appointments (Appointment Admin Service)

| Table | Purpose | Key Columns | Key Constraints |
|-------|---------|-------------|-----------------|
| `appointment_slot_configs` | Recurring slot rules | `slot_config_id` (PK), `slot_config_uuid` (UK), `tenant_uuid`, `facility_id` (FK→org), `department_id` (FK→org), `provider_id` (FK→provider), `resource_id` (FK→facility_resources), `recurrence_rule` (RRULE string), `start_time`, `end_time`, `slot_duration_minutes`, `max_appointments_per_slot`, `timezone`, `effective_from`, `effective_to`, `status` (PENDING_APPROVAL/ACTIVE/REJECTED/EXPIRED/SUPERSEDED), `created_by`, `approved_by`, `approved_on` | `uk_slot_config_uuid`, FK→`facilities`, FK→`departments`, **no FK to provider/org resources** |
| `appointment_slots` | Generated time slots | `slot_id` (PK), `slot_uuid` (UK), `tenant_uuid`, `slot_config_id` (FK), `facility_id`, `department_id`, `provider_id`, `resource_id`, `slot_date`, `start_time`, `end_time`, `max_appointments`, `status` (AVAILABLE/BOOKED/BLOCKED/CANCELLED), `current_bookings` | `uk_slot_uuid`, `uq_slot_config_date_time` (`slot_config_id`,`slot_date`,`start_time`), FK→`appointment_slot_configs` |
| `appointments` | Booked appointments | `appointment_id` (PK), `appointment_uuid` (UK), `tenant_uuid`, `patient_id` (FK→patient), `slot_id` (FK), `facility_id`, `department_id`, `provider_id`, `booking_reason`, `notes`, `status` (BOOKED/CHECKED_IN/COMPLETED/CANCELLED/NO_SHOW/RESCHEDULED), `token_number`, `booked_by`, `booked_on`, `checked_in_on`, `completed_on` | `uk_appointment_uuid`, `uq_slot_appointment` (`slot_id`,`appointment_id` unique), FK→`appointment_slots`, **no FK to patient/provider/org** |
| `appointment_status_history` | Status audit | `history_id` (PK), `appointment_id` (FK), `from_status`, `to_status`, `changed_by`, `changed_on`, `reason` | FK→`appointments` |
| `facility_closures` | Closure dates | `closure_id` (PK), `closure_uuid` (UK), `tenant_uuid`, `facility_id` (FK→org), `closure_date`, `reason`, `is_recurring`, `recurrence_pattern`, `status` | `uk_closure_uuid`, FK→`facilities` |
| `facility_resources` | Bookable resources (rooms, equipment) | `resource_id` (PK), `resource_uuid` (UK), `tenant_uuid`, `facility_id` (FK→org), `resource_code` (UK), `resource_name`, `resource_type`, `status` | `uk_resource_uuid`, `uq_facility_resource_code` (`facility_id`,`resource_code`), FK→`facilities` |
| `appointment_reminders` | Reminder schedule | `reminder_id` (PK), `tenant_uuid`, `appointment_id` (FK), `reminder_type` (EMAIL/SMS/WHATSAPP), `lead_time_minutes`, `status` (PENDING/SENT/FAILED), `sent_on` | FK→`appointments` |
| `appointment_cancellation_reasons` | Cancellation reason catalog | `reason_id` (PK), `tenant_uuid`, `reason_code` (UK), `reason_text`, `is_active` | `uk_cancellation_reason_code` |
| `appointment_token_counters` | Daily token sequence | `counter_id` (PK), `tenant_uuid`, `facility_id`, `slot_date`, `current_token` | `uk_tenant_facility_date_counter` (`tenant_uuid`,`facility_id`,`slot_date`) |
| `tenant_notification_settings` | Reminder lead times | `setting_id` (PK), `tenant_uuid` (UK), `reminder_lead_times` (JSON, e.g., `[1440, 120]`), `channels` (JSON), `status` | `uk_tenant_notification_setting` |

---

## 6. sp3digital_notifications (Notification Admin Service)

| Table | Purpose | Key Columns | Key Constraints |
|-------|---------|-------------|-----------------|
| `notification_templates` | Channel-specific templates | `template_id` (PK), `template_uuid` (UK), `tenant_uuid`, `event_code`, `channel` (EMAIL/SMS/WHATSAPP), `locale`, `subject_template`, `body_template`, `variables_schema` (JSON), `is_default`, `status` | `uk_template_uuid`, `uq_template_tenant_event_channel_locale` (`tenant_uuid`,`event_code`,`channel`,`locale`) |
| `notification_event_definitions` | Event catalog | `event_definition_id` (PK), `event_definition_uuid` (UK), `tenant_uuid`, `event_code` (UK), `event_name`, `description`, `default_channels` (JSON), `default_priority`, `status` | `uk_event_definition_uuid`, `uq_event_definition_tenant_code` (`tenant_uuid`,`event_code`) |
| `notifications` | Outbox queue | `notification_id` (PK), `notification_uuid` (UK), `tenant_uuid`, `event_code`, `recipient_user_id`, `recipient_patient_id`, `recipient_provider_id`, `channel`, `priority` (HIGH/NORMAL/LOW), `subject`, `body`, `variables` (JSON), `status` (QUEUED/SENDING/SENT/FAILED/SUPPRESSED), `scheduled_for`, `sent_on`, `attempt_count`, `max_retries`, `last_error` | `uk_notification_uuid`, `idx_notifications_tenant_status_scheduled` (`tenant_uuid`,`status`,`scheduled_for`) |
| `notification_attempts` | Delivery attempt log | `attempt_id` (PK), `notification_id` (FK), `provider` (EMAIL/SMS/WHATSAPP), `provider_message_id`, `provider_response` (JSON), `status` (SENT/FAILED), `attempted_on`, `error_message` | `idx_attempts_notification`, FK→`notifications` |
| `notification_suppressions` | Opt-out / block list | `suppression_id` (PK), `tenant_uuid`, `contact_value` (email/phone), `channel`, `reason`, `expires_on`, `status` | `uq_suppression_tenant_contact_channel` (`tenant_uuid`,`contact_value`,`channel`) |
| `tenant_channel_settings` | Per-tenant channel config | `setting_id` (PK), `tenant_uuid`, `channel` (EMAIL/SMS/WHATSAPP), `provider_config` (JSON, e.g., SMTP creds, Twilio SID), `is_enabled`, `default_from`, `status` | `uq_tenant_channel_setting` (`tenant_uuid`,`channel`) |

---

## Cross-Service Reference Integrity (Application-Level)

| Referencing Table | Referenced Table | Service | Validation Point |
|-------------------|------------------|---------|------------------|
| `organizations_users.organization_id` | `organizations.organization_id` | identity → organization | `organizationDirectory.js` on create/update |
| `facilities_users.facility_id` | `facilities.facility_id` | identity → organization | `organizationDirectory.js` |
| `patients.facility_id` | `facilities.facility_id` | patient → organization | `organizationDirectory.js` in `patientService.createPatient` |
| `providers.organization_id` | `organizations.organization_id` | provider → organization | `organizationDirectory.js` |
| `providers.facility_id` | `facilities.facility_id` | provider → organization | `organizationDirectory.js` |
| `appointment_slot_configs.facility_id` | `facilities.facility_id` | appointment → organization | `organizationDirectory.js` |
| `appointment_slot_configs.department_id` | `departments.department_id` | appointment → organization | `organizationDirectory.js` |
| `appointment_slot_configs.provider_id` | `providers.provider_id` | appointment → provider | `providerDirectory.js` |
| `appointment_slot_configs.resource_id` | `facility_resources.resource_id` | appointment → appointment | Local FK (enforced) |
| `appointments.patient_id` | `patients.patient_id` | appointment → patient | `patientDirectory.js` |
| `appointments.slot_id` | `appointment_slots.slot_id` | appointment → appointment | Local FK (enforced) |

> **Critical**: `appointment_slots.resource_id` → `facility_resources.resource_id` is the **only cross-table FK enforced at DB level** in appointment service (see `app.js:111` error handler for `SequelizeForeignKeyConstraintError`).

---

## Common Query Patterns & Indexing Strategy

| Pattern | Index |
|---------|-------|
| Tenant-scoped list | `idx_<table>_tenant_status` (`tenant_uuid`, `status`) |
| User-scoped list | `idx_<table>_user_status` (`user_id`, `status`) |
| Facility-scoped list | `idx_<table>_facility_status` (`facility_id`, `status`) |
| Status + date range | `idx_<table>_status_date` (`status`, `created_on` / `scheduled_for`) |
| UUID lookup | `uk_<table>_uuid` (unique) |
| Code lookup | `uk_<table>_code` / `uq_<table>_tenant_code` |

---

## Data Retention & Archival

| Service | Retention Policy | Mechanism |
|---------|------------------|-----------|
| identity | Audit logs: indefinite (compliance) | Manual partition/archive |
| patient | Configurable per tenant (`retention_years`) | `npm run anonymize` job scrubs PII, sets `status=ANONYMIZED` |
| notification | `NOTIFICATION_RETENTION_DAYS` (default 365) | `npm run purge` deletes old notifications + attempts |
| appointment | No automated retention | Manual cleanup |
| organization/provider | No automated retention | Manual cleanup |

---

## Schema Evolution Notes

- **No migration tooling** — model changes require coordinated deploy + manual SQL
- **Backward compatibility**: New columns must be nullable or have defaults
- **Enum-like `status` columns**: Use `VARCHAR(30)` not ENUM (allows new values without ALTER)
- **JSON columns**: Used for flexible config (`scopes`, `configuration`, `variables`, `availability_schedule`, `redirect_uris`) — no schema validation at DB level
- **Encrypted columns**: `client_secret_encrypted`, `secret_encrypted`, `encrypted_content` use `secretProtector.js` (AES-256-GCM with key versioning)

---

## Verified vs Inferred

| Finding | Status | Source |
|---------|--------|--------|
| 6 databases, one per service | **Verified** | SQL dumps in `database/` |
| No cross-database FKs | **Verified** | Dump inspection; only intra-db FKs present |
| Soft delete via `status` column | **Verified** | All model files + dump DDL |
| Dual-key pattern (id + uuid) | **Verified** | All model files |
| Geography FKs not enforced from facilities | **Verified** | `facility.model.js` comment + dump shows no FK |
| Appointment resource_id FK enforced | **Verified** | `app.js:111` error handler + dump shows FK |
| Patient MRN sequence per facility | **Verified** | `patient_number_sequence` table + `patientService.js` |
| Refresh token rotation + reuse detection | **Verified** | `authService.js:367-398` |
| MFA TOTP with backup codes | **Verified** | `mfaService.js` + `mfa_methods` table |
| Maker-checker slot config | **Verified** | `appointmentSlotConfigService.js` + `slot_config.status` enum |
| Notification outbox + worker polling | **Verified** | `notificationService.js` + `worker/dispatcher.js` |
| SUPERADMIN bypasses tenant scoping | **Verified** | `authentication.js:249` + `userService.js`/`tenantService.js` |
| Organization service authorize() not mounted | **Verified** | `app.js:92` (only `authenticate`) + `authentication.js:180-183` comment |
| Identity service dual-gate authZ | **Verified** | `authentication.js:256-297` (token + DB permissions) |
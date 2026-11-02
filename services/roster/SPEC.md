# Mosaic Roster — SPEC

- **Owns:** organizations, offices, departments, employees, app access with ADMIN flags, compensation, self-service profiles (including bank details), announcements, payslip history, avatars.
- **Publishes:** `roster.organizations`, `roster.offices`, `roster.departments`, `roster.employees`, `roster.employee-compensation`.
- **Consumes:** `paystream.payslips`.
- **API:** REST under `/api/v1/roster`.
- **UI:** `apps/roster-web` (`/roster`).

Every employee has implicit Roster access to their own profile. HR staff hold Roster access levels for everything else.

## Permissions (`RosterPermission`)

| Permission | Scope kind | Allows |
|---|---|---|
| `VIEW_EMPLOYEE` | ORG_OFFICE | view employee records (HR view) in scope |
| `MANAGE_EMPLOYEE` | ORG_OFFICE | create, edit, transfer, deactivate, reactivate, resend invite |
| `MANAGE_APP_ACCESS` | ORG_OFFICE | grant or remove app access and ADMIN flags |
| `VIEW_COMPENSATION` | ORG_OFFICE | see salary, allowances and unmasked bank details |
| `MANAGE_COMPENSATION` | ORG_OFFICE | edit salary and allowances |
| `MANAGE_ORG_STRUCTURE` | NONE | create and edit organizations, offices and departments |
| `MANAGE_ANNOUNCEMENTS` | NONE | post and archive announcements |
| `MANAGE_ACCESS_LEVELS` | NONE | manage Roster access levels |
| `VIEW_AUDIT` | NONE | audit log |

**Directory search** is open to every active employee and returns non-sensitive fields only: name, title, department, office, email, avatar.

## Data (`roster` DB)
- `organizations`: `id`, `code` unique, `name`, `country`, `status`
- `offices`: `id`, `org_id`, `name`, `city`, `status`
- `departments`: `code` PK (2–5 uppercase letters, immutable), `name`, `head_user_id`, `status`
- `employees`:
  - identity: `user_id` PK, `employee_number` unique, `first_name`, `last_name`, `email` unique
  - placement: `job_title`, `org_id`, `office_id`, `department_code`, `manager_user_id`
  - lifecycle: `hire_date`, `status`, `avatar_file_id`
  - `created_at`, `updated_at`
- `employee_number_history`: `user_id`, `employee_number`, `department_code`, `valid_from`, `valid_to`
- `department_sequences`: `department_code`, `next_value`. Numbers are never reused.
- `pending_transfers`: `user_id`, `to_department_code`, `to_office_id` (optional), `effective_month` (always the 1st), `created_by`
- `employee_app_access`: `user_id`, `app`, `admin`
- `employee_profiles`: `user_id`, `personal_email`, `phone`, `address`, `emergency_contact_name`, `emergency_contact_phone`, `bank_name`, `bank_branch`, `bank_account_name`, `bank_account_number` (encrypted)
- `compensations`: `user_id`, `currency`, `basic_salary` (encrypted), `effective_from`, `updated_by`
- `compensation_allowances`: `user_id`, `name`, `amount`, `taxable`
- `announcements`: `id`, `title`, `body` (markdown), `audience` (`ALL` or an org ID), `pinned`, `published_at`, `archived_at`, `author_user_id`
- `payslips`: `id`, `user_id`, `department_code`, `year`, `month`, `payload` jsonb, `published_at`, unique (`user_id`, `department_code`, `year`, `month`)
- `payslip_resets`: `department_code`, `year`, `month`, `reset_at`
- shared tables: access-control tables (ARCHITECTURE §4.3), `audit_log`, `outbox`, `processed_events`, `notifications`, `file_attachments`

## Business rules

### Company email
- Built as `lower(ascii(first)) + "." + lower(ascii(last)) + "@mosaichub.com"`. Diacritics are stripped, and anything other than letters and hyphens is removed.
- On a clash with any employee, active or inactive, append `2`, `3`, … to the last name part.
- The email is generated once and doesn't change when the name changes, unless HR presses "Regenerate email".

### Employee numbers
- Format: `MH-<DEPT>-<4-digit sequence per department>`, e.g. `MH-FIN-0042`.

### Department transfers
- HR schedules a transfer with an effective month, which is always the 1st of a month.
- A scheduler runs at 00:05 Africa/Accra on the 1st of each month and applies due transfers:
  1. new employee number from the new department's sequence
  2. close the old number in the history table
  3. update the department (and office, if one was given)
  4. publish `UPDATED`
- The effective month must be the next month or later. That way, each month's payroll for a department always sees a stable list of members.

### Deactivation
- Deactivation sets the status to `INACTIVE` and publishes `DEACTIVATED`. Employees are never hard-deleted.
- A department head or office can't be deactivated while it still has active members.

### App access
- `ROSTER` is always present and can't be removed.
- Changing access or ADMIN flags publishes `UPDATED`.

### Compensation topic
Any change to compensation **or** bank details publishes on `roster.employee-compensation`.

### Payslip consumer
- **PUBLISHED:** upsert by (`user_id`, `department_code`, `year`, `month`), unless `published_at` is older than a matching `payslip_resets.reset_at`. Then notify the employee: "Your March 2026 payslip is available."
- **RESET:** delete the payslips matching department + year + month, and record the reset.

### Payslip PDF
`GET /me/payslips/{id}/pdf` renders the stored payload with OpenPDF in MosaicHub styling. It isn't stored.

## Endpoints (all under `/api/v1/roster`)
- **Organization structure:** `GET|POST /organizations`, `PATCH /organizations/{id}`, and the same pattern for `/offices` and `/departments`.
- **Employees:**
  - `GET /employees` with filters: q, orgId, officeId, departmentCode, status, app. Scoped.
  - `POST /employees`, `GET|PATCH /employees/{userId}`
  - `POST /employees/{userId}/deactivate`, `/reactivate`, `/resend-invite`, `/regenerate-email`
  - `POST /employees/{userId}/transfers`, `DELETE /employees/{userId}/transfers/pending`
  - `PUT /employees/{userId}/app-access` with `[{app, admin}]`
  - `GET|PUT /employees/{userId}/compensation`
  - `GET /employees/{userId}/avatar`
- **Directory:** `GET /directory?q=` (any active employee).
- **Me:**
  - `GET|PATCH /me/profile` (including bank details)
  - `POST /me/avatar/upload-url`, `POST /me/avatar/confirm`
  - `GET /me/payslips?year=`, `GET /me/payslips/{id}/pdf`
  - `GET /me/summary`, `GET /me/notifications`, `POST /me/notifications/{id}/read`, `POST /me/notifications/read-all`
- **Announcements:** `GET /announcements` (visible to the user), `POST /announcements`, `PATCH /announcements/{id}`
- **Admin:** `GET|POST|PATCH|DELETE /access-levels`, `PUT /access-levels/{id}/members`, `GET /audit-log`

`/me/summary` for an HR user returns headcount and pending invites; for any employee it returns the latest payslip month.

## Screens (`apps/roster-web`)
1. **Employees list:** search, filter chips, status badges, "Add employee" button.
2. **Add/Edit employee:** a wizard with three steps, Basics → Placement → App access (ADMIN toggles per app). The generated email and number are previewed live.
3. **Employee detail** with tabs:
   - Overview
   - Employment (including transfer history and the scheduled transfer)
   - App access
   - Compensation (permission-gated)
   - Audit
4. **Organization structure:** organizations, then offices (as cards), then departments with code, head and headcount.
5. **Access levels:** list, editor (permission checkboxes grouped by area, org/office scope pickers), members.
6. **My profile:** personal details, bank details (masked after save), avatar upload, change password.
7. **My payslips:** grouped by year, each with a download button.
8. **Announcements:** manage screen for HR.
9. **Audit log:** filterable table.

## Tests
- Email generation, including clashes and diacritics.
- Number sequencing and transfer scheduling (with a controllable clock).
- Scope filtering on lists and single-record reads.
- Outbox events validated against the contracts.
- Payslip PUBLISHED/RESET ordering, including a late PUBLISHED arriving after a RESET.
- Avatar upload confirm, including content-type verification.

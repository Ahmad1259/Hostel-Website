# Database Schema (Logical Design)

Status: **Phase 1 draft — awaiting owner approval.**

Engine-agnostic logical model. Physical DDL (Oracle or PostgreSQL) is written in
Phase 4 once the database service is approved. Table names are lowercase; add a
`hostel_id` column wherever "tenant scope" is noted — this is how data isolation
is enforced at the data layer.

Conventions for every table: `id` (surrogate primary key), `created_at`,
`updated_at`, `created_by`, and for audited tables `row_version` for optimistic
locking.

## 1. Identity, tenancy and access

**users** — one login per person.
`id, email (unique, nullable until set), phone (unique), password_hash (nullable
if SSO), status (active|locked|deactivated), mfa_enabled, last_login_at`

**roles** — reference table: `platform_admin, hostel_owner, hostel_manager,
warden, student`.

**user_roles** — grants a role, optionally scoped to a hostel.
`user_id, role_id, hostel_id (null for platform_admin), granted_by, granted_at,
revoked_at`
Unique: `(user_id, role_id, hostel_id)` where not revoked. A student has exactly
one active `student` role with `hostel_id = null`.

**hostels** — a tenant.
`id, name, slug (unique), address, city, region, latitude, longitude,
description, policies_text, contact_phone, status (pending|active|suspended),
verification_status, is_location_verified`

**hostel_staff_assignments** — explicit staff-to-hostel link; the source of
tenant isolation for staff.
`id, hostel_id, user_id, role_id, assigned_by, assigned_at, revoked_at`
Unique: `(hostel_id, user_id, role_id)` where `revoked_at is null`.
Rule: every staff query must join through this table or filter by `hostel_id`
from an active assignment. No cross-hostel reads or writes.

**sessions / refresh_tokens** — `id, user_id, token_hash, issued_at, expires_at,
revoked_at, ip_hash, user_agent_hash`. Only hashes stored.

**verification_challenges** — email/mobile OTP and password reset.
`id, user_id, purpose (email_verify|phone_verify|password_reset|mfa),
challenge_hash, expires_at, attempts, consumed_at`. Short TTL, single use.

**audit_events** — append-only. `id, actor_user_id, action, entity_type,
entity_id, hostel_id, before_json, after_json, ip_hash, occurred_at`.
Never updated or deleted by application code.

## 2. Student profile and admissions

**students** — profile tied to a student user.
`id, user_id (unique), full_name, cnic_masked, cnic_hash, date_of_birth,
institution_name, institution_verified, program, enrollment_year, guardian_name,
guardian_phone, guardian_relationship, address, profile_status`
Full CNIC is stored encrypted at rest (application-level) and never placed in
URLs, logs, exports, or frontend state; only masked value plus hash for lookup.

**admission_applications**
`id, student_id, hostel_id, room_type_requested, preferred_start_date,
message, status (draft|submitted|under_review|documents_required|approved|
rejected|withdrawn), submitted_at, decided_at, decided_by, decision_reason`
A submitted application does **not** reserve a bed and creates no bed hold.

**application_documents** — private uploads.
`id, application_id, hostel_id, doc_type (cnic|enrollment|guardian_consent|photo|
other), storage_key (opaque, non-public), file_hash, size_bytes, mime_type,
uploaded_by, uploaded_at, status (pending|accepted|rejected), reviewed_by,
reviewed_at, review_note`
Files live in object storage with private ACLs; access only via short-lived
signed URLs issued after a server-side permission check.

## 3. Inventory: rooms, beds, facilities

**room_types** — per hostel catalog definition.
`id, hostel_id, name (single|double|triple|quad|custom), capacity,
rent_per_person_monthly, description`

**rooms** — physical rooms.
`id, hostel_id, room_type_id, room_number, floor, wing/block,
washroom_type (attached|shared), rent_per_person_monthly, capacity,
status (active|under_maintenance|retired), readiness_status (ready|not_ready)`

**beds** — the bookable unit; availability is computed from bed status plus
active assignments, never stored as a free-floating counter.
`id, hostel_id, room_id, label, status (available|occupied|maintenance|reserved),
current_assignment_id (nullable, set only in the same transaction that creates
the active assignment)`
Unique: `(room_id, label)`.

**availability_snapshot** — cache for public search only.
`hostel_id, room_type_id, available_bed_count, as_of_at`. Regenerated
transactionally whenever beds/assignments change; the public site must show
`as_of_at` ("Availability last updated"). Search never invents numbers.

**facilities** — reference list (wifi, laundry, meals, generator, study_room,
security_cctv, water cooler, geyser, housekeeping…).

**hostel_facilities / room_facilities** — many-to-many with `verified`
(true|false) and `verified_by/verified_at`. Unverified facilities must display
as "unverified" and cannot appear in the "verified facilities" filter.

**room_media** — photos, video, genuine 360.
`id, hostel_id, room_id (nullable), bed_id (nullable), media_type
(photo|video|panorama_360|floor_plan), storage_key, alt_text, orientation,
captured_at, consent_confirmed, published, reviewed_by`
`media_type` is authoritative: a `photo` must never be rendered with 360
controls. `consent_confirmed` is required before publishing any image showing
identifiable residents or belongings; in practice, resident-identifying media
stays unpublished.

## 4. Occupancy lifecycle

**bed_assignments** — the fact that a person holds a bed for a period.
`id, student_id, hostel_id, room_id, bed_id, application_id (nullable),
status (reserved|active|ended|cancelled), start_date, end_date,
rent_per_person_monthly (rate locked at assignment), deposit_amount,
assigned_by, assigned_at, ended_by, ended_at, end_reason`
**Critical constraints:**
- Partial unique index on `bed_id` where `status in ('reserved','active')` —
  prevents two live assignments to one bed.
- Partial unique index on `student_id` where `status = 'active'` — a student
  holds at most one active bed.
- `(start_date, end_date)` must not overlap for the same `bed_id` (checked in
  the assigning transaction).
- Rate is copied into the row so a later rent change never rewrites history.

**stay_events** — check-in / check-out record.
`id, assignment_id, student_id, hostel_id, event_type
(check_in|check_out|early_checkout|absence_long), event_date, recorded_by,
room_readiness_after, notes`
Check-in transitions a `reserved` assignment to `active` and the bed to
`occupied`. Check-out ends the assignment, sets the bed `available` only when
`room_readiness_after = ready`, otherwise `maintenance`/`not_ready`.

**room_transfer_requests**
`id, student_id, hostel_id, from_assignment_id, to_room_id, to_bed_id,
requested_date, effective_date, reason, status (pending|approved|rejected|
completed|cancelled), approved_by, approved_at, completed_at,
new_assignment_id (set on completion)`
Completion is one transaction: end old assignment (preserving its rate and
history), create new assignment with the destination bed's current rate, flip
both beds' statuses, and write a `bed_transfer` audit event. Future challans use
the new rate only from `effective_date` forward; past challans are untouched.

## 5. Money: challans, payments, receipts, deposits

**fee_policies** — per hostel.
`id, hostel_id, billing_cycle (monthly), due_day_of_month, grace_period_days,
late_fee_type (none|fixed|percent), late_fee_value, allow_partial_payments,
min_partial_percent, currency (PKR default)`

**challans** — an amount owed; immutable once issued.
`id, challan_number (unique, human-readable, e.g. H012-2026-10-000345),
hostel_id, student_id, assignment_id, period_start, period_end,
head (rent|deposit|utilities|late_fee|other), amount, amount_paid, balance,
status (draft|issued|partially_paid|paid|void|adjusted), due_date,
issued_at, issued_by, void_reason, voided_by, voided_at`
Uniqueness: `(assignment_id, period_start, head)` — one rent challan per
assignment per period, which makes monthly generation idempotent and prevents
duplicate bills.
A challan is never deleted. Corrections create a linked adjustment (`void` +
new challan, or a negative `other` challan referencing the original) so the
trail stays readable and auditable.

**payment_submissions** — anything a person claims as payment.
`id, challan_id, hostel_id, student_id, method (bank_transfer|jazzcash|
easypaisa|cash|other), amount, claimed_at, reference_text (masked),
proof_media_id (private), currency, status (pending|verified|rejected),
verified_by, verified_at, rejection_reason`
Status starts `pending`. Uploading proof never changes a challan balance. Only a
`verified` submission, recorded by an authorized staff member, moves money.

**receipts** — official confirmation.
`id, receipt_number (unique), challan_id, payment_submission_id, hostel_id,
student_id, amount, received_at, issued_by`
Unique: `(payment_submission_id)` — one receipt per verified payment; and the
server re-checks that `challan.amount_paid + this amount <= challan.amount`, so
double-verification cannot over-collect.

**deposits** — refundable security money.
`id, student_id, hostel_id, assignment_id, amount, challan_id, status
(pending|held|partially_refunded|refunded|forfeited), forfeit_reason,
refunded_at, refunded_by`
Deposits are tracked separately from rent and are never counted as income.

**balance view** — `challan.amount - challan.amount_paid`, where `amount_paid`
is updated only inside the transaction that verifies a payment or issues an
adjustment. Outstanding balance is always derived from challan rows, never
from a cached field.

## 6. Notifications

**notification_preferences** — consent-first.
`id, student_id, channel (whatsapp|sms|email|in_app), phone_e164, email,
consent_granted_at, consent_source, stop_at, is_active`
No delivery without an active consent record.

**notification_events** — what should be sent.
`id, recipient_user_id, hostel_id, type (fee_due|fee_overdue|payment_verified|
payment_rejected|assignment_created|transfer_update|receipt_issued),
entity_type, entity_id, scheduled_for, dedupe_key (unique), status
(pending|sending|sent|failed|skipped), attempts, last_error, sent_at`
`dedupe_key` (e.g. `fee_due:challan:123:2026-10`) makes scheduler runs safe to
retry and prevents duplicate reminders. Before sending a fee reminder the job
re-reads the balance; if it is zero the event is marked `skipped`.

**notification_deliveries** — per attempt.
`id, event_id, channel, provider, provider_message_id, attempt_no, requested_at,
result (success|failure|unknown), error_code, cost_units`
The provider is an adapter interface. If no WhatsApp provider is available in
budget, delivery falls back to in-app + email and the record keeps an honest
`channel` value — the app never claims a message was sent.

## 7. Cross-cutting integrity rules

1. Bed availability, assignment overlap, challan uniqueness, and receipt
   double-issuance are enforced by **database constraints**, and additionally
   re-checked in server transactions. Application checks alone are not accepted.
2. Multi-step state changes (transfer, check-out, payment verification) run in a
   single transaction; on any failure nothing is half-written.
3. Money amounts use fixed-point decimal types, never floating point.
4. Every write path stamps `created_by`/`updated_by` and emits an audit event.
5. Data retention: student records persist through renewals; a new assignment
   does not erase the previous one — history stays queryable.

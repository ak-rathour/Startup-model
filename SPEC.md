# SPEC.md — LibraryHive MVP Specification

**Status: Authoritative.** This document must be detailed enough for a senior developer to
implement the MVP without guessing. It must agree with `FEATURES.md` and `AGENTS.md`. If an
implementation detail here conflicts with `FEATURES.md`, the conflict must be raised, not
silently resolved.

## 1. Product Overview

LibraryHive is a production-oriented platform connecting library owners and students, covering
discovery, seat booking, membership, fees, payments, attendance, and complaints.

## 2. Problem Statement

Small/medium paid libraries run attendance, fees, and renewals manually, causing revenue loss
(unbilled vacant seats, cash mismatches) and owner time cost. Students have no way to discover,
evaluate, or book a seat at a library remotely.

## 3. Target Users

**Owner:** operator of a paid library/reading room. **Student:** a person who uses or is
evaluating a paid library seat.

## 4. MVP Objective

A production-quality, deployable system usable by at least 3 real design-partner libraries,
covering every feature in `FEATURES.md` Section 3.

## 5. Business Hypothesis

Owners will pay for a system that replaces their register for attendance/fees/renewals *and*
helps them fill seats via discovery/booking; students will use online discovery + booking over
walking in cold.

## 6. User Roles

`OWNER` and `STUDENT` — mutually exclusive, set at registration, immutable thereafter for MVP.

## 7. Technology Decisions (evaluated per Section 14 of the source prompt)

| Concern | Decision | Rationale |
|---|---|---|
| Frontend | Next.js, TypeScript, Tailwind | SSR needed for public discovery/SEO; already validated for this project |
| Backend | Django + Django REST Framework | Relational domain (memberships, bookings, payments) fits Django ORM; admin panel aids design-partner support during pilot |
| Database | PostgreSQL (Supabase-managed) | Relational integrity for booking/payment constraints; managed reduces ops burden for a small team |
| File Storage | Supabase Storage | Bundled with the existing Postgres provider; S3-compatible; avoids a second storage vendor |
| Payment Gateway | Razorpay (single gateway only) | Standard for Indian small-business payments; webhook-based verification supported |
| Notifications | **WhatsApp + SMS**, both via a single BSP (MSG91) | MSG91 provides WhatsApp Business API and DLT-compliant SMS from one account/integration, avoiding two separate vendor integrations. WhatsApp is the primary channel (richer, cheaper per-message for utility category); SMS is sent as a guaranteed-delivery companion, since WhatsApp delivery can silently fail (uninstalled app, number not on WhatsApp) while SMS reaches any phone. An in-app `Notification` row is still created for every send, for audit/dedup — it is now a record, not the delivery mechanism itself. |
| Backend Hosting | Render | Django-compatible, simple deploy, acceptable free/low tier for pilot scale |
| Frontend Hosting | Vercel | Native Next.js fit |
| Auth | Django auth + DRF SimpleJWT, one coherent system | Avoids combining multiple auth providers per the explicit instruction |

## 8. Functional Requirements

### 8.1 Authentication (Feature A)
- Registration: `username`, `email`, `phone` (unique), `password` (min 8 chars), `role`
- Login: `username` + `password` → JWT access + refresh tokens
- Logout: client discards tokens; refresh token may be blacklisted server-side
- Passwords hashed via Django's default hasher (PBKDF2); never logged or stored in plaintext
- Every protected endpoint checks role AND object ownership, not role alone

### 8.2 Owner Library Profile (Feature B)
- One `Library` per `Owner` (enforced by a one-to-one/unique FK)
- Required before publish: `name`, `address`, `city`, `opening_time`, `closing_time`,
  `seat_capacity`, at least one photo with `is_cover=True`
- `is_published` boolean gates visibility in discovery/public search

### 8.3 Photo Management (Feature C)
- Accepted formats: JPEG, PNG, WebP. Max size: 5 MB per file.
- Stored in Supabase Storage under `libraries/{library_id}/{photo_id}.{ext}`; DB stores the
  reference, not the binary
- Exactly one `LibraryPhoto` per library may have `is_cover=True` (enforced at write time —
  setting a new cover unsets the previous one in the same transaction)
- Delete: removes both the storage object and the DB row; deleting the cover photo requires
  the request to specify a new cover, or the system auto-promotes the most recently uploaded
  remaining photo
- Public access: read is public (unauthenticated); write/delete requires the owning owner

### 8.4 Library Discovery (Feature D)
- `GET /api/libraries/` supports `?city=` filtering at minimum; returns only `is_published=True`
  libraries
- Each result includes: cover photo URL, name, city, seat availability summary
  (available/total)

### 8.5 Library Public Profile (Feature E)
- Full published fields + photo gallery + live seat grid + demo request affordance
- No login required to view

### 8.6 Student Management (Feature F)
- Owner's student list = all `Membership` rows for their `Library`, joined to `User`
- Owner cannot query/list students at another owner's library (object-level permission)

### 8.7 Offline + Online Registration (Feature G)
- `User.account_status`: `PENDING_CLAIM` | `ACTIVE`
- Owner-created student: `User` row created with `account_status=PENDING_CLAIM`,
  unusable password (`set_unusable_password()`), required `phone`
- Claim flow: a registration request with a phone matching a `PENDING_CLAIM` user does not
  create a new `User` — it prompts a claim (verify phone via OTP or a simple verification step,
  then set a real password), flipping `account_status` to `ACTIVE`
- If an `ACTIVE` student (already registered, possibly at a different library) is added by an
  owner at a new library, the system links the existing `User` to a new `Membership` — it does
  not create a duplicate `User`
- Uniqueness is enforced on `User.phone` at the database level in all cases

### 8.8 Membership Management (Feature H)
- Status is **derived**, not manually set: `ACTIVE` while `expiry_date` is more than 7 days
  away; `EXPIRING_SOON` within 7 days; `EXPIRED` once past `expiry_date`
- Renewal: creates/updates the `FeeRecord` for the new period and sets a new `expiry_date`;
  status recalculates automatically from the new date
- Unique constraint: one active `Membership` per (`student`, `library`) pair

### 8.9 Automatic Membership Expiry Notification (Feature I)
- **Trigger:** a scheduled daily job (Django management command run via the host's scheduler,
  e.g. Render Cron) scans all `ACTIVE`/`EXPIRING_SOON` memberships
- **Timing:** fires once when `expiry_date - today == 7 days`, and once again on
  `expiry_date == today` if still unrenewed
- **Recipient:** the library's owner
- **Content:** `"{student_name}'s membership expires in {N} days."` (or "expires today")
- **Delivery:** WhatsApp message (via MSG91's WhatsApp Business API, using a pre-approved
  utility template) **and** SMS (via MSG91's DLT-registered SMS API) sent to the owner's
  phone, in parallel, for the same trigger. An in-app `Notification` row is created regardless
  of delivery outcome, so the owner dashboard always shows the notification even if both
  channels fail
- **Duplicate prevention:** `Notification.dedup_key = f"{membership_id}:{trigger_type}:
  {expiry_date}"` is unique — the daily job upserts on this key before attempting delivery, so
  re-running the job never sends a second WhatsApp/SMS pair for the same (membership, trigger,
  expiry cycle), even if the job crashes mid-run and is retried. Renewing the membership
  changes `expiry_date`, which naturally produces a fresh, distinct dedup key for the next
  cycle
- **Delivery failure handling:** WhatsApp and SMS are attempted independently — if WhatsApp
  fails (e.g., number not on WhatsApp) but SMS succeeds, the notification still counts as
  delivered; `Notification` stores per-channel delivery status (`whatsapp_status`,
  `sms_status`) for visibility, but does not retry a channel that already succeeded on the
  other
- **Timezone:** all scheduling in the library's configured timezone (default: `Asia/Kolkata`)
- **Setup lead time — flag before build starts:** both channels require upfront registration
  that is *not* same-day: the WhatsApp utility template must be submitted and approved by Meta
  (via MSG91), and the SMS sender ID/template must be DLT-registered with Indian telecom
  regulators (via MSG91). Both should be started on Day 1 of implementation, not when this
  feature is reached, or they become a late-stage blocker.

### 8.10 Fee Management (Feature J)
- `FeeRecord`: `amount`, `due_date`, `status` (`pending`/`paid`/`overdue`), `payment_date`
- Every `FeeRecord` belongs to exactly one `Membership`
- Status transitions are timestamped for audit

### 8.11 Online Payment (Feature K)
- Single gateway: **Razorpay**
- Flow: `POST /api/payments/initiate/` creates a Razorpay order server-side → frontend opens
  Razorpay checkout with that order ID → on completion, Razorpay calls the backend webhook
  (`POST /api/payments/webhook/`) → backend verifies the payment signature server-side using
  Razorpay's SDK → only on verified success is the associated `FeeRecord` marked paid and the
  `Membership` activated/renewed
- The frontend's own "success" callback is **never** trusted to activate anything — it only
  shows a "processing" state until the webhook-driven backend state changes
- Idempotency: webhook handler is idempotent on `gateway_payment_id` — a duplicate webhook
  delivery for the same payment is a no-op on the second call
- Failure/cancellation: `Payment.status` set to `failed`/`cancelled`; membership state
  unchanged; student sees a clear retry option
- Refunds: not built; if Razorpay's basic dashboard-initiated refund is used manually by the
  team, it does not need in-app handling for MVP — document this as a manual, out-of-band
  process

### 8.12 Attendance (Feature L)
- `POST /api/memberships/{id}/check-in/`: rejected with a clear error if an open
  (`check_out_time IS NULL`) `Attendance` row already exists for that membership
- `POST /api/memberships/{id}/check-out/`: rejected if no open row exists
- Enforced via a partial unique constraint (one open row per membership) at the database level

### 8.13 Seat Management & Visual Seat Grid (Features M, N)
- `Seat`: `library` (FK), `seat_number`, `status` (`available`/`occupied`/`reserved`/
  `disabled`) — `status` is computed at read time from current bookings/attendance, with
  `disabled` as the one owner-settable override
- Grid endpoint returns all seats for a library with current computed status for a given date
- Disabling a seat with an active booking is blocked by the API unless the request explicitly
  confirms cancellation of that booking

### 8.14 Seat Selection & Hourly/Shift Booking (Features O, P)
- Shifts (owner-configurable per library, default three): Morning (6–12), Afternoon (12–18),
  Evening (18–23)
- `POST /api/bookings/`: `seat_id`, `membership_id`, `date`, `shift` (or `start_time`/
  `end_time` for hourly libraries)
- **Conflict prevention:** a partial unique index on (`seat_id`, `date`, `shift`) where
  `status='booked'` — this is enforced by the database, not just application logic, so two
  simultaneous requests cannot both succeed
- A rejected booking returns a clear conflict error, never a silent failure

### 8.15 Demo / Visit Request (Feature Q)
- `DemoRequest.status`: `PENDING` → `ACCEPTED` | `REJECTED`; student may also transition
  `PENDING → CANCELLED`
- No other transitions permitted

### 8.16 Complaint / Issue Management (Feature R)
- Categories: Seat problem, AC/Fan, Cleanliness, Internet, Other
- `Complaint.status`: `OPEN → IN_PROGRESS → RESOLVED` (forward-only)
- Only a student with a (current or past) `Membership` at that library may file a complaint
  there

### 8.17 Owner Dashboard (Feature S)
- Real aggregate queries only: total students, active/expiring/expired memberships, today's
  attendance, available/occupied seats, pending demo requests, pending complaints, recent
  payments
- Zero-data states render an explicit empty state

### 8.18 Student Dashboard (Feature T)
- Current membership + expiry, active bookings, attendance history, demo request status,
  payment history, complaint status — scoped strictly to the authenticated student

## 9. Non-Functional Requirements

- All list endpoints paginated (default page size 20)
- API response time target: under 500ms for non-payment endpoints at pilot scale (3–5
  libraries)
- All timestamps stored in UTC, rendered in the library's local timezone on the frontend
- Mobile-responsive UI for all public and student-facing pages at minimum

## 10. Complete User Journeys

See `FEATURES.md` Section 16 for the eight critical journeys. Each must have at least one
corresponding integration test (Section 45).

## 11–12. Authentication & Authorization

- JWT access token (short-lived) + refresh token (longer-lived), via
  `djangorestframework-simplejwt`
- Every view enforces: (a) authentication where required, (b) role check, (c) object-level
  ownership check (custom DRF permission classes) — role check alone is never sufficient

## 13. Role-Based Access Control

See `FEATURES.md` Section 14 (User-Role Matrix) — implemented via DRF permission classes per
endpoint, not via frontend route guarding alone.

## 14. Database Architecture

Single PostgreSQL database, one schema. No sharding, no read replicas for MVP. Relational
design chosen because nearly every entity has a strict foreign-key relationship (Membership →
Attendance/FeeRecord/Booking; Library → Seat/Photo) where referential integrity matters
(e.g., preventing an orphaned booking).

## 15–16. Database Entities & Relationships

| Entity | Key Fields | Relationships / Constraints |
|---|---|---|
| `User` | id (UUID, PK), username, email, phone (unique), password (hashed), role, account_status (PENDING_CLAIM/ACTIVE), exam_category (nullable), target_year (nullable), created_at | Base identity for both roles |
| `Library` | id (PK), owner (FK → User, unique), name, description, address, city, contact_phone, opening_time, closing_time, facilities, seat_capacity, is_published, created_at, updated_at | One owner → one library |
| `LibraryPhoto` | id (PK), library (FK), image_url, is_cover, uploaded_at | Exactly one `is_cover=True` per library |
| `Seat` | id (PK), library (FK), seat_number, is_disabled, created_at | Unique (`library`, `seat_number`) |
| `Membership` | id (PK), student (FK → User), library (FK), monthly_fee, start_date, expiry_date, registered_by (online/offline), created_at | Unique active (`student`, `library`); status is derived, not stored |
| `Booking` | id (PK), seat (FK), membership (FK), date, shift (or start_time/end_time), status (booked/cancelled), created_at | Partial unique (`seat`, `date`, `shift`) where `status='booked'` |
| `FeeRecord` | id (PK), membership (FK), amount, due_date, status, payment (FK → Payment, nullable), payment_date (nullable) | One membership → many fee records |
| `Payment` | id (PK), membership (FK), amount, gateway_order_id, gateway_payment_id (unique), status (created/success/failed/cancelled), verified, created_at | Idempotent on `gateway_payment_id` |
| `Attendance` | id (PK), membership (FK), check_in_time, check_out_time (nullable) | Partial unique: one open row per membership |
| `DemoRequest` | id (PK), library (FK), student_name, student_phone, preferred_datetime, status, created_at | Not linked to `User` initially |
| `Complaint` | id (PK), membership (FK), category, description, image_url (nullable), status, created_at, updated_at | Forward-only status |
| `Notification` | id (PK), owner (FK → User), membership (FK, nullable), type, message, dedup_key (unique), whatsapp_status (sent/failed/not_attempted), sms_status (sent/failed/not_attempted), sent_at | Unique `dedup_key` prevents duplicate sends; per-channel status tracks WhatsApp/SMS delivery independently |

## 17. Constraints Summary

- `User.phone` — unique (prevents duplicate identities across all registration paths)
- `Library.owner` — unique (one library per owner)
- `LibraryPhoto` — at most one `is_cover=True` per library
- `Seat(library, seat_number)` — unique
- `Membership(student, library)` — unique while active
- `Booking(seat, date, shift)` — unique while `status='booked'` (the seat-conflict guard)
- `Attendance` — at most one row per membership with `check_out_time IS NULL`
- `Payment.gateway_payment_id` — unique (webhook idempotency)
- `Notification.dedup_key` — unique (notification idempotency)

## 18–21. API Architecture, Endpoints, Request/Response, Validation

All endpoints under `/api/`. JSON request/response. Standard REST verbs. Every endpoint listed
below must exist and match this contract; no endpoints for future/out-of-scope features.

| Method & Path | Auth | Purpose |
|---|---|---|
| POST /api/auth/register/ | None | Register (handles claim flow if phone is PENDING_CLAIM) |
| POST /api/auth/login/ | None | Login → JWT pair |
| GET /api/libraries/ | Public | Discovery/search (`?city=`) |
| POST /api/libraries/ | OWNER | Create own library |
| GET /api/libraries/{id}/ | Public | Public profile |
| PATCH /api/libraries/{id}/ | OWNER (own) | Edit profile |
| POST /api/libraries/{id}/photos/ | OWNER (own) | Upload photo |
| DELETE /api/photos/{id}/ | OWNER (own library) | Delete photo |
| PATCH /api/photos/{id}/set-cover/ | OWNER (own library) | Set as cover |
| POST /api/libraries/{id}/students/ | OWNER (own) | Add walk-in/existing student, create Membership |
| GET /api/libraries/{id}/students/ | OWNER (own) | List students |
| GET /api/memberships/{id}/ | OWNER (own lib) / STUDENT (own) | Membership detail |
| POST /api/memberships/{id}/renew/ | OWNER (own lib) | Renew membership |
| POST /api/memberships/{id}/fees/ | OWNER (own lib) | Create FeeRecord |
| PATCH /api/fees/{id}/ | OWNER (own lib) | Update fee status |
| GET /api/memberships/{id}/fees/ | OWNER (own lib) / STUDENT (own) | List fees |
| POST /api/payments/initiate/ | STUDENT | Create Razorpay order |
| POST /api/payments/webhook/ | Gateway (signature-verified) | Verify + activate/renew |
| GET /api/memberships/{id}/payments/ | OWNER (own lib) / STUDENT (own) | Payment history |
| POST /api/memberships/{id}/check-in/ | STUDENT (own) | Check in |
| POST /api/memberships/{id}/check-out/ | STUDENT (own) | Check out |
| GET /api/libraries/{id}/attendance/today/ | OWNER (own) | Today's attendance |
| GET /api/libraries/{id}/seats/ | Public | Seat grid (with status for a given date) |
| POST /api/libraries/{id}/seats/ | OWNER (own) | Add seat |
| PATCH /api/seats/{id}/ | OWNER (own lib) | Enable/disable |
| POST /api/bookings/ | STUDENT | Create booking (conflict-checked) |
| GET /api/memberships/{id}/bookings/ | OWNER (own lib) / STUDENT (own) | List bookings |
| POST /api/libraries/{id}/demo-requests/ | Public | Submit request |
| GET /api/libraries/{id}/demo-requests/ | OWNER (own) | List requests |
| PATCH /api/demo-requests/{id}/ | OWNER (own) | Accept/reject |
| POST /api/memberships/{id}/complaints/ | STUDENT (own) | File complaint |
| GET /api/libraries/{id}/complaints/ | OWNER (own) | List complaints |
| PATCH /api/complaints/{id}/ | OWNER (own) | Update status |
| GET /api/notifications/ | OWNER | List own notifications |
| GET /api/owner/dashboard/ | OWNER | Summary |
| GET /api/student/dashboard/ | STUDENT | Own summary |

**Validation:** all input validated server-side (DRF serializers); reject malformed dates,
negative amounts, invalid enum values with `400` and a field-level error body.

## 22. Error Handling

Standard error shape: `{"detail": "human-readable message", "field_errors": {...}}`. `401` for
missing/invalid auth, `403` for authenticated-but-unauthorized, `404` for not-found-or-not-
yours (do not leak existence of other owners' resources), `409` for conflicts (double booking,
duplicate payment webhook already processed).

## 23–40. Per-Feature Detailed Behavior

Covered in Section 8 (Functional Requirements) above, organized by feature.

## 41. Security

- Passwords hashed (Django default), never logged
- JWT secrets and all credentials in environment variables, never committed
- CORS restricted to the deployed frontend origin(s) only
- Every endpoint enforces object-level ownership (Section 12/13)
- File uploads validated by content type and size before storage (Section 8.3)
- Payment webhook signature verified before trusting any payload (Section 8.11)
- No sensitive data (passwords, tokens, payment details) in logs

## 42. File Storage

Supabase Storage, bucket per environment. Public read for library photos; write/delete
restricted to the owning owner via signed backend-issued requests (frontend never gets direct
write credentials to storage).

## 43. Deployment

Frontend → Vercel (auto-deploy from `main`). Backend → Render (auto-deploy from `main`,
migrations run as a release step). Database → Supabase Postgres. Scheduled notification job →
Render Cron Job calling the Django management command daily.

## 44. Environment Variables

| Variable | Where | Purpose |
|---|---|---|
| `DATABASE_URL` | Backend | Postgres connection |
| `DJANGO_SECRET_KEY` | Backend | Django signing key |
| `DJANGO_DEBUG` | Backend | `False` in production |
| `ALLOWED_HOSTS` | Backend | Production host allowlist |
| `CORS_ALLOWED_ORIGINS` | Backend | Frontend origin(s) |
| `RAZORPAY_KEY_ID` / `RAZORPAY_KEY_SECRET` | Backend | Payment gateway |
| `SUPABASE_STORAGE_URL` / `SUPABASE_STORAGE_KEY` | Backend | File storage |
| `MSG91_AUTH_KEY` | Backend | MSG91 account authentication |
| `MSG91_WHATSAPP_TEMPLATE_ID` | Backend | Approved WhatsApp utility template for expiry reminders |
| `MSG91_SMS_TEMPLATE_ID` | Backend | DLT-registered SMS template for expiry reminders |
| `NEXT_PUBLIC_API_URL` | Frontend | Backend API base URL |

## 45. Testing

Critical-path tests only (per `AGENTS.md` philosophy): auth + role/ownership permission
boundaries, duplicate check-in prevention, duplicate-phone/claim-flow correctness, booking
conflict prevention, payment webhook idempotency, notification dedup. One integration test per
journey in Section 10.

## 46. Production Readiness

See `FEATURES.md` Section 17 (Definition of Done) — identical checklist, authoritative there.

## 47. Monitoring / Logging

Basic error logging (Django's default logging to stdout, captured by the hosting platform) is
sufficient for MVP. No dedicated APM/monitoring service — would be over-engineering at this
scale.

## 48. MVP Acceptance Criteria

See `FEATURES.md` Sections 10–11 (per-feature acceptance criteria and edge cases) —
authoritative there.

## 49. Explicitly Excluded Features

See `FEATURES.md` Sections 12–13.

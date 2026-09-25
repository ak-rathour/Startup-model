# FEATURES.md — LibraryHive MVP

**Status: Authoritative.** This document, together with `SPEC.md` and `AGENTS.md`, is the
strict source of truth for the LibraryHive MVP. No feature listed here as P0 may be removed,
downgraded, or silently deferred during implementation.

## 1. Product Overview

LibraryHive is a two-sided platform connecting **Library Owners** (small/medium paid
libraries and reading rooms) with **Students**. Owners manage their library's operations
digitally instead of on paper; students discover, evaluate, book, and use libraries through
the platform.

## 2. MVP Objective

Deliver a **production-quality** MVP — real database, real auth, real payments, real file
storage, real notifications — usable by at least 3 real library owners as design partners.
Small and complete, not large and half-finished.

## 3. Complete MVP Feature List

| ID | Feature | Priority | Primary User(s) | Problem Solved | User Value |
|---|---|---|---|---|---|
| A | Authentication | P0 | Owner, Student | No secure, role-aware entry point | Safe, correct access to the right side of the product |
| B | Owner Library Profile | P0 | Owner | No digital record of the library's core info | Single place to manage what students see |
| C | Library Photo Upload | P0 | Owner, Student | Students can't evaluate a library remotely | Visual trust before visiting |
| D | Library Discovery | P0 | Student | No way to find/compare libraries before visiting | Students find a fitting library without cold-visiting |
| E | Library Public Profile | P0 | Student | No detailed remote view of a library | Informed decision before booking/visiting |
| F | Student Management | P0 | Owner | No centralized student records | Replaces the register for day-to-day student data |
| G | Offline + Online Registration | P0 | Owner, Student | Most students still join in person | One system, no duplicate identities |
| H | Membership Management | P0 | Owner, Student | Manual renewal tracking causes missed/unbilled lapses | Clear lifecycle, fewer missed renewals |
| I | Automatic Membership Expiry Notification | P0 | Owner | Owners forget expiring memberships | Proactive renewal opportunity, less revenue leakage |
| J | Fee Management | P0 | Owner, Student | Register/cash mismatches at month-end | Accurate, reconciled fee records |
| K | Online Payment | P0 | Student | No way to pay without visiting in person | Convenient renewal/join, verified revenue |
| L | Attendance | P0 | Student, Owner | Manual attendance checking is slow and error-prone | Real-time visibility, no register flipping |
| M | Seat Management | P0 | Owner | No structured view of seat inventory/status | Owner knows exact capacity and state |
| N | Visual Seat Grid | P0 | Owner, Student | A bare seat count hides which seats are actually free | Concrete, at-a-glance availability |
| O | Seat Selection | P0 | Student | No way to pick a specific seat remotely | Student books the seat they actually want |
| P | Hourly / Shift Booking | P0 | Student, Owner | Many libraries bill by hour/shift, not just monthly | Matches how owners actually operate |
| Q | Demo / Visit Request | P0 | Student, Owner | No low-friction way to try before joining | Bridges discovery to membership |
| R | Complaint / Issue Management | P0 | Student, Owner | Problems go unreported or get lost verbally | Tracked, resolvable issue history |
| S | Owner Dashboard | P0 | Owner | No single operational overview | One screen for what needs attention today |
| T | Student Dashboard | P0 | Student | No single view of own status | Membership, fees, bookings in one place |

All 20 features are **P0**. None are deferred.

## 4. Feature Priority

Every feature in Section 3 is required for MVP completion — there is no P1 tier in this scope.
Sequencing for implementation is governed by the dependency matrix (Section 9/15), not by
priority, since priority is uniform.

## 5–7. Owner/User, Problem Solved, User Value

Covered per-feature in the Section 3 table above.

## 8. Workflows

**B — Owner Library Profile:** Owner logs in → creates library (name, description, address,
timings, facilities, seat capacity) → library exists in `unpublished` state until at least one
cover photo is set → owner publishes.

**C — Photo Upload:** Owner uploads image(s) → each stored as a `LibraryPhoto` → owner marks
one as cover → students see cover photo on discovery cards and full gallery on the profile
page → owner may delete non-cover photos, or reassign cover before deleting the current one.

**D → E — Discovery to Profile:** Student browses/searches (by city at minimum) → sees library
cards (cover photo, name, city, seat availability summary) → opens a card → full public
profile (gallery, description, timings, facilities, seat grid, demo request option).

**G — Registration:** *Offline:* Owner adds a student (name, phone, exam info optional) →
system creates a `User` with `account_status = PENDING_CLAIM` (no usable password) → student
later registers online with the same phone → system detects the pending record and converts
it to `ACTIVE` (claim flow) instead of creating a duplicate. *Online:* Student registers
directly → `ACTIVE` from creation.

**H — Membership Lifecycle:** `ACTIVE` → (within 7 days of `expiry_date`) `EXPIRING_SOON` →
(past `expiry_date`, unrenewed) `EXPIRED` → renewal payment/record → back to `ACTIVE` with a
new `expiry_date`.

**I — Expiry Notification:** Scheduled process checks memberships daily → for each membership
crossing the 7-day-before-expiry threshold or the expiry date itself, create exactly one
`Notification` per (membership, trigger type, expiry_date) → deliver via **WhatsApp and SMS**
to the owner's phone (both attempted independently) → also shown in the owner
dashboard/notification list as a permanent record regardless of delivery outcome.

**K — Payment:** Student selects a plan (new membership or renewal) → backend creates a
payment order with the gateway → student completes payment on the gateway's page → gateway
calls the backend webhook/callback → backend independently verifies the payment with the
gateway (never trusts the frontend) → on verified success, `FeeRecord` marked paid and
`Membership` activated/renewed.

**L — Attendance:** Student checks in (blocked if an open check-in already exists for that
membership) → checks out (closes the open record) → owner sees today's attendance list.

**M/N/O/P — Seats & Booking:** Owner configures seat count → each seat is a row in `Seat` →
owner can disable/enable a seat → student views the visual grid for a chosen date → selects an
available seat + shift (Morning/Afternoon/Evening) or specific time → backend checks for a
conflicting booking on that seat/date/shift → if free, `Booking` is created; if not, request is
rejected with a clear error, not a silent failure.

**Q — Demo Request:** Student submits (name, phone, preferred date/time) from a library's
public profile → owner sees it as `PENDING` → accepts/rejects → student sees updated status.

**R — Complaint:** Student (must have an active/past membership at that library) selects a
category, adds a description, optionally attaches an image → submits → owner sees it as
`OPEN` → owner moves it to `IN_PROGRESS` → `RESOLVED` → student sees the current status at
each stage.

## 9. Dependencies

| Feature | Depends On |
|---|---|
| A | — (foundation) |
| B | A |
| C | B |
| D | B (public; no auth required to view) |
| E | B, C |
| F | A, B |
| G | A, F |
| H | G |
| I | H |
| J | H |
| K | H, J |
| L | H |
| M | B |
| N | M |
| O | N, P |
| P | M |
| Q | D |
| R | H |
| S | F, H, J, L, M, Q, R (aggregates) |
| T | H, J, L, Q, R (aggregates) |

## 10–11. Acceptance Criteria & Key Edge Cases

**A — Authentication**
- Registration requires unique phone; login returns a valid session/JWT with correct role
- A STUDENT token can never access an OWNER-only endpoint, and vice versa
- Edge case: registering with a phone already in `PENDING_CLAIM` status triggers the claim
  flow (Section 8/G), not a duplicate-user error

**B — Owner Library Profile**
- Owner can create exactly one library; a second creation attempt is rejected
- All required fields (name, address, timings, seat capacity) must be present before publish
- Edge case: unpublished library never appears in discovery or public search

**C — Photo Upload**
- Only image formats (JPEG/PNG/WebP) accepted; oversized files rejected with a clear error
- Exactly one photo can be marked cover at a time
- Edge case: deleting the current cover photo requires selecting a new cover first, or the
  system auto-promotes the next available photo

**D — Discovery**
- Search/browse returns only published libraries
- City filter returns correct, non-empty results for a valid city
- Edge case: zero results shows an explicit empty state, not a blank screen

**E — Public Profile**
- All public fields render without requiring login
- Seat grid reflects live availability at page load

**F — Student Management**
- Owner sees only students with a membership at their own library
- Edge case: a student with expired membership still appears in the list, tagged `EXPIRED`

**G — Offline + Online Registration**
- Two registrations with the same phone never create two `User` rows
- Edge case: owner adds a student whose phone is already `ACTIVE` from a different library —
  system links the existing student to a new `Membership` at this library, not a new `User`

**H — Membership Management**
- Status is always correctly derived from `expiry_date` vs. current date
- Renewal extends `expiry_date` and returns status to `ACTIVE`
- Edge case: renewing an already-`ACTIVE` (not yet expiring) membership is allowed and simply
  extends the date

**I — Expiry Notification**
- Exactly one 7-day-reminder per membership per expiry cycle (see `Notification.dedup_key` in
  `SPEC.md`)
- Edge case: if a membership is renewed before the 7-day reminder fires, the reminder for the
  *old* expiry date must not fire after renewal

**J — Fee Management**
- Every `FeeRecord` is tied to exactly one `Membership`
- Status transitions (pending → paid, pending → overdue) are auditable (timestamped)

**K — Online Payment**
- Membership is activated/renewed only after backend-verified success, never on frontend
  confirmation alone
- Edge case: duplicate webhook delivery for the same payment must not double-activate or
  create two `FeeRecord`s (idempotent by gateway payment ID)
- Edge case: failed/cancelled payment leaves the membership state unchanged, with a clear
  status shown to the student

**L — Attendance**
- At most one open (`check_out_time IS NULL`) attendance row per membership at any time
- Check-out without a prior open check-in is rejected

**M/N — Seat Management & Grid**
- Grid always reflects current seat count and per-seat status
- Disabling a seat with an active booking on it is blocked or requires explicit confirmation
  (documented in `SPEC.md`)

**O/P — Seat Selection & Booking**
- Two students can never successfully book the same seat for an overlapping date/shift
  (enforced at the database level, not just the UI)
- Edge case: two simultaneous booking requests for the same seat/slot — exactly one succeeds,
  the other receives a clear conflict error

**Q — Demo Request**
- Status only moves `PENDING → ACCEPTED/REJECTED`; no other transitions
- Edge case: student can cancel a still-pending request

**R — Complaint**
- Only a student with a membership at that library can file a complaint there
- Status only moves forward (`OPEN → IN_PROGRESS → RESOLVED`), never backward silently

**S/T — Dashboards**
- All figures come from real queries; zero-data states render an explicit empty state, never
  a fabricated placeholder number

## 12. Future / P2 Features (not in MVP)

Exam-based student community · exam/target-based offers · AI assistant/recommendations/demand
prediction · advanced analytics · advanced marketing automation · referral system · advanced
coupons · personalized offers · multi-branch enterprise management · complex subscription
billing · multiple payment gateways · advanced payment reconciliation · complex refund
management · advanced floor-plan editor · advanced scheduling engine.

## 13. Explicitly Out of Scope

Same list as Section 12. None of these may be implemented, even in a reduced form, without
explicit approval that amends this document.

## 14. User-Role Matrix

| Feature | Owner Access | Student Access |
|---|---|---|
| A. Auth | Full (own account) | Full (own account) |
| B. Library Profile | Full (own library) | Read-only (public) |
| C. Photos | Full (own library) | Read-only |
| D. Discovery | N/A | Full |
| E. Public Profile | N/A | Full (read-only) |
| F. Student Management | Full (own library's students) | None |
| G. Registration | Create (walk-in) | Self (online) |
| H. Membership | Full (own library) | Read-only (own) |
| I. Notifications | Read (own library) | None |
| J. Fees | Full (own library) | Read-only (own) |
| K. Payment | Read (own library's records) | Initiate + read (own) |
| L. Attendance | Read (own library) | Self check-in/out |
| M/N. Seats/Grid | Full (own library) | Read-only |
| O/P. Booking | Read (own library) | Create/read (own) |
| Q. Demo Requests | Full (own library) | Create/read (own) |
| R. Complaints | Full (own library) | Create/read (own) |
| S. Owner Dashboard | Full (own) | None |
| T. Student Dashboard | None | Full (own) |

## 15. Feature Dependency Matrix

See Section 9 (table form is the dependency matrix; a directed graph would show A as the
single root feeding B/F, which then fan out to every other feature).

## 16. Critical End-to-End Journeys

1. **Owner onboarding:** Register → Create Library → Upload Photos → Configure Seats →
   Publish Library
2. **Student discovery to booking:** Register → Discover Library → View Photos/Details →
   View Seat Grid → Select Seat → Select Shift/Time → Book
3. **Offline student lifecycle:** Owner Adds Offline Student → Creates Membership → Records
   Fee → Student Later Registers/Claims Account
4. **Payment activation:** Student Selects Membership → Pays Online → Backend Verifies →
   Membership Activated
5. **Expiry notification cycle:** Membership Approaches Expiry → System Detects Window →
   Owner Notified → Duplicate Notification Prevented
6. **Attendance:** Student Checks In → Attends → Checks Out → Owner Views Attendance
7. **Complaint resolution:** Student Reports Complaint → Owner Views → Owner Updates Status →
   Student Views Resolution
8. **Demo request:** Student Requests Demo → Owner Accepts/Rejects → Student Sees Status

## 17. MVP Definition of Done

- [ ] Authentication, authorization, owner isolation, and student isolation all verified
- [ ] Library creation, editing, photo upload/display, and cover photo all work
- [ ] Discovery and public profile work end-to-end
- [ ] Student management, offline registration, online registration, and account claiming all
      work with no duplicate identities
- [ ] Membership, fees, online payment (with backend verification), and renewal all work
- [ ] Expiry detection and automatic owner notification work, with duplicates prevented
- [ ] Attendance check-in/check-out works with duplicate prevention
- [ ] Seat management, visual seat grid, seat selection, and booking work with conflict
      prevention enforced at the database level
- [ ] Demo requests and complaint management both work end-to-end
- [ ] Owner and student dashboards show real data with correct empty states
- [ ] Real database, real file storage, no core feature uses fake/mock data
- [ ] Security, error, loading, and empty states all handled; responsive UI
- [ ] Production deployment works with securely configured environment variables

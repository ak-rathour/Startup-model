# AGENTS.md — LibraryHive Engineering Conventions

**Status: Authoritative.** Defines how developers and AI coding agents build LibraryHive. Must
agree with `FEATURES.md` (what) and `SPEC.md` (how it behaves). This file governs how it's
built.

## 1. Architecture

```
Browser
   ↓
Next.js (TypeScript, Tailwind) — Vercel
   ↓  (JSON over HTTPS)
Django REST Framework — Render
   ↓
Django ORM
   ↓
PostgreSQL — Supabase (managed)

+ Supabase Storage (library photos)
+ Razorpay (payment gateway, webhook-verified)
+ MSG91 (WhatsApp Business API + DLT SMS, single BSP for both channels)
+ Render Cron (daily expiry-notification job)
```

No microservices, no message queue, no separate AI/ML service. One backend service, one
frontend service, one database.

## 2. Directory Structure

```
backend/
├── config/            # settings, root urls, wsgi/asgi
├── accounts/           # User model, auth
├── libraries/           # Library, LibraryPhoto, Seat
├── memberships/         # Membership, registration/claim logic
├── attendance/
├── fees/                # FeeRecord, Payment
├── bookings/             # Booking, conflict logic
├── demo_requests/
├── complaints/
├── notifications/         # Notification model + daily management command
├── manage.py
├── requirements.txt
└── .env.example

frontend/
└── src/
    ├── app/
    │   ├── page.tsx                 # homepage
    │   ├── login/ register/
    │   ├── libraries/ libraries/[id]/
    │   ├── owner/  (dashboard, library, students, attendance, fees,
    │   │            demo-requests, complaints)
    │   └── student/ (dashboard, requests, bookings)
    ├── components/
    └── lib/
        └── api.ts       # single typed API client
```

One Django app per domain concern (Section 8 of `SPEC.md` maps 1:1 to these apps). Do not put
unrelated models in the same app.

## 3. Backend Conventions

- One `ModelViewSet` (or explicit `APIView` where a viewset doesn't fit) per resource
- Permission classes are explicit per view — never rely on a global default alone for
  object-level checks (owner-own-library, student-own-membership)
- Serializers validate everything the database constraint doesn't already guarantee (e.g.,
  file type/size for photos, positive amounts for fees)
- Business rules that determine correctness (membership status derivation, booking conflict,
  notification dedup, payment verification) live in model methods or a `services.py` per app —
  not duplicated in the view layer

## 4. Frontend Conventions

- All backend calls go through `src/lib/api.ts` — no scattered `fetch()` calls inside
  components
- Server components for public/discovery pages (SEO matters here per `SPEC.md` Section 7);
  client components only where interactivity requires it (forms, check-in button, seat
  selection)
- Tailwind only; no separate CSS-in-JS library

## 5. API Conventions

- REST, JSON, plural resource nouns, nested where the relationship is owned
  (`/libraries/{id}/photos/`, not a flat `/photos/?library_id=`)
- Every list endpoint paginated
- Every error response follows the shape in `SPEC.md` Section 22

## 6. Database Conventions

- UUID primary keys on all user-facing entities (not sequential integers)
- Every foreign key has an explicit `on_delete` policy, chosen deliberately (e.g., deleting a
  `Membership` should not silently cascade-delete `Payment` history — use `PROTECT` or
  `SET_NULL` as appropriate, documented at the model)
- Constraints from `SPEC.md` Section 17 are enforced as actual database constraints
  (`UniqueConstraint`, partial indexes), not only application-level checks

## 7. Authentication & Authorization Rules

- One system only: Django auth + DRF SimpleJWT. Do not add a second auth provider.
- Every protected view checks role AND object ownership. A permission class that only checks
  role is incomplete — reviewers should reject it.

## 8. Validation

Validate server-side always. Frontend validation is a UX convenience, never the security
boundary (this applies especially to payment success and file uploads — see `SPEC.md` 8.3,
8.11).

## 9. Error Handling

Return the standard error shape (`SPEC.md` Section 22). Never leak whether a resource exists
when the requester isn't authorized to see it — return `404`, not `403`, for another owner's
private resources.

## 10. Testing

Critical paths only, per `SPEC.md` Section 45. Do not write exhaustive tests for every getter/
setter — write tests for the rules that, if broken, cause data corruption or a security
failure (duplicate identity, duplicate booking, duplicate payment activation, duplicate
notification, cross-owner/cross-student data leakage).

## 11. Security

- No secrets in source control, ever — `.env` files are gitignored, `.env.example` documents
  the shape only
- Payment webhook signatures verified before any state change
- File uploads validated before storage (Section 8.3 of `SPEC.md`)

## 12. File Upload Rules

Backend validates content type and size before forwarding to Supabase Storage. Never trust a
client-provided MIME type alone — check actual file content.

## 13. Payment Implementation Rules

- Only Razorpay. No second gateway, no wallet, no subscription billing engine.
- Membership state changes only from the verified webhook handler, never from a frontend
  callback.
- Webhook handler must be idempotent (Section 17 of `SPEC.md`).

## 14. Notification Implementation Rules

- WhatsApp + SMS, both via MSG91 (Section 7/8.9 of `SPEC.md`). Do not add a second BSP or a
  separate email channel without an explicit scope change to `FEATURES.md`.
- Both channels are attempted independently for every trigger; a failure on one channel must
  not block or retry the other.
- The WhatsApp template and the SMS DLT template/sender ID must be registered and approved
  before this feature can be tested end-to-end — start this registration on Day 1 of the
  relevant sprint, not when the notification feature is reached, since approval is not
  same-day.
- The daily job must be safe to re-run without creating duplicate sends (dedup key checked
  before attempting delivery, not after).
- Always create the in-app `Notification` record regardless of delivery outcome, so the owner
  dashboard remains a reliable audit trail even if both channels fail for a given send.

## 15. Git Workflow

- `main` is protected — no direct pushes
- Feature branches: `feature/<area>-<short-description>` (e.g.,
  `feature/bookings-conflict-check`)
- Fix branches: `fix/<area>-<short-description>`
- Every change lands via PR with review before merge

## 16. Commit Conventions

Conventional commits: `feat(scope): ...`, `fix(scope): ...`, `refactor(scope): ...`,
`test(scope): ...`, `docs(scope): ...`, `chore(scope): ...`. Never `"update"`, `"fix stuff"`,
`"final"`, `"changes"`. One logical change per commit.

## 17. Pull Request Rules

Every PR states: what changed, why, how it was tested, migration info if the schema changed,
and any known limitations. Before merge: it runs, tests pass, no secrets committed, no
unrelated changes bundled in.

## 18. Environment Variables

See `SPEC.md` Section 44 for the full list. `.env.example` in each of `backend/` and
`frontend/` must stay in sync with what the code actually reads.

## 19. Documentation

`README.md` covers human setup. This file (`AGENTS.md`) covers engineering rules. `SPEC.md`
covers exact behavior. `FEATURES.md` covers what's in scope. Keep all four consistent — a code
change that alters documented behavior must update `SPEC.md` in the same PR.

## 20. AI Coding Agent Rules

- Read `SPEC.md` and `FEATURES.md` before implementing anything.
- Follow `FEATURES.md` strictly — every P0 feature listed there must be built.
- Do not invent requirements not present in `SPEC.md`/`FEATURES.md`.
- Do not remove, downgrade, or silently defer a required MVP feature.
- Do not move a required MVP feature to future/P2.
- Do not implement anything from the Future/P2 or Out-of-Scope lists unless explicitly asked.
- Do not introduce dependencies, services, or architecture not named in this file or `SPEC.md`
  Section 7.
- Do not use mock/hardcoded data for any completed feature — dashboards and lists must query
  real data.
- Do not create a fake or stub API where a real one is specified.
- Do not bypass authentication or authorization, even temporarily "to test."
- Never trust frontend validation for anything security- or money-relevant; validate on the
  backend (Section 8 above).
- Never trust a frontend payment-success signal — activation happens only from the verified
  webhook (`SPEC.md` 8.11).
- Enforce booking-conflict and duplicate-check-in prevention at the database level, not only
  in application code.
- Enforce notification deduplication via the documented `dedup_key`, not an ad hoc check.
- Run tests after any meaningful change; fix failures rather than skipping or deleting the
  test.
- Do not modify `FEATURES.md` or `SPEC.md` to make implementation easier — if an implementation
  conflict is found, surface it explicitly instead of quietly resolving it in either direction.
- Preserve existing working functionality when modifying code — inspect before editing, patch
  rather than rewrite where an existing implementation already meets `SPEC.md`.
- Prefer the simplest production-safe solution described in `SPEC.md` over a more elaborate
  alternative; do not over-engineer beyond what `SPEC.md` specifies.

## 21. Definition of Done (engineering)

A task is done when: it matches its `SPEC.md` section exactly, it has the critical-path tests
described in Section 10 above, it passes review against the checklist in Section 17, and it
does not regress any previously-passing item in `FEATURES.md` Section 17.

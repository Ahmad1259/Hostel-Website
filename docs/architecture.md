# Architecture

Status: **Phase 1 draft.** Application architecture is decided; the database
service decision (task 1.3) is finalized in `free-tier-cost-review.md` and
mirrored here. Sections marked PENDING are completed after verified research.

## 1. Shape of the system

One repository, one deployable application, three surfaces:

```
[Public site]        [Student portal]     [Staff portals]
  discovery,           application,         owner / manager / warden /
  tours, search        payments, history    admin operations
        \                  |                  /
         +-------- single SPA (React + Vite + TypeScript) --------+
                              |
                   REST API (Node.js + Express + TypeScript)
                    authn/z - tenant guard - validation - audit
                              |
                Relational DB (Oracle or PostgreSQL - see section 4)
                              |
              Private object storage (documents, payment proofs,
              room media) reached only via signed, expiring URLs
                              |
              Notification scheduler (provider-agnostic adapters)
```

Rationale for one app rather than microservices: a small team (and coding
agents) on free tiers cannot afford service sprawl. Boundaries are enforced as
modules inside one codebase, so splitting later stays possible but is not
required today.

## 2. Stack (decided)

- **Frontend:** React 18 + Vite + TypeScript, plain CSS with design tokens
  (no heavy UI kit, to keep the identity original and the bundle small).
  Router with route-level role guards; all authoritative checks server-side.
- **Backend:** Node.js + Express + TypeScript. Layered modules:
  `auth`, `hostels`, `discovery`, `admissions`, `occupancy`, `billing`,
  `notifications`, `audit`. Each module exposes a typed service interface;
  controllers stay thin.
- **Validation:** schema-first (`zod` on both sides) against shared types in
  `packages/contracts` - the contract directory referenced in
  `docs/COLLABORATION.md`.
- **DB access:** a thin repository layer with parameterized SQL; migrations as
  versioned SQL files in `db/migrations/` (engine-neutral where possible,
  engine-specific files where constraints require it).
- **Object storage:** S3-compatible free/cheap option or the chosen cloud's
  native bucket service; private ACLs, server-issued signed URLs only.
- **Scheduler:** an in-process job runner (e.g. `node-cron`) for fee checks and
  notification dispatch - chosen so it works inside a single free web service
  without a paid worker tier; revisit if the host sleeps (see section 5).

## 3. Security architecture

- **Authentication:** email/phone + password (argon2id hashing), server-side
  sessions via httpOnly secure cookies + rotating refresh tokens; short-lived
  OTP challenges for verification and reset. Google login only after real OAuth
  credentials exist. MFA as TOTP where supported.
- **Authorization:** permission constants generated from
  `docs/role-permission-matrix.md` are checked in middleware on every sensitive
  route; UI guards are cosmetic only.
- **Tenant isolation:** a `requireHostelScope` middleware resolves the caller's
  active `hostel_staff_assignments` and injects allowed hostel IDs; every query
  in staff modules must receive its hostel filter from that context - raw
  `hostel_id` from request bodies is never trusted. Automated tests attempt
  cross-hostel access on every staff endpoint and must fail.
- **Input surface:** request validation, upload MIME/size allowlists, image
  re-encoding for media, rate limiting on auth and upload routes, secrets via
  environment only.
- **Audit:** append-only `audit_events` for every state change of money,
  assignments, roles and documents.

## 4. Data strategy - production choice deferred (owner decision D-006)

Research completed 2026-10-26 (task 1.3); full evidence in
`docs/free-tier-cost-review.md`. Two viable $0 paths remain:

- **Path A - Oracle-first:** OCI Always Free ARM VM (2 OCPU/12 GB after the
  June 2026 reduction) running Node/Express + Oracle Database 23ai + our
  scheduler, with daily dumps to separate storage. Requires an owner-registered
  OCI account (card on file for the hold), a local Docker install for dev, and
  acceptance of free-instance reclaim risk during the pilot.
- **Path C - Postgres-first (fallback):** Neon free Postgres (1 GB, long-lived)
  + Render free web service (cold starts; no free cron - reminders via an
  external once-daily trigger on a protected endpoint). Simplest operations;
  deviates from the brief's Oracle preference, which master-prompt section 7
  allows if documented and approved.

Recommendation: **Path A** if the owner can register OCI and tolerate pilot-era
outage risk; otherwise **Path C** to keep momentum, keeping the schema
engine-neutral so an Oracle migration stays a DDL task. Both preserve every
integrity rule in `database-schema.md`.
## 5. Deployment topology (zero-budget-first)

- Develop locally with Node 24; local database for the chosen engine (or the
  cloud free DB with a local fallback for offline work).
- Deploy frontend + API to a free tier whose current terms are verified at
  deploy time (never assumed).
- If the chosen app host sleeps on inactivity, the API is kept warm only within
  that host's fair-use rules; otherwise the scheduler design moves to
  DB-side jobs or an external free cron trigger hitting a protected endpoint.
- Every environment (local / preview / production) has its own config file
  template (`.env.example`), and production secrets exist only in the host's
  secret store.

## 6. What is explicitly NOT in this architecture

- No microservices, Kubernetes, or message queues at this scale.
- No unofficial WhatsApp Web automation anywhere in billing or notifications.
- No fabricated seed content presented as real listings.
- No client-side payment status changes, ever.

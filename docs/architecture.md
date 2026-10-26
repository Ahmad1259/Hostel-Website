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

## 4. Data strategy - PENDING final choice

Local machine has Node 24 + Git but **no Docker and no Python**, which shapes
the options. The full comparison with sources and limits is in
`docs/free-tier-cost-review.md`. Summary of paths:

- **Path A - Oracle Cloud Always Free Autonomous DB (23ai) + free app host.**
  Matches the brief's Oracle requirement; connectivity via the thin driver over
  HTTPS with wallet credentials. Risks to verify: signup requirements (payment
  method), free resource ceilings, reachability from a third-party free host.
- **Path B - Oracle 23ai Free on Always-Free ARM compute** hosting app + DB
  together. Self-contained and genuinely Oracle; but requires installing Docker
  on the laptop for local development and more self-management (backups, TLS).
- **Path C - PostgreSQL on a long-lived free tier** (e.g. Supabase/Neon free
  plan) with Oracle-compatible discipline kept in the schema; migration path
  to Oracle documented. Cheapest to operate; deviates from the brief's stated
  preference - allowed by section 7 of the master prompt if Oracle's complexity
  is shown to be excessive, **with owner approval**.

Whatever is chosen, the logical model in `database-schema.md` does not change;
only DDL, drivers and ops scripts do.

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

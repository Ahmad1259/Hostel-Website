# Master Task Board

Maintained by the manager role. Dependency graph matters more than the list:
never start a task whose dependencies are not merged and green.

Legend: ⬜ not started · 🔵 in progress · 🟡 awaiting review · ✅ done & merged

## Phase 1 — Foundations

| # | Task | Depends on | Status |
|---|---|---|---|
| 1.1 | Inspect environment, clone repo, establish `main` | — | ✅ |
| 1.2 | Verify available skills; record what was actually used | — | ✅ |
| 1.3 | Research Oracle free options + hosting alternatives (with sources) | — | 🔵 |
| 1.4 | `docs/architecture.md` incl. DB decision + alternatives | 1.3 | ⬜ |
| 1.5 | `docs/database-schema.md` + ERD | — | 🟡 draft |
| 1.6 | `docs/role-permission-matrix.md` | — | 🟡 draft |
| 1.7 | `docs/free-tier-cost-review.md` | 1.3 | ⬜ |
| 1.8 | `docs/backup-and-recovery.md` | 1.4 (engine choice) | 🟡 draft, engine section pending |
| 1.9 | `docs/COLLABORATION.md`, task board, decision log | — | ✅ |
| 1.10 | **GATE: owner approves architecture & stack** | 1.4, 1.5, 1.6, 1.7 | ⬜ |

## Phase 2 — Design system & public prototype

| # | Task | Depends on | Status |
|---|---|---|---|
| 2.1 | Design direction research (current guidance, not templates) | 1.10 | ⬜ |
| 2.2 | Design tokens: typography, color, spacing, motion rules | 2.1 | ⬜ |
| 2.3 | Static prototype: home, search/listing, hostel detail, tour view | 2.2 | ⬜ |
| 2.4 | States: loading, empty, error, success | 2.3 | ⬜ |
| 2.5 | Browser audit at 372/768/1280px + reduced-motion + a11y pass | 2.3, 2.4 | ⬜ |
| 2.6 | **GATE: design review, de-genericize weak areas** | 2.5 | ⬜ |

## Phase 3 — Public discovery (real data plumbing)

3.1 repo scaffold (app + db migrations layout) · 3.2 hostels/rooms/beds/facilities/media CRUD APIs with contracts · 3.3 search & filter API incl. availability snapshot · 3.4 connect public pages to API · 3.5 media viewer honoring true media types · **GATE: filters return only real, verified data; availability timestamp visible**

## Phase 4 — Auth, roles, admissions

4.1 sessions, password policy, rate limiting, audit log · 4.2 role grants + tenant isolation middleware (tests prove cross-hostel reads fail) · 4.3 student registration/portal · 4.4 applications + private document upload/verification · 4.5 optional Google login only when properly configured · **GATE: QA attacks own-the-system attempts; server-side checks verified**

## Phase 5 — Occupancy

5.1 bed inventory statuses · 5.2 assignment transaction + DB constraints (no double-active bed) · 5.3 check-in/out · 5.4 renewals preserving history · 5.5 transfer flow per schema doc · **GATE: constraint-violation tests pass; history immutable**

## Phase 6 — Billing

6.1 fee policies + monthly challan generation (idempotent, unique per period) · 6.2 payment submissions (pending-only) · 6.3 verification workflow + receipts (one per submission, balance-safe) · 6.4 deposits · 6.5 outstanding-balance view · **GATE: proof upload never marks paid; duplicate verification blocked under concurrency**

## Phase 7 — Notifications

7.1 provider-independent adapter interface · 7.2 scheduler: balance re-check, dedupe key, retries with limits, consent check · 7.3 chosen provider integration **only if verified within budget**, else documented fallback · **GATE: no duplicate reminders; unsent ≠ sent in records**

## Phase 8 — Hardening & release

8.1 independent QA sweep (all roles, failure scenarios) · 8.2 security review incl. tenant-isolation attacks · 8.3 backup restore drill · 8.4 production build + deployment runbook with verified free-tier steps · **GATE: zero critical/high open defects; production-ready claim only if all evidence recorded**

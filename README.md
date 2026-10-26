# Hostel Discovery & Long-Term Accommodation Platform

A multi-hostel accommodation platform for students in Pakistan: public hostel
discovery with authentic room tours, admissions with document verification,
bed/occupancy management, monthly challans with verified payments and receipts,
and automated fee reminders — with strict data isolation between hostels.

**Current status: Phase 1 — architecture and data model drafted, awaiting owner
approval.** No application code exists yet, and by design none will before the
Phase 1 gate passes.

## Development plan (phased, gated)

| Phase | Deliverable | Gate to proceed |
|---|---|---|
| 1 | Workspace, architecture, schema, cost review, collaboration rules | Owner approves architecture + DB/hosting strategy |
| 2 | Original design system + browser-tested public prototype | Design review at mobile/tablet/desktop; no fake content |
| 3 | Public discovery, filtering, real room tours with genuine media | Filters reflect real data; availability timestamps shown |
| 4 | Auth, roles, applications, private document verification | Role tests pass; tenant isolation proven; no public doc URLs |
| 5 | Beds, assignments, check-in/out, renewals, transfers | DB constraints block double-assignment in tests |
| 6 | Challans, payment verification, receipts, deposits | Proof upload ≠ paid; duplicate-payment tests pass |
| 7 | Notification scheduler + provider adapters | Balance re-checked before send; dedupe proven; honest fallback |
| 8 | Independent QA, security review, restore drill, deployment docs | Zero critical/high defects; verified free-tier limits |

## Repository map

```
docs/
  architecture.md              system design and chosen services (with sources)
  database-schema.md           full logical data model
  entity-relationship-diagram.md
  role-permission-matrix.md    the authoritative access rules
  free-tier-cost-review.md     verified free-tier limits and trade-offs
  backup-and-recovery.md       backup policy + mandatory restore drills
  COLLABORATION.md             git/branch/worktree rules for agents
  TASKS.md                     master task board and dependency graph
  DECISIONS.md                 decision log (what/why/when)
  qa-reports/                  QA findings by severity, with evidence
```

## Ground rules

- Real data only in the product; no fabricated testimonials, ratings, or
  availability.
- Money records are immutable and auditable; corrections are new records.
- Nothing paid is activated without the owner's explicit approval.
- A feature is "working" only when tested in a real browser/database, not when
  generated.

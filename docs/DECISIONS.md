# Decision Log

Append-only. One entry per consequential technical or product decision.

---

**D-001 · 2026-10-26 · Repository and baseline**
Decision: single repository `Ahmad1259/Hostel-Website`, `main` as the stable
branch, feature branches per `docs/COLLABORATION.md`, worktrees for parallel
agent work on one machine.
Why: repo was empty; Git must be the shared source of truth across accounts.
Status: adopted.

**D-002 · 2026-10-26 · Logical data model first**
Decision: full logical schema + ERD drafted before any framework or DDL exists
(`docs/database-schema.md`).
Why: bed-assignment uniqueness, challan immutability and tenant scoping are
schema-level guarantees; retrofitting them after code exists is how platforms
get money bugs.
Status: adopted, pending owner approval.

**D-003 · 2026-10-26 · Payment model separation**
Decision: `payment_submissions` (claims) and `receipts` (confirmed money) are
separate entities; only staff verification moves a challan balance, inside one
transaction.
Why: master requirement §9 — a screenshot must never mean "paid".
Status: adopted.

**D-004 · 2026-10-26 · Database engine — Oracle vs alternatives**
Pending task 1.3 research results. Constraints being weighed: no Docker and no
Python on this machine; owner budget ≈ zero; free tiers with inactivity deletes
are unsafe for real student data; Oracle Autonomous DB Free requires verifying
current signup terms and connectivity from a free external host.
Options under evaluation: (A) Oracle Cloud Always Free Autonomous DB 23ai +
free app host; (B) Oracle 23ai Free running on Always-Free ARM compute (needs
Docker install on laptop for local dev); (C) PostgreSQL on a genuinely
long-lived free tier (Supabase/Neon) with Oracle migration path documented.
Decision and evidence will be recorded in `docs/architecture.md` and
`docs/free-tier-cost-review.md`.
Status: open — owner approval required before Phase 2.

**D-005 · 2026-10-26 · WhatsApp**
No unofficial WhatsApp Web automation in the core design. Notification service
is provider-independent with honest delivery records; a real provider is
selected (or explicitly deferred) during Phase 7 after verifying current costs.
Status: adopted as policy.

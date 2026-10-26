# Backup and Recovery Plan

Status: **Phase 1 draft.** The exact commands depend on the approved database
service (see `free-tier-cost-review.md`); the policy below applies either way.

## Why this matters here

This platform holds admission records, identity documents, bed history and
financial challans. Losing or corrupting that data breaks real people's housing
and a hostel's accounting. "The hosting provider backs it up for you" is not
acceptable - we verify our own backups.

## Policy

| Data | What | Frequency | Retention | Storage location |
|---|---|---|---|---|
| Database | Full logical dump | Daily | 30 days rolling | Separate bucket/account from the app |
| Database | Pre-migration snapshot | Before every schema change | Until next release + 30 days | Same as above |
| Transaction deltas | Paid challan/receipt change window | Hourly if the free tier supports it, else daily with the full dump | 30 days | Same as above |
| Documents bucket | Private student documents | Incremental copy | 90 days versioned | Object storage with versioning enabled |
| Repository | Git history | Continuous (it is git) | Forever | GitHub (private) |
| Secrets | Env template values | On change, manually | - | Owner password manager, never in git |

Recovery objectives (honest for a free-tier system): **RPO <= 24 hours** (could
lose up to a day of records if disaster strikes right before a daily dump),
**RTO <= 4 hours** (restore is manual). If the business later needs tighter
numbers, paid backups are the way - documented in `free-tier-cost-review.md`.

## Verification (this is the part most people skip)

1. A restore drill is mandatory in Phase 8 and quarterly afterwards:
   download the newest dump into a throwaway local database, run integrity
   queries (row counts, challan balance sums vs. payment totals, one bed = one
   active assignment), and record results in `docs/qa-reports/`.
2. An untested backup is treated as *no backup*. If a drill fails, that is a
   critical defect and blocks release.
3. Backup files contain personal data. They must be encrypted at rest and their
   storage location must have access restricted to the admin.

## Free-tier realities we will not pretend away

- Some free database services pause or delete data after inactivity (common on
  free app hosts). Mitigations considered: keep-alive ping within fair-use
  rules, choosing services whose free tier is explicitly long-lived, or running
  the database on an Always-Free compute instance we control. The final choice
  is recorded in `architecture.md` with the current terms verified at that time.
- Object-storage free allowances are small. Media policy (photo count/size per
  hostel) must stay within them; limits are documented, not discovered by
  accident when a bill or a lockout appears.

# Free-Tier Cost Review (verified 2026-10-26)

All limits below were checked against official documentation on 2026-10-26 by a
research pass with source links. Free tiers change often - **re-verify each
before sign-up, and never enter a payment method without the owner's explicit
approval.**

## 1. Oracle options

### Oracle Cloud Always Free Autonomous Database (23ai / now "AI Database 26ai" family)
- 2 free instances per tenancy, 20 GB storage each, ~30 concurrent sessions max.
- HTTP layer (ORDS/APEX/Database Actions) throttles at roughly 3-6 concurrent
  users (HTTP 429). Oracle's own docs state the Always Free ATP is **not
  intended for production use**.
- Instance **auto-stops after 7 days without connections** and the service can
  be **reclaimed after 90 cumulative inactive days**.
- Sign-up requires a **credit/debit card** (hold only, no charge unless you
  upgrade). Free Tier is unavailable in some regions and the home region choice
  is permanent.
- Connectivity from an external host: works over the public TCPS endpoint
  (outbound from the app is fine). The wallet is optional when mTLS is set to
  "Not Required" - node-oracledb **thin mode** connects with TLS + username and
  password only, no Oracle Client libs, no Python needed.
  Sources: docs.oracle.com - autonomous-always-free, signingup, freetier,
  connecting-python-tls, node-oracledb connection docs.

### Oracle Database 23ai/26ai Free (local edition)
- Limits: 2 CPU threads, 2 GB RAM, 12 GB user data, one instance per machine.
- It is **not a hosted cloud service** - it runs only where you install it.
- Available as a container image; a Windows-native install avoids Docker, but
  this machine currently has **no Docker and no Python** (Node 24 is installed).

### OCI Always Free Compute (VMs)
- **Changed June 2026:** Ampere ARM allowance reduced from 4 OCPU/24 GB to
  **2 OCPU/12 GB** (1,500 OCPU-hrs + 9,000 GB-hrs per month), 200 GB boot
  volume total; 2 x86 micro VMs still free.
- One ARM VM *can* host the app + Oracle DB together for $0, but free ARM
  capacity is scarce in some regions and Oracle **reclaims idle free instances**
  (widely reported). Reclaim = outage or data loss without disciplined backups.
  Source: infoq.com 2026/07 oracle-cloud-free-tier-limits; docs.oracle.com freetier.

## 2. Application hosting

| Service | 2026 reality | Verdict for us |
|---|---|---|
| Render free | 0.1 CPU / 512 MB, sleeps after 15 min idle, 750 instance-hrs/mo; **workers and cron jobs are NOT free**; free Postgres deletes after 30 days | Acceptable for a pilot API (cold-start delay), not for scheduled jobs |
| Railway | No true free tier; one-time $5 trial / $0 plan with $1 non-rollover credit | Excluded |
| Fly.io | Free tier removed (late 2024); card + ~$2 minimum | Excluded |
| Vercel Hobby | Serverless only, 300s max function, crons **once/day**, **non-commercial use only** | Excluded for a commercial platform |
| Netlify Free | Static + serverless (30s functions); no long-running server | Frontend static hosting candidate only |
| OCI Always Free VM | $0 Linux ARM VM, full control, app + DB + cron in one place | Only $0 option that supports everything, with reclaim risk to manage |

Key honest conclusion: **no free app host outside Oracle gives us a
24/7 server + database + scheduled jobs for $0.** The combinations that work at
exactly $0/month are (A) one OCI Always Free ARM VM running everything, or
(B) free-frontend + Render-free-API + Neon-Postgres with degraded cold starts
and no reliable scheduler.

## 3. Free PostgreSQL (fallback engines)

- **Neon**: 1 GB / project, 20 GB total across 100 projects, 100 compute-hrs/mo,
  autosuspends after ~5 min idle, pairs well with any external host. Long-lived
  free plan; better fit than Supabase for our "always-on-ish" needs.
- **Supabase**: 500 MB DB, 2 projects, **pauses after 1 week inactivity** -
  pause behavior is hostile to real resident data availability.

## 4. WhatsApp / messaging - major finding

- Meta **removed free messaging**: template (utility) messages are billed
  per-message since 1 Oct 2025, and since **1 Oct 2026 even replies inside the
  24-hour service window are billed**. Reported Pakistan utility rates ~ $0.015
  per message (secondary source, unverified against Meta's card directly).
- Unofficial WhatsApp automation (WhatsApp Web libraries) violates WhatsApp
  Terms and leads to **permanent number bans**. It will not touch billing or
  reminders.
- Practical $0 phase: **in-app notifications + email** (free provider within
  limits), with the WhatsApp adapter interface ready so a paid Meta Business
  account can be connected later, when the platform has revenue and the owner
  explicitly approves costs.

## 5. Costs that are unavoidable eventually (approval required before any of it)

| Item | Approx. | When needed |
|---|---|---|
| Domain name | ~$10-15/yr | Real launch |
| WhatsApp Business API | per-message, Pakistan utility rate | Only with revenue |
| Object storage beyond free allowance | usage-based | As media grows |
| Backup beyond free limits | few $/mo | Once real records exist |
| VPS if free VM is reclaimed/stops being offered | ~$5/mo | Production reliability |

## 6. Decision input (recommended, pending owner approval)

**Path A - Oracle-first, $0 (recommended pilot):** one OCI Always Free ARM VM
runs Node/Express + Oracle DB (23ai container) + our scheduler; frontend static
files served by the same VM or Netlify. Daily dumps + document copies to a
separate free storage bucket. Mitigates inactivity reclamation with a compliant
keep-alive and disciplined restores. Requires: owner provides a card for OCI
sign-up (hold only), accepts reclaim/outage risk during pilot, installs Docker
Desktop locally (or we use a cloud devcontainer) for a local Oracle instance.

**Path C - Postgres-first, $0 (lowest ops burden):** Vercel/Netlify static
frontend is out for commercial use, so: Render free web service (cold starts) +
Neon free Postgres + reminders via an external free cron hitting a protected
endpoint once daily. Oracle compatibility rules (constraints, transactions,
immutability) stay identical in the schema; migration to Oracle later is a DDL
task, not a redesign.

The master prompt prefers Oracle but authorizes documenting trade-offs and
proposing alternatives (section 7). Both paths are buildable on the same logical
schema already drafted. The choice hinges on: willingness to register OCI with
a card + manage a Linux VM (A), versus accepting Postgres + degraded free app
hosting for simplicity (C).

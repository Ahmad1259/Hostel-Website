# Collaboration and Git Rules (all contributors, human or agent)

This repository is developed by several authorized accounts/agents. Git is the
single source of truth. These rules prevent agents from destroying each other's
work.

## Branches

- `main` — always releasable. Only the manager agent merges into it.
- `feature/<role>/<task-id>-<short-name>` — e.g. `feature/backend/schema-phase1`.
- `fix/<role>/<short-name>` — defects found by QA.
- Never force-push to `main`. Never rewrite shared history without explicit
  owner authorization.

## Worktrees (recommended for parallel work on one machine)

```
git worktree add ../hostel-feat-backend feature/backend/<task>
git worktree add ../hostel-feat-design  feature/design/<task>
```
Each agent works in its own worktree directory, commits, pushes its branch, and
the manager reviews and merges. Agents on *different* accounts must push to this
same remote — separate accounts do not share a workspace automatically.

## File ownership (avoid two agents editing one file)

| Area | Owner role | Typical files |
|---|---|---|
| Architecture, schema migrations, shared contracts | Backend/DB agent | `docs/architecture.md`, `db/migrations/*`, `server/src/contracts/*` |
| Public pages, design system | Product Design agent | `web/src/design-system/*`, `web/src/pages/public/*` |
| Auth, portals, operations flows | Hostel Operations agent | `server/src/modules/{auth,admissions,occupancy,billing}/*` |
| Tests, QA reports | QA/Security agent | `tests/*`, `docs/qa-reports/*` — QA does not fix product code, it reports |
| Merges, task board, decision log | Manager agent | `docs/TASKS.md`, `docs/DECISIONS.md` |

If a change needs to touch another area's files, it is split into two commits
or coordinated by the manager first.

## Contracts before code

Before Phase 3, the manager publishes in `server/src/contracts/`:
- OpenAPI spec for every endpoint,
- shared TypeScript types,
- the permission matrix as machine-readable constants mirroring
  `docs/role-permission-matrix.md`.
Frontend and backend are built against the contract, not against each other's
guessed behavior.

## Commit discipline

- Coherent commits, imperative description: `Add bed-assignment uniqueness constraints`.
- Each commit states: task id, files changed, and any decision it implements.
- Run lint, typecheck, and the tests related to your change before committing.
- **Never commit**: `.env`, secrets, real student data, database dumps,
  payment proofs. `.gitignore` covers these; review stays alert anyway.

## Definition of done (per task)

1. Code meets the contract and permission matrix.
2. Tests covering the happy path and at least the failure paths listed in the
   task exist and pass.
3. No new console/network errors in a real browser check (UI tasks).
4. Manager reviewed the diff; QA severity report updated if defects found.
5. Release gates in `docs/TASKS.md` not marked passed on claims — only on
   executed evidence.

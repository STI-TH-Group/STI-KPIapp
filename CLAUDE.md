# CLAUDE.md

Guidance for Claude Code and contributors working in the **STI KPI App** repository.

## What this repository is

The STI KPI App: a Tier 1, standalone intranet app at `http://sti-apps:8080/kpi`, governed by
**STI-STD-2026-001 STI Application Development Standard**. The design is in
`Doc/STI_KPI_App_TDD.html` (read it before changing anything) and the clickable mock-up is in
`Doc/STI_KPI_App_Prototype.html`. Status: design and prototype only; no production code yet.

## Standard rule: always record changes in `changes/`

**Every change goes into the matching `changes/<area>.md` file, in the same commit or PR as
the change.** Commits, PRs and Claude Code sessions all follow this rule. A PR that changes
files but has no change-log entry is not complete.

1. Pick the area or areas the change touches from the table below. A change that spans
   several areas gets an entry in each, and each entry points to the others.
2. Add the entry at the **top** of the file's `## Log` section, so the newest comes first.
3. Use the entry format below. Write in English and add Thai where the readers need it.
4. Record decisions, open items that were closed, and anything that changes the design. When
   the design changes, also update `Doc/STI_KPI_App_TDD.html` and raise its version number.
5. Never log secrets, passwords, connection strings or personal data about named staff.

| Area | File | Covers |
|------|------|--------|
| Design & documentation | `changes/design-docs.md` | TDD, prototype, README, this file |
| Backend | `changes/backend.md` | `STI.Kpi.Api`: controllers, business rules, scoring, background jobs |
| Frontend | `changes/frontend.md` | `STI.Kpi.Web`: React screens, TH/EN text, branding |
| Database | `changes/database.md` | `sti_kpi` schema, numbered migrations, seed data |
| Auth & security | `changes/auth-security.md` | STI Main SSO, local admin login, roles, privacy, audit |
| ERP integration | `changes/erp-integration.md` | Resin DW and `pps2` read-only feeds, import job |
| Deployment & ops | `changes/deployment-ops.md` | IIS, App Pool, YARP gateway, ports, config, TLS, SMTP |

Add a new area file (and a row here and in `changes/README.md`) only when a change fits none of
the existing areas.

### Entry format

```markdown
### YYYY-MM-DD: Short title
- **What:** what changed, in one or two lines
- **Why:** the reason, decision or request behind it
- **Files:** `path/one`, `path/two`
- **Impact:** what someone else must do or watch for (or "None")
- **Ref:** PR/commit, related entries in other area files, TDD section
```

## Working notes

- Thai text: write files as UTF-8. Do not use PowerShell to read or write files containing Thai.
  On Windows run Python with `PYTHONUTF8=1 PYTHONIOENCODING=utf-8`.
- Branding follows the STI brand system (Navy `#1B2D5B`, Tech Cyan `#00A3E0`, Precision Green
  `#00A878`, Precision Silver `#8C8C8C`; logo v2 only).
- Work on a branch and merge into `main` through a pull request.

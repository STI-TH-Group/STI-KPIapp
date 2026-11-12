# Backend (STI.Kpi.Api) · Change log

**Covers:** ASP.NET Core API: controllers, repositories, scoring rules, background jobs, logging, Excel.
**TDD reference:** section 2 (Architecture), section 3 (Scoring rules) and section 4 (Modules, flows and API).

Newest log entry first. Rule and entry format: [`../CLAUDE.md`](../CLAUDE.md). The baseline below is TDD v0.5 (`Doc/STI_KPI_App_TDD.html`); keep it current as changes land.

## Design baseline (TDD v0.5)

### Stack and role
- ASP.NET Core on **.NET 10**, built from `templates/tier1-core-dotnet`. Hosted on IIS (App Pool `STI.Kpi`, port **5230**, `PathBase = /kpi`).
- The **only writer** to `sti_kpi`. It owns business rules, scoring, permissions, sign-in, jobs, and all database and ERP access. It never renders pages.
- Data access with **Dapper + MySqlConnector**. Logging with **Serilog** (30-day files). Excel with **ClosedXML** (template import, board and review export).
- Serves the React build from `wwwroot`: one IIS site, one address, no CORS.
- `Auth:Mode`: `sso` in production, `local` for development before the STI Main registration exists.

### Solution layout
```
src/STI.Kpi.Api/
├─ Common/      ApiResponse, PageQuery, ExceptionMiddleware, ErpDate, PasswordHasher
├─ Auth/        SsoController, StiMainClient (exchange, JWKS cache, userinfo), LocalLoginController,
│               JwtService, AuthSessionRepository, KpiPolicies.cs   (template's LdapService removed)
├─ Features/    Register/ Targets/ Entry/ Scoring/ Reviews/ YearEnd/ Boards/ Admin/ People/
│               (Controller + Repository + Dtos per folder)
├─ Jobs/        ReminderJob, ErpImportJob, PeriodCloseJob  (IHostedService)
├─ Database/Migrations/  001_core.sql · 003_kpi_org.sql · 004_kpi_register.sql …
└─ wwwroot/     build output of STI.Kpi.Web
```

### Modules and API groups
| Module | Main API | Phase |
|---|---|---|
| M7 Company board | `GET /api/boards/company?period=&layer=` (any signed-in user) | 1 |
| M1 KPI register | `/api/kpis` (CRUD), `/api/kpis/tree?cycle=`, `/api/kpis/split?cycle=&unit=` | 1 |
| M2 Targets and approval | `POST /api/kpis/{id}/submit` · `/approve` · `/return`, each with `{ comment }` | 1 |
| M3 Monthly results | `GET/PUT /api/entries?unit=&period=` · `POST /api/attachments` (multipart) | 1 |
| Goal + Base layers | `/api/base-kpis` · `POST /api/base-kpis/seed?cycle=2027` | 1 |
| M4/M5 Quarterly review | `/api/reviews?unit=&quarter=` · `/actions` · `/change-requests` | 2 |
| M6 Year-end review | `/api/year-end?unit=&year=` · `/sign` | 3 |
| M8 Admin | `/api/admin/cycles` · `/periods` · `/org-units` · `/rules` · `/assignments` · `/audit` | 1–3 |
| M9 My KPIs | `/api/people/me/kpis` | 4 (2028) |
| Auth | `/sso/start`, `/sso/callback`, `/api/auth/session`, `/me`, `/signed-out`, `/local-login`, `/logout` (see [auth-security.md](auth-security.md)) | 1 |

Example calls:
```
GET  /api/kpis?cycle=2027&unit=INJ&layer=goal&strategy=S2&status=approved&page=1&pageSize=50
GET  /api/kpis/tree?cycle=2027            → goal → strategy → company KPI → team KPI
GET  /api/entries?unit=INJ&period=2027-05 → grid rows with target, last value, RAG
PUT  /api/entries                         → [{ kpiId, periodId, value, comment }]
GET  /api/kpis/split?cycle=2027&unit=INJ  → { baseWeight, goalWeight, count, range, warning }
```

### API contract (`Standard/docs/02-api-contract.md`)
- Envelope `{ success, message, data, pagination? }`. `message` is shown to staff verbatim, in the user's language (`Accept-Language: th|en`).
- Paths are lower-kebab plural; query and JSON are camelCase.
- Lists take `page`, `pageSize` (cap 200), `search` and `sort` (whitelist per endpoint).
- `GET /api/health` is anonymous and reports `database`, `dw` and `sso` (STI Main JWKS reachable).
- Status codes: 400 (field errors in `data`), 401, 403, 404, and **409** for "period is locked" or "KPI changed since you opened it" (optimistic check on `updated_at`).

### Business rules
- **Goal KPI:** approval rejects an orphan (no parent KPI and no strategy). **Base KPI:** needs no parent; an optional `strategy_id` marks it goal-linked.
- **Targets are versioned:** each change adds a `kpi_target` row and never overwrites the original. Changes are approved at a quarterly review (`change_request`).
- **Monthly entry:** due by the 5th working day. A comment is required for amber and red. ERP KPIs are read-only. Locked periods reject writes.
- **Soft warnings (never a block):** Base/Goal weight split outside the range for the level, and more than 8 KPIs in a set.
- **Reviews:** open actions carry forward to the next quarter. At year end, Base KPIs default to `keep`.

### Scoring (`Features/Scoring/ScoringService.cs`)
This class holds the rules. `scoring.ts` mirrors them only for instant feedback while typing, and both are tested against the same table of cases. The server value is the one stored.

| Direction | Achievement |
|---|---|
| up | actual ÷ target; if target = 0, any positive actual = cap |
| down | target ÷ actual; if actual ≤ 0, cap |
| range | 1.0 inside [target_low, target_high]; outside, distance-based (rule still to agree with HR) |
| pass | pass = 1.0, fail = 0 |

- Defaults: green ≥ 100%, amber ≥ 90%, cap 120%, bands 110/100/90/80 → score 5/4/3/2/1 (`scoring_rule`, per cycle).
- The period value uses the KPI's aggregation: `avg` of months with data, or `last` for cumulative KPIs.
- Team score = Σ(weight × score) ÷ Σ(weight), over approved KPIs with a result, across both layers.
- The final assessment score is the reviewer's, else the self-assessment, else the system score.
- RAG and achievement are **stored at save time**, so a later rule change cannot rewrite history.
- First test cases come from the prototype's `ach`, `scoreOf`, `periodVal` and `teamScore` with its sample data.

### Background jobs
| Job | When | Does |
|---|---|---|
| `ErpImportJob` | 1st working day, 08:00 | Imports the closed month's ERP KPIs (see [erp-integration.md](erp-integration.md)) |
| `ReminderJob` | Around the entry deadline | Late-entry reminders by email (TH + EN); "not updated" flag on the board |
| `PeriodCloseJob` | After each period | Closes and locks periods |

### Build
`npm run build` (`tsc -b` clean), then `dotnet publish -c Release` with 0 errors and 0 warnings. Never build on the server.

## Open items
- [ ] Range-direction rule outside [target_low, target_high] (agree with HR)
- [ ] Final scoring scale and RAG thresholds (business)
- [ ] Map AD roles by group DN or SID in `KpiPolicies.cs` (see [auth-security.md](auth-security.md))

## Log

### 2026-11-12: Design baseline recorded from TDD v0.5
- **What:** Recorded the backend design: stack, layout, modules and API, contract, business rules, scoring, jobs.
- **Why:** Starting point for change tracking before Phase 1 (MVP, Jan–Mar 2027).
- **Files:** `changes/backend.md`
- **Impact:** None; no code yet.
- **Ref:** TDD v0.5 sections 2–4; [design-docs.md](design-docs.md)

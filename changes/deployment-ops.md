# Deployment & ops · Change log

**Covers:** IIS hosting, App Pool, YARP gateway, ports, server folders, production config, network, TLS, SMTP, logging, backup, roadmap.
**TDD reference:** section 2 (Deployment), section 6 (NFRs) and section 7 (roadmap, open items).

Newest log entry first. Rule and entry format: [`../CLAUDE.md`](../CLAUDE.md). The baseline below is TDD v0.5 (`Doc/STI_KPI_App_TDD.html`); keep it current as changes land.

## Design baseline (TDD v0.5)

### Identity
- **STI KPI** (product mark STI K), code name `STI.Kpi.*`. A **Tier 1 (Core)** standalone intranet app at `http://sti-apps:8080/kpi`.
- Governed by **STI-STD-2026-001**, STI Application Development Standard (27 Aug 2026). Where the TDD and the standard disagree, the standard wins.
- STI Main links to `/kpi`; it does not host it.

### Hosting
| Item | Value |
|---|---|
| Ports | API **5230**, Vite dev **5330**, recorded in `Standard/docs/06-deploy.md` |
| Gateway (YARP :8080) | Routes `kpi-bare` (`/kpi`) and `kpi-all` (`/kpi/{**catch-all}`) to cluster `kpi` → `http://localhost:5230/` |
| Path base | `App:PathBase = /kpi`; `base: '/kpi/'` in Vite; router `basename="/kpi"` |
| App Pool | `STI.Kpi`: No Managed Code, AlwaysRunning, recycles at **04:30** (after the DW ETL) |
| Server folders | `D:\STI\apps\kpi\{releases, current, shared\{config, logs, data\uploads}}` |
| One site | The backend serves the frontend from `wwwroot`: one IIS site, no CORS, a single deploy |

### Configuration (never in the repository)
- **KPI side**, `shared\config\appsettings.Production.json`: `Auth:Mode = sso`, `Auth:Sso:Issuer` (STI Main's `SSO_ISSUER`), `Auth:Sso:BaseUrl`, `Auth:Sso:ClientId`, `Auth:Sso:ClientSecret`, and KPI's own `Auth:Jwt:Key`.
- **STI Main side**, one entry in `config/sso-clients.json` on STI Main's backend config volume, then restart STI Main:
  ```jsonc
  { "clientId": "sti-kpi", "name": "STI KPI", "secret": "<≥ 32 chars, out of band>",
    "redirectUris": ["https://sti-apps:8080/kpi/sso/callback"],
    "signoutUrl": "https://sti-apps:8080/kpi/api/auth/signed-out",
    "redirectUriOptional": false }
  ```
  Both URLs must be **absolute** and match KPI exactly; the host shown is a placeholder until the production hostname is confirmed. STI Main endpoints sit under `/api/` (see [auth-security.md](auth-security.md)).

### Network
- KPI (IIS) calls STI Main (Linux Docker host) over HTTPS for the code exchange, JWKS and userinfo. STI Main calls KPI's sign-out URL. Both hops cross hosts, so check the firewall and certificate trust.
- **TLS on the gateway is required before go-live.**

### Build and release
- `npm run build` (`tsc -b` clean), copy into `wwwroot`, then `dotnet publish -c Release` with 0 errors and 0 warnings. **Never build on the server.**

### Non-functional requirements
| Area | Requirement |
|---|---|
| Users | About 20 active KPI owners and Council members; all staff can read the board |
| Volume | About 300 KPIs × 12 months a year; no caching layer |
| Response | Board and entry grid < 1 s on the intranet |
| Offline intranet | No CDN at runtime: fonts, icons and charts bundled |
| Email reminders | SMTP relay (to confirm with IT); TH + EN text |
| Logging | Serilog, 30-day files under `shared\logs`; never log tokens or secrets |
| Backup | `sti_kpi` nightly, plus `shared\config` and `shared\data\uploads` |
| Health | `GET /api/health` (anonymous): `database`, `dw`, `sso` |

### Roadmap
| Phase | When | Scope |
|---|---|---|
| 0 · Define | Nov–Dec 2026 | Roles, scoring and review forms agreed; standard KPI Excel template as fallback |
| 1 · MVP | Jan–Mar 2027 | M1, M2, M3, M7, basic M8. All team KPIs in the app by end of Q1 |
| 2 · Reviews | Apr–Jun 2027 | M4 assessment, M5 quarterly review. Q2 review in the app |
| 3 · Year end | Jul–Dec 2027 | M6, change requests, reminders, ERP feed, exports |
| 4 · Individuals | 2028 | M9, privacy, HR calibration. Pilot one division in H1, all staff in H2 |

## Open items
- [ ] IT: register `sti-kpi` in STI Main and share `SSO_ISSUER` and the production hostname
- [ ] IT: open HTTPS both ways between the KPI server and STI Main; confirm certificate trust
- [ ] IT: enable TLS on the gateway (blocks go-live)
- [ ] IT: provide an SMTP relay for reminders
- [ ] Decision requested: confirm Tier 1 and a standalone app at `/kpi`
- [ ] Decision requested: adopt the standard's stack in place of the plan's technology section
- [ ] Decision requested: allocate ports 5230 / 5330 and the `/kpi` gateway route
- [ ] Business: who builds it, and who owns the process

## Log

### 2026-11-12: Design baseline recorded from TDD v0.5
- **What:** Recorded the deployment design: hosting, gateway, config on both sides, network, build, NFRs, roadmap.
- **Why:** Starting point for change tracking before Phase 1.
- **Files:** `changes/deployment-ops.md`
- **Impact:** None yet. TLS and the STI Main registration must be done before go-live.
- **Ref:** TDD v0.5 sections 2, 6, 7; [auth-security.md](auth-security.md)

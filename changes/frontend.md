# Frontend (STI.Kpi.Web) · Change log

**Covers:** React SPA: screens, navigation, TH/EN text, STI branding, accessibility.
**TDD reference:** section 2 (Solution layout), section 4 (Modules) and section 5 (UI, branding and language).

Newest log entry first. Rule and entry format: [`../CLAUDE.md`](../CLAUDE.md). The baseline below is TDD v0.5 (`Doc/STI_KPI_App_TDD.html`); keep it current as changes land.

## Design baseline (TDD v0.5)

### Stack and role
- **React 19 + Vite + TypeScript + Tailwind 4**. `npm run dev` on port **5330**, proxying `/api` to 5230.
- Owns screens, navigation, TH/EN text and instant feedback while typing. It **never** touches the database or decides permissions; route guards and hidden buttons are UX only.
- `npm run build` output goes into the backend's `wwwroot`. Set `base: '/kpi/'` in `vite.config.ts` and `basename="/kpi"` on the router.
- `src/` holds `lib/api.ts`, `stores/auth.tsx`, `i18n/index.ts`, `components/`, `pages/` and `features/scoring.ts`.

### Pages (one per prototype screen)
| Module | Page | Phase |
|---|---|---|
| M7 Company board | `BoardPage` | 1 |
| M1 KPI register | `RegisterPage`, `KpiDrawer` | 1 |
| M2 Targets and approval | `KpiDrawer` → approval tab | 1 |
| M3 Monthly results | `EntryPage` | 1 |
| Goal + Base layers | `BaseKpiPage`, layer view | 1 |
| M4/M5 Quarterly review | `ReviewPage` | 2 |
| M6 Year-end review | `YearEndPage` | 3 |
| M8 Admin | `Admin*Pages` | 1–3 |
| M9 My KPIs | `MyKpiPage` | 4 (2028) |

- Every list screen has the standard's four states: loading, error with retry, empty, data. A new search resets to page 1.
- The register, entry grid and boards show a **Base / Goal badge** and a layer filter. The KPI tree shows Goal KPIs under their strategy, and goal-linked Base KPIs with a link marker.
- The draft and approval views show the Base/Goal weight split, with an **amber notice** when it is outside range. Submit and approve still work.
- The entry grid shows target, last value and RAG. A comment is required for amber and red. ERP KPIs are read-only, but owners can comment on them.

### Sign-in behaviour (see [auth-security.md](auth-security.md))
- With no token, or on a 401, send the browser to `/kpi/sso/start?returnUrl=…`. This is silent while the STI Main session is alive.
- After the callback, call `POST /api/auth/session` and send the app token as `Authorization: Bearer`, as the standard's `lib/api.ts` does. No token ever appears in a URL.
- On a 403, show the bilingual message from the API.
- Sign-out sends the user on to STI Main to end that session too.
- The local admin login page `/kpi/admin/login` is not linked from the staff navigation.

### Branding and language
- Brand tokens only, in Tailwind `@theme` (`index.css`): Navy `#1B2D5B`, Tech Cyan `#00A3E0`, Precision Green `#00A878`, Precision Silver `#8C8C8C`.
- Fonts: Rajdhani (headings), Open Sans (EN body), Sarabun (Thai), bundled with `@fontsource`. Bootstrap Icons bundled. **No CDN at runtime.**
- Logo: STI v2 plus the app's own **STI K** mark (`Template/Branding/k/`). No STI-X or Resin marks inside the app.
- **Thai is the default.** Every string is a TH/EN pair in `i18n/index.ts`.
- Every value column uses the `.num` class (tabular figures).
- Charts are small in-house SVG components (the prototype's `chart()`, ported). No charting library unless a screen needs one.
- Shared components come from the template: `KpiCard`, `StatusBadge`, `PaginationBar`. Promote the RAG dot, team score card and entry grid to `Standard/packages/sti-ui/` once a second app needs them.

### Devices and accessibility
- Office PCs and supervisor phones or tablets: layouts work at 768 px, with touch targets ≥ 44 px.
- Visible focus, `aria-label` on icon buttons, `role="alert"` on errors (standard section 05).
- Board and entry grid respond in under 1 s on the intranet.

### Porting from the prototype (`Doc/STI_KPI_App_Prototype.html`)
1. Port the prototype's strings into `i18n/index.ts`. They are already TH/EN pairs.
2. Port its CSS tokens into Tailwind `@theme`, and build pages from the template's shared components.
3. Port `ach`, `scoreOf`, `periodVal` and `teamScore` to `features/scoring.ts`, using the prototype's sample data as the first test cases.
4. Replace the sample `KPIS`, `UNITS` and `STRATS` with API calls.

The prototype is vanilla JS in one file. It is the reference for screens and wording, not code to migrate.

## Open items
- [ ] Assignment picker: no staff search is available to KPI yet, so admins may have to type usernames (see [auth-security.md](auth-security.md))

## Log

### 2026-11-12: Design baseline recorded from TDD v0.5
- **What:** Recorded the frontend design: stack, pages, sign-in behaviour, branding, accessibility, porting steps.
- **Why:** Starting point for change tracking before Phase 1.
- **Files:** `changes/frontend.md`
- **Impact:** None; no code yet.
- **Ref:** TDD v0.5 sections 4–5; [design-docs.md](design-docs.md)

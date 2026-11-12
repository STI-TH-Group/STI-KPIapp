# Design & documentation · Change log

**Covers:** TDD, prototype, README, CLAUDE.md, the change logs and other project documents.

Newest log entry first. Rule and entry format: [`../CLAUDE.md`](../CLAUDE.md).

## Design baseline

### Documents
| Document | Version | Status |
|---|---|---|
| `Doc/STI_KPI_App_TDD.html`: technical design (EN + TH headings) | **0.5**, 12 Nov 2026 | For IT review. Governed by STI-STD-2026-001 |
| `Doc/STI_KPI_App_Prototype.html`: clickable mock-up of the nine screens | — | Design reference for screens and wording (vanilla JS, not migrated as code) |
| `changes/*.md`: per-area design baseline and change log | — | Kept current with every change |

### What the TDD covers
| TDD section | Area file |
|---|---|
| 1. Overview: scope, Tier 1, standalone app at `/kpi` | [deployment-ops.md](deployment-ops.md) |
| 2. Architecture: stack, solution layout, deployment | [backend.md](backend.md), [frontend.md](frontend.md), [deployment-ops.md](deployment-ops.md) |
| 3. Data model: tables, Base and Goal KPIs, scoring, ERP feed | [database.md](database.md), [backend.md](backend.md), [erp-integration.md](erp-integration.md) |
| 4. Modules, flows and API | [backend.md](backend.md), [frontend.md](frontend.md) |
| 5. UI, branding and language | [frontend.md](frontend.md) |
| 6. Security, access and NFRs | [auth-security.md](auth-security.md), [deployment-ops.md](deployment-ops.md) |
| 7. Roadmap, risks and open items | Open items are split across the area files |

### Scope
- **In scope:** KPI register, target approval, monthly actuals with RAG and evidence, scoring, quarterly and year-end reviews, dashboards, and the ERP feed for production KPIs (2027). Individual KPIs, self and manager assessment, HR calibration and privacy controls (2028).
- **Out of scope:** payroll and bonus calculation, a full HR system, replacing the ERP.
- The two-layer framework: **Base KPI + Goal KPI = Total KPI**, tracked monthly and reviewed quarterly and at year end.

### Sources the TDD is built from
- `STI_KPI_App_Design.md` (draft v0.1, 30 Oct 2026) and `STI_KPI_WebApp_Plan.md` (draft v0.1, 27 Oct 2026). The TDD replaces the plan's technology section.
- `STI_KPI_Framework_Summary_EN-TH.md` (two-layer KPI framework, Oct 2026)
- STI-STD-2026-001 STI Application Development Standard (27 Aug 2026), including `docs/03-auth.md`
- STI Main: `STI-TH-Group/Mainpage`, `SSO-Handshake.md` and `Backend/sso/`

## Open items
- [ ] TDD v0.6: fold in the eight STI Main differences recorded in [auth-security.md](auth-security.md)
- [ ] Business: scoring scale, RAG thresholds and the range rule
- [ ] Business: fiscal year, number of teams and owners, who builds it, process owner
- [ ] Sign-off on the six decisions requested in TDD section 7 (listed in the area files)

## Log

### 2026-11-12: Area files filled from TDD v0.5 and checked against STI Main
- **What:** Added a design baseline and an open-items checklist to every area file, from TDD v0.5. Checked the SSO design against the STI Main code and recorded the differences.
- **Why:** Each area file should show the agreed design as well as its changes, and KPI must work with STI Main as it is built.
- **Files:** `changes/*.md`, `CLAUDE.md`
- **Impact:** The TDD needs a v0.6 for the STI Main differences (see [auth-security.md](auth-security.md)).
- **Ref:** [backend.md](backend.md), [frontend.md](frontend.md), [database.md](database.md), [auth-security.md](auth-security.md), [erp-integration.md](erp-integration.md), [deployment-ops.md](deployment-ops.md)

### 2026-11-12: Change-log rule and per-area change logs
- **What:** Added `CLAUDE.md` with the standing rule to record every change in `changes/<area>.md`, and created one log per area of work.
- **Why:** To keep one traceable record of changes per area from the start of the project.
- **Files:** `CLAUDE.md`, `changes/*.md`
- **Impact:** Every later commit or PR must add an entry to the matching area file.
- **Ref:** Branch `docs/design-and-prototype`

### 2026-11-12: Technical design v0.5 and prototype added
- **What:** Added the technical design document (version 0.5, for IT review) and the interactive HTML prototype.
- **Why:** Baseline design for IT review before Phase 1 (MVP, Jan–Mar 2027).
- **Files:** `Doc/STI_KPI_App_TDD.html`, `Doc/STI_KPI_App_Prototype.html`, `README.md`
- **Impact:** The decisions requested in TDD section 7 need sign-off before build starts.
- **Ref:** Branch `docs/design-and-prototype`; TDD sections 1–7

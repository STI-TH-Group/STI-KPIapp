# ERP integration · Change log

**Covers:** Read-only feeds from the Resin data warehouse and the `pps2` fallback; `ErpImportJob`.
**TDD reference:** section 3 (ERP and data-warehouse feed) and section 7 (risks, open items).

Newest log entry first. Rule and entry format: [`../CLAUDE.md`](../CLAUDE.md). The baseline below is TDD v0.5 (`Doc/STI_KPI_App_TDD.html`); keep it current as changes land.

## Design baseline (TDD v0.5)

### Scope
- Production KPIs have `source = erp` and are **never typed in**: OEE, %NG, breakdown downtime, on-time delivery and first-lot pass.
- KPI reads the ERP and **never writes to it**. The feed is planned for Phase 3 (Jul–Dec 2027).

### Sources
| Order | Source | Access |
|---|---|---|
| Preferred | **Resin DW** (MySQL 8, nightly ETL 02:30–04:00) | Read-only account. The DW already cleans VARCHAR dates, Buddhist-era years and mixed collations |
| Fallback until the DW is live | **`pps2`** | SELECT-only queries through the read-only ERP connection, using `ErpDate` and exclusive upper bounds. **No views or other objects are ever created in `pps2`** |

### Import job
- `ErpImportJob` runs on the **1st working day at 08:00** for the month just closed.
- It writes `kpi_actual` rows, with RAG and achievement computed at save time, and logs one `erp_import_run` (`started_at`, `finished_at`, `period_id`, `rows`, `status`, `message`), which Admin shows.
- Each ERP KPI's query is named by `kpi.source_ref` and lives in code, in `ErpKpiQueries.cs`, parameterised. **No SQL is stored in the database.**
- `GET /api/health` reports `dw` status.
- An owner can **comment** on an ERP value but cannot overwrite it. A wrong value is fixed at the source and re-imported.
- The `STI.Kpi` App Pool recycles at 04:30, after the DW ETL.

### Known limits and risks
- Hourly production and downtime data are complete only from **Apr–May 2026**, so earlier OEE baselines are not comparable.
- The Resin DW go-live date is unknown. Until then, the feed uses the SELECT-only `pps2` fallback.

## Open items
- [ ] IT: confirm the Resin DW go-live date
- [ ] Define the query for each ERP KPI (`source_ref` → `ErpKpiQueries.cs`)
- [ ] Decision requested: the ERP feed reads the Resin DW, with `pps2` SELECT-only as the fallback

## Log

### 2026-11-12: Design baseline recorded from TDD v0.5
- **What:** Recorded the ERP feed design: sources, import job, rules, known limits.
- **Why:** Starting point for change tracking. The feed is planned for Phase 3 (Jul–Dec 2027).
- **Files:** `changes/erp-integration.md`
- **Impact:** None; no code yet.
- **Ref:** TDD v0.5 section 3; [backend.md](backend.md), [database.md](database.md)

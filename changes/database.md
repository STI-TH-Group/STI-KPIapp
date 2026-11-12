# Database (sti_kpi) · Change log

**Covers:** MySQL 8 schema `sti_kpi`, numbered migrations, seed and reference data, backup.
**TDD reference:** section 3 (Data model).

Newest log entry first. Rule and entry format: [`../CLAUDE.md`](../CLAUDE.md). The baseline below is TDD v0.5 (`Doc/STI_KPI_App_TDD.html`); keep it current as changes land.

## Design baseline (TDD v0.5)

### Conventions (`Standard/docs/04-mysql.md`)
- MySQL 8, schema **`sti_kpi`**, `utf8mb4_unicode_ci`. KPI owns its database. **ERP schemas are never written to.**
- Tables are `snake_case` and singular, use real `DATETIME`, and have `id BIGINT AUTO_INCREMENT`.
- Every business table has `created_at`, `updated_at`, `created_by`, `updated_by` (AD username) and `is_active` (soft delete).
- User-facing names come in pairs: `name` + `name_th`.
- Migrations are numbered SQL files in `Database/Migrations/00N_*.sql`: `001_core.sql` from the template, then `003_kpi_org.sql`, `004_kpi_register.sql`, and so on.
- Usernames (owners, entry staff, reviewers) are AD logins, **not** foreign keys.

### Tables
| Group | Table | Key columns / notes |
|---|---|---|
| Org, time | `org_unit` | `code` UNIQUE (e.g. INJ), `name`, `name_th`, `parent_id`, `level` (company / division / team) |
| | `org_unit_role` | `org_unit_id`, `username`, `role` (owner / reviewer / member) |
| | `cycle` | `year`, `status` (setting / open / closed / locked); one per year |
| | `period` | `cycle_id`, `kind` (month / quarter / year), `seq`, `starts_on`, `entry_due_on` (5th working day), `status`; locked periods reject writes |
| | `strategy` | `code` (G, S1–S4), `name`, `name_th`, `cycle_id` |
| KPI definition | `kpi` | `cycle_id`, `code` (UNIQUE per cycle), `org_unit_id`, `definition`, `formula`, `unit`, `direction` (up / down / range / pass), `aggregation` (avg / last / sum), `frequency`, `source` (manual / erp / excel), `source_ref`, `status`, `layer` (base / goal), `parent_kpi_id`, `strategy_id`, `base_kpi_code` |
| | `kpi_target` | `kpi_id`, `version`, `baseline`, `target`, `target_low`/`target_high`, `weight`, `milestones_json`, `approved_by`, `approved_at`, `change_request_id`. **A new row per change**; the original is never overwritten |
| | `kpi_assignment` | `kpi_id`, `username`, `role` (entry) |
| | `scoring_rule` | `cycle_id`, `green_min`, `amber_min`, `cap`, `bands_json`, `layer_split_json` |
| Results | `kpi_actual` | `kpi_id`, `period_id`, `value` DECIMAL(18,4), `is_pass`, `rag`, `achievement`, `comment`, `source`, `entered_by`, `entered_at`. **UNIQUE (kpi_id, period_id)**. RAG and achievement are stored at save time |
| | `attachment` | `owner_type` (actual / review), `owner_id`, `file_name`, `stored_path`, `content_type`, `size_bytes`, `sha256`. Files live on disk, never in the DB, and the owner link is not enforced by a foreign key |
| | `assessment` | `kpi_id`, `period_id`, `kind` (system / self / reviewer), `score` 1–5, `comment`. UNIQUE (kpi_id, period_id) |
| Review | `review` | `org_unit_id`, `period_id`, `stage` (draft / sent / signed), `highlights`, `issues`, `attendees`, owner/reviewer/exec signatures. One per team per quarter or year |
| | `review_action` | `review_id`, `action`, `owner_name`, `due_on`, `status` (open / done / carried) |
| | `change_request` | `kpi_id`, `from_target`, `to_target`, `reason`, `status`, `decided_in_review_id` |
| | `year_end_decision` | `kpi_id`, `cycle_id`, `decision` (keep / change / retire), `note` |
| | `erp_import_run` | `started_at`, `finished_at`, `period_id`, `rows`, `status`, `message` |
| Platform | `local_user` | From `001_core.sql`; local admin accounts only |
| | `auth_session` | `session_id` (= KPI token `jti`), `username`, `auth_type` (AD / LOCAL), `sti_sid`, `sti_jti`, `issued_at`, `expires_at`, `revoked_at`, `client_ip`. One row per sign-in. `sti_sid` and `sti_jti` hold STI Main's `sid` and `jti` claims |
| | `audit_log` | Every create, update, submit, approve and sign, and every read of `person_*`: who, what, before/after JSON, when |
| | `schema_migration` | From `001_core.sql` |
| Reference | `base_kpi` | One-off import of the 2026 inventory (`code`, `dept`, `name_th`, `rule`, `months_json`, `decision`). Read-only. Kept KPIs are copied into `kpi` as 2027 Base KPIs |
| 2028 | `person_kpi`, `person_assessment` | `username`, `kpi_id` or own definition, `share`, `weight`, scores. Privacy rules are in [auth-security.md](auth-security.md) |

### Seed and reference data
- `scoring_rule` defaults: green ≥ 100, amber ≥ 90, cap 120, bands 110/100/90/80 → 5/4/3/2/1.
- `layer_split_json` starting values (Base / Goal weight): Manager 50–60% / 40–50%, Supervisor ~70% / ~30%, Operator 80–90% / 10–20%. The division and team range is still to agree; until it is set, no warning shows.
- The 2026 base-KPI inventory workbook goes into `base_kpi`, then is seeded as 2027 Base KPIs (`layer = base`, with `base_kpi_code` pointing back).
- Strategies G and S1–S4 per cycle. Org units run company → division → team.

### Volume and backup
- About 300 KPIs × 12 months a year; no caching layer is needed.
- `sti_kpi` is backed up nightly, together with `shared\config` and `shared\data\uploads`.

## Open items
- [ ] Which 2026 base KPIs are kept for 2027 (drives the `base_kpi` seed)
- [ ] Base/Goal weight range for division and team sets (`layer_split_json`)
- [ ] Fiscal year, number of teams and owners (drives the `cycle`, `period` and `org_unit` seed)

## Log

### 2026-11-12: Design baseline recorded from TDD v0.5
- **What:** Recorded the data model: conventions, all tables, seed data, volume and backup.
- **Why:** Starting point for change tracking before the first migration.
- **Files:** `changes/database.md`
- **Impact:** None; no migrations yet.
- **Ref:** TDD v0.5 section 3; [design-docs.md](design-docs.md)

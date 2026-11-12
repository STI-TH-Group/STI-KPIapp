# Auth & security · Change log

**Covers:** sign-in through STI Main (SSO), local admin login, app tokens and sessions, roles, authorisation, privacy, audit, transport and secrets.
**TDD reference:** section 6 (Security, access and NFRs) and section 7 (risks).
**Integration partner:** STI Main, repo `STI-TH-Group/Mainpage` (`SSO-Handshake.md`, `Backend/sso/`).

Newest log entry first. Rule and entry format: [`../CLAUDE.md`](../CLAUDE.md). The baseline below is TDD v0.5 (`Doc/STI_KPI_App_TDD.html`), checked against the STI Main code. Keep it current as changes land.

## Design baseline

### Principles
- Staff sign in **only through STI Main** with their AD account. KPI has no staff login form, never sees an AD password and never talks to Active Directory.
- KPI's own login page is for **local admin accounts only**.
- STI Main passes the user's **profile**; it does not decide anyone's rights in KPI. KPI maps the profile (AD groups) to its own roles in its own database.
- KPI stores only per-KPI and per-unit assignments, keyed by username. It keeps no second user list.
- **Every permission is enforced on the server.**
- KPI has its own App Pool and its own JWT signing key, so a leak elsewhere cannot mint a token that reads 2028 individual results.

### STI Main SSO contract (as implemented in Mainpage)
| Item | Value |
|---|---|
| Client id | `sti-kpi`, registered in STI Main's `config/sso-clients.json` (on the backend config volume, not in git) |
| Registration | `{ clientId, name, secret (≥ 32 chars), redirectUris: [absolute URL], signoutUrl: absolute URL, redirectUriOptional: false }`. STI Main restarts to load it. The old `permissions` field is no longer used |
| JWKS | `GET /api/.well-known/jwks.json` (anonymous). All current public keys, newest first |
| Authorize | `GET /api/sso/authorize?client_id=sti-kpi&redirect_uri=…&state=…`. `state` must match `^[\w.~-]{8,256}$`. No prompt if the user already has an STI Main session |
| Code | One-time, **60 s**, held in STI Main's memory, bound to the client and `redirect_uri` |
| Exchange | `POST /api/sso/exchange`, server-to-server, client secret by **Basic auth** (or in the body). **`redirectUri` is required** and must equal the one used at authorize. Errors: `401 invalid_client`, `400 invalid_code` (a reused code also returns 400) |
| Token | **RS256**, `kid` in header, `iss` = STI Main's `SSO_ISSUER`, `aud = sti-kpi`, `sub` = username, `jti`. Life = **min(8 h, end of the user's STI Main sign-in)** |
| Claims | `username`, `full_name`, `ou`, `ous`, `groups` (CN only), `auth_type`, plus `group_details: [{ name, dn, sid }]`, `sid` (STI Main sign-in id), and `dept`/`role` (**STI Main's own, for reference only; never used for KPI authorisation**) |
| Userinfo | `GET /api/sso/userinfo` with `Authorization: Bearer <STI Main token>`. `401 signed_out` after sign-out, `401 invalid_token` after expiry, `503 permissions_unavailable` |
| Sign-out ping | STI Main calls KPI's `signoutUrl` with `POST { username, jti, signedOutAt }` and Basic `sti-kpi:<secret>`. Fire-and-forget (5 s timeout, no retry), **only on an explicit sign-out** (not idle expiry or the 8 h cap). The list of tokens to notify survives an STI Main restart (MySQL `sso_issued_tokens`) |
| Profile failure | If the LDAPS profile read fails, STI Main does not sign the user in (`503 profile_unavailable`), so KPI never receives an empty `groups` caused by a directory failure |
| Key rotation | New key added to JWKS beside the old one; the old one is removed after 8 h + 10 min. KPI refetches JWKS when it sees an unknown `kid` |

### Staff sign-in flow (KPI side)
1. The SPA has no token (or gets a 401) and goes to `/kpi/sso/start?returnUrl=…`. KPI stores a random `state` and the return URL in a short-lived HttpOnly cookie, then redirects to STI Main's authorize URL with the registered `redirect_uri`.
2. STI Main signs the user in if needed (Kerberos, then LDAPS profile) and redirects to `/kpi/sso/callback?code&state`.
3. The callback checks `state`, then calls the exchange with the client secret and the same `redirectUri`.
4. KPI validates the token against the cached JWKS (10 min cache, refetched on an unknown `kid`): `alg` must be RS256, `iss` = configured issuer, `aud = sti-kpi`, `exp` with 2 min skew. Anything else is refused and logged.
5. KPI writes an `auth_session` row (`sti_sid` = `sid`, `sti_jti` = `jti`) and mints its own **HS256 app token** with `Auth:Jwt:Key`: `username`, `full_name`, `ou`, `ous`, `groups`, `auth_type = AD`. It expires no later than the STI Main token.
6. Handoff: a single-use, 60 s HttpOnly cookie, then `POST /api/auth/session` returns the app token in the body; the SPA sends it as `Authorization: Bearer`. **No token ever appears in a URL**, and the STI Main token never reaches the browser.

KPI endpoints: `GET /sso/start`, `GET /sso/callback`, `POST /api/auth/session`, `GET /api/auth/me`, `POST /api/auth/signed-out` (STI Main only), `POST /api/auth/local-login` (local admin only), `POST /api/auth/logout`.

### Sign-out and session checks
- **Ping:** `POST /kpi/api/auth/signed-out` sets `revoked_at` on the matching `auth_session`; the next request with that token gets 401.
- **Userinfo is the authority:** the ping fires only on an explicit sign-out and is never retried, so KPI calls `/api/sso/userinfo` with the stored STI Main token at most every 15 minutes per active session. `401 signed_out` or `401 invalid_token` revokes the KPI session.
- **Sign-out in KPI** revokes the KPI session, then sends the browser to STI Main so the user can end that session too.
- **Revocation check:** every request checks the token's `jti` against `auth_session` (cached in memory for 60 s).
- **Groups are read at sign-in:** AD group changes take effect at the next STI Main sign-in.

### Roles
| Role | Source | Permissions |
|---|---|---|
| Executive (MD) | AD group `STI_KPI_Executive` | Read all; approve company and strategy KPIs; executive sign-off |
| Strategy Council | AD group `STI_KPI_Council` | Read all; approve division KPIs; run and sign quarterly reviews |
| HR / KPI admin | AD group `STI_KPI_Admin` | Cycles, periods, org units, rules, assignments, audit log |
| KPI owner | `org_unit_role` | Draft, submit, self-assess, sign as owner for their units |
| Data entry | `kpi_assignment` | Enter actuals and evidence for assigned KPIs |
| Reviewer | `org_unit_role` | Assess and approve for the units they review |
| All staff | Signed in through STI Main | Read the company board and their own unit |
| Local admin | `local_user` (`auth_type = LOCAL`) | Admin module and health only |

- Map the three AD groups by **`group_details[].dn` or `.sid`**, not by CN alone: a group with the same CN in another OU would otherwise match.
- Approval routes: team KPI → division head; division KPI → Council; strategy KPI → Executive.
- Admin lists assignments whose username has not signed in for 90 days (from `auth_session`) so HR can check them.

### Local admin login (`/kpi/admin/login`)
- For setup before SSO is live, and for IT access when STI Main or AD is down. Not linked from staff navigation.
- `POST /api/auth/local-login` checks `local_user` only. It never binds to AD and never accepts AD credentials.
- At most two accounts (e.g. `kpi.admin`, held by IT), **PBKDF2-SHA256, 210,000 iterations**, per-user salt, rotated when staff change.
- Token: `auth_type = LOCAL`, role `LocalAdmin`, valid **2 hours**. It opens Admin and health only. It cannot enter results, approve, assess, sign or read `person_*`; those policies require `auth_type = AD`.
- Rate limit of 5 attempts per 15 min per IP, then a 15 min lock. Every attempt is audited. Optional IT-subnet allow-list.

### Authorisation and data protection
- Policies in `KpiPolicies.cs`: `CanApprove`, `CanEnter(kpiId)`, `CanReview(unitId)`, `IsAdmin`. React route guards are UX only.
- 401 (missing, expired or revoked token) → silent SSO restart. 403 → bilingual message, e.g. "คุณไม่มีสิทธิ์บันทึกผล KPI ของทีมนี้ / You are not assigned to enter results for this team."
- **2028 privacy:** `person_*` rows are readable only by the person, their reviewer line, `STI_KPI_Admin` and `STI_KPI_Executive`. Scope is filtered **in SQL**, not after loading. Reads are audited as well as writes.
- Uploads: ≤ 10 MB, pdf/jpg/png/xlsx, stored on disk with a SHA-256 hash.
- Audit: all writes, every sign-in attempt and every read of individual data go to `audit_log`.
- Transport: HTTPS end to end; **no go-live without TLS on the gateway**.
- Secrets: the SSO client secret and `Auth:Jwt:Key` live only in `shared\config`, never in the repo. Tokens and secrets are never logged; logs carry `username` and `jti` only.

### Risks
| Risk | Response |
|---|---|
| STI Main down: staff cannot sign in | Existing sessions run to expiry; local admin still works; health reports `sso` |
| SSO not yet tested end to end | Develop with `Auth:Mode = local`; run the seven acceptance checks in `SSO-Handshake.md` §10 against `sti-kpi` early in Phase 1 |
| Missed sign-out (ping is fire-and-forget, explicit sign-out only) | Userinfo check every 15 min; token life capped at 8 h |
| AD group name clash across OUs | Map roles by DN or SID from `group_details` |
| Privacy of individual results (2028) | SQL-level scope filters, read auditing; HR owns the rules before M9 starts |

## Differences found between TDD v0.5 and STI Main (to fold into the next TDD version)
1. **Group mapping:** the TDD maps roles by AD group name. STI Main says to map on `dn` or `sid` from `group_details`.
2. **Per-app permissions:** the TDD's open question "STI Main per-app permissions or AD groups?" is settled. STI Main has no per-app permission list (its `permissions` field was retired on 23 Oct 2026), so KPI uses AD groups.
3. **Sign-out ping:** the TDD says pings can be lost when STI Main restarts. That is now fixed: the notify list is stored in MySQL. Pings are still fire-and-forget and are sent only on an explicit sign-out, so the 15-minute userinfo check stays.
4. **Exchange request:** `redirectUri` is required in the exchange call. The TDD does not mention it.
5. **Paths:** STI Main serves SSO under `/api/` (`/api/sso/authorize`, `/api/sso/exchange`, `/api/sso/userinfo`, `/api/.well-known/jwks.json`). The handshake draft's `/portal/...` paths are not the implementation.
6. **Registration:** `redirectUris` and `signoutUrl` must be **absolute** http(s) URLs. The TDD's `signoutUrl = /kpi/api/auth/signed-out` must be written as a full URL.
7. **Token life:** the token lasts the shorter of 8 h and the rest of the user's STI Main sign-in, not a flat 8 h.
8. **Staff search:** STI Main has `POST /api/ad/users/search`, but only for its own admin pages under STI Main's permissions. KPI cannot call it, so the picker question stays open.

## Open items
- [ ] IT: register `sti-kpi` in STI Main's `sso-clients.json` (secret, absolute callback URL, absolute sign-out URL) and share `SSO_ISSUER`
- [ ] IT: create AD groups `STI_KPI_Executive`, `STI_KPI_Council`, `STI_KPI_Admin`, and give KPI their DNs or SIDs
- [ ] Username lookup for the assignment picker: ask STI Main for a staff search endpoint that SSO clients may call, or have admins type usernames
- [ ] Run the seven SSO acceptance checks against `sti-kpi`
- [ ] Update the TDD (section 6) with the differences above
- [ ] Link to pay (changes the approval and privacy rules for 2028)
- [ ] Decision requested: staff sign in only through STI Main; the login page is for local admins only
- [ ] Decision requested: approve the AD groups and the assignment model
- [x] ~~AD groups or STI Main per-app permissions?~~ Settled: AD groups (STI Main has no per-app permissions)

## Log

### 2026-11-12: Baseline checked against STI Main code
- **What:** Checked the SSO design against `Mainpage` (`SSO-Handshake.md` §14 and `Backend/sso/`, commit `d3148b5`). Recorded the contract as implemented and eight differences from TDD v0.5.
- **Why:** KPI must talk to STI Main as STI Main actually behaves, not only as the handshake draft describes.
- **Files:** `changes/auth-security.md`
- **Impact:** The TDD needs a section 6 update. Role mapping must use group DN or SID. The registration needs absolute URLs. The per-app permission question is closed.
- **Ref:** [deployment-ops.md](deployment-ops.md), [design-docs.md](design-docs.md)

### 2026-11-12: Design baseline recorded from TDD v0.5
- **What:** Recorded the auth and security design: SSO flow, sessions, local admin, roles, policies, privacy, secrets, risks.
- **Why:** Starting point for change tracking before Phase 1.
- **Files:** `changes/auth-security.md`
- **Impact:** None; no code yet.
- **Ref:** TDD v0.5 sections 6–7

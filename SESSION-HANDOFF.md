# SESSION HANDOFF

Date: 2026-02-22

## Current state (authoritative)

- Deployment workflow is now centered on root command runner: `~/apps/Website/aera`.
- Docs are updated and aligned with the new command flow:
    - `DEPLOY-CHEATSHEET.md` (primary runbook)
    - `README.md` (full guide)
- Production is backend-first with explicit health gating before client startup.
- SMTP is production SMTP-only (Mailhog removed from production surfaces).
- Edge maintenance mode is implemented (clean 503 maintenance page + toggle commands).

## Key changes completed in this session

### 1) Unified root CLI (`./aera`)

Implemented/available commands include:

- Deploy/core: `doctor`, `network:ensure`, `backend:up`, `backend:wait`, `backend:retry`, `client:up`, `edge:up`, `cert:init`, `edge:restart`, `deploy:all`
- Visibility: `status`, `logs:backend`, `logs:client`, `logs:edge`
- SMTP ops: `smtp:configure`, `smtp:show`, `smtp:test`
- Maintenance ops: `maintenance:on`, `maintenance:off`, `maintenance:status`

Important behavior:

- `smtp:configure` updates backend `.env.production` keys and by default recreates only `REPLACE_WITH_API_SERVICE` + `REPLACE_WITH_CONTAINER_OR_SERVICE_NAME`.
- Optional `--worker-only` exists for minimal-impact mail path updates.

### 2) Production SMTP hardening + dynamic provider switching

- Production Mailhog references removed from:
    - `backend/docker-compose.production.yml`
    - `backend/.env.production`
    - `backend/.env.production.example`
- Added `EMAIL_SECURE` support to SMTP runtime:
    - `backend/api/src/utils/emailTransporter.util.ts`
    - `backend/docker-compose.production.yml` env passthrough
    - `backend/.env.production` + `.env.production.example`

Result:

- Switching providers (cPanel ↔ M365 etc.) is now command-based, no code edit required.

### 3) Maintenance mode (edge)

- Nginx maintenance toggle via file flag: `/etc/nginx/maintenance/enabled`
- Nginx serves controlled 503 page during maintenance:
    - `proxy/prod/nginx/prod.conf.template`
- Static maintenance assets folder mounted into edge container:
    - `proxy/prod/docker-compose.edge.prod.yml`
    - `proxy/prod/nginx/maintenance/*`
- Maintenance page now includes branded design + logo:
    - `proxy/prod/nginx/maintenance/maintenance.html`
    - `proxy/prod/nginx/maintenance/maintenance-logo.png`

Important:

- Logo route standardized as `/maintenance-logo.png` (works with configured Nginx location).

## Assumed VPS structure

```text
~/apps/Website/
  aera
  backend/
  client/
  proxy/
```

---

## Update: Full stack recovery + test-domain go-live (2026-02-25)

### Completed in this session

- Backend authentication/startup blocker resolved; backend is stable/healthy.
- Client SSR crash fixed and deployed:
    - Homepage/public payload null-guard hardening in client.
    - Pulled on VPS and rebuilt client image from latest source.
- Edge/TLS brought online successfully for test domain:
    - Bootstrap cert path handled.
    - Let’s Encrypt certificate successfully issued for `REPLACE_WITH_SECONDARY_DOMAIN`.
    - HTTPS serving correctly (`301` http→https, `200` over https).
- Proxy certbot sidecar stability fixes were applied in proxy compose and deployed.

### Notable code/deploy changes applied

- Client repo:
    - `5542378` — SSR/public page payload guard hardening (prevents `heroSection` undefined crash).
- Proxy repo:
    - Certbot service command/entrypoint fixes in `prod/docker-compose.edge.prod.yml` so renewal loop runs under shell correctly.

### Current runtime state (authoritative)

- `backend`: healthy (`mongo`, `redis`, `rabbitmq`, `api`, `worker`).
- `client`: healthy (`REPLACE_WITH_CONTAINER_OR_SERVICE_NAME`).
- `edge`: healthy (`REPLACE_WITH_EDGE_CONTAINER`) and serving TLS on `REPLACE_WITH_SECONDARY_DOMAIN`.
- `certbot`: configured and running with renewal loop behavior corrected.

### Current domain mode (important)

- Active primary/testing domain for this VPS: `REPLACE_WITH_SECONDARY_DOMAIN`.
- `proxy/prod/.env.edge.prod` is currently set for test-domain-first operation.
- `REPLACE_WITH_PRIMARY_DOMAIN` cutover is deferred until content/admin setup is complete.

### Resume here next session (backend seeding + content ops)

Run from `~/apps/Website`:

```bash
./aera status

# Seed default division + admin user (set strong values)
ADMIN_PASSWORD='<STRONG_PASSWORD>' \
ADMIN_EMAIL='REPLACE_WITH_EMAIL_ADDRESS' \
ADMIN_FIRSTNAME='System' \
ADMIN_LASTNAME='Admin' \
./aera content:seed:admin

# Clear backend content cache
./aera content:cache:clear
```

Optional verification after seeding:

```bash
./aera logs:backend
curl -Ik https://REPLACE_WITH_SECONDARY_DOMAIN
```

### Planned later cutover (no migration)

When ready to switch primary domain to `REPLACE_WITH_PRIMARY_DOMAIN`:

- Update `proxy/prod/.env.edge.prod`:
    - `PRIMARY_DOMAIN=REPLACE_WITH_PRIMARY_DOMAIN`
    - `SECONDARY_DOMAIN=REPLACE_WITH_SECONDARY_DOMAIN`
    - `TLS_CERT_DOMAIN=REPLACE_WITH_PRIMARY_DOMAIN`
- Then run:

```bash
cd ~/apps/Website
EMAIL=<ops_email> ./aera cert:init
./aera edge:restart
./aera status
```

## Recommended deployment flow now (VPS)

From `~/apps/Website`:

```bash
./aera doctor
./aera backend:up
./aera backend:wait
./aera client:up
./aera edge:up
EMAIL=<YOUR_EMAIL> ./aera cert:init
./aera edge:restart
./aera status
```

If backend health wait times out:

```bash
./aera backend:retry
./aera client:up
```

## SMTP runbook (VPS)

### Configure (example cPanel)

```bash
./aera smtp:configure --provider=cpanel --host=<smtp_host> --port=<465_or_587> --secure=<true_or_false> --user=<mailbox> --password='<password>' --from='"Atomic Energy Regulatory Authority" <REPLACE_WITH_EMAIL_ADDRESS>'
./aera smtp:show
./aera smtp:test --to=<recipient>
```

### Switch provider later (example M365)

```bash
./aera smtp:configure --provider=m365 --user=<mailbox> --password='<password>' --from='"Atomic Energy Regulatory Authority" <REPLACE_WITH_EMAIL_ADDRESS>'
./aera smtp:test --to=<recipient>
```

## Maintenance runbook (VPS)

Before planned disruptive work:

```bash
./aera maintenance:on
./aera maintenance:status
```

After work completes:

```bash
./aera maintenance:off
./aera maintenance:status
```

## Domain/TLS notes

- Dual-domain edge still supported via:
    - `PRIMARY_DOMAIN`
    - `SECONDARY_DOMAIN`
    - `TLS_CERT_DOMAIN`
- First-time cert caveat remains valid:
    - If real certs are absent, bootstrap temporary self-signed certs in cert volume, then run `make prod-cert-init`.

## Resume checklist for next session

- Confirm DNS for active domain(s) resolves to VPS public IP.
- Confirm firewall allows `22/80/443`.
- Confirm env files exist and are populated:
    - `backend/.env.production`
    - `client/.env.production`
    - `proxy/prod/.env.edge.prod`
- Confirm root CLI exists/executable:
    - `~/apps/Website/aera`
    - optional symlink: `/usr/local/bin/aera`
- If validating maintenance UI, ensure logo path is `/maintenance-logo.png` and file exists in `proxy/prod/nginx/maintenance/`.

## Quick sanity commands

```bash
cd ~/apps/Website
./aera doctor
./aera status
./aera maintenance:status
./aera smtp:show
```

---

## Update: Multi-repo split + Ops repo (2026-02-22)

### What changed

- Project is now split across GitHub repos:
    - `Aera-Website-API` (backend)
    - `Aera-Website-Client` (client)
    - `Aera-Website-Proxy` (proxy)
    - `Aera-Website-Ops` (runbooks + `aera` orchestrator)
- Operational docs moved to Ops repo and updated:
    - `DEPLOY-CHEATSHEET.md`
    - `README.md`
    - `SESSION-HANDOFF.md`

### Root command strategy in production

- Canonical CLI file is now managed in Ops repo: `Aera-Website-Ops/aera`.
- VPS root launcher should be bootstrapped/updated by downloading single file into `~/apps/Website/aera`.
- This preserves Laravel-style command UX from workspace root (`./aera ...`) while keeping source of truth in Ops repo.

Bootstrap/update command:

```bash
cd ~/apps/Website
curl -fsSL https://raw.githubusercontent.com/Lusekero/Aera-Website-Ops/main/aera -o ./aera
chmod +x ./aera
```

Optional global symlink:

```bash
sudo ln -sf ~/apps/Website/aera /usr/local/bin/aera
```

### Environment-file security hardening

Audit result across repos:

- Backend had tracked real `.env.production` (contains secrets) — must remain untracked.
- Proxy had tracked real env files (`dev/.env.edge.dev`, `prod/.env.edge.prod`, `prod/.env.edge.prod.local`) — now intended to remain untracked.
- Client uses `.env.example` pattern (safe baseline).

Policy moving forward:

- Commit only `*.example` env templates.
- Keep real env files local/VPS only.
- Rotate any credentials that were previously tracked.

### MongoDB password rotation guidance added to docs

- Added procedure for existing production DB after env duplication:
    - Run `changeUserPassword(...)` inside running Mongo container.
    - Update backend `MONGO_INITDB_ROOT_PASSWORD` and `MONGODB_URI` in `.env.production` via `./aera env:set`.
    - Recreate/wait backend stack.
- Clarified first-time deploy case: set env before first `backend:up` (no rotation step needed on fresh volume).

### Next-session checklist

- Ensure VPS runbooks use the single-file `aera` bootstrap from Ops GitHub.
- Confirm root `~/apps/Website/aera` is executable and matches latest Ops commit.
- Confirm no real `.env*` files are tracked in API/Proxy repos before future pushes.
- If secret exposure occurred historically, complete credential rotation:
    - Mongo/RabbitMQ passwords
    - JWT and refresh-token keys
    - Email OTP secret
    - Public API key
    - reCAPTCHA secret

---

## Update: VPS auth + deploy continuation (2026-02-23)

### Where we left off

- Local repos are clean/synced to `main` after push/merge flow:
    - API: `Aera-Website-API`
    - Client: `Aera-Website-Client` (feature branch merged into `main` and pushed)
    - Proxy: `Aera-Website-Proxy`
    - Ops: `Aera-Website-Ops`
- Requested TS prop issues under client resources pages were fixed.
- VPS prep reached cheat-sheet Step 4 (clone/update repos under `~/apps/Website`).
- Blocker identified and resolved in docs: GitHub no longer accepts account password for git over HTTPS; use SSH key auth (or PAT).

### What was updated in docs this session

- `DEPLOY-CHEATSHEET.md` now includes a reusable SSH setup block under Section 4:
    - generate key
    - add key to agent
    - register GitHub host
    - add public key to GitHub
    - verify with `ssh -T git@github.com`
- Added quick commands to switch existing repo remotes from HTTPS to SSH.

### Resume commands (from VPS)

```bash
cd ~/apps/Website

# If SSH auth is not configured yet, run the new SSH block in DEPLOY-CHEATSHEET Section 4.

# Then continue Step 4 using SSH
GITHUB_OWNER="Lusekero"
BACKEND_REPO="Aera-Website-API"
CLIENT_REPO="Aera-Website-Client"
PROXY_REPO="Aera-Website-Proxy"
GIT_PROTOCOL="ssh"

# Run clone_or_update block from DEPLOY-CHEATSHEET.md
```

After repos are present, continue normal deploy flow:

```bash
cd ~/apps/Website
./aera doctor
./aera network:ensure
./aera backend:up
./aera backend:wait
./aera client:up
./aera edge:up
./aera status
```

### Notes

- If VPS was rebooted after Docker install, reconnect via SSH and continue from the resume commands above.
- If `ssh -T git@github.com` fails, fix SSH auth first before retrying clone/update.

---

## Update: MongoDB password rotation & backend stack (2026-02-23)

### Where to resume

- MongoDB password rotation is in progress or just completed.
- The following commands are the next checkpoint:

```bash
# 1. Change MongoDB root password inside the running container:
docker exec -i REPLACE_WITH_MONGO_CONTAINER mongosh \
  -u "$MONGO_INITDB_ROOT_USERNAME" \
  -p "$MONGO_INITDB_ROOT_PASSWORD" \
  --authenticationDatabase admin \
  --eval "db.getSiblingDB('admin').changeUserPassword('$MONGO_INITDB_ROOT_USERNAME', '$NEW_PASS')"

# 2. Update backend env with new password and URI:
cd ~/apps/Website
./aera env:set --target=backend --profile=prod \
  --set MONGO_INITDB_ROOT_PASSWORD="$NEW_PASS" \
  --set MONGODB_URI="mongodb://$MONGO_INITDB_ROOT_USERNAME:$NEW_PASS@REPLACE_WITH_MONGO_SERVICE:27017/$MONGO_INITDB_DATABASE"

# 3. Restart backend stack:
./aera backend:up
./aera backend:wait
./aera status
```

**Resume here next session to complete or verify MongoDB password rotation and backend stack health.**

---

## Update: VPS backend troubleshooting checkpoint (2026-02-24)

### Completed in this session

- Pushed backend `main` commits used in this troubleshooting run:
    - `d21041e` — production API healthcheck timing tuned.
    - `c4270ae` — synced local backend config updates.
    - `02fc49f` — API startup failure handling now exits non-zero and prints `[API STARTUP FAILURE] ...`.
- VPS pulled latest backend and rebuilt API/worker images.

### Findings (authoritative)

- API no longer fails silently; logs now expose cause.
- First failure seen: `Password contains unescaped characters` (URI encoding issue).
- URI encoding was corrected for `MONGODB_URI` and `RABBITMQ_URL`.
- Current blocker after encoding fix: `[API STARTUP FAILURE] Authentication failed.`
- `mongosh` check with env credentials failed (`MongoServerError: Authentication failed`).
- Mongo volume was reset and recreated (`production_mongo_data`), but API health still did not pass before pause.

### Last known runtime state

- `mongo`, `redis`, `rabbitmq` healthy.
- `REPLACE_WITH_API_CONTAINER` restart loop (`starting`/`unhealthy`).
- API logs repeatedly show `Authentication failed.`

### First commands to run next session (fresh bootstrap path)

```bash
cd ~/apps/Website

# Backup env
cp backend/.env.production backend/.env.production.bak.$(date +%F-%H%M%S)

# Full backend reset with volumes
docker compose --env-file backend/.env.production -f backend/docker-compose.production.yml down -v --remove-orphans

# Load env context
cd backend
set -a
source .env.production
set +a
cd ..

# Use simple bootstrap passwords (first successful bring-up)
NEW_MONGO_PASS='AeraMongoPass2026'
NEW_RMQ_PASS='AeraRabbitPass2026'

./aera env:set --target=backend --profile=prod   --set MONGO_INITDB_ROOT_PASSWORD="$NEW_MONGO_PASS"   --set RABBITMQ_PASSWORD="$NEW_RMQ_PASS"   --set MONGODB_URI="mongodb://${MONGO_INITDB_ROOT_USERNAME}:${NEW_MONGO_PASS}@REPLACE_WITH_MONGO_SERVICE:27017/${MONGO_INITDB_DATABASE}?authSource=admin"   --set RABBITMQ_URL="amqp://${RABBITMQ_USER}:${NEW_RMQ_PASS}@REPLACE_WITH_RABBITMQ_SERVICE:${RABBITMQ_PORT}"

./aera backend:up
./aera backend:wait
```

### If still unhealthy, collect immediately

```bash
cd ~/apps/Website
./aera status
docker compose --env-file backend/.env.production -f backend/docker-compose.production.yml logs --tail=120 REPLACE_WITH_API_SERVICE
docker inspect REPLACE_WITH_API_CONTAINER --format 'status={{.State.Status}} exit={{.State.ExitCode}} restarts={{.RestartCount}} health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}}'

cd ~/apps/Website/backend
set -a; source .env.production; set +a
docker exec -i REPLACE_WITH_MONGO_CONTAINER mongosh   -u "$MONGO_INITDB_ROOT_USERNAME"   -p "$MONGO_INITDB_ROOT_PASSWORD"   --authenticationDatabase admin   --eval 'db.adminCommand({ ping: 1 })'
```

---

## Update: Production login 403 (`MISSING_API_KEY`) triage checkpoint (2026-02-25)

### What was completed

- Confirmed backend accepts login requests when `x-api-key` is present (curl test moved past API-key layer).
- Identified real issue as frontend request path intermittently missing API key header in production flow.
- Implemented/pushed client hardening on `main` (repo: `Aera-Website-Client`):
    - Commit: `c576a94`
    - `client/nuxt.config.ts`
        - `publicApiKey` now resolves from both `NUXT_PUBLIC_API_KEY` and fallback `PUBLIC_API_KEY`.
    - `client/composables/ApiClient.ts`
        - Header key resolution hardened with fallback chain and `.trim()` before injecting `x-api-key`.
- VPS already pulled commit and restarted client container.
- Environment parity on VPS verified:
    - `backend/.env.production` `PUBLIC_API_KEY` matches
    - `client/.env.production` `NUXT_PUBLIC_API_KEY` matches
    - `docker exec REPLACE_WITH_CONTAINER_OR_SERVICE_NAME printenv` shows `NUXT_PUBLIC_API_KEY` present.
- Edge was restarted after checks.

### Important operational note (copy/paste trap)

- A recurring blocker was shell syntax errors caused by copying markdown-rendered command links into SSH, e.g. `[.env.production](...)`.
- Only run plain-text commands in terminal.

### Current suspected gap

- `./aera client:up` recreates containers but does not force image rebuild.
- If client image was built before `c576a94`, old bundle may still be serving (header fix not active).

### Resume from here (first commands next session)

Run exactly on VPS:

```bash
cd ~/apps/Website/client

ENV_FILE=".env.production"
COMPOSE_FILE="docker-compose.prod.yml"
SERVICE="nuxt-prod"

docker compose --env-file "$ENV_FILE" -f "$COMPOSE_FILE" build --no-cache "$SERVICE"

cd ~/apps/Website
./aera client:down
./aera client:up
./aera edge:restart
./aera status
```

### Verification checklist after rebuild

1. Browser (incognito / hard refresh):
    - Submit login form.
    - In DevTools Network, inspect `POST /api/v1/private-access/auth/login`.
    - Confirm request includes header `x-api-key`.
2. API logs during login attempt:

```bash
docker logs -f REPLACE_WITH_API_CONTAINER | grep -E "MISSING_API_KEY|INVALID_API_KEY|private-access/auth/login"
```

### Decision point after verification

- If `x-api-key` is present and 403 disappears, continue with normal login QA (reCAPTCHA/user credentials).
- If `MISSING_API_KEY` persists, capture one failing request’s full request headers from browser DevTools and continue from that evidence.

---

## Update: Docker Auth Session Stabilization + Remember-Me (2026-03-12)

### Scope completed

- Backend and client auth/session flow was hardened and pushed to `main`.
- Validation was Docker-first (production image builds succeeded for both backend and client).

### Commits pushed

- Backend (`Aera-Website-API`): `383d881` - fix(auth): stabilize refresh sessions and remember-me behavior
- Client (`Aera-Website-Client`): `78b9769` - fix(auth): auto-refresh expired tokens during navigation and API calls
- Related prior client production image/IPX fix already on `main`: `55512ca`

### Backend changes (authoritative)

- Added refresh endpoint:
    - `POST /api/v1/private-access/auth/refresh-token`
    - file: `backend/api/src/routes/private/auth.routes.ts`
- Preserved `rememberMe` through device verification and 2FA continuation:
    - `backend/api/src/controllers/private/auth/deviceVerification.controller.ts`
    - `backend/api/src/services/private/auth/login.service.ts`
- Fixed refresh token rotation matching by using deterministic HMAC hash lookup:
    - `backend/api/src/services/private/auth/refreshToken.service.ts`
- Refresh cookie issuance now occurs for both remembered and non-remembered logins after successful 2FA.
- Session timing moved to env-driven values with defaults:
    - `ACCESS_TOKEN_EXPIRE_PERIOD=3600`
    - `REFRESH_TOKEN_EXPIRE_SESSION_PERIOD=43200`
    - `REFRESH_TOKEN_EXPIRE_REMEMBER_ME_PERIOD=2592000`
    - refs: `backend/.env.example`, `backend/.env.production.example`, `backend/api/src/utils/constants.util.ts`

### Client changes (authoritative)

- Added refresh-on-expiry retry for private API requests:
    - `client/composables/AuthFetch.ts`
- Route middleware now attempts cookie-based refresh before forcing logout:
    - `client/middleware/auth.ts`
- On refresh failure, client clears session state and redirects to login (`sessionExpired=true`).

### Resulting behavior

- Session is now activity-based:
    - Access token expires normally.
    - Active private usage triggers transparent refresh using HttpOnly refresh cookie.
    - Inactive users eventually expire when refresh window closes.
- Keep-me-logged-in now extends refresh window, rather than frequent forced re-login.

### Docker validation completed

- Backend build passed:

```bash
docker compose --env-file backend/.env.production -f backend/docker-compose.production.yml build REPLACE_WITH_API_SERVICE REPLACE_WITH_CONTAINER_OR_SERVICE_NAME
```

- Client build passed:

```bash
docker compose --env-file client/.env.production -f client/docker-compose.prod.yml build nuxt-prod
```

- Build note:
    - `NUXT_PUBLIC_THEME_COLOR` warning appears if unset (non-blocking).

### Resume commands on VPS (production rollout)

```bash
cd ~/apps/Website
git -C backend pull --ff-only
git -C client pull --ff-only

# Optional explicit session policy (recommended)
./aera env:set --target=backend --profile=prod \
  --set ACCESS_TOKEN_EXPIRE_PERIOD=3600 \
  --set REFRESH_TOKEN_EXPIRE_SESSION_PERIOD=43200 \
  --set REFRESH_TOKEN_EXPIRE_REMEMBER_ME_PERIOD=2592000

./aera backend:build --no-cache
./aera backend:up
./aera backend:wait

./aera client:build --no-cache
./aera client:up

./aera edge:restart
./aera status
```

### Post-deploy verification checklist

1. Login with Keep Me Logged In unchecked and confirm normal login + expected timeout behavior.
2. Login with Keep Me Logged In checked and confirm session survives beyond access-token expiry.
3. Verify refresh route activity in backend logs:

```bash
docker logs -f REPLACE_WITH_API_CONTAINER | grep -E "refresh-token|Unauthorized|Token has expired"
```

4. Confirm protected route navigation no longer force-logs out active users.

---

## Update: Session Closeout - IPX Upload Rendering Resolved (2026-03-12)

### Final issue addressed

- Production images still failed to render through Nuxt IPX on:
    - `/_ipx/_/uploads/images/heroSection/...`
- Diagnostic result isolated root cause:
    - Direct edge uploads route worked (`200`).
    - IPX route failed (`403`).
    - Client container -> backend upload fetch failed with:
        - `Forbidden host` / `HOST_NOT_ALLOWED`.

### Root cause

- Backend host allowlisting was enabled (`ENFORCE_ALLOWED_HOSTS=true`) but internal Docker hostnames were not included in `ALLOWED_HOSTS`.
- IPX internal fetches originated with host `REPLACE_WITH_API_SERVICE` (and `REPLACE_WITH_API_CONTAINER` alias), which backend rejected.

### Production fix applied

- Updated backend production env allowlist to include internal API hosts:

```bash
./aera env:set --target=backend --profile=prod \
  --set ALLOWED_HOSTS='REPLACE_WITH_SECONDARY_DOMAIN,REPLACE_WITH_API_SERVICE,REPLACE_WITH_API_CONTAINER'
```

- Restarted backend and re-verified.

### Verification evidence (authoritative)

- `direct=200`:
    - `https://REPLACE_WITH_SECONDARY_DOMAIN/api/v1/uploads/images/heroSection/banner-1773215921623-823233085.jpeg`
- `ipx=200`:
    - `https://REPLACE_WITH_SECONDARY_DOMAIN/_ipx/_/uploads/images/heroSection/banner-1773215921623-823233085.jpeg`
- from `REPLACE_WITH_CONTAINER_OR_SERVICE_NAME` container to backend uploads:
    - `status=200` for `http://REPLACE_WITH_API_SERVICE:4000/api/v1/uploads/images/heroSection/banner-1773215921623-823233085.jpeg`

### Current status at closeout

- Auth/session flow fixes are deployed and verified.
- Keep-me-logged-in and activity-based session refresh behavior are deployed.
- Nuxt image rendering for API uploads is working in production.
- No active blockers at session close.

---

## Update: Full Content Seeding Infrastructure (2026-06-22)

### Scope completed

All content under `backend/api/src/content-seeds/` — both `administration/` and `pages/` — is now fully seedable via service-aligned scripts. The seeding infrastructure was built from scratch and validated end-to-end via `npx ts-node` runtime execution.

### What was built

#### Seed runtime utilities (new files)

- `backend/api/src/scripts/seed/helpers/runtime.util.ts`
    - MongoDB connection/disconnection, URI rewriting for local host
    - `resolveOrCreateDivisionId`, `resolveAuthorId`
    - `toNumber` numeric coercion helper
    - `shouldRefreshCache` guard for container-only Redis
- `backend/api/src/scripts/seed/helpers/cache-refresh.util.ts`
    - `guardedRefresh(...)` wrapper — skips Redis refresh when `REPLACE_WITH_REDIS_SERVICE` host is unreachable in local environments
    - Entity-specific refresh functions
    - `refreshPageRelatedCaches()` for full page graph

#### Enhanced page content seeder

- `backend/api/src/scripts/seed/create-page-content.script.ts`
    - Expanded to seed full nested graph per page:
        - Page record
        - Schema Markup + SameAs + Sections + Contact Points
        - Social Media Meta Tags
        - Hero Section + Banners
        - Content Sections + Content Blocks
        - Block children: images, videos, list items, additional information
    - `normalizePageSeed()` handles both `{ page: {...} }` and flat seed shapes
    - `toNumber()` auto-coercion of numeric section fields

#### New seed scripts

| Script                                             | Purpose                                                                   |
| -------------------------------------------------- | ------------------------------------------------------------------------- |
| `seed/seed-page-content-by-slug.script.ts`         | Canonical single-page seeder by slug (replaces all old per-page wrappers) |
| `seed/seed-all-pages-content.script.ts`            | Batch seeder for all 20 page payloads                                     |
| `seed/seed-board-members.script.ts`                | Board members (7 records) with Joi validation + image constraints         |
| `seed/seed-vacancies.script.ts`                    | Vacancies (4 records)                                                     |
| `seed/seed-procurements.script.ts`                 | Procurements (5 records) with title/attachment max-length clamping        |
| `seed/seed-authorized-service-providers.script.ts` | Providers with image path validation                                      |
| `seed/seed-articles.script.ts`                     | News articles                                                             |
| `seed/seed-events.script.ts`                       | Events                                                                    |
| `seed/seed-all-content.script.ts`                  | Master orchestrator: runs all seeders in dependency order                 |

#### Seed constant exports added

- `administration/boardMember.ts`, `administration/procurement.ts`, `administration/vacancy.ts` — arrays now exported
- All page content-seed files now export top-level named constants used by orchestrator scripts

#### Script cleanup

- Removed 11 obsolete per-page wrapper scripts (e.g. `seed-about-page.script.ts`, `seed-contact-page.script.ts`, etc.) that imported non-existent `.page` modules
- Replaced all old page command routing with single canonical `seed-page-content-by-slug.script.ts` entrypoint

#### Command wiring

- `backend/api/package.json` — updated to remove stale per-page npm scripts, added new canonical seeder scripts
- `backend/scripts/aera` — updated `content:seed:page` case to route through slug-based script, updated help text and examples with all valid slugs
- `backend/scripts/aera` — `content:seed:all`, `content:seed:pages`, `content:seed:board-members`, `content:seed:vacancies`, `content:seed:procurements`, `content:seed:authorized-service-providers`, `content:seed:articles`, `content:seed:events` all wired

#### Documentation

- `backend/docs/operations/CONTENT_SEEDING_GUIDE.md` — comprehensive new guide (quick start, all seeders, validation rules, post-seed verification curls, cache behavior, troubleshooting, common workflow recipes)
- `backend/docs/operations/DATA_MIGRATION_AND_SEEDING.md` — updated from sparse stub to practical quick-start + pointer to guide
- `backend/docs/operations/README.md` — linked new guide
- `backend/docs/operations/CONTENT_OPERATIONS.md` — scoped to manual API workflow, with pointer to seeding guide
- `backend/api/src/scripts/SEED_SCRIPTS_REFERENCE.md` — quick reference living alongside the scripts
- `backend/api/src/scripts/docs/SCRIPT_USAGE.md` — removed stale "all content creation scripts have been removed" claim

### Validation outcome (authoritative)

All seeders were validated via `npx ts-node` runtime execution:

- `seed-all-pages-content` — 20 pages created/updated, full nested graph (schema markup, social meta, hero sections, sections/blocks/children)
- `seed-board-members` — 7 records upserted
- `seed-vacancies`, `seed-procurements` — all records upserted after Joi validation fixes
- `seed-authorized-service-providers`, `seed-articles`, `seed-events` — all records upserted
- `seed-all-content` (full orchestrator) — completed end-to-end successfully:

```
Connecting to MongoDB...
Seeding base content (divisions, activities, partners)...
Seeding all page content...
Seeding administration (board members, vacancies, procurements)...
Seeding authorized service providers...
Seeding articles and events...
All content seeds completed successfully.
```

Idempotent re-run confirmed: all 7 board members show `Updated: 7, Created: 0` on subsequent runs.

### Important notes on seeding behavior

- **Idempotent:** All seeders use upsert logic — safe to re-run
- **Validation:** Service-layer Joi schemas are enforced at seed time (image path regex, field max-lengths, required fields)
- **Cache refresh:** Guarded wrapper skips Redis refresh when `REPLACE_WITH_REDIS_SERVICE` host is unreachable locally (production Docker runs refresh normally)
- **Local MongoDB URI rewriting:** `MONGODB_URI` `REPLACE_WITH_MONGO_SERVICE:27017` is automatically rewritten to `localhost:${DEV_MONGO_HOST_PORT}` for local script runs

### Known remaining items

- Legacy `create-sample-articles.script.ts` in `backend/api/src/scripts/` is not wired to any current command; can be removed or superseded by `seed-articles.script.ts` in a future cleanup pass
- Seeding scripts are excluded from `tsconfig.json` (`src/scripts/**/*`); `npm run typecheck` does not validate them — `npx ts-node` runtime execution is the authoritative test path

### Resume from here next session (VPS content seeding)

Content seeding has only been validated locally. To apply on VPS:

```bash
cd ~/apps/Website

# Pull latest backend source
git -C backend pull --ff-only

# Optional: rebuild backend image to pick up any code changes
./aera backend:build
./aera backend:up
./aera backend:wait

# Seed all content (idempotent - safe to run on first deploy or re-run)
./aera content:seed:all

# Or selectively:
./aera content:seed:divisions
./aera content:seed:pages
./aera content:seed:board-members
./aera content:seed:vacancies
./aera content:seed:procurements
./aera content:seed:authorized-service-providers
./aera content:seed:articles
./aera content:seed:events

# Clear cache after seeding if needed
./aera content:cache:clear

# Verify via public endpoint
curl -s -H "x-api-key: $PUBLIC_API_KEY" https://REPLACE_WITH_SECONDARY_DOMAIN/api/v1/public/pages/about | head -c 500
```

### Post-seeding verification checklist (VPS)

- [ ] Run `./aera content:seed:all` — no errors, "completed successfully" message
- [ ] Curl `GET /api/v1/public/pages/about` and confirm nested sections/blocks in response
- [ ] Curl `GET /api/v1/public/about/board-members` and confirm 7 members returned
- [ ] Curl `GET /api/v1/public/articles` and confirm articles returned
- [ ] Browser test: navigate all seeded pages on frontend and confirm content renders
- [ ] Confirm Nuxt IPX image rendering still works for any seeded images

---

## Update: Newsletter Publish System Design (2026-06-26)

### Goal

Implement a robust newsletter publish flow that sends the monthly newsletter (including populated articles and optional events) to all verified subscribers at scale without blocking API responsiveness.

### Existing foundation confirmed

- RabbitMQ email queue and worker are already in place:
    - `backend/api/src/worker.ts`
    - queue: `email_queue`
    - retry logic already exists (up to 3 attempts)
- Redis-backed email template loading is already in place:
    - `backend/api/src/utils/redisClient.util.ts`
    - templates folder: `backend/api/src/emails/templates/`
- Subscription verification and confirmation email flows are already queue-driven:
    - `backend/api/src/emails/subscriptionVerification.email.ts`
    - `backend/api/src/emails/subscriptionConfirmation.email.ts`
- Newsletter CRUD and monthly uniqueness checks are already implemented:
    - `backend/api/src/services/private/newsletter/newsletter.service.ts`
    - `backend/api/src/services/private/newsletter/verification.service.ts`

### Gaps identified before broadcast implementation

1. Subscriber audience filtering
    - Current subscription cache loader (`Subscription.find()`) does not explicitly filter to verified subscribers.
    - Newsletter blasts must target only `isVerified: true` recipients.

2. Producer connection overhead
    - Current email producer pattern opens/closes RabbitMQ connection per send call.
    - This is acceptable for transactional emails but inefficient for high-volume campaign fan-out.

3. Worker throughput
    - Worker currently uses `prefetch(1)` which is conservative and may be too slow for large sends.

4. Publish operation lifecycle
    - Newsletter service has create/update/delete, but no dedicated publish endpoint and no campaign-level tracking.

5. Public links in newsletter email
    - Newsletter email content needs stable frontend URLs for article/event read-more actions.

### Recommended architecture (queue-first, non-blocking)

#### A. Publish API endpoint

- Add `PUT /private-access/newsletter/publish/:id`.
- Validate:
    - newsletter exists
    - not already published
    - contains at least one article
- Transition newsletter state to publishing/published workflow state.
- Enqueue campaign job and return `202 Accepted` quickly.

#### B. Two-stage queue strategy

1. Campaign queue (`newsletter_publish_queue`)
    - One job per newsletter publish action.
    - Worker prepares audience and fan-out.

2. Recipient queue (`newsletter_recipient_queue`)
    - One job per recipient email send.
    - Enables horizontal scaling and retry isolation.

#### C. Campaign worker responsibilities

- Load newsletter with fully populated article and event details.
- Build canonical campaign payload snapshot.
- Query verified subscribers only.
- Chunk recipients and enqueue recipient jobs in batches.
- Update campaign counters/progress.

#### D. Recipient worker responsibilities

- Render newsletter template with dynamic sections:
    - Articles section always rendered.
    - Events section rendered conditionally only if events exist.
- Send using transporter with retries and failure classification.
- Persist send result (`sent`/`failed`) for observability.

### Data model additions recommended

1. `newsletter_campaigns`
    - `newsletterId`
    - `issueDate`
    - `status` (`queued`, `processing`, `completed`, `failed`)
    - `queuedCount`, `sentCount`, `failedCount`
    - `startedAt`, `completedAt`
    - `initiatedBy`

2. (Optional but recommended) `newsletter_campaign_recipients`
    - `campaignId`
    - `email`
    - `status`
    - `attempts`
    - `lastError`
    - unique index on (`campaignId`, `email`) for idempotency

### Idempotency and reliability requirements

- Use recipient-level idempotency key (`campaignId + email`) to prevent duplicates.
- Use dead-letter queue for exhausted retries.
- Ensure publish endpoint is safe against duplicate clicks/race conditions.
- Maintain explicit campaign state transitions.

### Performance and delivery tuning

- Enable SMTP pooling for campaign sends in `emailTransporter`.
- Increase worker prefetch/concurrency gradually (e.g., 5 -> 10 -> 20) based on provider limits.
- Batch queue publish operations to reduce broker round trips.
- Keep API request path non-blocking (enqueue and return).

### Suggested implementation phases

#### Phase 1 (foundation)

- Add publish endpoint, controller method, and service method.
- Add verified-subscriber-only query path.
- Add campaign queue job enqueue.

#### Phase 2 (email content and fan-out)

- Add newsletter email HTML template (`newsletterBroadcast.template.html`).
- Build server-side section renderers for article/event blocks.
- Add campaign worker + recipient job fan-out.

#### Phase 3 (tracking and operability)

- Add campaign persistence and progress tracking.
- Add admin-visible publish result summary (queued/sent/failed).
- Add failure reporting and replay controls.

#### Phase 4 (scale hardening)

- SMTP pool tuning, queue prefetch tuning, and load testing.
- Add DLQ monitoring/alerts.
- Add runbook for campaign retries and incident handling.

### Definition of done (newsletter publish)

- Publish click triggers non-blocking enqueue and immediate API response.
- Only verified subscribers receive campaign email.
- Email includes article section with read-more links.
- Event section is optional and only rendered when newsletter has events.
- Campaign completion metrics are visible and auditable.
- Retries and idempotency prevent duplicate sends.

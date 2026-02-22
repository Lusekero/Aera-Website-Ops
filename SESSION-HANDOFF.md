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

- `smtp:configure` updates backend `.env.production` keys and by default recreates only `aera-api` + `aera-worker`.
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
./aera smtp:configure --provider=cpanel --host=<smtp_host> --port=<465_or_587> --secure=<true_or_false> --user=<mailbox> --password='<password>' --from='"Atomic Energy Regulatory Authority" <info@aera.org.mw>'
./aera smtp:show
./aera smtp:test --to=<recipient>
```

### Switch provider later (example M365)

```bash
./aera smtp:configure --provider=m365 --user=<mailbox> --password='<password>' --from='"Atomic Energy Regulatory Authority" <info@aera.org.mw>'
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

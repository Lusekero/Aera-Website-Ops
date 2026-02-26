# AERA VPS Deployment Day Cheat Sheet

Use this on deployment day. Command-first, minimal explanation.

---

## 1) SSH from local

```bash
ssh -p <SSH_PORT> <SSH_USERNAME>@<VPS_IP>
```

---

## 2) One-time VPS base prep (if not done)

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install ca-certificates curl gnupg lsb-release git ufw fail2ban make
sudo ufw allow <SSH_PORT>/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status
```

---

## 3) One-time Docker install (if not done)

Check first:

```bash
if command -v docker >/dev/null 2>&1 && docker compose version >/dev/null 2>&1; then
  echo "Docker + Docker Compose already installed"
  docker --version
  docker compose version
else
  echo "Docker and/or Docker Compose missing"
fi
```

Install only if missing:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
```

---

## 4) Place code as siblings

Target:

```text
~/apps/Website/
  backend/
  client/
  proxy/
```

Create and verify structure (first-time VPS setup):

```bash
mkdir -p ~/apps/Website
```

Clone/update directly from GitHub (recommended):

```bash
cd ~/apps/Website

# Change only these if your org/repo names differ
GITHUB_OWNER="Lusekero"
BACKEND_REPO="Aera-Website-API"
CLIENT_REPO="Aera-Website-Client"
PROXY_REPO="Aera-Website-Proxy"

# Optional: use "ssh" if your VPS has deploy keys configured
GIT_PROTOCOL="https"   # https | ssh

repo_url() {
  local repo="$1"
  if [ "$GIT_PROTOCOL" = "ssh" ]; then
    echo "git@github.com:${GITHUB_OWNER}/${repo}.git"
  else
    echo "https://github.com/${GITHUB_OWNER}/${repo}.git"
  fi
}

clone_or_update() {
  local dir="$1"
  local repo="$2"
  local url
  url="$(repo_url "$repo")"

  if [ -d "$dir/.git" ]; then
    echo "[update] $dir <- $url"
    git -C "$dir" fetch --all --prune
    git -C "$dir" pull --ff-only
  else
    echo "[clone]  $dir <- $url"
    rm -rf "$dir"
    git clone "$url" "$dir"
  fi
}

clone_or_update backend "$BACKEND_REPO"
clone_or_update client "$CLIENT_REPO"
clone_or_update proxy "$PROXY_REPO"

ls -la ~/apps/Website
```

Direct clone commands (simple one-time setup):

```bash
cd ~/apps/Website
git clone https://github.com/Lusekero/Aera-Website-API.git backend
git clone https://github.com/Lusekero/Aera-Website-Client.git client
git clone https://github.com/Lusekero/Aera-Website-Proxy.git proxy
ls -la ~/apps/Website
```

If HTTPS git auth fails (`Invalid username or token`), configure SSH once on the VPS, then use `GIT_PROTOCOL="ssh"` in the clone/update block:

```bash
# 1) Generate SSH key (press Enter for default path; optional passphrase)
ssh-keygen -t ed25519 -C "<your_github_email>"

# 2) Start agent and load key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 3) Add GitHub host key
ssh-keyscan github.com >> ~/.ssh/known_hosts

# 4) Print public key and add it in GitHub:
#    GitHub -> Settings -> SSH and GPG keys -> New SSH key
cat ~/.ssh/id_ed25519.pub

# 5) Verify auth (expected message: You've successfully authenticated)
ssh -T git@github.com
```

If you are connected as `root`, use this exact root-safe block:

```bash
set -euo pipefail

# 1) Create SSH key (press Enter at prompt to use /root/.ssh/id_ed25519)
ssh-keygen -t ed25519 -C "<your_github_email>"

# 2) Start agent and load key
eval "$(ssh-agent -s)"
ssh-add /root/.ssh/id_ed25519

# 3) Trust GitHub host key with proper permissions
mkdir -p /root/.ssh && chmod 700 /root/.ssh
ssh-keyscan -H github.com >> /root/.ssh/known_hosts
chmod 600 /root/.ssh/known_hosts

# 4) Print public key and add in GitHub -> Settings -> SSH and GPG keys
cat /root/.ssh/id_ed25519.pub

# 5) Verify SSH auth to GitHub
ssh -T git@github.com
```

If `ssh-keygen` says the file already exists, type `n` (do not overwrite), then either reuse existing key with `ssh-add /root/.ssh/id_ed25519` or create a new key file:

```bash
ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519_github -C "<your_github_email>"
ssh-add /root/.ssh/id_ed25519_github
cat /root/.ssh/id_ed25519_github.pub
```

Quick remote switch (existing repos already cloned with HTTPS):

```bash
cd ~/apps/Website
git -C backend remote set-url origin git@github.com:Lusekero/Aera-Website-API.git
git -C client remote set-url origin git@github.com:Lusekero/Aera-Website-Client.git
git -C proxy remote set-url origin git@github.com:Lusekero/Aera-Website-Proxy.git
git -C backend remote -v
git -C client remote -v
git -C proxy remote -v
```

If any folder already exists, remove it first or use the clone/update block above.

If repos were cloned/uploaded elsewhere, move them into place:

```bash
mv ~/apps/backend ~/apps/Website/backend 2>/dev/null || true
mv ~/apps/client ~/apps/Website/client 2>/dev/null || true
mv ~/apps/proxy ~/apps/Website/proxy 2>/dev/null || true
```

Verify:

```bash
ls -la ~/apps/Website
```

Install global `aera` command (optional, recommended):

First bootstrap `aera` into root from Ops repo:

```bash
cd ~/apps/Website
curl -fsSL https://raw.githubusercontent.com/Lusekero/Aera-Website-Ops/main/aera -o ./aera
chmod +x ./aera
```

Update later (same command):

```bash
cd ~/apps/Website
curl -fsSL https://raw.githubusercontent.com/Lusekero/Aera-Website-Ops/main/aera -o ./aera
chmod +x ./aera
```

Then install global command:

```bash
cd ~/apps/Website
sudo ln -sf ~/apps/Website/aera /usr/local/bin/aera
```

Verify from any directory:

```bash
aera help
```

If repo path changes later, relink quickly:

```bash
sudo rm -f /usr/local/bin/aera
sudo ln -sf ~/apps/Website/aera /usr/local/bin/aera
```

---

## 4) Pre-deploy sanity checks

# Backend: remove dev configs and examples

rm -f backend/docker-compose.yml backend/.env.example

# Client: remove dev Dockerfile, dev compose, and example env

rm -f client/Dockerfile.dev client/docker-compose.dev.yml client/.env.example

# Proxy: remove the entire dev folder

rm -f proxy/dev proxy/prod/.env.edge.prod.local.example proxy/prod/docker-compose.edge.prod.local.yml

---

## 4A) Alias quick reference (root `aera`)

Run all commands from `~/apps/Website` unless noted.

Core stack lifecycle:

```bash
./aera doctor
./aera network:ensure
./aera backend:up
./aera backend:wait
./aera client:up
./aera edge:up
./aera status
```

Bring stacks down / restart:

```bash
./aera backend:down
./aera client:down
./aera edge:down

./aera backend:restart
./aera client:restart
./aera edge:restart
```

Logs:

```bash
./aera logs:backend
./aera logs:client
./aera logs:edge
```

Namespaced passthrough aliases (single command surface):

```bash
./aera backend:cmd <backend_command> [args...]
./aera client:cmd <client_command> [args...]
./aera edge:cmd <make_target> [args...]
```

Examples:

```bash
./aera backend:launch:status
./aera backend:queue:health
./aera client:deploy:test
./aera client:maintenance:down
./aera edge:cmd status
```

Day-0 minimal sequence (copy/paste):

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

Beginner/Junior quick start (fresh VPS):

```bash
mkdir -p ~/apps/Website
cd ~/apps/Website
git clone https://github.com/Lusekero/Aera-Website-API.git backend
git clone https://github.com/Lusekero/Aera-Website-Client.git client
git clone https://github.com/Lusekero/Aera-Website-Proxy.git proxy
cd ~/apps/Website
./aera doctor
./aera network:ensure
./aera backend:up && ./aera backend:wait && ./aera client:up && ./aera edge:up
./aera status
```

If you need to stop everything safely:

```bash
cd ~/apps/Website
./aera edge:down
./aera client:down
./aera backend:down
./aera status
```

---

## 5) Configure production env files

- `~/apps/Website/backend/.env.production`
- `~/apps/Website/client/.env.production`
- `~/apps/Website/proxy/prod/.env.edge.prod`

Create env files from examples (recommended first step):

```bash
cd ~/apps/Website

# Backend
cp -n backend/.env.example backend/.env
if [ ! -f backend/.env.production ]; then cp backend/.env.production.example backend/.env.production; fi

# Client
cp -n client/.env.example client/.env
if [ ! -f client/.env.production ]; then cp client/.env.production.example client/.env.production; fi

# Proxy (dev + prod)
cp -n proxy/dev/.env.edge.dev.example proxy/dev/.env.edge.dev
if [ ! -f proxy/prod/.env.edge.prod ]; then cp proxy/prod/.env.edge.prod.example proxy/prod/.env.edge.prod; fi

# Optional local-prod variant (if you use local prod override)
cp -n proxy/prod/.env.edge.prod.local.example proxy/prod/.env.edge.prod.local
```

Notes:

- `cp -n` means "do not overwrite" existing env files.
- If you need to refresh from examples, use `cp -f` intentionally.

MongoDB password rotation after env duplication (existing production DB):

```bash
cd ~/apps/Website/backend
set -euo pipefail
source .env.production

read -s -p "New Mongo root password: " NEW_PASS; echo

docker exec -i aera-mongo-prod mongosh \
  -u "$MONGO_INITDB_ROOT_USERNAME" \
  -p "$MONGO_INITDB_ROOT_PASSWORD" \
  --authenticationDatabase admin \
  --eval "db.getSiblingDB('admin').changeUserPassword('$MONGO_INITDB_ROOT_USERNAME', '$NEW_PASS')"

cd ~/apps/Website
./aera env:set --target=backend --profile=prod \
  --set MONGO_INITDB_ROOT_PASSWORD="$NEW_PASS" \
  --set MONGODB_URI="mongodb://$MONGO_INITDB_ROOT_USERNAME:$NEW_PASS@aera-mongo:27017/$MONGO_INITDB_DATABASE"

./aera backend:up
./aera backend:wait
./aera status
```

For first-time deploy on a brand-new Mongo volume, set the password in env before first `./aera backend:up` and skip `changeUserPassword`.

Quick file checks before first deploy:

```bash
test -f ~/apps/Website/backend/.env.production && echo "backend env ok" || echo "missing backend .env.production"
test -f ~/apps/Website/client/.env.production && echo "client env ok" || echo "missing client .env.production"
test -f ~/apps/Website/proxy/prod/.env.edge.prod && echo "proxy env ok" || echo "missing proxy .env.edge.prod"
```

Set env values with one common command style (backend/client/proxy, dev/prod):

```bash
cd ~/apps/Website

# Backend production secrets/JWT/API keys
./aera env:set --target=backend --profile=prod --set JWT_SECRET='<jwt_secret>' --set REFRESH_TOKEN_ENCRYPTION_KEY='<refresh_secret>' --set PUBLIC_API_KEY='<public_api_key>'

# Or auto-generate strong backend secrets directly into prod env file
./aera backend:secrets:generate --profile=prod

# Sync backend PUBLIC_API_KEY into client NUXT_PUBLIC_API_KEY (same profile)
./aera client:api-key:sync --profile=prod

# Backend development values
./aera env:set --target=backend --profile=dev --set JWT_SECRET='dev-jwt-secret' --set API_DOMAIN=localhost

# Client production/public env
./aera env:set --target=client --profile=prod --set NUXT_PUBLIC_API_BASE_URL=/api/v1 --set NUXT_PUBLIC_UPLOADS_PATH=/uploads --set NUXT_PUBLIC_APP_ORIGIN=https://test.aera.org.mw
./aera env:set --target=client --profile=prod --set NUXT_PUBLIC_RECAPTCHA_SITE_KEY='<site_key_from_recaptcha>'
./aera env:set --target=client --profile=prod --set RECAPTCHA_SITE_KEY='<site_key_from_recaptcha>'

# Proxy production routing/env
./aera env:set --target=proxy --profile=prod --set PRIMARY_DOMAIN=test.aera.org.mw --set SECONDARY_DOMAIN=aera.org.mw --set TLS_CERT_DOMAIN=test.aera.org.mw

# Read back values
./aera env:get --target=backend --profile=prod --get JWT_SECRET --get PUBLIC_API_KEY

# Remove deprecated key
./aera env:unset --target=backend --profile=prod --unset OLD_LEGACY_SECRET
```

Runtime key names expected by code:

| Layer         | Keys expected by runtime                                                                                                                  | Note                                                                                                  |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Backend/API   | `JWT_SECRET`, `REFRESH_TOKEN_ENCRYPTION_KEY`, `EMAIL_OTP_SECRET`, `PUBLIC_API_KEY`, `RECAPTCHA_SECRET`                                    | Use `./aera backend:secrets:generate --profile=<dev                                                   | prod>` for generated ones; set provider-issued keys (like reCAPTCHA) explicitly. |
| Backend SMTP  | `EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_SECURE`, `EMAIL_USER`, `EMAIL_PASSWORD`, `EMAIL_FROM`                                                  | Use `./aera smtp:configure ...`                                                                       |
| Client (Nuxt) | `NUXT_PUBLIC_API_BASE_URL`, `NUXT_PUBLIC_UPLOADS_PATH`, `NUXT_PUBLIC_APP_ORIGIN`, `NUXT_PUBLIC_API_KEY`, `NUXT_PUBLIC_RECAPTCHA_SITE_KEY` | `NUXT_PUBLIC_API_KEY` must match backend `PUBLIC_API_KEY` (`./aera client:api-key:sync --profile=<dev | prod>`).                                                                         |
| Proxy edge    | `PRIMARY_DOMAIN`, `SECONDARY_DOMAIN`, `TLS_CERT_DOMAIN`, `CLIENT_UPSTREAM`, `API_UPSTREAM`, `CLIENT_MAX_BODY_SIZE`                        | These are routing/runtime vars, not generated secrets.                                                |

If you still need custom app-specific keys not listed above:

```bash
./aera client:secrets:generate --profile=prod --secret CUSTOM_CLIENT_SECRET
./aera proxy:secrets:generate --profile=prod --secret CUSTOM_EDGE_SECRET --format=base64
```

Environment separation rule:

- `--profile=dev` writes only to dev env files (`backend/.env`, `client/.env`, `proxy/dev/.env.edge.dev`)
- `--profile=prod` writes only to production env files (`backend/.env.production`, `client/.env.production`, `proxy/prod/.env.edge.prod`)
- Dev and prod are intentionally separate and should be run independently (not simultaneously on the same deployment path).

Test-first values in proxy env:

```dotenv
PRIMARY_DOMAIN=test.aera.org.mw
SECONDARY_DOMAIN=aera.org.mw
TLS_CERT_DOMAIN=test.aera.org.mw
CLIENT_UPSTREAM=nuxt-prod:3000
API_UPSTREAM=aera-api-prod:4000
CLIENT_MAX_BODY_SIZE=200m
```

Important first-issuance note (staged DNS):

- `./aera cert:init` requests certs for `TLS_CERT_DOMAIN` plus distinct `PRIMARY_DOMAIN`/`SECONDARY_DOMAIN` values.
- If only the test domain currently points to VPS, set `SECONDARY_DOMAIN` equal to `PRIMARY_DOMAIN` temporarily (or to the same test domain) before first `cert:init`.
- After `aera.org.mw` DNS points to VPS, restore `SECONDARY_DOMAIN=aera.org.mw` and re-run `./aera cert:init`.

Configure SMTP from cPanel (production email sending):

1. In cPanel, open `Email Accounts` for your mailbox (example: `info@aera.org.mw`) and click `Connect Devices` / `Set Up Mail Client`.
2. Copy Outgoing (SMTP): host, port, encryption mode, username, password.
3. Map values into `~/apps/Website/backend/.env.production`:

```dotenv
EMAIL_HOST=<cpanel_smtp_host>
EMAIL_PORT=<465_or_587>
EMAIL_SECURE=<true_for_465_or_false_for_587>
EMAIL_USER=<full_mailbox_address>
EMAIL_PASSWORD=<mailbox_password>
EMAIL_FROM='"Atomic Energy Regulatory Authority" <info@aera.org.mw>'
```

Apply and verify:

```bash
cd ~/apps/Website
./aera smtp:configure --provider=cpanel --host=<cpanel_smtp_host> --port=<465_or_587> --secure=<true_or_false> --user=<full_mailbox_address> --password='<mailbox_password>' --from='"Atomic Energy Regulatory Authority" <info@aera.org.mw>'
./aera smtp:show
./aera smtp:test --to=<your_test_email>
./aera logs:backend
```

Platform switch later (example to Microsoft 365) is a single command + restart:

```bash
cd ~/apps/Website
./aera smtp:configure --provider=m365 --user=<full_mailbox_address> --password='<mailbox_password>' --from='"Atomic Energy Regulatory Authority" <info@aera.org.mw>'
./aera smtp:test --to=<your_test_email>
```

Tip: use the exact host/port/encryption pair from provider docs. Common values are `465` (SSL/TLS, secure=true) or `587` (STARTTLS, secure=false).
`./aera smtp:configure` updates `.env.production` and recreates only backend runtime services (`aera-api`, `aera-worker`) by default.
Advanced option: add `--worker-only` to restart just `aera-worker` (use only if your email sending path is worker-only).

---

## 6) DNS check from local machine

```bash
dig +short test.aera.org.mw
```

Must resolve to VPS public IP.

---

## 7) Deploy (run on VPS)

Run from workspace root:

```bash
cd ~/apps/Website
./aera doctor
```

### 7A) Backend deploy (independent)

Optional safety pre-create shared network (idempotent):

```bash
./aera network:ensure
```

```bash
./aera doctor
./aera backend:up
./aera backend:wait
./aera status
```

If backend wait times out on first boot:

```bash
./aera backend:retry
./aera status
```

### 7B) Client deploy (independent, after backend healthy)

```bash
./aera client:up
./aera status
```

### 7C) Proxy deploy (independent, after backend+client)

Edge first-time TLS bootstrap (only needed before real cert exists):

```bash
cd ~/apps/Website/proxy
source prod/.env.edge.prod

docker run --rm -v aera_certbot_certs:/etc/letsencrypt alpine:3.20 sh -c "
  set -e
  apk add --no-cache openssl >/dev/null
  mkdir -p /etc/letsencrypt/live/${TLS_CERT_DOMAIN}
  openssl req -x509 -nodes -newkey rsa:2048 -days 1 \
    -keyout /etc/letsencrypt/live/${TLS_CERT_DOMAIN}/privkey.pem \
    -out /etc/letsencrypt/live/${TLS_CERT_DOMAIN}/fullchain.pem \
    -subj '/CN='${TLS_CERT_DOMAIN}
"
cd ~/apps/Website
```

```bash
./aera edge:up
EMAIL=<YOUR_EMAIL> ./aera cert:init
./aera edge:restart
./aera status
```

If cert issuance fails for a secondary domain not yet pointed to VPS, temporarily set `SECONDARY_DOMAIN` to match the active domain and retry `cert:init`.

Or run full flow in one command (first-time friendly):

```bash
cd ~/apps/Website
EMAIL=<YOUR_EMAIL> ./aera deploy:all
```

### 7D) Independent re-deploy commands (modular)

Backend only:

```bash
cd ~/apps/Website
./aera backend:build --no-cache
./aera backend:up
./aera backend:wait
```

Client only:

```bash
cd ~/apps/Website
./aera client:up
```

Proxy only:

```bash
cd ~/apps/Website
./aera edge:build --no-cache
./aera edge:up
./aera edge:restart
```

### 7E) Backend content/admin bootstrap (optional)

Use these from `~/apps/Website` to keep one Laravel-like command style:

```bash
ADMIN_PASSWORD='StrongPassword123!' ADMIN_EMAIL='admin@aera.org.mw' ADMIN_FIRSTNAME='System' ADMIN_LASTNAME='Admin' ./aera content:seed:admin
./aera content:cache:clear
```

Equivalent backend-local commands (same operation):

```bash
cd ~/apps/Website/backend
ADMIN_PASSWORD='StrongPassword123!' ADMIN_EMAIL='admin@aera.org.mw' ADMIN_FIRSTNAME='System' ADMIN_LASTNAME='Admin' ./scripts/aera content:seed:admin
./scripts/aera content:cache:clear
```

Equivalent raw compose sequence (manual mode, backend → client → proxy):

Run these from `~/apps/Website/proxy`:

```bash
cd ~/apps/Website/proxy
```

Optional safety pre-create (idempotent; safe to skip when starting backend first):

```bash
docker network create aera_network_prod || true
```

```bash
docker compose --env-file ../backend/.env.production -f ../backend/docker-compose.production.yml up -d
```

Wait for backend services to become healthy before starting client:

```bash
docker compose --env-file ../backend/.env.production -f ../backend/docker-compose.production.yml ps
```

```bash
timeout 600 bash -c 'until [ "$(docker inspect -f "{{.State.Health.Status}}" aera-mongo-prod 2>/dev/null)" = "healthy" ] && [ "$(docker inspect -f "{{.State.Health.Status}}" aera-redis-prod 2>/dev/null)" = "healthy" ] && [ "$(docker inspect -f "{{.State.Health.Status}}" aera-rabbitmq-prod 2>/dev/null)" = "healthy" ] && [ "$(docker inspect -f "{{.State.Health.Status}}" aera-api-prod 2>/dev/null)" = "healthy" ]; do echo "Waiting for backend health checks..."; sleep 5; done'
```

If the wait times out, inspect backend status/logs before starting client:

```bash
docker compose --env-file ../backend/.env.production -f ../backend/docker-compose.production.yml ps && docker compose --env-file ../backend/.env.production -f ../backend/docker-compose.production.yml logs --tail=200
```

Then wait 5 minutes before retrying backend startup + health wait:

```bash
sleep 300 && docker compose --env-file ../backend/.env.production -f ../backend/docker-compose.production.yml up -d && timeout 600 bash -c 'until [ "$(docker inspect -f "{{.State.Health.Status}}" aera-mongo-prod 2>/dev/null)" = "healthy" ] && [ "$(docker inspect -f "{{.State.Health.Status}}" aera-redis-prod 2>/dev/null)" = "healthy" ] && [ "$(docker inspect -f "{{.State.Health.Status}}" aera-rabbitmq-prod 2>/dev/null)" = "healthy" ] && [ "$(docker inspect -f "{{.State.Health.Status}}" aera-api-prod 2>/dev/null)" = "healthy" ]; do echo "Waiting for backend health checks (retry)..."; sleep 5; done'
```

Final guard (abort if any backend service is not healthy):

```bash
bash -ec '[ "$(docker inspect -f "{{.State.Health.Status}}" aera-mongo-prod 2>/dev/null)" = "healthy" ] && [ "$(docker inspect -f "{{.State.Health.Status}}" aera-redis-prod 2>/dev/null)" = "healthy" ] && [ "$(docker inspect -f "{{.State.Health.Status}}" aera-rabbitmq-prod 2>/dev/null)" = "healthy" ] && [ "$(docker inspect -f "{{.State.Health.Status}}" aera-api-prod 2>/dev/null)" = "healthy" ] || { echo "Backend is not fully healthy. Aborting before client startup."; exit 1; }'
```

```bash
AERA_SHARED_NETWORK=aera_network_prod docker compose --env-file ../client/.env.production -f ../client/docker-compose.prod.yml up -d
```

```bash
source prod/.env.edge.prod

docker run --rm -v aera_certbot_certs:/etc/letsencrypt alpine:3.20 sh -c "
  set -e
  apk add --no-cache openssl >/dev/null
  mkdir -p /etc/letsencrypt/live/${TLS_CERT_DOMAIN}
  openssl req -x509 -nodes -newkey rsa:2048 -days 1 \
    -keyout /etc/letsencrypt/live/${TLS_CERT_DOMAIN}/privkey.pem \
    -out /etc/letsencrypt/live/${TLS_CERT_DOMAIN}/fullchain.pem \
    -subj '/CN='${TLS_CERT_DOMAIN}
"
```

```bash
docker compose -f prod/docker-compose.edge.prod.yml up -d
```

```bash
make prod-cert-init EMAIL=<YOUR_EMAIL>
```

```bash
docker compose -f prod/docker-compose.edge.prod.yml restart aera-edge-prod
```

```bash
make status
curl -I http://test.aera.org.mw
curl -I https://test.aera.org.mw
```

---

## 8) Logs/troubleshooting

```bash
cd ~/apps/Website/proxy && docker compose -f prod/docker-compose.edge.prod.yml logs -f aera-edge-prod
```

```bash
cd ~/apps/Website/backend && docker compose -f docker-compose.production.yml logs -f
```

```bash
cd ~/apps/Website/client && docker compose -f docker-compose.prod.yml logs -f
```

---

## 9) Planned maintenance window (clean 503 page)

Use these from `~/apps/Website`:

```bash
./aera maintenance:on
./aera maintenance:status
```

When maintenance is complete:

```bash
./aera maintenance:off
./aera maintenance:status
```

Behavior:

- Users see a controlled maintenance page (`503 Service Unavailable`) instead of raw `502`/network errors.
- ACME challenge path remains available for certificate renewal.
- Toggle is file-flag based (`proxy/prod/nginx/maintenance/enabled`) and can be changed without stopping proxy.

---

## 10) Later cutover to main domain (same VPS, no DB migration)

Edit `~/apps/Website/proxy/prod/.env.edge.prod`:

```dotenv
PRIMARY_DOMAIN=aera.org.mw
SECONDARY_DOMAIN=test.aera.org.mw
# keep or change TLS_CERT_DOMAIN intentionally
```

Then:

```bash
cd ~/apps/Website/proxy
make prod-cert-init EMAIL=<YOUR_EMAIL>
make prod-restart
```

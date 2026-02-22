# AERA VPS Deployment Guide (Beginner Friendly)

This guide shows you how to:

1. SSH into your VPS using VS Code
2. Prepare the VPS safely for Docker deployment
3. Deploy the `backend`, `client`, and `proxy` stacks
4. Issue production TLS certificates for `test.aera.org.mw` (and later `aera.org.mw`)

---

## 0) What you need before starting

- VPS details from your provider:
    - IP address
    - SSH port
    - SSH username
    - SSH password
- Domain DNS control for `test.aera.org.mw`
- This project structure on your local machine:
    - `Website/backend`
    - `Website/client`
    - `Website/proxy`

> Security note: do **not** commit or share VPS passwords in files, screenshots, or chat logs.

---

## 1) Connect to VPS with VS Code (Remote SSH)

### Step 1. Install VS Code extension

- Install **Remote - SSH** (Microsoft).

### Step 2. Ensure local SSH client exists

- Linux/macOS:
    ```bash
    ssh -V
    ```
- Windows PowerShell:
    ```powershell
    ssh -V
    ```
    If command is missing, install OpenSSH client first.

### Step 3. Add SSH host config on your local machine

Edit `~/.ssh/config` and add:

```sshconfig
Host aera-vps
  HostName <VPS_IP>
  User <SSH_USERNAME>
  Port <SSH_PORT>
  PreferredAuthentications password
  PubkeyAuthentication yes
```

Then secure permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
```

### Step 4. Connect from VS Code

1. Press `F1`
2. Run: `Remote-SSH: Connect to Host...`
3. Choose `aera-vps`
4. Enter VPS password when prompted

You are now working directly on the VPS filesystem in VS Code.

---

## 2) First-time VPS preparation

Run these commands on the VPS terminal (inside VS Code Remote or normal SSH):

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install ca-certificates curl gnupg lsb-release git ufw fail2ban make
```

### Firewall (important)

Open SSH, HTTP, HTTPS:

```bash
sudo ufw allow <SSH_PORT>/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status
```

---

## 3) Install Docker Engine + Compose plugin

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Add your SSH user to docker group and re-login once:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker --version
docker compose version
```

---

## 4) Put project code on VPS

Choose one method.

### Option A (recommended): clone from Git

```bash
mkdir -p ~/apps/Website && cd ~/apps/Website
git clone https://github.com/Lusekero/Aera-Website-API.git backend
git clone https://github.com/Lusekero/Aera-Website-Client.git client
git clone https://github.com/Lusekero/Aera-Website-Proxy.git proxy
```

### Option B: upload existing folders

From local machine, copy your `Website` folder to VPS (scp/rsync).

Target structure on VPS should be:

```text
~/apps/Website/
  backend/
  client/
  proxy/
```

If you used Option A with `~/apps/Website` directly, you can skip this move step.
If you cloned into `~/apps` first, move repos into sibling layout:

```bash
mkdir -p ~/apps/Website
mv ~/apps/backend ~/apps/Website/
mv ~/apps/client ~/apps/Website/
mv ~/apps/proxy ~/apps/Website/
```

### Optional: install global aera command (recommended)

Bootstrap `aera` into root from Ops repo:

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

Run once on VPS so you can use `aera` from any directory:

```bash
cd ~/apps/Website
sudo ln -sf ~/apps/Website/aera /usr/local/bin/aera
```

Verify:

```bash
aera help
```

If the project path changes later, relink the command:

```bash
sudo rm -f /usr/local/bin/aera
sudo ln -sf ~/apps/Website/aera /usr/local/bin/aera
```

---

## 5) Configure production environment files

Create env files from examples first:

```bash
cd ~/apps/Website

# Backend
cp -n backend/.env.example backend/.env
cp -n backend/.env.production.example backend/.env.production

# Client
cp -n client/.env.example client/.env
cp -n client/.env.production.example client/.env.production

# Proxy (dev + prod)
cp -n proxy/dev/.env.edge.dev.example proxy/dev/.env.edge.dev
cp -n proxy/prod/.env.edge.prod.example proxy/prod/.env.edge.prod

# Optional local-prod variant
cp -n proxy/prod/.env.edge.prod.local.example proxy/prod/.env.edge.prod.local
```

`cp -n` avoids overwriting existing env files.

### Backend env

Create/edit:

- `~/apps/Website/backend/.env.production`

Ensure production DB, Redis, RabbitMQ, JWT, API keys, CORS/allowed-host values are correct.

### Client env

Create/edit:

- `~/apps/Website/client/.env.production`

Keep API base relative (`/api/v1`) for same-origin behind proxy.

### Proxy env

Create/edit:

- `~/apps/Website/proxy/prod/.env.edge.prod`

For test-first rollout, set:

```dotenv
PRIMARY_DOMAIN=test.aera.org.mw
SECONDARY_DOMAIN=aera.org.mw
TLS_CERT_DOMAIN=test.aera.org.mw
CLIENT_UPSTREAM=nuxt-prod:3000
API_UPSTREAM=aera-api-prod:4000
CLIENT_MAX_BODY_SIZE=200m
```

Later, when going live on main domain, swap primary/secondary and (optionally) keep TLS lineage stable.

### Unified env command style (recommended)

From `~/apps/Website`, manage backend/client/proxy env keys with one command pattern:

```bash
./aera env:set --target=backend --profile=prod --set JWT_SECRET='<jwt_secret>' --set REFRESH_TOKEN_ENCRYPTION_KEY='<refresh_secret>' --set PUBLIC_API_KEY='<public_api_key>'
./aera client:api-key:sync --profile=prod
./aera env:set --target=client --profile=prod --set NUXT_PUBLIC_API_BASE_URL=/api/v1 --set NUXT_PUBLIC_UPLOADS_PATH=/uploads
./aera env:set --target=proxy --profile=prod --set PRIMARY_DOMAIN=test.aera.org.mw --set TLS_CERT_DOMAIN=test.aera.org.mw

./aera env:get --target=backend --profile=prod --get JWT_SECRET --get PUBLIC_API_KEY
./aera env:unset --target=backend --profile=prod --unset OLD_LEGACY_SECRET
```

Use `--profile=dev` for development env files (`backend/.env`, `client/.env`, `proxy/dev/.env.edge.dev`).

---

## 6) DNS and pre-flight checks

Before certificate issuance, from your local machine:

```bash
dig +short test.aera.org.mw
```

This **must** return your VPS public IP.

Also verify ports are reachable (80/443 open in cloud firewall + UFW).

---

## 7) Production deployment command sequence

All commands below run on VPS:

```bash
cd ~/apps/Website/proxy
```

### Step 1. Create shared prod Docker network (one-time)

```bash
docker network create aera_network_prod || true
```

### Step 2. Start backend + client only

```bash
docker compose --env-file ../backend/.env.production -f ../backend/docker-compose.production.yml up -d
AERA_SHARED_NETWORK=aera_network_prod docker compose --env-file ../client/.env.production -f ../client/docker-compose.prod.yml up -d
```

### Step 3. One-time TLS bootstrap (prevents first-start cert-file crash)

Your prod Nginx expects cert files at `/etc/letsencrypt/live/<TLS_CERT_DOMAIN>/...`.
Create temporary self-signed files in the cert volume first:

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

### SMTP via cPanel (no Google Workspace/M365 required)

In cPanel:

1. Go to `Email Accounts` for your mailbox (for example `info@aera.org.mw`)
2. Open `Connect Devices` / `Set Up Mail Client`
3. Copy Outgoing (SMTP) values: host, port, encryption, username, password

Set in backend production env (`~/apps/Website/backend/.env.production`):

```dotenv
EMAIL_HOST=<cpanel_smtp_host>
EMAIL_PORT=<465_or_587>
EMAIL_SECURE=<true_for_465_or_false_for_587>
EMAIL_USER=<full_mailbox_address>
EMAIL_PASSWORD=<mailbox_password>
EMAIL_FROM='"Atomic Energy Regulatory Authority" <info@aera.org.mw>'
```

Apply quickly with unified CLI:

```bash
cd ~/apps/Website
./aera smtp:configure --provider=cpanel --host=<cpanel_smtp_host> --port=<465_or_587> --secure=<true_or_false> --user=<full_mailbox_address> --password='<mailbox_password>' --from='"Atomic Energy Regulatory Authority" <info@aera.org.mw>'
./aera smtp:show
./aera smtp:test --to=<your_test_email>
```

Switching later to another provider (example Microsoft 365) is one command + restart:

```bash
cd ~/apps/Website
./aera smtp:configure --provider=m365 --user=<full_mailbox_address> --password='<mailbox_password>' --from='"Atomic Energy Regulatory Authority" <info@aera.org.mw>'
./aera smtp:test --to=<your_test_email>
```

The `smtp:configure` command updates `.env.production` and recreates only backend runtime services (`aera-api`, `aera-worker`) by default.
Advanced option: use `--worker-only` to recreate only `aera-worker` when you are certain email sending is worker-only.

### Step 4. Start prod edge

```bash
docker compose -f prod/docker-compose.edge.prod.yml up -d
```

### Step 5. Issue real Let's Encrypt cert

```bash
make prod-cert-init EMAIL=<YOUR_EMAIL>
```

### Step 6. Restart edge to load real cert

```bash
docker compose -f prod/docker-compose.edge.prod.yml restart aera-edge-prod
```

### Optional: maintenance mode during planned changes

From `~/apps/Website`:

```bash
./aera maintenance:on
./aera maintenance:status
```

Disable when done:

```bash
./aera maintenance:off
./aera maintenance:status
```

This keeps edge online but serves a controlled `503` maintenance page instead of raw gateway errors.

### Step 7. Verify services

```bash
make status
curl -I http://test.aera.org.mw
curl -I https://test.aera.org.mw
```

Expected:

- HTTP redirects to HTTPS
- HTTPS returns `200` or expected app response

---

## 8) Ongoing operations

### View logs

```bash
cd ~/apps/Website/proxy
docker compose -f prod/docker-compose.edge.prod.yml logs -f aera-edge-prod
```

```bash
cd ~/apps/Website/backend
docker compose -f docker-compose.production.yml logs -f
```

```bash
cd ~/apps/Website/client
docker compose -f docker-compose.prod.yml logs -f
```

### Restart full prod stack (from proxy)

```bash
cd ~/apps/Website/proxy
make prod-restart
```

---

## 9) Switch from test domain to live domain later (no DB migration)

When ready:

1. Update DNS so `aera.org.mw` points to same VPS
2. Edit `proxy/prod/.env.edge.prod`:
    - `PRIMARY_DOMAIN=aera.org.mw`
    - `SECONDARY_DOMAIN=test.aera.org.mw`
    - Keep or change `TLS_CERT_DOMAIN` intentionally
3. Re-issue certificate including both domains:

```bash
cd ~/apps/Website/proxy
make prod-cert-init EMAIL=<YOUR_EMAIL>
make prod-restart
```

No database migration is needed because data stays in the same Docker volumes on the same VPS.

---

## 10) Recommended security follow-up (after first successful deploy)

- Set up SSH key login and disable password login
- Disable root SSH login
- Keep automatic security updates enabled
- Add regular backups for DB + uploads volumes
- Add uptime monitoring and alerting

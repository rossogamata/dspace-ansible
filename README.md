# DSpace 9 Ansible Deployment

Ansible automation for a full DSpace 9.x stack on a single Ubuntu 22.04/24.04 server.
Covers PostgreSQL 16, Apache Solr 9.8, DSpace backend (Spring Boot), Angular SSR frontend (PM2), and Nginx with Let's Encrypt TLS.

---

## Table of Contents

1. [Architecture overview](#architecture-overview)
2. [Repository structure](#repository-structure)
3. [Component versions](#component-versions)
4. [Prerequisites](#prerequisites)
5. [First-time setup](#first-time-setup)
6. [Running the playbook](#running-the-playbook)
7. [Key variables reference](#key-variables-reference)
8. [Role descriptions](#role-descriptions)
9. [How-to guides](#how-to-guides)
10. [Debug procedures](#debug-procedures)

---

## Architecture overview

```
Browser / Client
      │ HTTPS 443
      ▼
   Nginx (reverse proxy + TLS termination)
      ├── /server  ──► DSpace Backend  (Spring Boot, port 8080)
      │                     │
      │                PostgreSQL 16 (port 5432)
      │                Apache Solr 9.8 (port 8983)
      │
      └── /       ──► Angular SSR frontend (PM2 cluster, port 4000)
                            │ SSR requests back to backend via localhost
                            └── http://localhost:8080/server
```

The Angular frontend performs server-side rendering (SSR). Its internal requests
always go to `localhost:8080` via `/etc/hosts` pinning, not through Nginx, avoiding
TLS cert issues on intranet deployments.

---

## Repository structure

```
dspace-ansible/
├── site.yml                        # Main playbook — runs all roles in order
├── ansible.cfg                     # Ansible defaults (inventory, remote user, etc.)
├── requirements.yml                # Ansible Galaxy dependencies
│
├── inventory/
│   └── hosts.yml                   # Target host(s) and SSH connection vars
│
├── group_vars/
│   ├── all.yml                     # All public variables (versions, paths, URLs)
│   ├── vault.yml                   # Encrypted secrets (do not commit plaintext)
│   └── vault.yml.example           # Template — copy and fill in, then encrypt
│
├── roles/
│   ├── common/                     # Base packages, Java 17, Maven 3.9, Node.js 22, PM2
│   ├── postgresql/                 # PostgreSQL 16 (PGDG repo), dspace DB + user
│   ├── solr/                       # Apache Solr 9.8 system service
│   ├── dspace_backend/             # Maven build, ant install, systemd service
│   ├── dspace_frontend/            # npm install, ng build, PM2 start
│   └── nginx/                      # Nginx config, optional Let's Encrypt cert
│
└── README/
    ├── ansible-usage.md            # Step-by-step Ansible workflow
    └── manual-install.md           # Manual install walkthrough (useful for debugging)
```

### Role internals

Each role follows the standard Ansible layout:

```
roles/<name>/
├── tasks/main.yml       # All tasks
├── defaults/main.yml    # Role-level variable defaults
├── handlers/main.yml    # Service restart / reload handlers
└── templates/           # Jinja2 config file templates (.j2)
```

---

## Component versions

| Component         | Version   | Notes                                              |
|-------------------|-----------|----------------------------------------------------|
| DSpace backend    | 9.1       | Spring Boot 3.5.3, Java 17                        |
| DSpace frontend   | 9.1       | Angular 18, SSR via `@angular/ssr`                |
| Java              | 17 (JDK)  | `openjdk-17-jdk` from Ubuntu repos               |
| Apache Maven      | 3.9.14    | Manual install — apt ships 3.6 which is too old  |
| Apache Solr       | 9.8.0     | Installed via `install_solr_service.sh`           |
| PostgreSQL        | 16        | Installed from PGDG apt repo                      |
| Node.js           | 22.x      | Installed via NodeSource setup script             |
| PM2               | latest    | npm global install, cluster mode                  |
| Nginx             | latest    | From Ubuntu repos                                 |
| Certbot           | latest    | `python3-certbot-nginx` plugin                   |

---

## Prerequisites

**Control machine** (where you run Ansible):

- Ansible 2.14+
- `ansible-galaxy install -r requirements.yml` (installs `community.general`, `ansible.posix`, `community.postgresql`)
- SSH access to the target server (key-based recommended)

**Target server:**

- Ubuntu 22.04 or 24.04, fresh install
- 4 GB RAM minimum (Maven build needs ~2–3 GB)
- 20 GB disk free (source + build artifacts + Solr data)
- Port 80 and 443 open (required for Let's Encrypt)
- Resolvable FQDN pointing to the server's public IP (required for Let's Encrypt)

---

## First-time setup

### 1. Configure inventory

Edit `inventory/hosts.yml` — replace the example host with your server:

```yaml
all:
  hosts:
    dspace-server:
      ansible_host: 192.168.200.33
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
      ansible_become: true
      ansible_become_method: sudo
      ansible_become_pass: "{{ vault_sudo_pass }}"
```

### 2. Configure variables

Edit `group_vars/all.yml` — at minimum set these values:

```yaml
dspace_server_url: "https://dspace.example.com/server"
dspace_ui_url:     "https://dspace.example.com"
dspace_name:       "My DSpace Repository"

dspace_admin_email:     "admin@example.com"
dspace_admin_firstname: "DSpace"
dspace_admin_lastname:  "Administrator"

nginx_backend_fqdn:       "dspace.example.com"
nginx_frontend_fqdn:      "dspace.example.com"
nginx_use_letsencrypt:    true
nginx_letsencrypt_email:  "admin@example.com"
```

### 3. Set up secrets

```bash
cp group_vars/vault.yml.example group_vars/vault.yml
# Edit vault.yml and fill in real values:
#   vault_db_password, vault_dspace_admin_password, vault_sudo_pass
ansible-vault encrypt group_vars/vault.yml
```

### 4. Install Galaxy dependencies

```bash
ansible-galaxy install -r requirements.yml
```

---

## Running the playbook

### Full deploy

```bash
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass
```

Or with a vault password file (non-interactive, useful for CI):

```bash
ansible-playbook -i inventory/hosts.yml site.yml --vault-password-file ~/.vault_pass.txt
```

### Dry run

```bash
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass --check --diff
```

### Run a single role (via tags)

```bash
# Only reconfigure nginx
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass --tags nginx

# Only reconfigure frontend
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass --tags dspace_frontend
```

### Force full rebuild

Set `dspace_force_reinstall: true` in `group_vars/all.yml` (or pass as extra var),
then re-run. This triggers a Maven rebuild + `ant update` even if artifacts already exist.

```bash
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass \
  -e dspace_force_reinstall=true
```

---

## Key variables reference

All variables live in `group_vars/all.yml`. Secrets are in `group_vars/vault.yml` (encrypted).

| Variable | Default | Description |
|---|---|---|
| `dspace_version` | `9.1` | DSpace backend + frontend version |
| `dspace_dir` | `/dspace` | Final installation directory |
| `dspace_build_dir` | `/opt/dspace-build` | Temporary source + build area |
| `dspace_ui_dir` | `/opt/dspace-ui` | Angular production deployment root |
| `dspace_server_url` | — | Public HTTPS URL for the REST API |
| `dspace_ui_url` | — | Public HTTPS URL for the frontend |
| `dspace_name` | — | Repository display name |
| `dspace_force_reinstall` | `false` | Force Maven rebuild even if jar exists |
| `maven_version` | `3.9.14` | Maven version to install manually |
| `postgresql_version` | `16` | PostgreSQL major version |
| `db_name` | `dspace` | Database name |
| `db_username` | `dspace` | Database user |
| `db_password` | `{{ vault_db_password }}` | From vault |
| `solr_version` | `9.8.0` | Apache Solr version |
| `dspace_ui_port` | `4000` | Port the Angular SSR process listens on |
| `dspace_ui_pm2_instances` | `max` | PM2 cluster workers (or a number) |
| `nginx_backend_fqdn` | — | FQDN for the backend (`/server`) |
| `nginx_frontend_fqdn` | — | FQDN for the frontend (`/`) |
| `nginx_use_letsencrypt` | `true` | Obtain a cert from Let's Encrypt |
| `nginx_letsencrypt_email` | — | Email for cert expiry notifications |
| `nginx_ssl_cert` | — | Path to cert (when not using LE) |
| `nginx_ssl_key` | — | Path to key (when not using LE) |
| `max_upload_size` | `512MB` | Max file upload size (nginx + Spring Boot) |
| `handle_prefix` | `123456789` | DSpace Handle prefix |

---

## Role descriptions

### `common`

Runs first on every host. Installs:

- Base packages (`curl`, `wget`, `git`, `unzip`, `ca-certificates`, Java 17, Ant)
- Apache Maven 3.9.14 (downloaded from `downloads.apache.org`, symlinked to `/opt/maven`, added to `PATH`)
- Node.js 22.x (via NodeSource setup script)
- PM2 (global npm install)
- `/etc/hosts` entry: `127.0.0.1 <nginx_frontend_fqdn>` — required so Angular SSR's
  internal HTTP requests resolve to `localhost` instead of an external IP

### `postgresql`

- Adds the [PGDG](https://apt.postgresql.org/) apt repository and signing key (Ubuntu ships PG 14; PG 16 requires PGDG)
- Installs `postgresql-16`
- Creates the `dspace` database and user with the vaulted password

### `solr`

- Downloads Apache Solr 9.8.0 tarball and runs `install_solr_service.sh`
  (creates `solr` OS user, installs to `/opt/solr-9.8.0`, registers systemd service)
- Templates `/etc/default/solr.in.sh` with `SOLR_OPTS=-Dsolr.config.lib.enabled=true`
  (required for DSpace's `<lib>` tags in `solrconfig.xml`)
- Solr cores are created by the `dspace_backend` role after `ant fresh_install`

### `dspace_backend`

- Downloads DSpace 9.1 source tarball from GitHub
- Templates `local.cfg` and `local.properties` (identical content; `.properties`
  extension required by Spring Boot 3.5 which cannot load `.cfg`)
- Runs `mvn package` (15–30 minutes, skipped if jar exists)
- Runs `ant fresh_install` (first time) or `ant update` (forced reinstall)
- Runs `dspace database migrate`
- Copies DSpace Solr configsets to `/var/solr/data` and creates Solr cores
- Creates the DSpace administrator account (idempotent via DB check)
- Installs and starts the `dspace-backend` systemd service

The systemd service passes critical config via JVM `-D` flags (highest priority in
Spring Boot, overrides any file-based config):

```
-Ddspace.dir=... -Ddspace.server.url=... -Ddspace.ui.url=...
-Drest.cors.allowed-origins=... -Dsolr.server=...
--spring.config.import=file:///dspace/config/local.properties
```

### `dspace_frontend`

- Downloads the `dspace-angular` 9.1 source tarball from GitHub
- Runs `npm install`
- Builds the browser bundle: `ng build --configuration production`
- Builds the SSR server bundle: `ng run dspace-angular:server:production`
  (produces `dist/server/main.js`)
- Syncs `dist/` to `/opt/dspace-ui/dist/`
- Templates `config.prod.yml` (backend URL, SSR listen address `0.0.0.0:4000`)
- Templates `dspace-ui.json` PM2 ecosystem file with `NODE_TLS_REJECT_UNAUTHORIZED=0`
  (required for intranet self-signed / internal CA certs)
- Starts the app with PM2 in cluster mode and saves the process list for reboots

### `nginx`

Two-phase deployment to handle the Let's Encrypt chicken-and-egg problem:

1. Deploy a minimal HTTP-only config → start nginx → run `certbot certonly --nginx`
2. Deploy the full HTTPS config with the obtained cert → reload nginx

On subsequent runs the cert exists so phase 1 is skipped entirely.

The Nginx config merges backend and frontend into a single `server {}` block when
`nginx_backend_fqdn == nginx_frontend_fqdn`:

```
location /server { proxy_pass http://127.0.0.1:8080; }
location /       { proxy_pass http://127.0.0.1:4000; }
```

---

## How-to guides

### Renew the Let's Encrypt certificate manually

```bash
sudo certbot renew --nginx
sudo systemctl reload nginx
```

Certbot auto-renewal is registered by the installer — check with `systemctl status certbot.timer`.

### Upgrade DSpace to a new version

1. Update `dspace_version` (and `dspace_angular_version` if different) in `group_vars/all.yml`
2. Set `dspace_force_reinstall: true`
3. Run the playbook — Maven will rebuild and `ant update` will be called

### Change a config value without rebuilding

Most runtime values (URLs, passwords, Solr URL, upload size) are passed as JVM `-D`
flags in the systemd service and as `local.properties` via Spring Boot config import.
Just re-run the playbook (or only the `dspace_backend` tag) and the service will restart
with the new values — no Maven rebuild needed.

### Use your own TLS certificate instead of Let's Encrypt

In `group_vars/all.yml`:

```yaml
nginx_use_letsencrypt: false
nginx_ssl_cert: /etc/ssl/certs/my-dspace.crt
nginx_ssl_key:  /etc/ssl/private/my-dspace.key
```

Copy your cert and key to those paths on the server before running the playbook.

### Deploy frontend changes only

```bash
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass \
  --tags dspace_frontend -e dspace_force_reinstall=true
```

### Check the PM2 process list

```bash
sudo -u dspace pm2 list
sudo -u dspace pm2 logs dspace-ui --lines 50
```

---

## Debug procedures

### Backend not starting

```bash
# Service status
sudo systemctl status dspace-backend

# Full journal logs (last 200 lines)
sudo journalctl -u dspace-backend -n 200 --no-pager

# Check if port 8080 is listening
ss -tlnp | grep 8080
```

Common causes:
- **Spring Boot startup timeout**: `TimeoutStartSec=180` — if the machine is slow the service may be killed before it's ready. Increase the value in the unit file.
- **`.cfg` extension error** (`File extension is not known to any PropertySourceLoader`): the `--spring.config.import` flag must point to a `.properties` file, not `.cfg`.
- **Wrong Solr URL**: check `-Dsolr.server=` in `journalctl` output matches the running Solr.
- **DB connection refused**: verify PostgreSQL is running and the `dspace` user can connect: `PGPASSWORD=<pwd> psql -h 127.0.0.1 -U dspace -d dspace -c 'SELECT 1'`

### Frontend (PM2) not starting

```bash
# PM2 process list
sudo -u dspace pm2 list

# Application logs
sudo -u dspace pm2 logs dspace-ui --lines 100

# Structured log files
sudo tail -f /var/log/dspace/dspace-ui-error.log
sudo tail -f /var/log/dspace/dspace-ui-out.log

# Check if port 4000 is listening
ss -tlnp | grep 4000
```

Common causes:
- **Script not found** (`dist/server/main.js`): the SSR build did not complete. Check that `ng run dspace-angular:server:production` succeeded during the playbook run.
- **EADDRNOTAVAIL** on port 4000: `ui.host` in `config.prod.yml` is set to the server's FQDN (not a local interface). Must be `0.0.0.0`.
- **500 error** ("undefined doesn't contain link sites"): SSR cannot reach the backend. Check that `/etc/hosts` has `127.0.0.1 <nginx_frontend_fqdn>` and the backend is responding on port 8080.
- **SSL errors in PM2 logs** (UNABLE_TO_VERIFY_LEAF_SIGNATURE): `NODE_TLS_REJECT_UNAUTHORIZED` is not `0` in the PM2 ecosystem file. Check `dspace-ui.json`.

### Nginx not loading / 502 Bad Gateway

```bash
# Test config syntax
sudo nginx -t

# Service status + error log
sudo systemctl status nginx
sudo tail -f /var/log/nginx/error.log

# Reload without downtime
sudo systemctl reload nginx
```

Common causes:
- **502 on `/server`**: backend on port 8080 is down. Check `dspace-backend` service.
- **502 on `/`**: PM2 on port 4000 is down. Check PM2.
- **"welcome to nginx" on `/`**: two server blocks with the same `server_name` and no `location /` in the backend block. The fix is to merge them into one block.

### Solr not available

```bash
sudo systemctl status solr
sudo journalctl -u solr -n 100 --no-pager

# Test Solr HTTP API
curl http://localhost:8983/solr/admin/cores?action=STATUS
```

Common causes:
- `/var/solr/logs` directory does not exist (created by the `solr` role but may be missing on fresh installs where the install script ran before the role).
- `SOLR_OPTS` missing `solr.config.lib.enabled=true` — DSpace configsets use `<lib>` tags that Solr 9.8 blocks by default.

### PostgreSQL connection issues

```bash
sudo systemctl status postgresql
sudo -u postgres psql -c '\l'

# Verify dspace user and DB
PGPASSWORD=<pwd> psql -h 127.0.0.1 -U dspace -d dspace -c 'SELECT version();'

# Check pg_hba.conf allows md5/scram for local connections
sudo cat /etc/postgresql/16/main/pg_hba.conf
```

### Full stack health check

```bash
# All four services should be active (running)
systemctl is-active postgresql solr dspace-backend nginx

# Ports in use
ss -tlnp | grep -E '5432|8983|8080|4000|443|80'

# Quick HTTP smoke tests
curl -sk https://localhost/server/api | python3 -m json.tool | head -20
curl -sk https://localhost/ | grep -i dspace
```

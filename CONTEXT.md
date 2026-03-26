# DSpace Ansible — AI Context

This file captures the full context needed to continue development of this
Ansible project in a new AI conversation. Read it before making changes.

---

## What this project does

Automates a complete DSpace 9.x installation on a **single Ubuntu 24.04 host**.
It installs and configures all components end-to-end:

| Component | Implementation |
|---|---|
| Java 17, Maven, Ant | Ubuntu `apt` (openjdk-17-jdk, maven, ant) |
| Node.js 22.x | NodeSource repo |
| PM2 | global `npm install -g pm2` |
| PostgreSQL 16 | Ubuntu `apt` (version included in Ubuntu 24.04) |
| Apache Solr 9.8 | Official `install_solr_service.sh` → systemd |
| DSpace backend (REST API) | Built from GitHub source tarball → systemd |
| DSpace frontend (Angular) | Built from GitHub source tarball → PM2 cluster |
| Nginx | Reverse proxy with TLS; optional Let's Encrypt |

---

## Repository structure

```
dspace-ansible/
├── ansible.cfg                  # inventory path, stdout_callback=yaml
├── requirements.yml             # community.postgresql + community.general
├── site.yml                     # single play: hosts=dspace, all 6 roles
├── inventory/
│   └── hosts.yml                # default: localhost (ansible_connection=local)
├── group_vars/
│   ├── all.yml                  # ALL variables — edit this first
│   └── vault.yml.example        # copy → vault.yml, then ansible-vault encrypt
└── roles/
    ├── common/          # Java, Maven, Ant, Node.js 22, PM2, dspace OS user
    ├── postgresql/      # install PG 16, configure, create dspace DB + user
    ├── solr/            # install Solr 9.8 via service script, SOLR_OPTS
    ├── dspace_backend/  # download source, mvn build, ant install, systemd
    ├── dspace_frontend/ # download angular source, npm build, PM2
    └── nginx/           # reverse proxy, optional Let's Encrypt
```

Each role has: `tasks/main.yml`, `handlers/main.yml`, `defaults/main.yml`,
and `templates/` where needed.

---

## Source of DSpace code

This project does **not bundle** DSpace source code.
It downloads releases directly from GitHub at run time:

- **Backend:** `https://github.com/DSpace/DSpace/releases/download/dspace-{{ dspace_version }}/dspace-{{ dspace_version }}-src-release.tar.gz`
- **Frontend:** `https://github.com/DSpace/dspace-angular/releases/download/dspace-{{ dspace_version }}/dspace-angular-{{ dspace_version }}.tar.gz`

The DSpace source repo lives at `/home/enduro/viti/DSpace` (DSpace 9.2,
backend only — no Angular). Key files:
- `dspace/config/local.cfg.EXAMPLE` — template for all config keys
- `dspace/config/dspace.cfg` — all defaults (85 KB)
- `docker-compose.yml` — shows Solr 9.8, 6 cores, env variable encoding
- `Install.md` — hand-written full installation guide (created alongside this project)

---

## Key design decisions

1. **Idempotency:** Maven build and `ant fresh_install` are skipped if
   `{{ dspace_dir }}/webapps/server-boot.jar` already exists.
   Set `dspace_force_reinstall: true` to re-run them (runs `ant update` on
   reinstall, not `ant fresh_install`).

2. **Solr cores:** DSpace 9.2 uses **6 cores** (not 4):
   `search`, `statistics`, `oai`, `authority`, `qaevent`, `suggestion`.
   Configsets are copied from `{{ dspace_dir }}/solr/` after `ant fresh_install`,
   then each core is created via `solr create` CLI.

3. **Solr 9.8 mandatory flag:** `solr.config.lib.enabled=true` must be in
   `SOLR_OPTS` (set in `/etc/default/solr.in.sh` via template).

4. **DSpace environment variable encoding:**
   Dots → `__P__`, dashes → `__D__` (used in Docker env; not needed in Ansible
   since we template `local.cfg` directly).

5. **Admin account creation:** Uses `dspace create-administrator -e -f -l -p -c`.
   Guarded by a `psql` count query so it's idempotent.

6. **Backend service:** Spring Boot runnable JAR at
   `{{ dspace_dir }}/webapps/server-boot.jar` → systemd unit `dspace-backend`.

7. **Frontend service:** PM2 cluster mode (`instances: max`) pointing to
   `dist/server/main.js` (Angular SSR entry point).

8. **Nginx:** Single config file handles both `nginx_backend_fqdn` (port 443 → 8080)
   and `nginx_frontend_fqdn` (port 443 → 4000). `nginx_use_letsencrypt: true`
   triggers certbot automatically.

9. **Secrets:** `db_password` and `dspace_admin_password` come from
   `group_vars/vault.yml` (ansible-vault encrypted). Referenced as
   `{{ vault_db_password }}` and `{{ vault_dspace_admin_password }}`.

---

## Variables — the ones that MUST be changed before first run

All in `group_vars/all.yml`:

```yaml
dspace_server_url: "https://api.example.com/server"   # real FQDN
dspace_ui_url:     "https://www.example.com"          # real FQDN
dspace_name:       "DSpace at My University"
dspace_admin_email: "admin@example.com"
nginx_backend_fqdn: "api.example.com"
nginx_frontend_fqdn: "www.example.com"
nginx_use_letsencrypt: false   # set to true if DNS is live
```

And `group_vars/vault.yml` (from vault.yml.example):
```yaml
vault_db_password: "strongpassword"
vault_dspace_admin_password: "strongpassword"
```

---

## How to run

```bash
# 1. Install Ansible collections
ansible-galaxy collection install -r requirements.yml

# 2. Set up inventory (already defaults to localhost)
# For a remote server, edit inventory/hosts.yml

# 3. Set up vault
cp group_vars/vault.yml.example group_vars/vault.yml
# edit vault.yml with real passwords
ansible-vault encrypt group_vars/vault.yml

# 4. Edit group_vars/all.yml (URLs, FQDNs, site name)

# 5. Run
ansible-playbook site.yml --ask-vault-pass

# Run a single role only
ansible-playbook site.yml --tags postgresql --ask-vault-pass

# Force rebuild after an upgrade
ansible-playbook site.yml -e dspace_force_reinstall=true --ask-vault-pass
```

---

## What is NOT yet done / known gaps

- **Cron tasks:** Scheduled DSpace maintenance tasks (index-discovery, filter-media,
  subscription-send, etc.) are documented in `../DSpace/Install.md` §11 but not
  yet implemented in this Ansible project. A `cron` role should be added.
- **Assetstore backup:** Not configured. Production deployments need an NFS mount
  or S3 assetstore (`dspace/config/modules/assetstore.cfg`).
- **GeoIP database:** Usage statistics by location requires a MaxMind
  `GeoLite2-City.mmdb` file. Not automated here.
- **`ansible.posix` collection:** `dspace_frontend` role uses
  `ansible.posix.synchronize`. Add `ansible.posix` to `requirements.yml` if needed.
- **Handle server:** Handle.net registration and server setup not included.
- **SAML/Shibboleth/OIDC auth:** Only password auth is enabled by default.
  Additional auth modules are in `dspace/config/modules/authentication-*.cfg`.
- **PostgreSQL 16 assumption:** `postgresql_conf_dir` defaults to
  `/etc/postgresql/16/main`. If Ubuntu ships a different version, update
  `postgresql_version` in `group_vars/all.yml`.
- **Single-domain variant:** Current nginx config uses two separate FQDNs (one
  for backend, one for frontend). A single-FQDN setup (`example.com/server` and
  `example.com/`) is possible — adjust the nginx template.

---

## Related files in the DSpace source repo

| File | Purpose |
|---|---|
| `/home/enduro/viti/DSpace/Install.md` | Hand-written installation guide (reference) |
| `/home/enduro/viti/DSpace/docker-compose.yml` | Shows Solr 9.8, 6 cores, env variable syntax |
| `/home/enduro/viti/DSpace/dspace/config/local.cfg.EXAMPLE` | Authoritative list of all config keys |
| `/home/enduro/viti/DSpace/dspace/config/dspace.cfg` | All defaults |

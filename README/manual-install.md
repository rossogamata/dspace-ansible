# Manual Installation (DSpace)

This repository mainly contains Ansible automation for a DSpace deployment. Manual installation is not the primary workflow, but these notes can help understand what gets provisioned.

## 1. Prerequisites

- Linux VM (Ubuntu 22.04+ recommended)
- `openjdk-17-jdk`
- `postgresql` and `postgresql-contrib`
- `nginx`
- `tomcat` (or similar application server if used by DSpace version)
- `curl`, `wget`, `unzip`, `git`

## 2. PostgreSQL

1. Create user and database:

```bash
sudo -u postgres createuser -P dspace_user
sudo -u postgres createdb -O dspace_user dspace
```

2. Ensure `pg_hba.conf` allows local password auth, then `sudo systemctl restart postgresql`.

## 3. Solr

1. Download Solr and configure core for DSpace.
2. Adjust `solr.in.sh` or equivalent env file for memory tuning.

## 4. DSpace application

1. Clone DSpace source or download a release.
2. Configure `${dspace.install.dir}/config/local.cfg` (or new config format) with DB/Solr endpoints.
3. Build with Maven: `mvn package -DskipTests`.
4. Deploy built webapp artifacts to the app server.

## 5. NGINX frontend

1. Create reverse proxy rule to backend and UI ports.
2. Enable config, test with `nginx -t`, and restart.

## 6. Verify

- Access UI: `http://<host>/`.
- Backend API: `http://<host>/rest/`.

> NOTE: This repo encodes these steps in Ansible roles; prefer that method for repeatable, idempotent deployment.

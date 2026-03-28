# Manual Installation (DSpace)

This repository uses Ansible for automation. The manual steps below capture roughly what happens under the hood and offer a fallback for debugging.

## 1. Prerequisites (Ubuntu 22.04+)

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk git curl wget unzip maven nginx postgresql postgresql-contrib
```

Optionally install a servlet container (Tomcat 10 example):

```bash
sudo apt install -y tomcat10
```

Confirm Java:

```bash
java -version
mvn -v
```

## 2. PostgreSQL setup

1. Set password and database:

```bash
sudo -u postgres psql -c "CREATE USER dspace_user WITH PASSWORD 'dspace_password';"
sudo -u postgres psql -c "CREATE DATABASE dspace OWNER dspace_user;"
```

2. Install required PostgreSQL extensions:

```bash
sudo -u postgres psql -d dspace -c "CREATE EXTENSION pgcrypto;"
```

3. Edit PostgreSQL auth for local password login:

```bash
sudo sed -i "s/^local.*all.*all.*peer/local all all md5/" /etc/postgresql/*/main/pg_hba.conf
sudo systemctl restart postgresql
```

4. Verify DB connection:

```bash
PGPASSWORD=dspace_password psql -h localhost -U dspace_user -d dspace -c 'SELECT 1;'
```

## 3. Solr setup

1. Download and install Solr:

```bash
wget https://downloads.apache.org/lucene/solr/9.9.0/solr-9.9.0.tgz
tar xzf solr-9.9.0.tgz
sudo bash solr-9.9.0/bin/install_solr_service.sh solr-9.9.0.tgz
sudo systemctl enable --now solr
```

2. Create DSpace core (example `dspace`):

```bash
sudo su - solr -c "/opt/solr/bin/solr create -c dspace -n data_driven_schema_configs"
```

3. Adjust `solr.in.sh` memory settings if needed (e.g., `SOLR_JAVA_MEM="-Xms512m -Xmx1024m"`).

## 4. DSpace application build & config

1. Clone DSpace source:

```bash
git clone https://github.com/DSpace/DSpace.git
cd DSpace
git checkout dspace-9.1
```

2. Configure DSpace: copy template and set DB/Solr:

```bash
cp [dspace]/config/local.cfg [dspace]/config/local.cfg.bak
# (edit the file; if using new DSpace 7, use config/dspace.cfg and curation-ui config as applicable)
```

Example values in `config/local.cfg`:

```ini
db.server=localhost
db.name=dspace
db.username=dspace_user
db.password=dspace_password
solr.server=http://localhost:8983/solr/dspace
```

3. Build:

```bash
mvn -U package -DskipTests
```

4. Install:

```bash
mkdir -p /opt/dspace
cp -r [dspace]/dspace/target/dspace-*-release /opt/dspace
# adapt to dspace version and target paths
```

## 5. Backend and frontend services

1. Deploy backend webapp to Tomcat (or your container):

```bash
sudo cp [dspace]/dspace/target/dspace/WEB-INF/lib/*.war /var/lib/tomcat10/webapps/
sudo systemctl restart tomcat10
```

2. Deploy frontend UI as static content under nginx or a front-end server.

## 6. NGINX reverse proxy (example)

Create `/etc/nginx/sites-available/dspace`:

```nginx
server {
  listen 80;
  server_name dspace.example.com;

  location / {
    proxy_pass http://localhost:8080/; # backend
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location /static/ {
    root /opt/dspace/ui; # frontend static
  }
}
```

Enable and test:

```bash
sudo ln -s /etc/nginx/sites-available/dspace /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## 7. Verify

```bash
curl -I http://localhost/
curl -I http://localhost/rest/
sudo systemctl status postgresql solr tomcat10 nginx
```

## 8. Vault-aware configuration

The preferred approach for production deployments in this repo is using Ansible Vault for secrets (DB password, admin password, sudo password):

- Copy vault example and encrypt:
  - `cp group_vars/vault.yml.example group_vars/vault.yml`
  - `ansible-vault encrypt group_vars/vault.yml`
- Add secrets in `group_vars/vault.yml` (e.g., `vault_db_password`, `vault_dspace_admin_password`, `vault_sudo_pass`).
- Use `--ask-vault-pass` or `--vault-password-file` when running playbooks.

## Latest updates

- Added explicit `ansible_become_pass` binding to `vault_sudo_pass` in inventory/hosts.yml.
- Added a reusable nginx reverse proxy example and service verification commands.

> NOTE: This repository’s Ansible playbook automates these steps with idempotency and role-based variables. Use `ansible-usage.md` for preferred workflow.


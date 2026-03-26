# Ansible Usage (DSpace deployment)

This repository is built around Ansible roles that configure and deploy DSpace infrastructure:
- `roles/postgresql` - database service
- `roles/solr` - Solr search service
- `roles/dspace_backend` - DSpace Java backend service
- `roles/dspace_frontend` - DSpace UI service
- `roles/nginx` - web reverse proxy

## 1. Requirements

- Control host with Ansible 2.14+
- Inventory file: `inventory/hosts.yml`
- Optional group vars in `group_vars/all.yml` and `group_vars/vault.yml.example`

Install dependencies:

```bash
ansible-galaxy install -r requirements.yml
```

## 2. Configure inventory & vars

- Edit `inventory/hosts.yml` with target hosts and SSH connection info.
- Copy `group_vars/vault.yml.example` to `group_vars/vault.yml` and set secrets (DB passwords, etc.).

## 3. Run playbook

```bash
ansible-playbook -i inventory/hosts.yml site.yml
```

For a dry run:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --check
```

## 4. Role-level execution

Run only a single role if needed:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --tags postgresql
```

(Requires tags in playbook tasks if used.)

## 5. Validating deployment

- Verify PostgreSQL is running and DSpace schema exists.
- Verify Solr core and accessible URL.
- Verify backend service via `systemctl status dspace-backend`.
- Verify frontend served by nginx.

## 6. Vault setup and become password

Set up secrets in `group_vars/vault.yml` and encrypt with Ansible Vault:

```bash
cp group_vars/vault.yml.example group_vars/vault.yml
ansible-vault encrypt group_vars/vault.yml
```

Then in `inventory/hosts.yml` map sudo password from vault:

```yaml
dspace-server:
  ansible_become: true
  ansible_become_method: sudo
  ansible_become_pass: "{{ vault_sudo_pass }}"
```

Run with vault password:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass
```

As alternative (non-interactive):

```bash
ansible-playbook -i inventory/hosts.yml site.yml --vault-password-file ~/.vault_pass.txt
```

## 7. Latest updates

- Added full vault-based sudo password support.
- Added explicit docs for `ansible_become_pass` in inventory.
- Added command examples to run playbook with `--ask-vault-pass` / `--vault-password-file`.
- Added `--check` dry run and backend/frontend quick validation commands.
- Updated DSpace source retrieval to use GitHub archive URL:
  `https://github.com/DSpace/DSpace/archive/refs/tags/dspace-{{ dspace_version }}.tar.gz` (fixes 404 for dspace-9.2 release artifacts).
- Updated to DSpace 9.1/9.2 compatibility in role defaults and templates.
- `group_vars/all.yml` now references vault secrets for:
  - `db_password` -> `vault_db_password`
  - `dspace_admin_password` -> `vault_dspace_admin_password`.
- Vault vars in `group_vars/vault.yml` include:
  - `vault_db_password`
  - `vault_dspace_admin_password`
  - `vault_sudo_pass`.

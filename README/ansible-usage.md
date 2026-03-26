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

# DSpace Ansible Deployment Repository

This repository contains an Ansible-based deployment for DSpace (backend, frontend, Solr, PostgreSQL, and nginx). It is designed as an infrastructure-as-code automation for reproducible installs.

## What is here

- `site.yml`: The main playbook coordinating the full environment.
- `roles/`: Each service is managed with its own role:
  - `postgresql`
  - `solr`
  - `dspace_backend`
  - `dspace_frontend`
  - `nginx`
- `inventory/hosts.yml`: Ansible inventory of target nodes.
- `group_vars/`: Shared variables and secrets template (`vault.yml.example`).
- `requirements.yml`: Ansible Galaxy role dependencies.
- `ansible.cfg`: Ansible configuration file.

## Getting started

1. Read `README/ansible-usage.md` for the primary Ansible workflow.
2. Read `README/manual-install.md` for a manual installation overview (helpful for understanding role behavior and debugging).

## Quick install

1. Create/validate `inventory/hosts.yml`.
2. Configure variables in `group_vars/*.yml`.
3. Install requirements: `ansible-galaxy install -r requirements.yml`.
4. Run: `ansible-playbook -i inventory/hosts.yml site.yml`.

## Notes

- Secrets should be managed with Ansible Vault (`ansible-vault encrypt group_vars/vault.yml`).
- The `vault_sudo_pass` variable is used for `ansible_become_pass` on the host.
- Use `--check` for dry-run verification.
- Rerun playbook after changed variables with `--diff` to validate idempotency.
- This repository is intended to be executed from a control machine with SSH access to managed hosts.

# DSpace Ansible Deployment Repository

This repository contains an Ansible-based deployment for DSpace (backend, frontend, Solr, PostgreSQL, and nginx). It is designed as an infrastructure-as-code automation for reproducible installs.

## What is here

- `site.yml`: The main playbook coordinating the full environment.
- `roles/`: Each service is managed with its own role:
  - `postgresql`
  - `solr`
  - `dspace_backend`
  - `dspace_frontend`
  - `nginx`
- `inventory/hosts.yml`: Ansible inventory of target nodes.
- `group_vars/`: Shared variables and secrets template (`vault.yml.example`).
- `requirements.yml`: Ansible Galaxy role dependencies.
- `ansible.cfg`: Ansible configuration file.

## Getting started

1. Read `README/ansible-usage.md` for the primary Ansible workflow.
2. Read `README/manual-install.md` for a manual installation overview (helpful for understanding role behavior and debugging).

## Quick install

1. Create/validate `inventory/hosts.yml`.
2. Configure variables in `group_vars/*.yml`.
3. Install requirements: `ansible-galaxy install -r requirements.yml`.
4. Run: `ansible-playbook -i inventory/hosts.yml site.yml`.

## Notes

- Secrets should be managed with Ansible Vault (`ansible-vault encrypt group_vars/vault.yml`).
- The `vault_sudo_pass` variable is used for `ansible_become_pass` in inventory.
- Use `--check` for dry-run verification.
- Rerun playbook after changed variables with `--diff` to validate idempotency.
- This repository is intended to be executed from a control machine with SSH access to managed hosts.

## Latest updates

- Consolidated `vault_sudo_pass` mapping in `inventory/hosts.yml`.
- Added `README/manual-install.md` and `README/ansible-usage.md` with specific command examples and vault flow.
- Added root `README.md` with doc links and quick start paths.
- Updated to DSpace 9.1 (9.2 source tarball not available).

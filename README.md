# DSpace Ansible Deployment (Root README)

This repository contains Ansible automation for deploying DSpace components (PostgreSQL, Solr, backend, frontend, nginx).

## Quick start

- `site.yml`: main playbook
- `roles/`: service roles
- `inventory/hosts.yml`: test/production hosts
- `group_vars/`: configuration + secrets
- `requirements.yml`: Galaxy role dependencies

## Documentation

- [Manual installation](README/manual-install.md) - manual install walkthrough
- [Ansible usage](README/ansible-usage.md) - ansible step-by-step usage
- [Internal repo landing doc](README/README.md) - expanded project explanation

## Run

1. `ansible-galaxy install -r requirements.yml`
2. Customize `inventory/hosts.yml` and `group_vars/*`
3. `ansible-playbook -i inventory/hosts.yml site.yml`

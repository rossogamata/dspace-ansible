# DSpace Ansible Deployment (Root README)

This repository contains Ansible automation for deploying DSpace components (PostgreSQL, Solr, backend, frontend, nginx).

## Quick start

- `site.yml`: main playbook
- `roles/`: service roles
- `inventory/hosts.yml`: test/production hosts
- `group_vars/`: configuration + secrets
- `requirements.yml`: Galaxy role dependencies

## Documentation

- `README/manual-install.md` - manual install walkthrough
- `README/ansible-usage.md` - ansible step-by-step usage
- `README/README.md` - expanded repo landing doc

## Run

1. `ansible-galaxy install -r requirements.yml`
2. Customize `inventory/hosts.yml` and `group_vars/*`
3. `ansible-playbook -i inventory/hosts.yml site.yml`

# Ansible Reference - PrestaShop + MySQL Deployment

Reference Ansible project: a **single top-level playbook** (`site.yml`)
deploys a PrestaShop + MySQL e-commerce platform on the environment of your
choice (production, staging, or any new inventory you add).

## Architecture

```text
┌────────────────────────────┐      ┌────────────────────────────┐
│   VM1 - MYSQL SERVER       │      │   VM2 - WEB SERVER         │
│   192.168.56.11            │◄─────│   192.168.56.12            │
│   Ubuntu 20.04 / 2 GB      │ 3306 │   Ubuntu 20.04 / 2 GB      │
│                            │      │                            │
│   Role: mysql              │      │   Role: prestashop         │
│   Hardened MySQL 8         │      │   Apache + PHP 7.4         │
│   + Ansible controller     │      │   PrestaShop 8.1.2         │
└────────────────────────────┘      └────────────────────────────┘
        Private network 192.168.56.0/24 (Vagrant/VirtualBox)
```

## Project structure

```text
project_ansible/
├── site.yml                  # SINGLE entry point (top-level, reusable)
├── ansible.cfg               # Ansible config (default inventory: production)
├── Vagrantfile               # Local lab: 2 VirtualBox VMs
│
├── inventories/              # 1 folder = 1 environment ("site")
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/all/vault.yml.example   # secrets (to encrypt)
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/all/vault.yml.example
│
├── group_vars/               # Variables shared per GROUP (all envs)
│   ├── mysql_servers.yml
│   └── web_servers.yml
├── host_vars/                # Variables per HOST (env-specific)
│   ├── mysql_prod.yml / mysql_staging.yml
│   └── web_prod.yml   / web_staging.yml
│
└── roles/
    ├── mysql/                # Hardened MySQL, tuning, backups (own README)
    └── prestashop/           # Apache + PHP + PrestaShop (own README)
```

## Quick start

```bash
# 1. Create the VMs (from Windows)
vagrant up

# 2. Connect to the controller (VM1)
vagrant ssh db
cd ansible

# 3. Copy the SSH key to the web server (password: vagrant)
ssh-copy-id vagrant@192.168.56.12

# 4. Create the environment vault
cp inventories/production/group_vars/all/vault.yml.example \
   inventories/production/group_vars/all/vault.yml
# edit the passwords, then:
ansible-vault encrypt inventories/production/group_vars/all/vault.yml

# 5. Deploy
ansible-playbook site.yml --ask-vault-pass
```

## One playbook, multiple environments

The same `site.yml` applies to any environment — the selected inventory
determines the target and its variables:

```bash
ansible-playbook site.yml -i inventories/production --ask-vault-pass
ansible-playbook site.yml -i inventories/staging    --ask-vault-pass

# Dry run (no changes applied)
ansible-playbook site.yml -i inventories/production --check --ask-vault-pass

# Partial deployment by tag or host
ansible-playbook site.yml --tags database --ask-vault-pass
ansible-playbook site.yml --tags web --limit web_prod --ask-vault-pass
```

To add an environment (e.g. `preprod`): duplicate an inventory folder, adapt
the IPs/hosts, create its vault — `site.yml` does not change.

## Variable hierarchy (from lowest to highest precedence)

1. `roles/*/defaults/main.yml` — universal non-sensitive defaults
2. `group_vars/<group>.yml` — shared by a group, all environments
3. `host_vars/<host>.yml` — specific to a host (prod vs staging)
4. `inventories/<env>/group_vars/all/vault.yml` — encrypted per-env secrets
5. `-e var=value` — one-off command-line override

Example: `mysql_max_connections` is 100 (default) → 200 on `mysql_prod`
(host_vars) → 50 on `mysql_staging` (host_vars).

## Security

- **No plaintext secrets in the repository**: passwords live in per-environment
  encrypted vaults (only the `.example` files are versioned).
- `no_log: true` on every task handling passwords.
- MySQL password never passed as a CLI argument (via `MYSQL_PWD` env var).
- MySQL application user restricted to its own database; anonymous users and
  the `test` database removed; `/root/.my.cnf` mode `0600`.

## Post-deployment verification

```bash
sudo systemctl status mysql        # on VM1
sudo systemctl status apache2      # on VM2
mysql -h 192.168.56.11 -u prestashop_user -p -e "SELECT 1;"   # from VM2
# From Windows: http://192.168.56.12 (or http://localhost:8081)
```

## Documentation

| Document | Contents |
| --- | --- |
| [roles/mysql/README.md](roles/mysql/README.md) | Documentation of the `mysql` role |
| [roles/prestashop/README.md](roles/prestashop/README.md) | Documentation of the `prestashop` role |
| [VARIABLES_HIERARCHY.md](VARIABLES_HIERARCHY.md) | Variables hierarchy and resolution (up to date) |

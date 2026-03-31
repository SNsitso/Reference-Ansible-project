# Ansible Project - Prestashop + MySQL Deployment

## Overview

This project automates the **complete and resilient deployment** of a PrestaShop e-commerce platform with MySQL using **Ansible** following professional best practices.

## Implementation Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEPLOYMENT FLOWS                             │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────┐       ┌──────────────────────────────────┐
│         VM1 - MYSQL SERVER       │       │      VM2 - WEB SERVER            │
│       192.168.56.11              │       │    192.168.56.12                 │
│       (Ubuntu 20.04)             │       │    (Ubuntu 20.04)                │
│       RAM: 2GB                   │       │    RAM: 2GB                      │
├──────────────────────────────────┤       ├──────────────────────────────────┤
│                                  │       │                                  │
│  Role: mysql                     │       │  Role: prestashop                │
│  ─────────────────────────────   │       │  ──────────────────────────────  │
│  Tasks:                          │       │  Tasks:                          │
│  • Install MySQL Server/Client   │       │  • Install Apache2               │
│  • Secure MySQL (users, perms)   │       │  • Install PHP 7.4/8.1           │
│  • Create database               │       │  • Install PHP extensions        │
│  • Create app user               │       │  • Download Prestashop           │
│  • Enable network connections    │       │  • Configure Apache VirtualHost  │
│  • Support backups (production)  │       │  • Optimize PHP configuration    │
│  • Configure slow query logs     │       │  • Test MySQL connection         │
│                                  │       │  • Configure permissions         │
│  Service: MySQL 3306             │       │                                  │
│  Database: prestashop_db         │       │  Service: Apache 80/443          │
│  User: prestashop_user           │       │  Platform: Prestashop 8.1.2      │
└──────────────────────────────────┘       └──────────────────────────────────┘
         │                                           │
         └───────────────── SSH ────────────────────┘
              Private Network (192.168.56.0/24)
                    MySQL Port 3306
```

### Technology Stack
- **Ansible**: Orchestration and configuration management
- **Vagrant**: VirtualBox for virtual machines
- **PrestaShop**: E-commerce platform
- **MySQL**: Database
- **Apache2 + PHP**: Web server

## Project Structure (Following Best Practices)

```
project_ansible/
├── playbooks/
│   └── deploy.yml                   # Main playbook
│
├── inventories/
│   ├── production/
│   │   └── hosts.yml               # Production Inventory (YAML)
│   └── staging/
│       └── hosts.yml               # Staging Inventory (YAML)
│
├── group_vars/
│   ├── mysql_servers.yml           # Common MySQL config
│   ├── web_servers.yml             # Common Web config
│   ├── vault.yml.example           # Production secrets template
│   └── vault_staging.yml.example   # Staging secrets template
│
├── host_vars/
│   ├── mysql_prod.yml              # Production MySQL specific config
│   ├── mysql_staging.yml           # Staging MySQL specific config
│   ├── web_prod.yml                # Production Web specific config
│   └── web_staging.yml             # Staging Web specific config
│
├── roles/
│   ├── mysql/
│   │   ├── README.md               # Complete role documentation
│   │   ├── meta/main.yml           # Metadata and requirements
│   │   ├── defaults/main.yml       # Universal default values
│   │   ├── handlers/main.yml       # Restarts and final tasks
│   │   ├── tasks/main.yml          # Installation tasks
│   │   └── templates/my.cnf.j2     # MySQL configuration template
│   │
│   └── prestashop/
│       ├── README.md               # Complete role documentation
│       ├── meta/main.yml           # Metadata and requirements
│       ├── defaults/main.yml       # Universal default values
│       ├── handlers/main.yml       # Apache restarts
│       ├── tasks/main.yml          # Installation tasks
│       └── templates/
│           ├── prestashop.conf.j2  # Apache VirtualHost
│           └── php.ini.j2          # Optimized PHP configuration
│
├── ansible.cfg                      # Ansible configuration (multi-env support)
├── README.md                        # This file
├── README_EN.md                     # English documentation
├── RESTRUCTURATION_SUMMARY.md       # Summary of changes
├── VARIABLES_HIERARCHY.md           # Variables hierarchy (FR)
├── VARIABLES_HIERARCHY_EN.md        # Variables hierarchy (EN)
├── STRUCTURE_VISUAL.txt             # Complete visual structure
├── Vagrantfile                      # VM configuration
└── (other logs/config files)
```

## Ansible Roles

### MySQL Role (VM1)

**Responsibilities**:
- Secure MySQL Server installation and configuration
- Creation of `prestashop_db` database
- Creation of `prestashop_user` application user
- Network connection configuration
- Backup management (Production only)
- Performance optimizations (Production vs Staging)

**Role Files**:
- `README.md`: Complete role documentation with examples
- `meta/main.yml`: Required variables and metadata
- `defaults/main.yml`: Universal default values
- `handlers/main.yml`: MySQL restarts
- `tasks/main.yml`: 16+ automated installation tasks
- `templates/my.cnf.j2`: MySQL configuration template

**Key Variables**:
```yaml
# group_vars/mysql_servers.yml (common to all MySQL)
mysql_port: 3306
mysql_bind_address: '0.0.0.0'
mysql_service_name: mysql

# host_vars/mysql_prod.yml (PRODUCTION specific)
mysql_database: prestashop_db
mysql_app_user: prestashop_user
mysql_max_connections: 200              # Production: high
mysql_innodb_buffer_pool_size: '1G'     # Production: large
mysql_backup_enabled: yes               # Backups enabled

# host_vars/mysql_staging.yml (STAGING specific)
mysql_database: prestashop_db_staging
mysql_app_user: prestashop_user_staging
mysql_max_connections: 50               # Staging: low
mysql_innodb_buffer_pool_size: '512M'   # Staging: small
mysql_backup_enabled: no                # No backups
```

### Prestashop Role (VM2)

**Responsibilities**:
- Apache2 installation with required modules (rewrite, headers)
- PHP 7.4/8.1 installation with required extensions
- PrestaShop 8.1.2 download and extraction
- Apache VirtualHost configuration
- Secure remote MySQL connection (VM1)
- PHP performance optimizations for e-commerce
- File permission management

**Role Files**:
- `README.md`: Complete documentation with troubleshooting
- `meta/main.yml`: Required variables and metadata
- `defaults/main.yml`: Universal default values
- `handlers/main.yml`: Apache restarts
- `tasks/main.yml`: 15+ automated installation tasks
- `templates/prestashop.conf.j2`: Apache VirtualHost
- `templates/php.ini.j2`: Optimized PHP configuration

**Key Variables**:
```yaml
# group_vars/web_servers.yml (common to all web)
apache_service_name: apache2
apache_modules:
  - rewrite
  - headers
php_packages:
  - php7.4-curl
  - php7.4-gd

# host_vars/web_prod.yml (PRODUCTION specific)
prestashop_domain: 192.168.56.12
prestashop_db_host: mysql_prod
php_memory_limit: 256M                  # Production: large
php_version: 7.4
mysql_backup_enabled: yes

# host_vars/web_staging.yml (STAGING specific)
prestashop_domain: 192.168.56.22
prestashop_db_host: mysql_staging
php_memory_limit: 128M                  # Staging: small
php_version: 7.4
mysql_backup_enabled: no
```

---

## Variables Architecture

### Priority Hierarchy (Cascade Resolution)

Variable hierarchy follows this order (least to most priority):

1. **Role Defaults** (`roles/*/defaults/main.yml`)
   - Universal default values
   - Apply everywhere without override

2. **Group Variables** (`group_vars/mysql_servers.yml`, `group_vars/web_servers.yml`)
   - Configuration common to ALL hosts in a group
   - Standard packages, services, paths
   - Overrides defaults for the group

3. **Host Variables** (`host_vars/mysql_prod.yml`, `host_vars/web_prod.yml`)
   - Configuration UNIQUE to each host
   - Production vs Staging optimizations
   - Secrets via Vault
   - Overrides group_vars

4. **Playbook Variables** (`playbooks/deploy.yml`, `-e` flag)
   - Command-line variables
   - Temporary override everything

### Example: How `mysql_max_connections` is Resolved

```yaml
# 1. Role default (base)
roles/mysql/defaults/main.yml:
  mysql_max_connections: 100

# 2. Group override (common)
group_vars/mysql_servers.yml:
  mysql_max_connections: 200

# 3. Host override (production specific)
host_vars/mysql_prod.yml:
  mysql_max_connections: 300

# 4. Runtime override (temporary)
$ ansible-playbook deploy.yml -e "mysql_max_connections=400"

# FINAL RESULT:
# - mysql_prod uses 300 (host_vars)
# - mysql_staging uses 200 (group_vars)
# - With -e flag, all use 400
```

---

## Secure Secrets Management

### Vault Structure

Sensitive passwords are stored **encrypted** with Ansible Vault:

```
group_vars/
├── mysql_servers.yml              # Normal variables (non-sensitive)
├── vault.yml                      # Secrets Production (ENCRYPTED)
├── web_servers.yml                # Normal variables (non-sensitive)
└── vault_staging.yml              # Secrets Staging (ENCRYPTED)

host_vars/
├── mysql_prod.yml                 # References vault
├── web_prod.yml                   # References vault
└── (other non-sensitive files)
```

### Usage

In host variables, reference vault secrets:

```yaml
# host_vars/mysql_prod.yml
mysql_root_password: "{{ vault_mysql_root_password }}"
mysql_app_password: "{{ vault_prestashop_db_password }}"
```

The `vault.yml` file (encrypted) contains:

```yaml
# group_vars/vault.yml
vault_mysql_root_password: "ReallySecurePassword123!"
vault_prestashop_db_password: "SecurePassword456!"
vault_prestashop_admin_password: "AdminPassword789!"
```

### Execution with Vault

```bash
# Prompt for password
ansible-playbook playbooks/deploy.yml -i inventories/production/ --ask-vault-pass

# Or with file
echo "my_vault_password" > .vault_pass
chmod 600 .vault_pass
ansible-playbook playbooks/deploy.yml --vault-password-file=.vault_pass
```

---

## Multi-Environment: Production vs Staging

### Separate Inventories

```
inventories/production/hosts.yml      # Production: mysql_prod, web_prod
inventories/staging/hosts.yml         # Staging: mysql_staging, web_staging
```

### Environment Variables

Configurations are **different** by environment:

| Configuration | Production | Staging |
|---------------|-----------|---------|
| **MySQL** |  |  |
| max_connections | 200 | 50 |
| buffer_pool_size | 1G | 512M |
| Backups | Yes | No |
| **PHP** |  |  |
| memory_limit | 256M | 128M |
| max_upload | 100M | 50M |
| **Prestashop** |  |  |
| Domain | 192.168.56.12 | 192.168.56.22 |
| Admin email | admin@prod.local | admin@staging.local |

### Deployment by Environment

```bash
# Production only
ansible-playbook playbooks/deploy.yml -i inventories/production/

# Staging only
ansible-playbook playbooks/deploy.yml -i inventories/staging/

# Simulation mode (dry-run)
ansible-playbook playbooks/deploy.yml -i inventories/production/ --check

# With encrypted secrets
ansible-playbook playbooks/deploy.yml -i inventories/production/ --ask-vault-pass
```

---

## Complete Role Documentation

Each role includes a **complete README.md** with:

### MySQL Role
- [roles/mysql/README.md](roles/mysql/README.md)
  - Description and features
  - Required and optional variables
  - Vault usage examples
  - Testing and validation instructions
  - Detailed troubleshooting guide
  - Security measures implemented
  - Performance optimization

### Prestashop Role
- [roles/prestashop/README.md](roles/prestashop/README.md)
  - Description and features
  - Required and optional variables
  - Environment-specific configuration
  - Basic and advanced usage
  - Testing instructions
  - Complete troubleshooting guide
  - Performance tuning
  - Security recommendations

---

## Additional Guides

| Document | Description |
|----------|------------|
| [VARIABLES_HIERARCHY.md](VARIABLES_HIERARCHY.md) | Complete variables hierarchy (FR) |
| [VARIABLES_HIERARCHY_EN.md](VARIABLES_HIERARCHY_EN.md) | Complete variables hierarchy (EN) |
| [README_EN.md](README_EN.md) | English documentation |
| [RESTRUCTURATION_SUMMARY.md](RESTRUCTURATION_SUMMARY.md) | Summary of restructuring changes |
| [STRUCTURE_VISUAL.txt](STRUCTURE_VISUAL.txt) | Visual project structure |

## Deployment Guide

### Prerequisites
- Vagrant installed
- VirtualBox installed
- Minimum 4 GB RAM available

### Deployment Steps
1. Create VMs with Vagrant
2. Connect to VM1 (Ansible controller)
3. Launch Ansible deployment
4. Test the website
5. Retrieve logs

*(Detailed instructions will be added after Vagrantfile creation)*

## Security

**Important**: Provided passwords are for demonstration only.

**Security Measures Implemented**:
- Dedicated MySQL user with limited privileges
- Removal of anonymous MySQL users
- Removal of test database
- .my.cnf file with restrictive permissions (0600)
- Secure Apache configuration

## Ports and Services

| Machine | Service | Port | Protocol | Description |
|---------|---------|------|----------|-------------|
| VM1 | MySQL | 3306 | TCP | Database |
| VM2 | Apache | 80 | HTTP | Prestashop web server |

## Inter-Server Communication

**How Prestashop communicates with MySQL**:
1. VM2 (Prestashop) connects to VM1 IP: `192.168.56.11:3306`
2. MySQL on VM1 accepts connections from VM2
3. Authentication with user `prestashop_user` / password `Prestashop@123`
4. Access to `prestashop_db` database

## Testing and Validation

**After Deployment**:
1. Verify MySQL on VM1: `sudo systemctl status mysql`
2. Verify Apache on VM2: `sudo systemctl status apache2`
3. Test DB connection: `mysql -h 192.168.56.11 -u prestashop_user -p`
4. Access website: http://192.168.56.12 (from Windows)

## Log Generation

```bash
# From VM1 (Ansible controller)
ansible-playbook deploy.yml -vvv > deployment.log 2>&1
```

Logs will include:
- All executed tasks
- Changes made
- Any errors
- Final deployment status

## Troubleshooting

### MySQL won't start
```bash
sudo systemctl status mysql
sudo journalctl -u mysql -n 50
```

### Apache won't start
```bash
sudo systemctl status apache2
sudo apache2ctl configtest
```

### Database connection issue
```bash
# Test from VM2
mysql -h 192.168.56.11 -u prestashop_user -pPrestashop@123
```

### Check logs
```bash
# Apache logs
tail -f /var/log/apache2/prestashop_error.log

# MySQL logs
tail -f /var/log/mysql/error.log
```

## Deliverables

The project will be delivered in `.zip` format containing:
- 2 Ansible roles (MySQL + Prestashop)
- Orchestration playbook
- Vagrantfile for VM creation
- Configuration files
- Real deployment logs (.txt)
- Complete documentation

## Author
DevOps Engineer - Ansible Exam
Date: December 2025

# Ansible Project - PrestaShop + MySQL Deployment

##  Overview

This project automates the **complete and resilient deployment** of an e-commerce platform using **PrestaShop** and **MySQL** with **Ansible**, following professional best practices.

### Technology Stack
- **Ansible** : Configuration management and orchestration
- **Vagrant** : VirtualBox virtual machines
- **PrestaShop** : E-commerce platform
- **MySQL** : Database server
- **Apache2 + PHP** : Web server

### Architecture

```
┌───────────────────────────┐         ┌───────────────────────────┐
│   VM1 - MYSQL SERVER      │         │   VM2 - WEB SERVER        │
│   192.168.56.11           │         │   192.168.56.12           │
│                           │         │                           │
│  • MySQL 3306             │         │  • Apache2 80/443         │
│  • Ansible Controller     │◄───────►│  • PHP 7.4/8.1            │
│  • 2GB RAM                │ Private │  • PrestaShop 8.1.2       │
│  • Ubuntu 20.04           │ Network │  • 2GB RAM                │
│                           │ SSH     │  • Ubuntu 20.04           │
└───────────────────────────┘         └───────────────────────────┘
```

## Deployment Architecture & Implementation Flows

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEPLOYMENT FLOWS                             │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────┐       ┌──────────────────────────────────┐
│         VM1 - MYSQL SERVER       │       │      VM2 - WEB SERVER            │
│       192.168.56.11              │       │    192.168.56.12                 │
│  Role: mysql                     │       │  Role: prestashop                │
│  Tasks:                          │       │  Tasks:                          │
│  • Install MySQL Server/Client   │       │  • Install Apache2               │
│  • Secure MySQL (users, perms)   │       │  • Install PHP 7.4/8.1           │
│  • Create database               │       │  • Install PHP extensions        │
│  • Create app user               │       │  • Download Prestashop           │
│  • Enable network connections    │       │  • Configure Apache VirtualHost  │
│  • Support backups (production)  │       │  • Optimize PHP configuration    │
│  • Configure slow query logs     │       │  • Test MySQL connection         │
└──────────────────────────────────┘       └──────────────────────────────────┘
         MySQL 3306                              Apache 80/443
```

##  Project Structure (Professional Best Practices)

```
project_ansible/
├── 📁 playbooks/
│   └── deploy.yml                   # Main playbook
│
├── 📁 inventories/
│   ├── production/
│   │   └── hosts.yml               # Production inventory (YAML)
│   └── staging/
│       └── hosts.yml               # Staging inventory (YAML)
│
├── 📁 group_vars/
│   ├── mysql_servers.yml           # MySQL common config (all MySQL)
│   ├── web_servers.yml             # Web common config (all Web)
│   ├── vault.yml.example           # Production secrets template
│   └── vault_staging.yml.example   # Staging secrets template
│
├── 📁 host_vars/
│   ├── mysql_prod.yml              # MySQL Production specific
│   ├── mysql_staging.yml           # MySQL Staging specific
│   ├── web_prod.yml                # Web Production specific
│   └── web_staging.yml             # Web Staging specific
│
├── 📁 roles/
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
│           └── php.ini.j2          # PHP configuration
│
├── 📄 ansible.cfg                   # Ansible configuration
├── 📄 README.md                     # Documentation (French)
├── 📄 README_EN.md                  # Documentation (English)
├── 📄 RESTRUCTURATION_SUMMARY.md    # Changes summary
├── 📄 VARIABLES_HIERARCHY.md        # Variables hierarchy (FR)
├── 📄 VARIABLES_HIERARCHY_EN.md     # Variables hierarchy (EN)
├── 📄 STRUCTURE_VISUAL.txt          # Visual structure overview
├── 📄 Vagrantfile                   # VM configuration
└── (other logs/config files)
```

---

##  Ansible Roles

###  MySQL Role (VM1)

**Responsibilities**:
- Installation and secure configuration of MySQL Server
- Creation of `prestashop_db` database
- Creation of `prestashop_user` application user
- Network connection configuration
- Backup management (Production only)
- Performance optimizations (Production vs Staging)

**Role Files**:
- `README.md` : Complete role documentation with examples
- `meta/main.yml` : Required variables and metadata
- `defaults/main.yml` : Universal default values
- `handlers/main.yml` : MySQL restarts
- `tasks/main.yml` : 16+ automated installation tasks
- `templates/my.cnf.j2` : MySQL configuration template

**Key Variables**:
```yaml
# group_vars/mysql_servers.yml (common to ALL MySQL hosts)
mysql_port: 3306
mysql_bind_address: '0.0.0.0'
mysql_service_name: mysql

# host_vars/mysql_prod.yml (PRODUCTION specific)
mysql_database: prestashop_db
mysql_app_user: prestashop_user
mysql_max_connections: 200           # Production: high
mysql_innodb_buffer_pool_size: '1G'  # Production: large
mysql_backup_enabled: yes            # Backups in prod

# host_vars/mysql_staging.yml (STAGING specific)
mysql_database: prestashop_db_staging
mysql_app_user: prestashop_user_staging
mysql_max_connections: 50            # Staging: low
mysql_innodb_buffer_pool_size: '512M' # Staging: small
mysql_backup_enabled: no             # No backups
```

### 🌐 PrestaShop Role (VM2)

**Responsibilities**:
- Apache2 installation with required modules (rewrite, headers)
- PHP 7.4/8.1 installation with required extensions
- PrestaShop 8.1.2 download and extraction
- Apache VirtualHost configuration
- Secure connection to remote MySQL (VM1)
- PHP performance optimizations for e-commerce
- File permission management

**Role Files**:
- `README.md` : Complete documentation with troubleshooting
- `meta/main.yml` : Required variables and metadata
- `defaults/main.yml` : Universal default values
- `handlers/main.yml` : Apache restarts
- `tasks/main.yml` : 15+ automated installation tasks
- `templates/prestashop.conf.j2` : Apache VirtualHost
- `templates/php.ini.j2` : Optimized PHP configuration

**Key Variables**:
```yaml
# group_vars/web_servers.yml (common to ALL web hosts)
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
php_memory_limit: 256M               # Production: large
php_version: 7.4
mysql_backup_enabled: yes

# host_vars/web_staging.yml (STAGING specific)
prestashop_domain: 192.168.56.22
prestashop_db_host: mysql_staging
php_memory_limit: 128M               # Staging: small
php_version: 7.4
mysql_backup_enabled: no
```

---

## Variables Architecture

### Priority Hierarchy (cascading resolution)

The variables hierarchy follows this order (from least to most priority):

1. **Role Defaults** (`roles/*/defaults/main.yml`)
   - Universal default values
   - Applied everywhere without override

2. **Group Variables** (`group_vars/mysql_servers.yml`, `group_vars/web_servers.yml`)
   - Configuration common to ALL hosts in a group
   - Packages, services, standard paths
   - Overrides defaults for the group

3. **Host Variables** (`host_vars/mysql_prod.yml`, `host_vars/web_prod.yml`)
   - Configuration UNIQUE to each host
   - Production vs Staging optimizations
   - Secrets via Vault
   - Overrides group_vars

4. **Playbook Variables** (`playbooks/deploy.yml`, `-e` flag)
   - Command-line variables
   - Temporarily override everything

### Example: How `mysql_max_connections` is resolved

```yaml
# 1. Role default (base)
roles/mysql/defaults/main.yml:
  mysql_max_connections: 100

# 2. Group override (common)
group_vars/mysql_servers.yml:
  mysql_max_connections: 200

# 3. Host override (production-specific)
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

## 🔐 Secure Secrets Management

### Vault Structure

Sensitive passwords are stored **encrypted** with Ansible Vault:

```
group_vars/
├── mysql_servers.yml              # Normal variables (non-sensitive)
├── vault.yml                      #  Production Secrets (ENCRYPTED)
├── web_servers.yml                # Normal variables (non-sensitive)
└── vault_staging.yml              #  Staging Secrets (ENCRYPTED)

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
# Ask for password
ansible-playbook playbooks/deploy.yml -i inventories/production/ --ask-vault-pass

# Or with password file
echo "my_vault_password" > .vault_pass
chmod 600 .vault_pass
ansible-playbook playbooks/deploy.yml --vault-password-file=.vault_pass
```

---

##  Multi-Environment: Production vs Staging

### Separate Inventories

```
inventories/production/hosts.yml      # Production: mysql_prod, web_prod
inventories/staging/hosts.yml         # Staging: mysql_staging, web_staging
```

### Environment-Specific Variables

Configurations are **different** per environment:

| Configuration | Production | Staging |
|---------------|-----------|---------|
| **MySQL** |  |  |
| max_connections | 200 | 50 |
| buffer_pool_size | 1G | 512M |
| Backups | Yes | No |
| **PHP** |  |  |
| memory_limit | 256M | 128M |
| max_upload | 100M | 50M |
| **PrestaShop** |  |  |
| Domain | 192.168.56.12 | 192.168.56.22 |
| Admin email | admin@prod.local | admin@staging.local |

### Deployment by Environment

```bash
# Production only
ansible-playbook playbooks/deploy.yml -i inventories/production/

# Staging only
ansible-playbook playbooks/deploy.yml -i inventories/staging/

# Dry-run (simulation)
ansible-playbook playbooks/deploy.yml -i inventories/production/ --check

# With encrypted secrets
ansible-playbook playbooks/deploy.yml -i inventories/production/ --ask-vault-pass
```

---

## 📚 Complete Role Documentation

Each role includes a **complete README.md** with:

### MySQL Role
- [roles/mysql/README.md](roles/mysql/README.md)
  - Description and features
  - Required and optional variables
  - Usage examples with Vault
  - Testing and validation instructions
  - Detailed troubleshooting guide
  - Implemented security measures
  - Performance optimization

### PrestaShop Role
- [roles/prestashop/README.md](roles/prestashop/README.md)
  - Description and features
  - Required and optional variables
  - Configuration per environment
  - Basic and advanced usage
  - Testing instructions
  - Complete troubleshooting guide
  - Performance tuning
  - Security recommendations

---

##  Additional Guides

| Document | Description |
|----------|------------|
| [VARIABLES_HIERARCHY.md](VARIABLES_HIERARCHY.md) | Complete variables hierarchy architecture (French) |
| [VARIABLES_HIERARCHY_EN.md](VARIABLES_HIERARCHY_EN.md) | Variables hierarchy architecture (English) |
| [README.md](README.md) | Complete documentation (French) |
| [RESTRUCTURATION_SUMMARY.md](RESTRUCTURATION_SUMMARY.md) | Restructuring changes summary |
| [STRUCTURE_VISUAL.txt](STRUCTURE_VISUAL.txt) | Visual project structure |

---

##  How to Use This Project

### Prerequisites

Before running this project, ensure you have:
- Vagrant 2.2+ installed
- VirtualBox 6.0+ installed
- 8GB RAM minimum available
- 10GB free disk space
- Internet connection active

### Quick Start

#### 1. Production Deployment

```bash
# Full deployment on Production
ansible-playbook playbooks/deploy.yml -i inventories/production/

# Simulation mode (dry-run) first
ansible-playbook playbooks/deploy.yml -i inventories/production/ --check

# With encrypted secrets
ansible-playbook playbooks/deploy.yml -i inventories/production/ --ask-vault-pass

# Verbose output for debugging
ansible-playbook playbooks/deploy.yml -i inventories/production/ -vvv
```

#### 2. Staging Deployment

```bash
# Deploy to Staging
ansible-playbook playbooks/deploy.yml -i inventories/staging/

# Staging with Vault
ansible-playbook playbooks/deploy.yml -i inventories/staging/ --ask-vault-pass
```

#### 3. Selective Deployment

```bash
# Deploy only MySQL
ansible-playbook playbooks/deploy.yml -i inventories/production/ --tags mysql

# Deploy only PrestaShop
ansible-playbook playbooks/deploy.yml -i inventories/production/ --tags prestashop

# Skip a specific task
ansible-playbook playbooks/deploy.yml -i inventories/production/ --skip-tags restart
```

#### 4. Verification

```bash
# Check inventory
ansible-inventory -i inventories/production/ --list

# Display variables for a host
ansible all -i inventories/production/ -m debug -a "var=vars"

# Validate syntax
ansible-playbook playbooks/deploy.yml --syntax-check
```

---

## 🔧 Troubleshooting

### Common Issues

#### "Unable to parse as an inventory source"
```bash
# Validate YAML syntax
yamllint inventories/production/hosts.yml
python3 -m yaml inventories/production/hosts.yml
```

#### "undefined variable"
```bash
# Check available variables
ansible-inventory -i inventories/production/ --host mysql_prod
```

#### "vault password not provided"
```bash
# Create password file
echo "your_vault_password" > .vault_pass
chmod 600 .vault_pass

# Add to .gitignore
echo ".vault_pass" >> .gitignore
```

---

## Best Practices in This Project

 **Variables Organization**
- Universal defaults in role defaults/
- Group common configs in group_vars/
- Host-specific configs in host_vars/
- Secrets encrypted with Vault

 **Multi-Environment**
- Separate inventories per environment
- Different optimizations per environment
- Easy to add new environments

 **Security**
- No hardcoded passwords
- Secrets encrypted with Vault
- Limited user privileges
- SSH key-based authentication

 **Documentation**
- README.md in each role
- Complete variable documentation
- Examples and use cases
- Troubleshooting guides

 **Modularity**
- Reusable roles
- Clear separation of concerns
- Easy to extend and customize

---

##  Support & Resources

- [Ansible Official Documentation](https://docs.ansible.com/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Role Documentation](roles/README.md)
- [Variables Architecture](VARIABLES_HIERARCHY_EN.md)

---

## Deployment Checklist

- [ ] Vagrant and VirtualBox installed
- [ ] 8GB RAM and 10GB disk space available
- [ ] Internet connection active
- [ ] Inventory files created and validated
- [ ] Vault secrets configured
- [ ] Playbook syntax validated
- [ ] Simulation (--check) successful
- [ ] Production deployment planned
- [ ] Backups configured
- [ ] Monitoring alerts set up

---

## 🎓 Next Steps (Optional)

After successful deployment, consider:
- **ansible-lint** : Validate code quality
- **Molecule** : Automated role testing
- **GitLab CI / GitHub Actions** : CI/CD pipeline
- **Terraform** : Infrastructure as Code
- **Prometheus** : Monitoring and alerting

---

**This project is production-ready and follows Ansible best practices!** 🚀


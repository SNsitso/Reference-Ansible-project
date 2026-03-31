# MySQL Role

## Role Architecture

```
┌────────────────────────────────────────────────────┐
│           ANSIBLE EXECUTION FLOW                   │
└────────────────────────────────────────────────────┘

PLAYBOOK (playbooks/deploy.yml)
        │
        ├─── hosts: mysql_servers
        │
        ├─── become: true
        │
        └─── roles:
             └─── mysql (THIS ROLE)

┌────────────────────────────────────────────────────┐
│  TARGET: VM1 - MySQL Server                        │
│  IP: 192.168.56.11                                 │
│  OS: Ubuntu 20.04                                  │
│  RAM: 2GB                                          │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  MYSQL ROLE TASKS                                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. Install MySQL Server & Client                 │
│  2. Start and enable MySQL service                │
│  3. Secure MySQL (remove anonymous users)         │
│  4. Remove test database                          │
│  5. Create prestashop_db database                 │
│  6. Create prestashop_user with limited privs     │
│  7. Configure network access (bind-address)       │
│  8. Set up configuration files (.my.cnf)          │
│  9. Enable optional backups (prod only)           │
│  10. Configure slow query logging                 │
│                                                    │
│  OUTPUT:                                           │
│  ✓ MySQL Server running on port 3306              │
│  ✓ Database: prestashop_db                        │
│  ✓ User: prestashop_user                          │
│  ✓ Secure, only needed privileges granted         │
│                                                    │
└────────────────────────────────────────────────────┘
```

## Description
This role installs, configures, and secures MySQL Server for Prestashop. It manages the installation of the MySQL server, creation of databases, users, and secure configuration.

## Features
- Installation of MySQL Server and client
- Secure configuration (removal of anonymous users)
- Creation of Prestashop database
- Creation of application user with limited privileges
- Configuration to accept network connections (bind-address)
- Management of configuration files (.my.cnf)
- Handlers for automatic restart
- Support for optional backup
- Configurable logs

## Required Variables

### Mandatory variables
```yaml
mysql_database: "prestashop_db"        # Name of the database
mysql_app_user: "prestashop_user"      # Application user
mysql_app_password: "SecurePassword"   # User password
```

### Optional variables (with default values)
```yaml
mysql_root_password: "RootP@ssw0rd123"
mysql_port: 3306
mysql_bind_address: '0.0.0.0'
mysql_max_connections: 200
mysql_innodb_buffer_pool_size: '1G'
mysql_backup_enabled: no
mysql_slow_query_log: yes
mysql_slow_query_time: 2
```

## Role Files

```
mysql/
├── README.md              # This file
├── defaults/main.yml      # Default variables
├── handlers/main.yml      # Handlers (restarts)
├── tasks/main.yml         # Installation tasks
├── templates/my.cnf.j2    # MySQL configuration template
└── meta/main.yml          # Metadata and dependencies
```

## Usage

### Usage in a playbook
```yaml
- name: Deploy MySQL
  hosts: mysql_servers
  become: true
  roles:
    - role: mysql
```

### Usage with variables
```yaml
- name: Deploy customized MySQL
  hosts: mysql_servers
  become: true
  vars:
    mysql_database: "my_shop_db"
    mysql_app_user: "shop_user"
    mysql_app_password: "MySecurePassword123"
    mysql_max_connections: 300
  roles:
    - role: mysql
```

### Sensitive variables (Vault)
For production environments, use Ansible Vault:

```bash
# Create a vault file with secrets
ansible-vault create group_vars/mysql_servers/vault.yml

# Contents of vault.yml file
vault_mysql_root_password: "MySuperSecureRootPassword"
vault_prestashop_db_password: "MySecureAppPassword"
```

Then reference in variables:
```yaml
mysql_root_password: "{{ vault_mysql_root_password }}"
mysql_app_password: "{{ vault_prestashop_db_password }}"
```

## Main Tasks

This role executes the following tasks:
1. Update APT cache
2. Installation of MySQL packages
3. Start and enable MySQL service
4. Configure root password
5. Create .my.cnf file for authentication
6. Removal of anonymous users
7. Removal of test database
8. Creation of Prestashop database
9. Creation of application user
10. Grant user privileges
11. Configure mysqld.cnf file
12. Restart MySQL (via handler)

## Tests

### Verify that MySQL is running
```bash
sudo systemctl status mysql
```

### Test MySQL connection
```bash
# With root user
mysql -u root -p

# With application user
mysql -h 192.168.56.11 -u prestashop_user -p prestashop_db
```

### Check logs
```bash
tail -f /var/log/mysql/error.log
```

### Validate configuration file
```bash
mysqld --validate-config-file=/etc/mysql/mysql.conf.d/mysqld.cnf
```

## Troubleshooting

### MySQL does not start
```bash
# Check status
sudo systemctl status mysql

# Check logs
sudo journalctl -u mysql -n 50

# Verify configuration
sudo mysqld --validate-config-file=/etc/mysql/mysql.conf.d/mysqld.cnf
```

### User connection failed
```bash
# Check existing users
mysql -u root -p -e "SELECT user, host FROM mysql.user;"

# Check privileges
mysql -u root -p -e "SHOW GRANTS FOR 'prestashop_user'@'%';"
```

## Security

Key security measures implemented:
- Anonymous user removal
- Test database removal
- Root password required
- Application user with limited privileges
- All privileges granted only for the specific database
- Secure .my.cnf file storage (permissions 0600)
mysql -u root -p -e "SHOW GRANTS FOR 'prestashop_user'@'%';"
```

### Problème d'espace disque
```bash
# Vérifier l'espace disponible
df -h /var/lib/mysql

# Trouver les requêtes lentes
mysqldumpslow /var/log/mysql/slow.log
```

## Performance et Optimisation

### Variables d'optimisation
- `mysql_max_connections`: Augmentez selon vos besoins
- `mysql_innodb_buffer_pool_size`: Réglez à ~80% de la RAM disponible
- `mysql_slow_query_log`: Activez pour le debugging des requêtes lentes

## Gestion des logs

### Apache logs
- Error log: `/var/log/mysql/error.log`
- Slow query log: `/var/log/mysql/slow.log` (si activé)

## Sécurité

### Mesures de sécurité implémentées
- Suppression des utilisateurs anonymes
- Suppression de la base de données test
- Mot de passe root configuré fortement
- Utilisateur application avec privilèges minimaux
- Fichier .my.cnf avec permissions restrictives (0600)
- Configuration SSL recommandée (non activée par défaut)

## Support et Contribution

Pour toute question ou contribution, veuillez contacter l'équipe DevOps.

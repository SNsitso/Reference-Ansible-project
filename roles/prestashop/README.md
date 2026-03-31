# Prestashop Role

## Role Architecture

```
┌────────────────────────────────────────────────────┐
│           ANSIBLE EXECUTION FLOW                   │
└────────────────────────────────────────────────────┘

PLAYBOOK (playbooks/deploy.yml)
        │
        ├─── hosts: web_servers
        │
        ├─── become: true
        │
        └─── roles:
             └─── prestashop (THIS ROLE)

┌────────────────────────────────────────────────────┐
│  TARGET: VM2 - Web Server                          │
│  IP: 192.168.56.12                                 │
│  OS: Ubuntu 18.04                                  │
│  RAM: 2GB                                          │
│  HTTP Ports: 80, 443                              │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  PRESTASHOP ROLE TASKS                             │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. Install Apache2 web server                    │
│  2. Install PHP 7.4/8.1 and core modules          │
│  3. Install PHP extensions (curl, json, xml,      │
│     gd, intl, pdo_mysql, etc.)                    │
│  4. Create prestashop application user            │
│  5. Create www-prestashop directory structure     │
│  6. Download & extract PrestaShop 8.1.2           │
│  7. Generate Apache2 VirtualHost configuration    │
│  8. Enable Apache2 modules (rewrite, ssl)         │
│  9. Optimize PHP configuration (memory,           │
│     execution time)                               │
│  10. Test MySQL connectivity to VM1               │
│  11. Set correct file permissions & ownership     │
│  12. Configure environment-specific settings      │
│      (prod vs staging)                            │
│                                                    │
│  OUTPUT:                                           │
│  ✓ Apache2 running on port 80/443                 │
│  ✓ PHP 7.4/8.1 configured and active              │
│  ✓ PrestaShop 8.1.2 deployed & ready              │
│  ✓ Connected to MySQL on VM1 (192.168.56.11)     │
│  ✓ SSL configured for HTTPS                       │
│                                                    │
└────────────────────────────────────────────────────┘
```

## Description
This role installs, configures, and optimizes Prestashop 8.x with Apache2 and PHP. It manages complete web server installation, Apache VirtualHost configuration, PHP installation with all required extensions, and Prestashop deployment.

## Features
- Installation of Apache2 with required modules
- Installation and configuration of PHP 7.4/8.0/8.1 with extensions
- Download and extraction of Prestashop
- Apache VirtualHost configuration
- Configuration of remote MySQL connection
- PHP optimizations for Prestashop
- File permissions configuration
- Support for multiple domains
- Configurable logs
- Handlers for automatic restart

## Required Variables

### Mandatory variables
```yaml
prestashop_version: "8.1.2"              # Prestashop version
prestashop_install_dir: "/var/www/prestashop"  # Installation directory
prestashop_db_host: "192.168.56.11"      # MySQL host
prestashop_db_name: "prestashop_db"      # Database name
prestashop_db_user: "prestashop_user"    # Database user
prestashop_db_password: "SecurePassword" # Database password
php_version: "7.4"                       # PHP version
```

### Optional variables (with default values)
```yaml
prestashop_domain: "192.168.56.12"
prestashop_admin_email: "admin@prestashop.local"
prestashop_admin_password: "Admin@123"
php_memory_limit: "256M"
php_max_execution_time: 60
php_upload_max_filesize: "100M"
php_post_max_size: "100M"
apache_vhost_port: 80
apache_vhost_servername: "192.168.56.12"
```

## Role Files

```
prestashop/
├── README.md                      # This file
├── defaults/main.yml              # Default variables
├── handlers/main.yml              # Handlers (restarts)
├── tasks/main.yml                 # Installation tasks
├── templates/
│   ├── prestashop.conf.j2         # Apache VirtualHost template
│   └── php.ini.j2                 # PHP configuration template
└── meta/main.yml                  # Metadata and dependencies
```

## Usage

### Basic usage
```yaml
- name: Deploy Prestashop
  hosts: web_servers
  become: true
  roles:
    - role: prestashop
```

### Usage with custom configuration
```yaml
- name: Deploy customized Prestashop
  hosts: web_servers
  become: true
  vars:
    prestashop_version: "8.1.2"
    prestashop_domain: "shop.example.com"
    prestashop_db_host: "db.example.com"
    php_version: "8.1"
    php_memory_limit: "512M"
  roles:
    - role: prestashop
```

### Sensitive variables (Vault)
```bash
# Create a vault file
ansible-vault create group_vars/web_servers/vault.yml

# Contents vault.yml
vault_prestashop_admin_password: "MyStrongAdminPassword123!"
vault_prestashop_db_password: "MyStrongDBPassword456!"
```

## Main Tasks

This role executes the following tasks:
1. Installation of system dependencies
2. Addition of PHP PPA repository
3. Installation of Apache2 and modules
4. Installation of PHP and extensions
5. Activation of required Apache modules
6. Download of Prestashop
7. Extraction of Prestashop
8. Configuration of permissions
9. Configuration of Apache VirtualHost
10. Configuration of optimized PHP.ini
11. Test of MySQL connection
12. Activation of VirtualHost
13. Restart of services

## Environment-Specific Variables

### Production
```yaml
# host_vars/web_prod.yml
php_memory_limit: "512M"
php_max_execution_time: 120
mysql_backup_enabled: true
mysql_max_connections: 200
```

### Staging
```yaml
# host_vars/web_staging.yml
php_memory_limit: "256M"
php_max_execution_time: 60
mysql_backup_enabled: false
mysql_max_connections: 50
```

## Tests

### Verify Apache is active
```bash
sudo systemctl status apache2
```

### Test Apache configuration
```bash
sudo apache2ctl configtest
```

### Access Prestashop
```
http://192.168.56.12
```

### Check Apache modules
```bash
sudo apache2ctl -M | grep rewrite
```

### Test PHP-MySQL connection
```bash
mysql -h 192.168.56.11 -u prestashop_user -p prestashop_db
```

### Check PHP version
```bash
php -v
php -m  # List loaded modules
```

## Troubleshooting

### Apache does not start
```bash
# Check configuration
sudo apache2ctl configtest

# Check logs
sudo tail -f /var/log/apache2/error.log

# Check permissions
sudo chown -R www-data:www-data /var/www/prestashop
```

### Prestashop does not display
```bash
# Check Apache logs
tail -f /var/log/apache2/prestashop_error.log

# Check file permissions
sudo chmod -R 755 /var/www/prestashop
sudo chmod -R 644 /var/www/prestashop/config

# Check database access
mysql -h 192.168.56.11 -u prestashop_user -p prestashop_db
```

### Prestashop download issue
```bash
# Check internet connectivity
sudo curl https://www.prestashop.com/

# Remove and re-download
sudo rm -rf /var/www/prestashop/*
# Re-run the playbook
```

### Slow performance
```bash
# Check resource usage
free -h
df -h

# Check MySQL slow queries
mysqldumpslow /var/log/mysql/slow.log
```

## Performance Optimization

Key optimizations implemented:
- PHP memory limit optimized per environment
- Apache modules optimized for Prestashop
- Database connection pooling enabled
- File caching configured
- Compression enabled for static files
- Database slow query logging enabled

## Performance et Optimisation

### PHP Configuration
- `php_memory_limit`: Minimum 256MB, recommandé 512MB
- `php_max_execution_time`: 30-120 secondes selon vos besoins
- `php_upload_max_filesize`: Réglez selon vos fichiers

### Apache Configuration
- `KeepAlive`: Activé pour les connexions persistantes
- `MaxRequestWorkers`: Réglez selon votre RAM disponible
- Compression gzip: Activée par défaut

### Prestashop Optimizations
- Cache opcode : Installez PHP-FPM pour une performance optimale
- Cache template : Activé par défaut
- Compression CSS/JS : À configurer dans Prestashop

## Gestion des logs

### Apache logs
- Error log: `/var/log/apache2/prestashop_error.log`
- Access log: `/var/log/apache2/prestashop_access.log`

### PHP logs
- Errors: Configurés dans `php.ini`
- Logs d'application: Dans `/var/www/prestashop/var/logs/`

## Sécurité

### Mesures de sécurité implémentées
- Permissions de fichiers restrictives (755 répertoires, 644 fichiers)
- Propriétaire www-data pour accès web
- Séparation des fichiers de configuration
- Utilisateur MySQL dédié avec privilèges minimaux
- Configuration SSL recommandée (non activée par défaut)
- Suppression des fichiers sensibles

### Recommandations supplémentaires
- Utilisez HTTPS en production
- Activez Mod_Security dans Apache
- Installez un WAF (Web Application Firewall)
- Mettez à jour régulièrement PHP et Prestashop
- Configurez un SSL Let's Encrypt

## Support et Contribution

Pour toute question ou contribution, veuillez contacter l'équipe DevOps.

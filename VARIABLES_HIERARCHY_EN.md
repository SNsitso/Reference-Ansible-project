---
# 🏗️ Variables Architecture - Hierarchy and Resolution

## Overview

Ansible resolves variables according to a **priority hierarchy**. Understanding this hierarchy is essential for correctly managing variables in your project.

---

## 📊 Variables Hierarchy (bottom to top)

```
Priority (bottom to top):

7. Extra variables (--extra-vars)              ← HIGHEST PRIORITY
6. Include vars / Import vars                  
5. Role vars & Include role defaults           
4. Host variables (host_vars/)                 
3. Group variables (group_vars/)              
2. Role defaults (roles/*/defaults/main.yml)   
1. Plugin defaults                             ← LOWEST PRIORITY
```

**Golden Rule**: The variable defined at the HIGHEST level wins.

---

## 🎯 Practical Example with Your Project

### Scenario: Variable `mysql_max_connections`

#### Level 1: Role Defaults (Universal default value)
```yaml
# roles/mysql/defaults/main.yml
mysql_max_connections: 100
```
✓ Applied if nothing else is defined

#### Level 2: Group Variables (Common to MySQL group)
```yaml
# group_vars/mysql_servers.yml
mysql_max_connections: 200
```
✓ Overrides default for ALL hosts in `mysql_servers` group

#### Level 3: Host Variables (Host-specific)
```yaml
# host_vars/mysql_prod.yml
mysql_max_connections: 300
```
✓ Overrides group_vars for `mysql_prod` ONLY

#### Level 4: Playbook (Extra Variables)
```bash
# Command line
ansible-playbook deploy.yml -e "mysql_max_connections=400"
```
✓ Temporarily overrides EVERYTHING for this execution

**Result**: `mysql_prod` will use `300`, other group hosts will use `200`

---

## 📁 Detailed Project Structure

### Resolution for `mysql_prod`

```
roles/mysql/defaults/main.yml
        ↓ (overridden by)
group_vars/mysql_servers.yml
        ↓ (overridden by)
host_vars/mysql_prod.yml
        ↓ (overridden by)
Playbook vars / Extra vars
        ↓
FINAL VALUE USED
```

### Resolution for `web_prod`

```
roles/prestashop/defaults/main.yml
        ↓ (overridden by)
group_vars/web_servers.yml
        ↓ (overridden by)
host_vars/web_prod.yml
        ↓ (overridden by)
Playbook vars / Extra vars
        ↓
FINAL VALUE USED
```

---

## 🗂️ Recommended Organization in Your Project

### Level 1: Universal Default Values
**File**: `roles/ROLE/defaults/main.yml`

**Contains**:
- Values valid for ALL hosts, ALL environments
- Reasonable defaults
- Generic numbers

**MySQL Example**:
```yaml
mysql_port: 3306                    # Same port everywhere
mysql_service_name: mysql           # Same service name
mysql_log_error: /var/log/mysql/error.log
```

### Level 2: Common Group Configuration
**File**: `group_vars/GROUP_NAME.yml`

**Contains**:
- Configuration common to ALL hosts in a group
- Packages, services, common modules
- Standard paths per group (e.g., /var/log/apache2)

**MySQL Example**:
```yaml
# group_vars/mysql_servers.yml
mysql_bind_address: '0.0.0.0'       # All MySQL accept network
mysql_packages:                      # Same packages everywhere
  - mysql-server
  - mysql-client
```

**Web Example**:
```yaml
# group_vars/web_servers.yml
apache_modules:                      # Same Apache modules everywhere
  - rewrite
  - headers
php_packages:                        # Common PHP extensions
  - php7.4-curl
  - php7.4-gd
```

### Level 3: Host-Specific Configuration
**File**: `host_vars/HOSTNAME.yml`

**Contains**:
- Configuration UNIQUE to this host
- Production vs Staging optimizations
- Passwords and secrets (via vault)
- Host-specific IPs and hostnames

**Production MySQL Example**:
```yaml
# host_vars/mysql_prod.yml
hostname: mysql-server-prod
mysql_max_connections: 200           # Higher for production
mysql_innodb_buffer_pool_size: '1G'  # Larger for production
mysql_root_password: "{{ vault_mysql_root_password }}"
mysql_backup_enabled: yes            # Backups in production
```

**Staging MySQL Example**:
```yaml
# host_vars/mysql_staging.yml
hostname: mysql-server-staging
mysql_max_connections: 50            # Lower in staging
mysql_innodb_buffer_pool_size: '512M' # Less in staging
mysql_backup_enabled: no             # No backups in staging
```

---

## 🔍 Debug: View Resolved Variables

### Display ALL variables for a host
```bash
ansible all -i inventories/production/ \
  -m debug \
  -a "var=vars" | grep -A 5 mysql_max_connections
```

### Display specific variable
```bash
ansible mysql_prod -i inventories/production/ \
  -m setup \
  -a filter=ansible_* | grep mysql
```

### In a playbook (for debugging)
```yaml
- name: "Display resolved variables"
  debug:
    msg: |
      mysql_port: {{ mysql_port }}
      mysql_max_connections: {{ mysql_max_connections }}
      mysql_innodb_buffer_pool_size: {{ mysql_innodb_buffer_pool_size }}
```

---

## 🎨 Variables Hierarchy Best Practices

### ✅ DO

1. **Put defaults in roles**
   ```yaml
   # roles/mysql/defaults/main.yml
   mysql_port: 3306  # Universal default value
   ```

2. **Put common configurations in group_vars**
   ```yaml
   # group_vars/mysql_servers.yml
   mysql_bind_address: '0.0.0.0'  # Common to ALL MySQL servers
   ```

3. **Put specifics in host_vars**
   ```yaml
   # host_vars/mysql_prod.yml
   mysql_max_connections: 300  # Unique to mysql_prod
   ```

4. **Use vault for secrets**
   ```yaml
   # host_vars/mysql_prod.yml
   mysql_root_password: "{{ vault_mysql_root_password }}"  # Encrypted!
   ```

### ❌ DON'T

1. **Don't put everything in defaults**
   ```yaml
   # roles/mysql/defaults/main.yml
   ❌ mysql_max_connections: 300  # Too specific!
   ```

2. **Don't write secrets in plain text**
   ```yaml
   # host_vars/mysql_prod.yml
   ❌ mysql_root_password: "MyRealPassword123"  # DANGER!
   ```

3. **Don't use global variables [all:vars]**
   ```ini
   ❌ [all:vars]
   ansible_user=vagrant  # Put in inventory per host
   ```

4. **Don't mix levels**
   ```yaml
   # group_vars/mysql_servers.yml
   ❌ mysql_prod_max_connections: 300  # Too specific!
   ```

---

## 🔐 Managing Secrets with Vault

### Recommended Structure

```
group_vars/
├── mysql_servers.yml           # Normal variables
├── vault.yml                   # ⚠️ ENCRYPTED SECRETS
├── web_servers.yml             # Normal variables
└── vault_staging.yml           # ⚠️ STAGING ENCRYPTED SECRETS
```

### Using in Variables

```yaml
# host_vars/mysql_prod.yml
mysql_root_password: "{{ vault_mysql_root_password }}"  # Reference to secret
mysql_database: "prestashop_db" # Normal variable
```

### Production Vault File

```yaml
# group_vars/vault.yml (encrypted with ansible-vault)
vault_mysql_root_password: "ReallySecurePassword123!"
vault_prestashop_db_password: "AnotherSecurePassword456!"
vault_prestashop_admin_password: "AdminPassword789!"
```

### Execution with Vault

```bash
# Ask for password
ansible-playbook playbooks/deploy.yml --ask-vault-pass

# Or with password file
ansible-playbook playbooks/deploy.yml --vault-password-file=.vault_pass
```

---

## 📊 Complete Matrix: Who Gets What?

### Case 1: Variable without override
```
Rule: defaults < group_vars < host_vars
```

| Variable | defaults | group_vars | host_vars | Result |
|----------|----------|-----------|-----------|--------|
| mysql_port | 3306 | (absent) | (absent) | 3306 |
| mysql_port | 3306 | 3307 | (absent) | 3307 |
| mysql_port | 3306 | 3307 | 3308 | 3308 |

### Case 2: Group-specific variable
```
group_vars: Applied to ALL hosts in group
```

| Group | mysql_servers | web_servers |
|-------|--------------|------------|
| group_vars | mysql_servers.yml | web_servers.yml |
| Affected hosts | mysql_prod, mysql_staging | web_prod, web_staging |

### Case 3: Host-specific variable
```
host_vars: Applied ONLY to that host
```

| Host | mysql_prod | mysql_staging | web_prod | web_staging |
|------|-----------|--------------|---------|------------|
| host_vars | mysql_prod.yml | mysql_staging.yml | web_prod.yml | web_staging.yml |

---

## 🧪 Testing Variable Resolution

### Validation script
```bash
#!/bin/bash
set -e

echo "=== Checking variable resolution ==="

# Production MySQL
echo "- mysql_prod:"
ansible mysql_prod -i inventories/production/ \
  -m debug \
  -a "msg='mysql_max_connections={{ mysql_max_connections }}'"

# Staging MySQL
echo "- mysql_staging:"
ansible mysql_staging -i inventories/staging/ \
  -m debug \
  -a "msg='mysql_max_connections={{ mysql_max_connections }}'"

# Production Web
echo "- web_prod:"
ansible web_prod -i inventories/production/ \
  -m debug \
  -a "msg='php_memory_limit={{ php_memory_limit }}'"

# Staging Web
echo "- web_staging:"
ansible web_staging -i inventories/staging/ \
  -m debug \
  -a "msg='php_memory_limit={{ php_memory_limit }}'"
```

---

## 🎓 Checklist: Well Organize Variables

- [ ] Defaults in `roles/*/defaults/main.yml` (universal)
- [ ] Group vars in `group_vars/` (group common)
- [ ] Host vars in `host_vars/` (host-specific)
- [ ] Secrets in encrypted vault (never plaintext)
- [ ] Variable documentation in role README
- [ ] Resolution tests with debug
- [ ] No duplication between levels
- [ ] Logical order: defaults < group < host

---

## 📚 Resources

- [Ansible Variable Precedence](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)
- [Using Vault in Playbooks](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
- [Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)

---

## 💡 Summary

| Level | File | Scope | Use For |
|-------|------|-------|---------|
| defaults | roles/*/defaults/main.yml | Universal | Default values |
| group_vars | group_vars/*.yml | Host group | Group common config |
| host_vars | host_vars/*.yml | Single host | Host-specific config |
| vault | (group/host)_vars/vault.yml | Sensitive variable | Encrypted secrets |
| extra-vars | -e "var=value" | Single execution | Temporary override |

---

**Master the variables hierarchy = Master Ansible!** 🚀


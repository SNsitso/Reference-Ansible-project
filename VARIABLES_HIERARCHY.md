---
# 🏗️ Architecture des Variables - Hiérarchie et Résolution

## Vue d'ensemble

Ansible résout les variables selon une **hiérarchie de priorité**. Comprendre cette hiérarchie est essentiel pour gérer correctement les variables dans votre projet.

---

## 📊 Hiérarchie des Variables (du bas au haut)

```
Priorité (de bas en haut):

7. Extra variables (--extra-vars)              ← HIGHEST PRIORITY
6. Include vars / Import vars                  
5. Role vars & Include role defaults           
4. Host variables (host_vars/)                 
3. Group variables (group_vars/)              
2. Role defaults (roles/*/defaults/main.yml)   
1. Plugi defaults                              ← LOWEST PRIORITY
```

**Règle d'or** : La variable définie au niveau MORE HAUT gagne.

---

## 🎯 Exemple Pratique avec Votre Projet

### Scénario: Variable `mysql_max_connections`

#### Niveau 1: Role Defaults (Valeur par défaut universelle)
```yaml
# roles/mysql/defaults/main.yml
mysql_max_connections: 100
```
✓ S'applique si rien d'autre n'est défini

#### Niveau 2: Group Variables (Commun au groupe MySQL)
```yaml
# group_vars/mysql_servers.yml
mysql_max_connections: 200
```
✓ Remplace le default pour TOUS les hôtes du groupe `mysql_servers`

#### Niveau 3: Host Variables (Spécifique à l'hôte)
```yaml
# host_vars/mysql_prod.yml
mysql_max_connections: 300
```
✓ Remplace les group_vars pour `mysql_prod` UNIQUEMENT

#### Niveau 4: Playbook (Extra Variables)
```bash
# En ligne de commande
ansible-playbook deploy.yml -e "mysql_max_connections=400"
```
✓ Remplace TOUT temporairement pour cette exécution

**Résultat** : `mysql_prod` utilisera `300`, autres hôtes du groupe utiliseront `200`

---

## 📁 Structure Détaillée de Votre Projet

### Résolution pour `mysql_prod`

```
roles/mysql/defaults/main.yml
        ↓ (remplacé par)
group_vars/mysql_servers.yml
        ↓ (remplacé par)
host_vars/mysql_prod.yml
        ↓ (remplacé par)
Playbook vars / Extra vars
        ↓
VALEUR FINALE UTILISÉE
```

### Résolution pour `web_prod`

```
roles/prestashop/defaults/main.yml
        ↓ (remplacé par)
group_vars/web_servers.yml
        ↓ (remplacé par)
host_vars/web_prod.yml
        ↓ (remplacé par)
Playbook vars / Extra vars
        ↓
VALEUR FINALE UTILISÉE
```

---

## 🗂️ Organisation Recommandée dans Votre Projet

### Niveau 1: Valeurs par Défaut Universelles
**Fichier** : `roles/ROLE/defaults/main.yml`

**Contient** :
- Valeurs valables pour TOUS les hôtes, TOUS les environnements
- Defaults « raisonnables »
- Chiffres génériques

**Exemple MySQL** :
```yaml
mysql_port: 3306                    # Même port partout
mysql_service_name: mysql           # Même nom de service
mysql_log_error: /var/log/mysql/error.log
```

### Niveau 2: Configuration Commune au Groupe
**Fichier** : `group_vars/GROUP_NAME.yml`

**Contient** :
- Configuration commune à TOUS les hôtes d'un groupe
- Packages, services, modules communs
- Paths standards par groupe (ex: /var/log/apache2)

**Exemple MySQL** :
```yaml
# group_vars/mysql_servers.yml
mysql_bind_address: '0.0.0.0'       # Tous les MySQL acceptent réseau
mysql_packages:                      # Même packages partout
  - mysql-server
  - mysql-client
```

**Exemple Web** :
```yaml
# group_vars/web_servers.yml
apache_modules:                      # Même modules Apache partout
  - rewrite
  - headers
php_packages:                        # Extensions PHP communes
  - php7.4-curl
  - php7.4-gd
```

### Niveau 3: Configuration Spécifique à l'Hôte
**Fichier** : `host_vars/HOSTNAME.yml`

**Contient** :
- Configuration UNIQUE à cet hôte
- Optimisations Production vs Staging
- Mots de passe et secrets (via vault)
- IPs et hostnames spécifiques

**Exemple MySQL Production** :
```yaml
# host_vars/mysql_prod.yml
hostname: mysql-server-prod
mysql_max_connections: 200           # Plus élevé pour production
mysql_innodb_buffer_pool_size: '1G'  # Plus grand pour production
mysql_root_password: "{{ vault_mysql_root_password }}"
mysql_backup_enabled: yes            # Backups en production
```

**Exemple MySQL Staging** :
```yaml
# host_vars/mysql_staging.yml
hostname: mysql-server-staging
mysql_max_connections: 50            # Moins en staging
mysql_innodb_buffer_pool_size: '512M' # Moins en staging
mysql_backup_enabled: no             # Pas de backups en staging
```

---

## 🔍 Debug: Voir les Variables Résolues

### Afficher TOUTES les variables d'un hôte
```bash
ansible all -i inventories/production/ \
  -m debug \
  -a "var=vars" | grep -A 5 mysql_max_connections
```

### Afficher une variable spécifique
```bash
ansible mysql_prod -i inventories/production/ \
  -m setup \
  -a filter=ansible_* | grep mysql
```

### Dans un playbook (pour debug)
```yaml
- name: "Afficher les variables résolues"
  debug:
    msg: |
      mysql_port: {{ mysql_port }}
      mysql_max_connections: {{ mysql_max_connections }}
      mysql_innodb_buffer_pool_size: {{ mysql_innodb_buffer_pool_size }}
```

---

## 🎨 Bonnes Pratiques de Hiérarchie des Variables

### ✅ À FAIRE

1. **Mettez les defaults dans les rôles**
   ```yaml
   # roles/mysql/defaults/main.yml
   mysql_port: 3306  # Valeur par défaut universelle
   ```

2. **Mettez les configurations communes dans group_vars**
   ```yaml
   # group_vars/mysql_servers.yml
   mysql_bind_address: '0.0.0.0'  # Commun à TOUS les MySQL
   ```

3. **Mettez les spécificités dans host_vars**
   ```yaml
   # host_vars/mysql_prod.yml
   mysql_max_connections: 300  # Unique à mysql_prod
   ```

4. **Utilisez vault pour les secrets**
   ```yaml
   # host_vars/mysql_prod.yml
   mysql_root_password: "{{ vault_mysql_root_password }}"  # Chiffré !
   ```

### ❌ À ÉVITER

1. **Ne mettez pas tout dans les defaults**
   ```yaml
   # roles/mysql/defaults/main.yml
   ❌ mysql_max_connections: 300  # Trop spécifique !
   ```

2. **N'écrivez pas les secrets en texte clair**
   ```yaml
   # host_vars/mysql_prod.yml
   ❌ mysql_root_password: "MyRealPassword123"  # DANGER !
   ```

3. **N'utilisez pas de variables globales [all:vars]**
   ```ini
   ❌ [all:vars]
   ansible_user=vagrant  # À mettre dans inventaire par hôte
   ```

4. **Ne mélangez pas les niveaux**
   ```yaml
   # group_vars/mysql_servers.yml
   ❌ mysql_prod_max_connections: 300  # Trop spécifique !
   ```

---

## 🔐 Gestion des Secrets avec Vault

### Structure Recommandée

```
group_vars/
├── mysql_servers.yml           # Variables normales
├── vault.yml                   # ⚠️ SECRETS CHIFFRÉS
├── web_servers.yml             # Variables normales
└── vault_staging.yml           # ⚠️ SECRETS STAGING CHIFFRÉS
```

### Utilisation dans les Variables

```yaml
# host_vars/mysql_prod.yml
mysql_root_password: "{{ vault_mysql_root_password }}"  # Référence au secret
mysql_database: "prestashop_db" # Variable normale
```

### Fichier Vault Production

```yaml
# group_vars/vault.yml (chiffré avec ansible-vault)
vault_mysql_root_password: "ReallySecurePassword123!"
vault_prestashop_db_password: "AnotherSecurePassword456!"
vault_prestashop_admin_password: "AdminPassword789!"
```

### Exécution avec Vault

```bash
# Demander le mot de passe
ansible-playbook playbooks/deploy.yml --ask-vault-pass

# Ou avec fichier de mot de passe
ansible-playbook playbooks/deploy.yml --vault-password-file=.vault_pass
```

---

## 📊 Matrice Complète: Qui prend Quoi?

### Cas 1: Variable sans override
```
Règle: defaults < group_vars < host_vars
```

| Variable | defaults | group_vars | host_vars | Résultat |
|----------|----------|-----------|-----------|----------|
| mysql_port | 3306 | (absent) | (absent) | 3306 |
| mysql_port | 3306 | 3307 | (absent) | 3307 |
| mysql_port | 3306 | 3307 | 3308 | 3308 |

### Cas 2: Variable spécifique à un groupe
```
group_vars: Appliqué à TOUS les hôtes du groupe
```

| Groupe | mysql_servers | web_servers |
|--------|--------------|------------|
| group_vars | mysql_servers.yml | web_servers.yml |
| Hôtes affectés | mysql_prod, mysql_staging | web_prod, web_staging |

### Cas 3: Variable spécifique à un hôte
```
host_vars: Appliqué UNIQUEMENT à cet hôte
```

| Hôte | mysql_prod | mysql_staging | web_prod | web_staging |
|------|-----------|--------------|---------|------------|
| host_vars | mysql_prod.yml | mysql_staging.yml | web_prod.yml | web_staging.yml |

---

## 🧪 Testing de la Résolution des Variables

### Script de validation
```bash
#!/bin/bash
set -e

echo "=== Vérification de la résolution des variables ==="

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

## 🎓 Checklist: Bien Organiser les Variables

- [ ] Defaults dans `roles/*/defaults/main.yml` (universelles)
- [ ] Group vars dans `group_vars/` (communes au groupe)
- [ ] Host vars dans `host_vars/` (spécifiques à l'hôte)
- [ ] Secrets dans vault chiffrés (jamais en clair)
- [ ] Documentation des variables dans role README
- [ ] Tests de résolution avec debug
- [ ] Pas de duplication entre les niveaux
- [ ] Ordre logique: defaults < group < host

---

## 📚 Ressources

- [Ansible Variable Precedence](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)
- [Using Vault in Playbooks](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
- [Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)

---

## 💡 Résumé

| Niveau | Fichier | Scope | Utilisez pour |
|--------|---------|-------|---------------|
| defaults | roles/*/defaults/main.yml | Universel | Valeurs par défaut |
| group_vars | group_vars/*.yml | Groupe d'hôtes | Config commune groupe |
| host_vars | host_vars/*.yml | Hôte unique | Config spécifique hôte |
| vault | (group/host)_vars/vault.yml | Variable sensible | Secrets chiffrés |
| extra-vars | -e "var=value" | Exécution unique | Override temporaire |

---

**Maîtrisez la hiérarchie des variables = Maîtrisez Ansible !** 🚀


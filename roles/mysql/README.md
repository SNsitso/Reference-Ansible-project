# Rôle Ansible : mysql

Installe et configure un serveur MySQL durci, prêt à servir une application
web distante (ici PrestaShop) : base de données applicative, utilisateur
dédié, écoute réseau, tuning par environnement et sauvegardes optionnelles.

## Fonctionnement

1. Installation des paquets (`mysql-server`, `mysql-client`, `python3-pymysql`).
2. Démarrage + activation du service.
3. Définition du mot de passe root (via socket Unix, donc idempotent) et
   dépôt de `/root/.my.cnf` (mode `0600`) pour la CLI.
4. Durcissement : suppression des utilisateurs anonymes et de la base `test`.
5. Création de la base applicative (charset/collation paramétrables) et de
   l'utilisateur applicatif avec privilèges limités à cette base.
6. Dépôt de `/etc/mysql/mysql.conf.d/99-ansible.cnf` (bind-address, port,
   max_connections, buffer pool, slow query log) — déclenche un restart via handler.
7. Sauvegarde quotidienne `mysqldump` par cron si `mysql_backup_enabled`.

Toutes les opérations d'administration passent par `login_unix_socket` : elles
fonctionnent avant comme après la pose du mot de passe root, sans `ignore_errors`.

## Variables

### Requises (aucun défaut — à fournir via Vault)

| Variable | Description |
| --- | --- |
| `mysql_root_password` | Mot de passe root MySQL |
| `mysql_app_password` | Mot de passe de l'utilisateur applicatif |

### Principales (defaults surchargables)

| Variable | Défaut | Description |
| --- | --- | --- |
| `mysql_database` | `prestashop_db` | Base applicative |
| `mysql_app_user` | `prestashop_user` | Utilisateur applicatif |
| `mysql_app_user_host` | `%` | Hôtes autorisés à se connecter |
| `mysql_app_privileges` | `ALL` | Privilèges sur la base applicative |
| `mysql_bind_address` | `0.0.0.0` | Adresse d'écoute |
| `mysql_port` | `3306` | Port d'écoute |
| `mysql_max_connections` | `100` | Tuning : connexions max |
| `mysql_innodb_buffer_pool_size` | `512M` | Tuning : buffer pool InnoDB |
| `mysql_slow_query_log` | `false` | Journal des requêtes lentes |
| `mysql_backup_enabled` | `false` | Cron `mysqldump` quotidien |
| `mysql_backup_dir` | `/var/backups/mysql` | Destination des dumps |

Voir [defaults/main.yml](defaults/main.yml) pour la liste complète.

## Exemple

```yaml
- hosts: mysql_servers
  become: true
  roles:
    - role: mysql
      vars:
        mysql_database: boutique_db
        mysql_app_user: boutique_user
        mysql_root_password: "{{ vault_mysql_root_password }}"
        mysql_app_password: "{{ vault_mysql_app_password }}"
```

## Vérification

```bash
sudo systemctl status mysql
sudo mysql -e "SHOW DATABASES;"                      # via /root/.my.cnf
mysql -h <ip> -u prestashop_user -p -e "SELECT 1;"   # depuis le serveur web
```

## Sécurité

- Aucun secret dans les defaults ni dans les logs (`no_log: true` sur les
  tâches manipulant des mots de passe).
- Utilisateur applicatif limité à SA base (pas de `*.*`).
- Utilisateurs anonymes et base `test` supprimés.
- `/root/.my.cnf` en `0600`.

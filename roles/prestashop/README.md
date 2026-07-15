# Rôle Ansible : prestashop

Installe Apache + PHP et déploie PrestaShop connecté à un serveur MySQL
distant. Le rôle vérifie la connectivité base de données en fin d'exécution.

## Fonctionnement

1. Installation des dépendances système (Apache, unzip, curl, mysql-client…).
2. Ajout du PPA PHP (`ppa:ondrej/php`) et installation de PHP + extensions
   requises par PrestaShop (curl, gd, intl, mbstring, mysql, xml, zip…).
3. Activation des modules Apache (`rewrite`, `headers`, `ssl`).
4. Téléchargement de l'archive PrestaShop depuis GitHub, extraction de
   l'archive externe puis de l'archive interne (idempotent via `creates:`).
5. Permissions : propriétaire `www-data`, mode symbolique `u=rwX,g=rX,o=rX`
   (les répertoires restent exécutables, les fichiers non).
6. VirtualHost Apache paramétré (port, domaine, logs dédiés), activation du
   site et désactivation du site par défaut (idempotent via `creates`/`removes`).
7. Configuration PHP dédiée (`99-prestashop.ini`) entièrement pilotée par variables.
8. Test de connexion à la base distante : le mot de passe transite par la
   variable d'environnement `MYSQL_PWD` (jamais en argument de commande) et la
   tâche est en `no_log`. Un échec de connexion fait échouer le déploiement.

## Variables

### Requises (aucun défaut — à fournir via Vault)

| Variable | Description |
| --- | --- |
| `prestashop_db_password` | Mot de passe de l'utilisateur MySQL applicatif |
| `prestashop_admin_password` | Mot de passe admin PrestaShop |

### Principales (defaults surchargables)

| Variable | Défaut | Description |
| --- | --- | --- |
| `prestashop_version` | `8.1.2` | Version téléchargée depuis GitHub |
| `prestashop_install_dir` | `/var/www/prestashop` | Répertoire d'installation |
| `prestashop_domain` | `localhost` | ServerName du VirtualHost |
| `prestashop_db_host` | `localhost` | Hôte MySQL distant |
| `prestashop_db_port` | `3306` | Port MySQL |
| `prestashop_db_name` | `prestashop_db` | Base de données |
| `prestashop_db_user` | `prestashop_user` | Utilisateur MySQL |
| `php_version` | `7.4` | Version PHP installée via PPA |
| `php_memory_limit` | `256M` | `memory_limit` |
| `php_upload_max_filesize` | `20M` | `upload_max_filesize` |
| `php_post_max_size` | `22M` | `post_max_size` |
| `php_max_execution_time` | `300` | `max_execution_time` |
| `apache_vhost_port` | `80` | Port du VirtualHost |
| `apache_modules` | `[rewrite, headers, ssl]` | Modules Apache activés |

Voir [defaults/main.yml](defaults/main.yml) pour la liste complète.

## Exemple

```yaml
- hosts: web_servers
  become: true
  roles:
    - role: prestashop
      vars:
        prestashop_domain: boutique.example.com
        prestashop_db_host: "{{ hostvars['mysql_prod'].ansible_host }}"
        prestashop_db_password: "{{ vault_mysql_app_password }}"
        prestashop_admin_password: "{{ vault_prestashop_admin_password }}"
```

## Vérification

```bash
sudo systemctl status apache2
sudo apache2ctl configtest
curl -I http://<ip>            # doit répondre 200/302
tail -f /var/log/apache2/prestashop_error.log
```

## Dépannage

- **Erreur 500** : vérifier `prestashop_error.log` et les permissions du
  répertoire d'installation.
- **Connexion DB échouée** : la tâche `Test connection...` échoue → vérifier
  que MySQL écoute (`bind-address`), que l'utilisateur autorise l'hôte (`%`)
  et que le port 3306 est joignable (`nc -zv <ip> 3306`).
- **Page d'installation absente** : l'archive interne `prestashop.zip` n'a
  peut-être pas été extraite — vérifier la présence de `index.php` dans
  `prestashop_install_dir`.

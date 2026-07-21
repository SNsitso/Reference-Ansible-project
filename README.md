# Ansible Reference - Déploiement PrestaShop + MySQL

Projet Ansible de référence : un **playbook unique haut niveau** (`site.yml`)
déploie une plateforme e-commerce PrestaShop + MySQL sur l'environnement de
votre choix (production, staging, ou tout nouvel inventaire que vous ajoutez).

> Documentation détaillée du projet et du processus : [PRESENTATION.md](PRESENTATION.md)
> Notes d'apprentissage / questions-réponses : [culture.md](culture.md)

## Architecture

```text
┌────────────────────────────┐      ┌────────────────────────────┐
│   VM1 - MYSQL SERVER       │      │   VM2 - WEB SERVER         │
│   192.168.56.11            │◄─────│   192.168.56.12            │
│   Ubuntu 20.04 / 2 GB      │ 3306 │   Ubuntu 20.04 / 2 GB      │
│                            │      │                            │
│   Rôle: mysql              │      │   Rôle: prestashop         │
│   MySQL 8 durci            │      │   Apache + PHP 7.4         │
│   + contrôleur Ansible     │      │   PrestaShop 8.1.2         │
└────────────────────────────┘      └────────────────────────────┘
        Réseau privé 192.168.56.0/24 (Vagrant/VirtualBox)
```

## Structure du projet

```text
project_ansible/
├── site.yml                  # Point d'entrée UNIQUE (haut niveau, réutilisable)
├── ansible.cfg               # Config Ansible (inventaire par défaut: production)
├── Vagrantfile               # Lab local : 2 VMs VirtualBox
│
├── inventories/              # 1 dossier = 1 environnement ("site")
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/all/vault.yml.example   # secrets (à chiffrer)
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/all/vault.yml.example
│
├── group_vars/               # Variables communes par GROUPE (tous envs)
│   ├── mysql_servers.yml
│   └── web_servers.yml
├── host_vars/                # Variables par HÔTE (spécifiques env)
│   ├── mysql_prod.yml / mysql_staging.yml
│   └── web_prod.yml   / web_staging.yml
│
└── roles/
    ├── mysql/                # MySQL durci, tuning, backups (README dédié)
    └── prestashop/           # Apache + PHP + PrestaShop (README dédié)
```

## Démarrage rapide

```bash
# 1. Créer les VMs (depuis Windows)
vagrant up

# 2. Se connecter au contrôleur (VM1)
vagrant ssh db
cd ansible

# 3. Copier la clé SSH vers le serveur web (mot de passe: vagrant)
ssh-copy-id vagrant@192.168.56.12

# 4. Créer le vault de l'environnement
cp inventories/production/group_vars/all/vault.yml.example \
   inventories/production/group_vars/all/vault.yml
# éditer les mots de passe, puis :
ansible-vault encrypt inventories/production/group_vars/all/vault.yml

# 5. Déployer
ansible-playbook site.yml --ask-vault-pass
```

## Un playbook, plusieurs environnements

Le même `site.yml` s'applique à n'importe quel environnement — c'est
l'inventaire qui décide de la cible et de ses variables :

```bash
ansible-playbook site.yml -i inventories/production --ask-vault-pass
ansible-playbook site.yml -i inventories/staging    --ask-vault-pass

# Simulation (aucun changement appliqué)
ansible-playbook site.yml -i inventories/production --check --ask-vault-pass

# Déploiement partiel par tag ou par hôte
ansible-playbook site.yml --tags database --ask-vault-pass
ansible-playbook site.yml --tags web --limit web_prod --ask-vault-pass
```

Pour ajouter un environnement (ex. `preprod`) : dupliquer un dossier
d'inventaire, adapter les IPs/hosts, créer son vault — `site.yml` ne change pas.

## Hiérarchie des variables (du moins au plus prioritaire)

1. `roles/*/defaults/main.yml` — défauts universels non sensibles
2. `group_vars/<groupe>.yml` — commun à un groupe, tous environnements
3. `host_vars/<hôte>.yml` — spécifique à un hôte (prod vs staging)
4. `inventories/<env>/group_vars/all/vault.yml` — secrets chiffrés par env
5. `-e var=valeur` — surcharge ponctuelle en ligne de commande

Exemple : `mysql_max_connections` vaut 100 (default) → 200 sur `mysql_prod`
(host_vars) → 50 sur `mysql_staging` (host_vars).

## Sécurité

- **Aucun secret en clair dans le dépôt** : les mots de passe vivent dans les
  vaults chiffrés par environnement (seuls les `.example` sont versionnés).
- `no_log: true` sur toutes les tâches manipulant des mots de passe.
- Mot de passe MySQL jamais passé en argument CLI (variable `MYSQL_PWD`).
- Utilisateur MySQL applicatif limité à sa base ; utilisateurs anonymes et
  base `test` supprimés ; `/root/.my.cnf` en `0600`.

## Vérification après déploiement

```bash
sudo systemctl status mysql        # sur VM1
sudo systemctl status apache2      # sur VM2
mysql -h 192.168.56.11 -u prestashop_user -p -e "SELECT 1;"   # depuis VM2
# Depuis Windows : http://192.168.56.12 (ou http://localhost:8081)
```

## Documentation

| Document | Contenu |
| --- | --- |
| [PRESENTATION.md](PRESENTATION.md) | Présentation détaillée du projet et du processus |
| [culture.md](culture.md) | Questions/réponses d'apprentissage Ansible |
| [roles/mysql/README.md](roles/mysql/README.md) | Documentation du rôle mysql |
| [roles/prestashop/README.md](roles/prestashop/README.md) | Documentation du rôle prestashop |
| [VARIABLES_HIERARCHY.md](VARIABLES_HIERARCHY.md) | Hiérarchie et résolution des variables (à jour) |
| `deploy_logs.txt`, `LOGS_DEPLOIEMENT_FINAL.txt` | Logs réels du déploiement de la version examen (archives) |

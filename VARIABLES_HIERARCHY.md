# Architecture des Variables - Hiérarchie et Résolution

> Guide aligné sur la structure actuelle du projet (playbook unique `site.yml`
> à la racine, vaults par environnement dans `inventories/<env>/group_vars/all/`).

## Vue d'ensemble

Ansible résout chaque variable selon une **hiérarchie de priorité**. Règle
d'or : **la définition la plus spécifique et la plus proche de l'exécution
gagne**. Comprendre cette hiérarchie est essentiel pour savoir où déclarer
quoi.

---

## Hiérarchie des variables (du moins au plus prioritaire)

```text
5. Extra variables (-e "var=valeur")             ← PRIORITÉ MAXIMALE
4. host_vars/<hôte>.yml                          (spécifique à un hôte)
3. group_vars/<groupe>.yml                       (commun à un groupe)
2. group_vars/all (dont vault.yml d'inventaire)  (commun à tous les hôtes)
1. roles/*/defaults/main.yml                     ← PRIORITÉ MINIMALE
```

(La liste officielle complète compte ~22 niveaux ; ceux-ci sont les seuls
utilisés dans ce projet.)

**Où ces fichiers sont-ils cherchés ?** Uniquement à côté du **playbook**
exécuté et à côté de **l'inventaire** utilisé. C'est pourquoi `site.yml` est
à la racine du projet (il rend visibles `group_vars/` et `host_vars/` racine)
et pourquoi les vaults vivent dans `inventories/<env>/group_vars/all/`.

---

## Exemple réel du projet : `mysql_max_connections`

```yaml
# 1. roles/mysql/defaults/main.yml (défaut universel)
mysql_max_connections: 100

# 2. host_vars/mysql_prod.yml (production : plus de charge)
mysql_max_connections: 200

# 3. host_vars/mysql_staging.yml (staging : ressources réduites)
mysql_max_connections: 50
```

```bash
# 4. Surcharge ponctuelle en ligne de commande (test de charge par ex.)
ansible-playbook site.yml -e "mysql_max_connections=400" --ask-vault-pass
```

**Résultat** :

| Contexte | Valeur utilisée | Vient de |
| --- | --- | --- |
| `mysql_prod` | 200 | host_vars |
| `mysql_staging` | 50 | host_vars |
| hôte MySQL sans host_vars | 100 | defaults du rôle |
| n'importe qui, avec `-e` | 400 | ligne de commande |

---

## Chaîne de résolution pour un hôte du projet

```text
Résolution pour mysql_prod (inventaire production) :

roles/mysql/defaults/main.yml                     mysql_max_connections: 100
        ↓ remplacé par
inventories/production/group_vars/all/vault.yml   vault_mysql_root_password: ***
        ↓ complété par
group_vars/mysql_servers.yml                      mysql_bind_address: 0.0.0.0
        ↓ remplacé par
host_vars/mysql_prod.yml                          mysql_max_connections: 200
        ↓ remplacé par
-e en ligne de commande (si fourni)
        ↓
                VALEURS FINALES UTILISÉES PAR LE PLAY
```

---

## Qui contient quoi (organisation du projet)

### Niveau 1 : defaults du rôle — `roles/<rôle>/defaults/main.yml`

Valeurs par défaut **universelles et non sensibles**. Le rôle doit pouvoir
fonctionner seul avec ces valeurs (comme un rôle publié sur Galaxy).

```yaml
# roles/mysql/defaults/main.yml
mysql_port: 3306
mysql_database: prestashop_db
mysql_max_connections: 100
# NB : aucun mot de passe ici — les secrets n'ont volontairement AUCUN défaut
```

### Niveau 2 : variables de groupe — `group_vars/<groupe>.yml`

Ce qui doit être **garanti sur tout le parc** d'un groupe, tous
environnements confondus. On n'y duplique pas les defaults.

```yaml
# group_vars/mysql_servers.yml
mysql_bind_address: '0.0.0.0'   # tout MySQL du parc accepte le réseau
mysql_remove_anonymous_users: true
```

### Niveau 3 : variables d'hôte — `host_vars/<hôte>.yml`

Ce qui est **unique à un hôte** : c'est ici que production et staging
divergent (tuning, chemins, domaines) et que les secrets sont référencés.

```yaml
# host_vars/mysql_prod.yml
mysql_max_connections: 200
mysql_innodb_buffer_pool_size: 1G
mysql_backup_enabled: true
mysql_root_password: "{{ vault_mysql_root_password }}"   # référence au vault
```

### Niveau 4 : secrets — `inventories/<env>/group_vars/all/vault.yml`

Un vault **par environnement**, chiffré avec `ansible-vault`, chargé
uniquement quand on cible cet inventaire. Mêmes noms de variables dans tous
les environnements, valeurs différentes :

```yaml
# inventories/production/group_vars/all/vault.yml (chiffré)
vault_mysql_root_password: "..."
vault_mysql_app_password: "..."
vault_prestashop_admin_password: "..."
```

```bash
# Création depuis le modèle versionné
cp inventories/production/group_vars/all/vault.yml.example \
   inventories/production/group_vars/all/vault.yml
ansible-vault encrypt inventories/production/group_vars/all/vault.yml

# Exécution
ansible-playbook site.yml -i inventories/production --ask-vault-pass
```

### Niveau 5 : extra vars — `-e`

Surcharge ponctuelle pour UNE exécution (debug, test). Gagne sur tout.

---

## Bonnes pratiques appliquées dans ce projet

### À faire

1. **Defaults complets et non sensibles dans les rôles** — le rôle est
   autonome et publiable.
2. **group_vars minimalistes** — uniquement ce qui diffère des defaults ou
   doit être garanti au niveau du groupe (pas de duplication).
3. **Différences d'environnement dans host_vars** — prod vs staging.
4. **Secrets uniquement via vault** — référencés par
   `"{{ vault_... }}"`, jamais de valeur en clair, jamais de défaut.
5. **Mêmes noms de variables vault partout** — la valeur change avec
   l'inventaire, pas le nom (`vault_mysql_app_password` en prod ET staging).

### À éviter

1. Une valeur spécifique à un environnement dans les defaults d'un rôle.
2. Un secret en clair à n'importe quel niveau (y compris les defaults —
   c'était le cas avant le refactor, voir PRESENTATION.md §8).
3. Des noms préfixés par environnement (`mysql_prod_max_connections`) — la
   hiérarchie fait déjà ce travail.
4. Un fichier `group_vars/` dont le nom ne correspond à aucun groupe : il
   n'est **jamais chargé** (c'était le bug de l'ancien `group_vars/vault.yml`).

---

## Debug : voir les variables résolues

```bash
# Une variable précise pour un hôte donné (exécuté depuis la racine du projet)
ansible mysql_prod -i inventories/production -m debug \
  -a "msg={{ mysql_max_connections }}"

ansible web_staging -i inventories/staging -m debug \
  -a "msg={{ php_memory_limit }}"

# Toutes les variables d'un hôte (verbeux)
ansible mysql_prod -i inventories/production -m debug -a "var=hostvars[inventory_hostname]"

# Vérifier l'inventaire et l'appartenance aux groupes
ansible-inventory -i inventories/production --graph
```

Attendu avec les valeurs actuelles : `mysql_prod` → 200, `mysql_staging` → 50,
`web_prod` → 256M, `web_staging` → 128M.

---

## Résumé

| Niveau | Fichier | Portée | Utiliser pour |
| --- | --- | --- | --- |
| defaults | `roles/*/defaults/main.yml` | Universelle | Défauts non sensibles |
| vault | `inventories/<env>/group_vars/all/vault.yml` | Tous les hôtes d'UN environnement | Secrets chiffrés |
| group_vars | `group_vars/<groupe>.yml` | Groupe, tous envs | Garanties de parc |
| host_vars | `host_vars/<hôte>.yml` | Un hôte | Différences prod/staging |
| extra-vars | `-e "var=valeur"` | Une exécution | Surcharge temporaire |

Pour le « pourquoi » détaillé de chaque mécanisme, voir
[culture.md](culture.md) (section « Anatomie du projet ») et
[PRESENTATION.md](PRESENTATION.md) §4.

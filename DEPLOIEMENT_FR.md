# 🚀 GUIDE DE DÉPLOIEMENT PAS À PAS

## 📋 Vue d'ensemble

Ce guide vous accompagne étape par étape pour déployer un site e-commerce Prestashop avec MySQL en utilisant Ansible et Vagrant.

**Architecture finale** :
```
┌─────────────────────┐         ┌─────────────────────┐
│   VM1 (db)          │         │   VM2 (web)         │
│  192.168.56.11      │◄───────►│  192.168.56.12      │
│                     │  réseau │                     │
│  MySQL 3306         │         │  Apache 80          │
│  + Ansible Control  │  SSH    │  Prestashop         │
└─────────────────────┘         └─────────────────────┘
```

---

## ⚙️ PRÉREQUIS

Avant de commencer, assurez-vous d'avoir :

- [x] **Vagrant** installé (version 2.2+)
- [x] **VirtualBox** installé (version 6.0+)
- [x] **8 GB RAM** minimum disponible
- [x] **10 GB** d'espace disque libre
- [x] Connexion internet active

### Vérification des prérequis

```powershell
# Vérifier Vagrant
vagrant --version

# Vérifier VirtualBox
vboxmanage --version
```

---

## 📝 ÉTAPE 1 : PRÉPARATION DU PROJET

### 1.1 Ouvrir PowerShell

```powershell
cd "C:\Users\Serge Nyuiadzi\Downloads\exam_ansible"
```

### 1.2 Vérifier les fichiers du projet

```powershell
Get-ChildItem
```

Vous devriez voir :
```
Vagrantfile
ansible.cfg
deploy.yml
inventory.ini
README.md
roles/
```

---

## 🖥️ ÉTAPE 2 : CRÉATION DES VMs AVEC VAGRANT

### 2.1 Démarrer les VMs

```powershell
vagrant up
```

**Ce qui se passe** :
- ✅ Téléchargement de Ubuntu 20.04 (première fois uniquement)
- ✅ Création de VM1 (db) - MySQL + Ansible
- ✅ Création de VM2 (web) - Prestashop
- ✅ Configuration réseau privé
- ✅ Installation d'Ansible sur VM1
- ✅ Synchronisation du projet dans VM1

**Durée estimée** : 5-10 minutes (première fois)

### 2.2 Vérifier l'état des VMs

```powershell
vagrant status
```

Résultat attendu :
```
db      running (virtualbox)
web     running (virtualbox)
```

---

## 🔑 ÉTAPE 3 : CONFIGURATION SSH

### 3.1 Se connecter à VM1 (contrôleur Ansible)

```powershell
vagrant ssh db
```

Vous êtes maintenant dans VM1 ! 🎉

### 3.2 Configurer l'accès SSH vers VM2

```bash
# Copier la clé SSH vers VM2
ssh-copy-id vagrant@192.168.56.12
```

Quand demandé :
- **Password** : `vagrant`
- Taper `yes` pour accepter la clé

### 3.3 Tester la connexion SSH

```bash
ssh vagrant@192.168.56.12 "echo 'Connexion réussie!'"
```

Si vous voyez "Connexion réussie!" → ✅ Parfait !

---

## 📦 ÉTAPE 4 : PRÉPARATION DU DÉPLOIEMENT ANSIBLE

### 4.1 Aller dans le dossier du projet

```bash
cd ~/ansible
ls -la
```

Vous devriez voir tous les fichiers du projet synchronisés depuis Windows.

### 4.2 Vérifier la configuration Ansible

```bash
# Vérifier la syntaxe du playbook
ansible-playbook deploy.yml --syntax-check
```

Résultat attendu : `playbook: deploy.yml`

### 4.3 Tester l'inventaire

```bash
# Lister les hôtes
ansible all -i inventory.ini --list-hosts
```

Résultat attendu :
```
hosts (2):
  db
  web
```

### 4.4 Tester la connectivité

```bash
# Ping les deux machines
ansible all -i inventory.ini -m ping
```

Résultat attendu :
```
db | SUCCESS => {"ping": "pong"}
web | SUCCESS => {"ping": "pong"}
```

✅ **Si tout est vert, vous êtes prêt !**

---

## 🚀 ÉTAPE 5 : DÉPLOIEMENT AVEC ANSIBLE

### 5.1 Lancer le déploiement avec logs

```bash
# Déploiement complet avec logs verbeux
ansible-playbook deploy.yml -vvv 2>&1 | tee deployment.log
```

**Ce qui va se passer** :

1. **Phase MySQL (VM1)** :
   - Installation de MySQL Server
   - Configuration sécurisée
   - Création de la base de données `prestashop_db`
   - Création de l'utilisateur `prestashop_user`
   - Configuration réseau (accepte connexions de VM2)

2. **Phase Prestashop (VM2)** :
   - Installation Apache + PHP 8.1
   - Téléchargement Prestashop 8.1.2
   - Configuration VirtualHost
   - Test de connexion à MySQL
   - Configuration finale

**Durée estimée** : 10-15 minutes

### 5.2 Suivre la progression

Vous verrez défiler :
- `TASK [mysql : ...]` - Installation MySQL
- `TASK [prestashop : ...]` - Installation Prestashop
- Messages de succès en vert
- Résumé final

### 5.3 Vérifier le résumé final

À la fin, vous devriez voir :
```
PLAY RECAP ***********************
db        : ok=XX   changed=YY
web       : ok=XX   changed=YY
```

✅ **Si "failed=0" partout → Déploiement réussi !**

---

## ✅ ÉTAPE 6 : VÉRIFICATION DU DÉPLOIEMENT

### 6.1 Vérifier MySQL sur VM1

```bash
# Vous êtes toujours dans VM1
sudo systemctl status mysql
```

Résultat attendu : `Active: active (running)`

```bash
# Tester la connexion à la base
mysql -u prestashop_user -pPrestashop@123 -e "SHOW DATABASES;"
```

Vous devriez voir `prestashop_db` ✅

### 6.2 Vérifier Apache sur VM2

```bash
# Depuis VM1, tester Apache sur VM2
ssh vagrant@192.168.56.12 "sudo systemctl status apache2"
```

Résultat attendu : `Active: active (running)`

### 6.3 Tester la connexion MySQL depuis VM2

```bash
# Vérifier que VM2 peut se connecter à MySQL de VM1
ssh vagrant@192.168.56.12 "mysql -h 192.168.56.11 -u prestashop_user -pPrestashop@123 -e 'SHOW DATABASES;'"
```

Si vous voyez `prestashop_db` → ✅ **Communication inter-serveurs OK !**

---

## 🌐 ÉTAPE 7 : ACCÈS AU SITE WEB

### 7.1 Depuis VM1, sortir vers Windows

```bash
exit
```

Vous êtes de retour dans PowerShell Windows.

### 7.2 Ouvrir le navigateur

Accédez à : **http://localhost:8080**

Vous devriez voir l'**assistant d'installation de Prestashop** ! 🎉

### 7.3 Suivre l'installation Prestashop

1. **Étape 1** : Choix de la langue → Français → Suivant

2. **Étape 2** : Accepter les licences → Suivant

3. **Étape 3** : Informations de la boutique
   - Nom : `Ma Boutique Test`
   - Pays : France
   - Email admin : `admin@test.com`
   - Mot de passe admin : `Admin@12345`
   → Suivant

4. **Étape 4** : Configuration de la base de données ⚠️ **IMPORTANT**
   - **Serveur de base de données** : `192.168.56.11`
   - **Nom de la base** : `prestashop_db`
   - **Utilisateur** : `prestashop_user`
   - **Mot de passe** : `Prestashop@123`
   - **Port** : `3306`
   → Tester la connexion → Suivant

5. **Étape 5** : Installation
   - Attendez la fin de l'installation (2-5 minutes)

6. **Étape 6** : Site prêt !

---

## 📊 ÉTAPE 8 : RÉCUPÉRATION DES LOGS

### 8.1 Se reconnecter à VM1

```powershell
vagrant ssh db
cd ~/ansible
```

### 8.2 Vérifier le fichier de log

```bash
# Voir la taille du fichier
ls -lh deployment.log

# Voir les dernières lignes
tail -50 deployment.log
```

### 8.3 Copier les logs vers Windows

```bash
# Depuis VM1
exit
```

```powershell
# Depuis Windows PowerShell
vagrant scp db:ansible/deployment.log ./deployment.log
```

Le fichier `deployment.log` est maintenant dans votre dossier Windows ! ✅

### 8.4 Créer un résumé des logs

```powershell
# Depuis Windows
vagrant ssh db -c "cd ~/ansible && cat deployment.log" > logs_complet.txt
```

---

## 🧪 ÉTAPE 9 : TESTS DE VALIDATION

### 9.1 Test 1 : Services actifs

```powershell
# MySQL
vagrant ssh db -c "sudo systemctl is-active mysql"

# Apache
vagrant ssh web -c "sudo systemctl is-active apache2"
```

Résultat attendu : `active` pour les deux ✅

### 9.2 Test 2 : Ports ouverts

```powershell
# MySQL port 3306
vagrant ssh db -c "sudo netstat -tlnp | grep 3306"

# Apache port 80
vagrant ssh web -c "sudo netstat -tlnp | grep :80"
```

### 9.3 Test 3 : Site web accessible

```powershell
# Tester depuis Windows
Invoke-WebRequest -Uri "http://localhost:8080" -UseBasicParsing
```

Si `StatusCode : 200` → ✅ Site accessible !

---

## 📦 ÉTAPE 10 : PRÉPARATION DU LIVRABLE

### 10.1 Créer l'archive finale

```powershell
# Fichiers à inclure
$files = @(
    "Vagrantfile",
    "ansible.cfg",
    "inventory.ini",
    "deploy.yml",
    "README.md",
    "DEPLOIEMENT.md",
    "deployment.log",
    "roles"
)

Compress-Archive -Path $files -DestinationPath "exam_ansible_final.zip" -Force
```

### 10.2 Contenu du livrable

```
exam_ansible_final.zip
├── Vagrantfile           ✅ Configuration VMs
├── ansible.cfg           ✅ Config Ansible
├── inventory.ini         ✅ Inventaire
├── deploy.yml            ✅ Playbook
├── README.md             ✅ Documentation
├── DEPLOIEMENT.md        ✅ Ce guide
├── deployment.log        ✅ Logs réels
└── roles/                ✅ 2 rôles (MySQL + Prestashop)
```

---

## 🛠️ COMMANDES UTILES

### Gestion des VMs

```powershell
# Voir l'état
vagrant status

# Arrêter les VMs
vagrant halt

# Redémarrer
vagrant reload

# Détruire les VMs
vagrant destroy

# SSH vers VM1 (db)
vagrant ssh db

# SSH vers VM2 (web)
vagrant ssh web
```

### Relancer le déploiement

```bash
# Depuis VM1
cd ~/ansible
ansible-playbook deploy.yml -vvv 2>&1 | tee deployment.log
```

### Voir les logs en temps réel

```bash
# Logs MySQL
vagrant ssh db -c "sudo tail -f /var/log/mysql/error.log"

# Logs Apache
vagrant ssh web -c "sudo tail -f /var/log/apache2/error.log"
```

---

## 🐛 DÉPANNAGE

### Problème : VMs ne démarrent pas

```powershell
# Vérifier VirtualBox
vboxmanage list vms

# Redémarrer VirtualBox
# Puis relancer
vagrant up
```

### Problème : Ansible ne trouve pas les hôtes

```bash
# Vérifier l'inventaire
ansible-inventory -i inventory.ini --list

# Tester ping
ansible all -m ping
```

### Problème : MySQL n'accepte pas les connexions

```bash
# Vérifier la config MySQL
vagrant ssh db -c "sudo cat /etc/mysql/mysql.conf.d/mysqld.cnf | grep bind-address"

# Devrait être : bind-address = 0.0.0.0
```

### Problème : Prestashop ne se connecte pas à MySQL

```bash
# Tester depuis VM2
vagrant ssh web -c "mysql -h 192.168.56.11 -u prestashop_user -pPrestashop@123 -e 'SELECT 1;'"
```

---

## ✅ CHECKLIST FINALE

Avant de soumettre votre projet :

- [ ] Les 2 VMs démarrent correctement
- [ ] MySQL fonctionne sur VM1
- [ ] Apache fonctionne sur VM2
- [ ] Communication DB ↔ Web opérationnelle
- [ ] Site accessible sur http://localhost:8080
- [ ] Logs `deployment.log` générés
- [ ] Archive ZIP créée avec tous les fichiers
- [ ] Documentation complète

---

## 🎓 RÉSUMÉ DE L'ARCHITECTURE

**Ce que vous avez déployé** :

1. **2 VMs distinctes** (respect de l'énoncé)
2. **VM1** : MySQL + Ansible Controller
3. **VM2** : Prestashop (serveur web)
4. **Communication réseau** : VM2 → VM1 (port 3306)
5. **Automatisation complète** avec Ansible
6. **Logs de déploiement** réels et documentés

**Félicitations ! Votre infrastructure e-commerce est opérationnelle !** 🎉


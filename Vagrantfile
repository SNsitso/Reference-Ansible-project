# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile pour déploiement Prestashop + MySQL
# Architecture: 2 VMs avec réseau privé
# VM1: MySQL + Ansible Controller
# VM2: Prestashop

Vagrant.configure("2") do |config|
  
  # Configuration de base commune
  config.vm.box = "ubuntu/focal64"  # Ubuntu 20.04 LTS
  config.vm.box_check_update = false
  
  # Configuration réseau privé
  # Réseau: 192.168.56.0/24
  
  #=============================================================================
  # VM1: MYSQL + ANSIBLE CONTROLLER
  #=============================================================================
  config.vm.define "db" do |db|
    # Configuration réseau
    db.vm.hostname = "mysql-server"
    db.vm.network "private_network", ip: "192.168.56.11"
    
    # SYNCHRONISATION AUTOMATIQUE DU PROJET
    # Le dossier Windows est synchronisé vers /home/vagrant/ansible dans la VM
    db.vm.synced_folder ".", "/home/vagrant/ansible", 
      owner: "vagrant", 
      group: "vagrant",
      mount_options: ["dmode=775,fmode=664"]
    
    # Configuration VirtualBox
    db.vm.provider "virtualbox" do |vb|
      vb.name = "exam_mysql"
      vb.memory = "2048"  # 2 GB RAM
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
    end
    
    # Provisioning: Installation des dépendances de base
    db.vm.provision "shell", inline: <<-SHELL
      echo "========================================="
      echo "Configuration de la VM MySQL"
      echo "========================================="
      
      # Mise à jour du système
      apt-get update -qq
      
      # Installation d'Ansible sur cette VM (contrôleur)
      echo "Installation d'Ansible..."
      apt-get install -y software-properties-common
      add-apt-repository --yes --update ppa:ansible/ansible
      apt-get install -y ansible
      
      # Installation de Python et dépendances
      apt-get install -y python3-pip python3-dev
      pip3 install pymysql
      
      # Configuration SSH pour Ansible
      echo "Configuration SSH..."
      sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
      systemctl restart sshd
      
      # Créer le répertoire pour les clés SSH
      mkdir -p /home/vagrant/.ssh
      chmod 700 /home/vagrant/.ssh
      chown vagrant:vagrant /home/vagrant/.ssh
      
      # Générer une clé SSH pour Ansible (sans passphrase)
      if [ ! -f /home/vagrant/.ssh/id_rsa ]; then
        sudo -u vagrant ssh-keygen -t rsa -b 2048 -f /home/vagrant/.ssh/id_rsa -N ""
      fi
      
      # Afficher les informations
      echo "========================================="
      echo "VM MySQL configurée avec succès!"
      echo "IP: 192.168.56.11"
      echo "Ansible installé: $(ansible --version | head -1)"
      echo "Projet synchronisé: /home/vagrant/ansible"
      echo "========================================="
    SHELL
    
    # Configuration finale (après création de la VM)
    db.vm.provision "shell", privileged: false, inline: <<-SHELL
      echo "========================================="
      echo "Configuration finale de l'environnement"
      echo "========================================="
      
      # Ajouter VM2 aux known hosts
      ssh-keyscan -H 192.168.56.12 >> ~/.ssh/known_hosts 2>/dev/null
      
      echo "========================================="
      echo "Environnement prêt pour Ansible!"
      echo "========================================="
      echo ""
      echo "Vos fichiers sont dans: /home/vagrant/ansible"
      echo ""
      echo "PROCHAINES ÉTAPES:"
      echo "1. vagrant ssh db"
      echo "2. cd ansible"
      echo "3. ssh-copy-id vagrant@192.168.56.12"
      echo "   (mot de passe: vagrant)"
      echo "4. Créer et chiffrer le vault (voir inventories/production/group_vars/all/vault.yml.example)"
      echo "5. ansible-playbook site.yml --ask-vault-pass -vv 2>&1 | tee deployment.log"
      echo ""
      echo "========================================="
    SHELL
  end
  
  #=============================================================================
  # VM2: PRESTASHOP (SERVEUR WEB)
  #=============================================================================
  config.vm.define "web" do |web|
    # Configuration réseau
    web.vm.hostname = "prestashop-server"
    web.vm.network "private_network", ip: "192.168.56.12"
    
    # Port forwarding pour accès depuis Windows
    web.vm.network "forwarded_port", guest: 80, host: 8081, auto_correct: true
    
    # Configuration VirtualBox
    web.vm.provider "virtualbox" do |vb|
      vb.name = "exam_prestashop"
      vb.memory = "2048"  # 2 GB RAM
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
    end
    
    # Provisioning: Configuration de base
    web.vm.provision "shell", inline: <<-SHELL
      echo "========================================="
      echo "Configuration de la VM Prestashop"
      echo "========================================="
      
      # Mise à jour du système
      apt-get update -qq
      
      # Installation de Python pour Ansible
      apt-get install -y python3 python3-apt
      
      # Configuration SSH
      echo "Configuration SSH..."
      sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
      systemctl restart sshd
      
      # Créer le répertoire .ssh pour vagrant
      mkdir -p /home/vagrant/.ssh
      chmod 700 /home/vagrant/.ssh
      chown vagrant:vagrant /home/vagrant/.ssh
      
      # Afficher les informations
      echo "========================================="
      echo "VM Prestashop configurée avec succès!"
      echo "IP: 192.168.56.12"
      echo "Accès web: http://localhost:8081 (ou http://192.168.56.12)"
      echo "========================================="
    SHELL
  end
end

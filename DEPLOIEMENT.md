# STEP-BY-STEP DEPLOYMENT GUIDE

## Overview

This guide walks you step-by-step through deploying a Prestashop e-commerce site with MySQL using Ansible and Vagrant.

**Final Architecture**:
```
VM1 (db)                    VM2 (web)
192.168.56.11               192.168.56.12

Network SSH Connection

MySQL 3306                  Apache 80
Ansible Controller          Prestashop
```

---

## Prerequisites

Before starting, ensure you have:

- [x] **Vagrant** installed (version 2.2+)
- [x] **VirtualBox** installed (version 6.0+)
- [x] **8 GB RAM** minimum available
- [x] **10 GB** free disk space
- [x] Active internet connection

### Verify Prerequisites

```powershell
# Check Vagrant
vagrant --version

# Check VirtualBox
vboxmanage --version
```

---

## STEP 1: PROJECT PREPARATION

### 1.1 Open PowerShell

```powershell
cd "C:\Users\Serge Nyuiadzi\Downloads\exam_ansible"
```

### 1.2 Verify project files

```powershell
Get-ChildItem
```

You should see:
```
Vagrantfile
ansible.cfg
deploy.yml
inventory.ini
README.md
roles/
```

---

## STEP 2: CREATE VMs WITH VAGRANT

### 2.1 Start the VMs

```powershell
vagrant up
```

**What happens**:
- Ubuntu 20.04 download (first time only)
- Creation of VM1 (db) - MySQL + Ansible
- Creation of VM2 (web) - Prestashop
- Private network configuration
- Ansible installation on VM1
- Project synchronization in VM1

**Estimated duration**: 5-10 minutes (first time)

### 2.2 Verify VM status

```powershell
vagrant status
```

Expected result:
```
db      running (virtualbox)
web     running (virtualbox)
```

---

## STEP 3: SSH CONFIGURATION

### 3.1 Connect to VM1 (Ansible controller)

```powershell
vagrant ssh db
```

You are now in VM1!

### 3.2 Configure SSH access to VM2

```bash
# Copy SSH key to VM2
ssh-copy-id vagrant@192.168.56.12
```

When prompted:
- **Password**: `vagrant`
- Type `yes` to accept the key

### 3.3 Test SSH connection

```bash
ssh vagrant@192.168.56.12 "echo 'Connection successful!'"
```

If you see "Connection successful!" -> Success!

---

## STEP 4: ANSIBLE DEPLOYMENT PREPARATION

### 4.1 Go to project folder

```bash
cd ~/ansible
ls -la
```

You should see all project files synchronized from Windows.

### 4.2 Verify Ansible configuration

```bash
# Check playbook syntax
ansible-playbook deploy.yml --syntax-check
```

Expected result: `playbook: deploy.yml`

### 4.3 Test inventory

```bash
# List all hosts
ansible all -i inventory.ini --list-hosts
```

Expected result:
```
hosts (2):
  db
  web
```

### 4.4 Test connectivity

```bash
# Ping both machines
ansible all -i inventory.ini -m ping
```

Expected result:
```
db | SUCCESS => {"ping": "pong"}
web | SUCCESS => {"ping": "pong"}
```

If all is green, you are ready!

---

## STEP 5: DEPLOYMENT WITH ANSIBLE

### 5.1 Launch deployment with logs

```bash
# Full deployment with verbose logs
ansible-playbook deploy.yml -vvv 2>&1 | tee deployment.log
```

**What will happen**:

1. **MySQL Phase (VM1)**:
   - MySQL Server installation
   - Secure configuration
   - Creation of `prestashop_db` database
   - Creation of `prestashop_user` user
   - Network configuration (accepts connections from VM2)

2. **Prestashop Phase (VM2)**:
   - Apache + PHP 8.1 installation
   - Prestashop 8.1.2 download
   - VirtualHost configuration
   - MySQL connection test
   - Final configuration

**Estimated duration**: 10-15 minutes

### 5.2 Follow progress

You will see:
- `TASK [mysql : ...]` - MySQL installation
- `TASK [prestashop : ...]` - Prestashop installation
- Success messages in green
- Final summary

### 5.3 Verify final summary

At the end, you should see:
```
PLAY RECAP ***********************
db        : ok=XX   changed=YY
web       : ok=XX   changed=YY
```

If "failed=0" everywhere -> Deployment successful!

---

## STEP 6: DEPLOYMENT VERIFICATION

### 6.1 Verify MySQL on VM1

```bash
# You are still in VM1
sudo systemctl status mysql
```

Expected result: `Active: active (running)`

```bash
# Test database connection
mysql -u prestashop_user -pPrestashop@123 -e "SHOW DATABASES;"
```

You should see `prestashop_db`

### 6.2 Verify Apache on VM2

```bash
# From VM1, test Apache on VM2
ssh vagrant@192.168.56.12 "sudo systemctl status apache2"
```

Expected result: `Active: active (running)`

### 6.3 Test MySQL connection from VM2

```bash
# Verify VM2 can connect to MySQL on VM1
ssh vagrant@192.168.56.12 "mysql -h 192.168.56.11 -u prestashop_user -pPrestashop@123 -e 'SHOW DATABASES;'"
```

If you see `prestashop_db` -> Inter-server communication OK!

---

## STEP 7: WEBSITE ACCESS

### 7.1 From VM1, exit to Windows

```bash
exit
```

You are back in Windows PowerShell.

### 7.2 Open browser

Go to: **http://localhost:8080**

You should see the **Prestashop installation wizard**!

### 7.3 Follow Prestashop installation

1. **Step 1**: Language selection -> French -> Next

2. **Step 2**: Accept licenses -> Next

3. **Step 3**: Store information
   - Name: `My Test Shop`
   - Country: France
   - Admin email: `admin@test.com`
   - Admin password: `Admin@12345`
   -> Next

4. **Step 4**: Database configuration - IMPORTANT
   - **Database server**: `192.168.56.11`
   - **Database name**: `prestashop_db`
   - **User**: `prestashop_user`
   - **Password**: `Prestashop@123`
   - **Port**: `3306`
   -> Test connection -> Next

5. **Step 5**: Installation
   - Wait for installation to complete (2-5 minutes)

6. **Step 6**: Site ready!

---

## STEP 8: LOG RETRIEVAL

### 8.1 Reconnect to VM1

```powershell
vagrant ssh db
cd ~/ansible
```

### 8.2 Verify log file

```bash
# Check file size
ls -lh deployment.log

# View last lines
tail -50 deployment.log
```

### 8.3 Copy logs to Windows

```bash
# From VM1
exit
```

```powershell
# From Windows PowerShell
vagrant scp db:ansible/deployment.log ./deployment.log
```

The `deployment.log` file is now in your Windows folder!

### 8.4 Create log summary

```powershell
# From Windows
vagrant ssh db -c "cd ~/ansible && cat deployment.log" > logs_complete.txt
```

---

## STEP 9: VALIDATION TESTS

### 9.1 Test 1: Active services

```powershell
# MySQL
vagrant ssh db -c "sudo systemctl is-active mysql"

# Apache
vagrant ssh web -c "sudo systemctl is-active apache2"
```

Expected result: `active` for both

### 9.2 Test 2: Open ports

```powershell
# MySQL port 3306
vagrant ssh db -c "sudo netstat -tlnp | grep 3306"

# Apache port 80
vagrant ssh web -c "sudo netstat -tlnp | grep :80"
```

Expected result: Both ports listed

### 9.3 Test 3: Database access

```powershell
vagrant ssh db -c "mysql -u prestashop_user -pPrestashop@123 prestashop_db -e 'SELECT 1;'"
```

Expected result: `1`

---

## TROUBLESHOOTING

### VM won't start
```powershell
vagrant destroy
vagrant up
```

### SSH connection refused
```powershell
# Reset SSH keys
vagrant ssh-config
```

### MySQL won't connect
```bash
# Check MySQL status
sudo systemctl status mysql
sudo journalctl -u mysql -n 20
```

### Apache won't start
```bash
# Check Apache syntax
sudo apache2ctl -t
sudo systemctl status apache2 -l
```

### Logs show errors
- Check the `deployment.log` file
- Look for FAILED tasks
- Fix the issue
- Re-run: `ansible-playbook deploy.yml`

---

## Rollback (if needed)

```powershell
# Destroy VMs
vagrant destroy -f

# Start fresh
vagrant up
```

---

## Clean Up

After testing:

```powershell
# Stop VMs without destroying
vagrant suspend

# Or completely remove
vagrant destroy

# Clean Vagrant cache
vagrant prune
```

---

## Summary

After completing all steps:
- Two VMs running (MySQL + Prestashop)
- Ansible configuration validated
- Prestashop accessible via browser
- Database properly connected
- All logs saved

Deployment successful!

---
title: Rapport de projet - Ansible AWX
---
<h1> <center>Proof Of Concept Ansible AWX</center> </h1>

> [name=Solen BELLOUATI] [time=03 mai 2020]

- [Présentation Ansible AWX](#pres)
- [Installation et déploiement de l’environnement de test](#inst)

> ### <center>Présentation Ansible AWX </center> <a id="pres"></a>

![](https://i.imgur.com/RmaTRNN.png =700x190)


**AWX** est la version **Open-Source** de l'outil d'orchestration **Ansible Tower** développé par **RedHat**.

Cet outils permet au premier abord de disposer d'un affichage Web clair et dynamique des actions se déroulant dans votre environnement Ansible. 
Son spectre de fonctionnalitées est bien plus vaste, il permet notamment la gestion de rôles et d'utilisateurs, la gestion et la journalisation des tâches en temps réel ainsi qu'en parallèle. 

**AWX** est un outil regorgeant de fonctionnalités innovantes, et nous allons les décortiquer afin de vérifier sa viabilité et son concept.

---

> ### <center>Installation et déploiement de l'environnement de test </center> <a id="inst"></a>

L'environnement de test est déployé sur une machine virtuelle, avec un système d'éxploitation **Debian 10**.

Cet environnement de test est composé de 10 containers Docker, répartis également entre Centos et Debian.
J'ai donc installé **Docker**, ainsi que **Docker-Compose**, grâce à un ancien projet (***https://github.com/solenb/wp-compose***), qui permettait d'installer et d'instancier un container WordPress. 

Ces containers ont été instanciés grâce à un script présent dans un TP de M.Pouchoulon (***TP3 M3206 Provisionning***), disponible dans le dêpot ***https://registry.iutbeziers.fr:5443/pouchou/tp3automatisation.git***.

#### **Contenu du dêpot**
```
root@debian:/home/vagrant/tp3automatisation# ls
ansible.cfg  apache2-lamp-debian.yml  create-cont.sh  hosts  httpd-lamp-centos.yml  info.php  install_ansible.sh
```

 + ***Première étape***
     - Installer Ansible `./install_ansible.sh`
     - Créer un paire de clés SSH `ssh-keygen -t ed25519`
     - La clef publique ainsi générée sera copiée dans les containers créés par le script create-cont.sh.Lancez le script ***create-cont*** via `./create-cont.sh`. Les actions suivantes vont être effectuées :
         - Télécharger les images Docker centos-ssh :7 et debian-ssh :8 depuis le registry de l’IUT de Béziers.
         - Créer 5 containers Docker Centos 7 et cinq containers Debian 8 qui vous serviront de cibles pourAnsible ( vous pouvez les listez via la commande "docker ps"). Centos 7 est un container qui estinstallé avec systemd.
         - Renseigner /etc/hosts avec les IP des containers pour pouvoir les joindre avec leurs noms via SSH.
         - Supprimerles containers Existants,supprimer les adresses existantes de votre /etc/hosts, supprimezle fichier know_hosts de SSH.
         - Copier votre clef publique SSH sur le container pour pouvoir travailler sans saisir de mots depasse.Vous pourrrez ainsi vous connecter en ssh sur tous les containers sans saisir de mot de passe.
         - Passez le stockage docker de AUFS vers device mapper - un bug existant pour aufs et centos
 + ***Deuxième étape***
     - Tester si les containers sont joignables grâce à un playbook Ansible basique
```
- hosts: container
  tasks:
      - ping:
```
```
root@debian:/home/vagrant/tp3automatisation# ansible-playbook ping.yml

PLAY [container] **************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************
[...]

TASK [ping] *******************************************************************************************************************************************************************************************************
[...]
PLAY RECAP ********************************************************************************************************************************************************************************************************
centos-0                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centos-1                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centos-2                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centos-3                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
centos-4                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
debian-0                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
debian-1                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
debian-2                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
debian-3                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
debian-4                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
``` 
Tous les containers sont maintenants joignables, l'environnement de test est maintenant en place. On va pouvoir s'atteler à la tâche d'installation de l'outil ***AWX***.



#### ***Premier visuel de l'interface*** (après déploiement)
> 
![](https://i.imgur.com/er5oz51.png)

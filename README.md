
# Rapport d'étude - Proof Of Concept AWX

> Solen BELLOUATI
> 13 juin 2020

***Sommaire***
[TOC]

> # Présentation Ansible AWX

![](https://i.imgur.com/RmaTRNN.png)


`AWX` est la version `Open-Source` de l'outil d'orchestration `Ansible Tower` développé par `RedHat`, il est la vitrine de ce dernier, ce qui signifie qu'il dispose des dernières nouveauté en premier.

Cet outils permet au premier abord de disposer d'un affichage Web clair et dynamique des actions se déroulant dans votre environnement Ansible. 
Son spectre de fonctionnalitées est bien plus vaste, il permet notamment la gestion de rôles et d'utilisateurs, la gestion et la journalisation des tâches en temps réel ainsi qu'en parallèle. 

`AWX` est un outil regorgeant de fonctionnalités innovantes, et nous allons les décortiquer afin de vérifier sa viabilité et son concept.

---

> ### Installation et déploiement de l'environnement de test 

L'environnement de test est déployé sur une machine virtuelle, avec un système d'éxploitation **Debian 10**.

Cet environnement de test est composé de 10 containers Docker, répartis également entre Centos et Debian.
J'ai donc installé **Docker**, ainsi que **Docker-Compose**, grâce à un ancien projet (***https://github.com/solenb/wp-compose***), qui permettait d'installer et d'instancier un container WordPress, réalisé lors des journées Hackaton. J'ai donc exécuté le script sans lancer l'instanciation du container Wordpress.

Ces containers ont été instanciés grâce à un script présent dans un TP de M.Pouchoulon (***TP3 M3206 Provisionning***), disponible dans le dêpot ***https://registry.iutbeziers.fr:5443/pouchou/tp3automatisation.git***.


 **Contenu du dêpot**

```
root@debian:/home/vagrant/tp3automatisation# ls
ansible.cfg  apache2-lamp-debian.yml  create-cont.sh  hosts  httpd-lamp-centos.yml  info.php  install_ansible.sh
```

### ***Déploiement d'un environnement de test***

Installer Ansible `./install_ansible.sh`
Créer un paire de clés SSH `ssh-keygen -t ed25519`
La clef publique ainsi générée sera copiée dans les containers créés par le script create-cont.sh.Lancez le script ***create-cont*** via `./create-cont.sh`. Les actions suivantes vont être effectuées :
- <p style="font-size:13px">Télécharger les images Docker centos-ssh :7 et debian-ssh :8 depuis le registry de l’IUT de Béziers.</p>
- <p style="font-size:13px">Créer 5 containers Docker Centos 7 et cinq containers Debian 8 qui vous serviront de cibles pourAnsible ( vous pouvez les listez via la commande "docker ps"). Centos 7 est un container qui estinstallé avec systemd.
- <p style="font-size:13px">Renseigner /etc/hosts avec les IP des containers pour pouvoir les joindre avec leurs noms via SSH.
- <p style="font-size:13px">Supprimerles containers Existants,supprimer les adresses existantes de votre /etc/hosts, supprimezle fichier know_hosts de SSH.
- <p style="font-size:13px">Copier votre clef publique SSH sur le container pour pouvoir travailler sans saisir de mots depasse.Vous pourrrez ainsi vous connecter en ssh sur tous les containers sans saisir de mot de passe.
- <p style="font-size:13px">Passez le stockage docker de AUFS vers device mapper - un bug existant pour aufs et centos

###  ***Test de l'environnement et de son accessibilité***

Tester si les containers sont joignables grâce à un playbook Ansible basique et la commande liée : 
```
- hosts: container
  tasks:
      - ping:
```
En ajoutant les groupes debian et centos au sous-groupe container, cela nous permet d'adresser toutes les machines en même temps de façon plus simple
```
[container:children]
debian
centos
```
Il suffit ensuite de lancer la commande `ansible-playbook` :
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

> #### Création de containers sur un VPS (modification du script de création des containers)
>
> Cette manipulation va permettre de rendre accessible le site AWX à tout le monde.
> Modification des commandes `docker run`, car les registres de l'iut ne sont pas accessibles. Récupération d'images Debian et Centos basiques.

```#! /bin/bash
if [ ! -f ~/.ssh/id_ed25519 ]
then 
	echo "pas de clefs ssh id_ed25519.pub"
        echo "Lancez ssh-keygen -t ed25519"
        echo "NE METTEZ PAS DE PASSPHRASE"
        exit 1
fi
cat ~/.ssh/id_ed25519 > /root/.ssh/authorized_keys
echo "effacement des containers existants"
echo "################################"
docker stop $(docker ps -a -q)
sleep 5s
docker rm $(docker ps -a -q)
sleep 10s
echo "effacement des network dockers existants"
echo "################################"
for net in  $(docker network ls -q);  do docker network rm $net; done
echo "################################"
echo "Creation des network dockers pour le TP"
echo "################################"
for x in $(seq 0 9); do docker network create --driver bridge brcent_n_$x ;done
for x in $(seq 0 9); do docker network create --driver bridge brddeb_n_$x ;done
echo "remise à zero de  /root/.ssh/known_hosts"
echo "################################"
> /root/.ssh/known_hosts
echo "supression des adresses des containers existants dans /etc/hosts"
echo "################################"
sed -i '/172.17/d'  /etc/hosts
echo "Création des containers Debian 8 et centos 6"
echo "################################"
for x in $(seq 0 4); do docker run -t -d -p 322$x:22 -v /root/.ssh:/root/.ssh --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro  --net brcent_n_$x  --name  centos-$x --hostname centos-$x  centos:latest ;done
for x in $(seq 0 4); do docker run -t -d -p 222$x:22 -v /root/.ssh:/root/.ssh  --net brddeb_n_$x  --name  debian-$x --hostname debian-$x  debian:latest ;done
echo "creation des ip des containers dans /etc/hosts"
echo "################################"
for x in $(seq 0 4); do echo "$(docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' centos-$x) centos-$x" >> /etc/hosts;done
for x in $(seq 0 4); do echo "$(docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' debian-$x) debian-$x"  >> /etc/hosts;done
echo "voila le boulot"
docker ps -
```
Au final le process est le même et les containers sont maintenant accessibles.
    
```/bin/bash
root@vps759131:~/projet2020/tp3automatisation# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                  NAMES
c9d87dbca422        debian:latest       "bash"              5 minutes ago       Up 5 minutes        0.0.0.0:2224->22/tcp   debian-4
aced38d80a33        debian:latest       "bash"              5 minutes ago       Up 5 minutes        0.0.0.0:2223->22/tcp   debian-3
6d350ff7f97f        debian:latest       "bash"              5 minutes ago       Up 5 minutes        0.0.0.0:2222->22/tcp   debian-2
ed5acd4e2101        debian:latest       "bash"              5 minutes ago       Up 5 minutes        0.0.0.0:2221->22/tcp   debian-1
525a83f69fff        debian:latest       "bash"              5 minutes ago       Up 5 minutes        0.0.0.0:2220->22/tcp   debian-0
2d4e513506ac        centos:latest       "/bin/bash"         5 minutes ago       Up 5 minutes        0.0.0.0:3224->22/tcp   centos-4
e11b63dcdb9d        centos:latest       "/bin/bash"         5 minutes ago       Up 5 minutes        0.0.0.0:3223->22/tcp   centos-3
2479878bd714        centos:latest       "/bin/bash"         5 minutes ago       Up 5 minutes        0.0.0.0:3222->22/tcp   centos-2
438c837402f0        centos:latest       "/bin/bash"         5 minutes ago       Up 5 minutes        0.0.0.0:3221->22/tcp   centos-1
1cec65b9c199        centos:latest       "/bin/bash"         5 minutes ago       Up 5 minutes        0.0.0.0:3220->22/tcp   centos-0
```
---

> ## Installation et déploiement de l'outil AWX

L'installation de l'outil Ansible AWX grâce à Docker Compose est très simple, il suffit de lancer un playbook présent dans les fichiers du dépot GitHub.

On commence par cloner le dépot `git clone https://github.com/ansible/awx.git`.

On se place ensuite dans le répertoire contenant le playbook d'installation `cd awx/installer`.

Enfin on lance le playbook d'installation `ansible-playbook -i inventory install.yml`.
    
Une fois l'installation terminée, on devrait apercevoir le message suivant grâce à la commande `docker logs -f awx_task` (il varient en fonction des versions).

```
Python 2.7.5 (default, Nov  6 2016, 00:28:07)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-11)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)

>>> <User: admin>
>>> Default organization added.
Demo Credential, Inventory, and Job Template added.
Successfully registered instance awx
(changed: True)
Creating instance group tower
Added instance awx to tower
(changed: True)
...
```
Afin de rendre la connection à l'interface plus sûre, il faut modifier dans le fichier inventory la variable `admin_password` , afin d'avoir un mot de passe fort.
Afin de se connecter à l'interface web, il faut rechercher l'adresse IP de la machine dans un navigeteur de recherche (en l'occurence Firefox avec l'adresse **192.168.1.79**).
    
#### ***Premier visuel de l'interface*** (après déploiement)
> 
![](https://i.imgur.com/er5oz51.png)
    
> #### ***Mise à jour de l'accéssiblité*** 
> Le site se trouve maintenant sur le nom de domaine awx.solenb.fr, il est accessible à tous (sous condition d'avoir un compte utilisateur)
>
> ![](https://i.imgur.com/OmF5Sj9.png)

    
---

> ## Présentation des fonctionnalitées de l'outil AWX

### Authentification
    - Lancement des playbooks
    - CRUD (Create Reload Update Delete) des jobs
    - Gestion des inventaires

Pour effectuer n'importe quelle action, il faut obligatoire s'identifier sur la platforme.
    
![](https://i.imgur.com/TLKlM0f.png)
#### Gestion des utilisateurs par l'administrateur
    
L'enregistrement des nouveaux utilisateurs se fait obligatoirement par l'administrateur, ce qui est long et fastidieux. Ces identifiants sont stockés sur une base de données, créée en même temps que le container AWX,  susceptible d'être vulnérable.

    
>Afin de créer de nouveaux utilisateurs, il faut se connecter en tant qu'administrateur. Ce qui est normal étant donné qu'il est la seule personne abilité à effectuer un complet CRUD (Create Read Update Delete) sur les utilisateurs. On se rend ensuite dans l'onglet `utilisateur`. Il ne nous reste plus qu'à cliquer sur le bouton vert +.
>
>![](https://i.imgur.com/01E9VwT.png)
>Il faut  maintenant rentrer ses identifiants, son mot de passe, son email ... On peut quand même aller plus loin que ça, et c'est là que tout l'intérêt des utilisateur est. On peut mettre en place une hiérarchie, en fonction du poste de l'utilisateur dans l'entreprise. ET en fonction de cette hiérarchie lui accorder des droits spécifiques
>
>
#### Authentification par application tierce
Un autre point positif est la possibilité d'une authentification sociale via notamment Github, AzureAD, GoogleAuth, ou encore des systèmes personnels comme LDAP, ou Radius.

>On teste la fonctionnalité en prenant l'authentification via Github. Avant tout il faut changer un paramètre afin que l'url de rappel soit connue de Github. On va ajouter l'url complet du site AWX, il va ensuite générer automatique l'url de rappel pour le token d'authentification Github.
>
>![](https://i.imgur.com/m2dJKz7.png)
>
>On se dirige ensuite sur le site Github, en ayant créé un compte sur la platforme au préalable, (https://github.com/settings/developers), dans les paramètres développeurs. On créer une nouvelle Oauth App.
>
> ![](https://i.imgur.com/slBhZ2p.png)
> 
> Les champs importants, ici sont l'url d'accueil de notre site, ainsi que l'url de rappel, que nous pouvons trouver sur la page d'authentification à l'onglet Github 
> 
>![](https://i.imgur.com/taP0Dju.png)
>
> Pour finir Github va automatiquement nous fournir une clé et un secret permettant l'authentification. La première page de connexion va aussi se voir implémenter un bouton permettant le nouveau mode de connexion.
> 
>![](https://i.imgur.com/in2iEZK.png)
>
> Ainsi n'importe qu'elle personne ayant un compte Github, pourra se créer un compte fonctionnel, mais restreint.
>
>![](https://i.imgur.com/xQsAZWN.png)
>
>On peut remarquer qu'en mettant connecté avec mon compte Github, l'utilisateur a été automatiquement créé. Et celui-ci ne dispose d'aucune permission, ce qui est modifiable.

### ***REST API***
    - Demande de token
    - Màj SCM (Supply Chain Management)
    - Collecte des flux d'activité
    - Lancement de playbooks

> Une API REST, signifiant **“Representational State Transfer”**, est une des plus communes. Les API REST imitent le mode de fonctionnement basique en informatique entre un serveur et un client. Chaque entité est "autonome" par rapport à l'autre, ce qui signifie qu'ils peuvent travailler **indépendamment**.
On dit qu'ils sont **"sans état"**, une reqête n'a d'impact sur la demande actuelle, et non sur les autres. 
Une API REST fonctionne aussi avec un système de mémoire cache, réduisant l'effort du serveur quand des requêtes similaires sont receptionnées.

Afin de visualiser l'arborescence de l'API d'AWX, nous allons effectuer une requête **curl**. Permettant de faire ressortir toutes les fonctionnalitées disponibles, avec une mise en forme jq (en ayant caché le mot de passe).
```bash
 bellouati@bellouati-HP-x360-310-G1-PC  ~  curl -u admin:$PASSWD -X GET http://awx.solenb.fr/api/v2/ | jq                                         ✔  2011  11:36:40
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1688  100  1688    0     0   4574      0 --:--:-- --:--:-- --:--:--  4586
{
  "ping": "/api/v2/ping/",
  "instances": "/api/v2/instances/",
  "instance_groups": "/api/v2/instance_groups/",
  "config": "/api/v2/config/",
  "settings": "/api/v2/settings/",
  "me": "/api/v2/me/",
  "dashboard": "/api/v2/dashboard/",
  "organizations": "/api/v2/organizations/",
  "users": "/api/v2/users/",
  "projects": "/api/v2/projects/",
  "project_updates": "/api/v2/project_updates/",
  "teams": "/api/v2/teams/",
  "credentials": "/api/v2/credentials/",
  "credential_types": "/api/v2/credential_types/",
  "credential_input_sources": "/api/v2/credential_input_sources/",
  "applications": "/api/v2/applications/",
  "tokens": "/api/v2/tokens/",
  "metrics": "/api/v2/metrics/",
  "inventory": "/api/v2/inventories/",
  "inventory_scripts": "/api/v2/inventory_scripts/",
  "inventory_sources": "/api/v2/inventory_sources/",
  "inventory_updates": "/api/v2/inventory_updates/",
  "groups": "/api/v2/groups/",
  "hosts": "/api/v2/hosts/",
  "job_templates": "/api/v2/job_templates/",
  "jobs": "/api/v2/jobs/",
  "job_events": "/api/v2/job_events/",
  "ad_hoc_commands": "/api/v2/ad_hoc_commands/",
  "system_job_templates": "/api/v2/system_job_templates/",
  "system_jobs": "/api/v2/system_jobs/",
  "schedules": "/api/v2/schedules/",
  "roles": "/api/v2/roles/",
  "notification_templates": "/api/v2/notification_templates/",
  "notifications": "/api/v2/notifications/",
  "labels": "/api/v2/labels/",
  "unified_job_templates": "/api/v2/unified_job_templates/",
  "unified_jobs": "/api/v2/unified_jobs/",
  "activity_stream": "/api/v2/activity_stream/",
  "workflow_job_templates": "/api/v2/workflow_job_templates/",
  "workflow_jobs": "/api/v2/workflow_jobs/",
  "workflow_approvals": "/api/v2/workflow_approvals/",
  "workflow_job_template_nodes": "/api/v2/workflow_job_template_nodes/",
  "workflow_job_nodes": "/api/v2/workflow_job_nodes/"
}

 
```

### ***Notifications Intégrées***
    - Slack
    - Email
    - Grafana
    - IRC
    - ...

> Les `notifications` sont un outil très important, surtout pour un administrateur. Elles permettent de tenir au courant toute personne travaillant sur un projet, en envoyant une `notification` à chaque action étant été efféctué.
Sur l'interface d'AWX, le système de `notification` est intégré nativement. De nombreux choix sont à votre disposition, comme l'intégration de `notification` par mail ou encore par Slack. C'est ce que nous allons tester ici.
    
![](https://i.imgur.com/Fs7IYU3.png)
    
Voilà comment cette interface se présente, pour le cas de Slack, il suffit de renseigner le canal de destination pour les notifications, et le token d'api Slack. Celui-ci est à obtenir en allant créer un application Slack sur leur interface web.
    
![](https://i.imgur.com/zmJY0PL.png)


Il faut donc créer une application Slack, qui va jouer le bot dans votre espce de travail. Vous devez lui assigner des droits, un espace de travail, un identifiant, et en plus (non obligatoire) un logo.
    
![](https://i.imgur.com/0mj4YxQ.png)

Finalement, on obtient ce précieux Token d'api Slack
    
![](https://i.imgur.com/6WlOOTL.png)

Pour finir, depuis l'interface d'AWX, on peut envoyer une notification test.
    
![](https://i.imgur.com/ERXeJRt.png)
    

Et voila le résultat sur Slack.

![](https://i.imgur.com/HJGwhRs.png)

### ***Organisation***
> L'organisation sur l'outil AWX, permet de mettre en place une vraie hiérarchie, avec un système de droit d'accés, donc une différenciation des rôle de chacun.
On peut créer tout type d'organisation, afin de refléter au mieux les utilisateurs de la platfome (workers).
    
Dans notre cas, nous avons créé une "Organization-test", permettant de mettre en avant les fonctionnalitées disponibles.
    
En premier lieu, on va se diriger dans la section utilisateur, où on peut en créer de nouveaux. Avec de nombreux attributs, comme son type (administrateur, auditeur, normal), son appartenance à une ou plusieurs équipes, ses droits, son mail. On peut voir en-dessous, l'utilisateur solenb, qui avait été crée grâce à l'intégration de l'authentification par Github.
    
![](https://i.imgur.com/zZTLyIT.png)

La deuxième manipulation à effectuer, avant la création de `l'organisation`, est la création d'une équipe. On peut ici sélectionner un groupe d'utilisateur, ainsi que leur permissions. 
    
![](https://i.imgur.com/qJGhaRa.png)
    
![](https://i.imgur.com/RBHzj30.png)



Dans la section `organisation`, tout ce qui peut la définir, les utilisateurs, les équipes, les inventaires et les projets .

![](https://i.imgur.com/0z7JCbX.png)

Pour finir, la création d'une `organisation` permet de poser un cadre aux actions d'un groupe, qui n'aura accès uniquement qu'aux choses nécessaires.

### ***Tâches de gestion***

> Les `Tâches de gestion`, sont des actions pré-programmées, qui interviennent pour nettoyer l'interface AWX, et rendre l'expérience utilisateur plus agréable. Il y en 4 chacune a un rôle précis, elles nettoient les choses pouvant vite devenir encombrantes, comme les jobs affichés ou l'activité du site. 
    
![](https://i.imgur.com/ZxwIAIJ.png)

Toutes les `Tâches de Gestions` sont modifiables, on peut selectionner les jours, les heures, la fréquence de répétition.

![](https://i.imgur.com/cRrg00r.png)

Ces `Tâches de Gestion` mettent en avant un système de crontab intégré à AWX, ce qui permet de lancer toute sorte de playbook à une date précise ou à intervalles réguliers. On retrouve cette fonction dans la section `Calendriers`.

![](https://i.imgur.com/inMONGr.png)

    
## **Cas pratique - Serveur Nextcloud**

A travers ce cas pratique, nous allons mettre en place un serveur `NextCloud`, et l'administrer via AWX.
Cela nous permettrait de tester l'interface AWX, et ses fonctionnalitées.
Le dêpot Git, contenant le playbook, se trouve à l'adresse suivant : 
 >   ***https://github.com/solenb/nextcloud-test***
 >   
    
>![](https://i.imgur.com/Eo1QIXx.png =100x100)
>
>***nextcloud.solenb.fr***
>
>Nextcloud  est à la base une platforme hébergé par l'utilisateur, afin de fairedu stockage de fichier, un peu comme Google Drive. Mais il reste un service opensource, et gratuit. 
Nextcloud a énormément évolué, grâce à toujours plus d'extensions, il est maintenant devenu une platforme de travail collaboratif, avec l'intégration d'éditeur de texte partagé, ou encore un service de visioconférence.

 >  Voici le schéma relatant à la chaine toutes les étapes devant être réalisées afin de déployer ce cas pratique.
 >   
 > ![](https://i.imgur.com/JwEYLTy.png)

Premièrement , on va se diriger dans la section `Projets` d'AWX. Cette partie nous permet de renseigner sur quel dêpot notre projet est. En l'occurence, un dêpot GitHub comportant le script d'installation du serveur NextCloud.
    
![](https://i.imgur.com/F13MnJe.png)

Il faut renseigner l'URL du dêpot GitHub, ainsi que la branche sur laquelle le travail se trouve. Et enregistrer.

On peut alors remarquer que notre `projet` est automatiquement ajouté aux projets enregistrés, et un petit voyant va nous indiquer si la récupération du projet est effective. 

![](https://i.imgur.com/mq4DwmN.png)

On se dirige ensuite dans la section `Informations d'identifications`. Où on va pouvoir rentrer les moyen de connexion à la machine qui va recevoir le playbook (machine NextCloud). On rempli donc l'user, le mot de passe et la clé ssh préalablement créée. 
    
![](https://i.imgur.com/8eg3WXD.png)

On peut alors remarquer que nos `informations d'identification` sont automatiquement ajoutées aux informations enregistrées, et un petit voyant va nous indiquer si la récupération du projet est effective. 
    
![](https://i.imgur.com/30W8gGV.png)

On se dirige maintenant sur l'onglet modèle afin de créer le playbook, nous permettant d'installer et de configurer Nextcloud.
Sur cette page il faut selectionner l'inventaire, ainsi que le projet précedemment créés. AWX va automatiquement aller chercher sur le répertoire distant les fichier de playbook en YAML (fichier de configuration d'Ansible), et les afficher.
On selectionne ici la verbosité désirée, pour cette fois nous allons selectionner 3, afin de debug le système.
    
![](https://i.imgur.com/98HWVZR.png)

Après de nombreux testes, modifications et ajustements des playbooks d'installation, le modèle fonctionne enfin correctement. L'interface graphique suivante, nous permet de suivre l'évolution du déroulement du playbook.
    
![](https://i.imgur.com/kFA69tz.png)

    
![](https://i.imgur.com/sIEWEGj.png)

Notre cas pratique est arrivé à sa conclusion, en définitive le système AWX est vraiment puissant, il permet d'ajouter beaucoup de fonctionnalitées au projet déjà complet Ansible.
Ce cas pratique est totalement fonctionnel, il permet d'organiser du travail collaboratif ou du stockage de documents, ou de media. Il restera accessible à toute personne voulant y accéder.

**Note**: Le logiciel NextCloud n'est pas accessible via le naviguateur Firefox, si vous voulez tester son accessibilité veuillez le faire avec Chrome, ou d'autres naviguateurs.
Un compte visiteur a été créé, il comporte ce sujet écrit et 1Gb de stockage (il est temporaire). Et voici les logs [visiteur:AWXprojet2020].

    
## ***Conclusion***
       
`AWX` est un projet très interressant, demandant énormément d'attention. Sa prise en main a quand même été longues, car toutes les outils de base d'Ansible sont ici re-modelés et re-travaillés.
    
Une fois les grandes lignes appréhendées, avec les concept de modèles, de projets et d'inventaires. 
On remarque quand même que l'interface a été soignée, en essayant d'apporter de la lisibilité, avec des graphiques, des options toujours avec des icones remarquables. On dénote quand même quelques problème d'affichage, avec des artefacts de menus.

On peut mettre en avant sa faible consommation en ressources, en effet le système ne nécessite que d'uniquement 4Gb de RAM, de 2 CPU et de 20Gb de stockage. Ce qui représente une petite structure et un très petit investissement, par rapport aux possibilitées accordées. Le système déployé au cours de ce rapport, fonctionnait sur un VPS de OVH à seulement 8€ par mois, ce qui représente une toute petite somme pour n'importe quelle entreprise.

AWX a été très facile à déployer, premièrement sur une machine virtuelle en local, afin de survoler le déploiement et les fonctionnalitées, puis sur un VPS à distance.
Grâce au playbook Ansible permettant l'installation automatisée tout est simplifié, il suffit au préalable d'installer Docker, Docker Compose et Ansible.    
Malheureusement ce n'est pas la même conclusion pour la migration du système, ou sa mise à jour. Effectivement AWX étant une version gratuite, il n'embarque pas toutes les fonctionnalitées de sa version payante Ansible Tower. Nottament dans les domaines cités, en effet pour migrer le système, il faut stopper tous les processus et exporter soit même les bases de données. Et pour la mise à jour il faut récupérer les bases de donner et changer tout le système, en récupérant la bonne version sur GitHub.

Les fonctionnalitées sont multiples, mais les plus importantes sont le système organisationnel, permettant une isolation des rôles des utilisateurs, ou encore le système de sauvegarde des traces système. Permetant de sauvegarder toute action et modification, ainsi que de notifier.
Mais la fonctionnalité majeure reste la centralisation des playbook, permettant une production, ainsi qu'un déploiement plus fluide et intuitif.

En conclusion `AWX` est un projet très interressant, comportant énormément de fonctionnalitées pouvant grandement faciliter la production et le déploiement de code, sous toutes ses formes. Même si on s'y perd parfois, à cause de la masse de focntion, et d'applications tierces ou de documentation.

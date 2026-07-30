
L'objectif est de pouvoir déployer sereinement, y compris un vendredi ou un week-end, sans craindre de casser la production.

- **Plus il y a de changements accumulés entre deux déploiements, plus il devient difficile d'identifier la source d'un problème** si quelque chose se passe mal.
- **Un time-to-market trop long ralentit l'entreprise** : plus le délai entre le développement d'une fonctionnalité et sa mise en production est long, plus l'organisation perd en réactivité.
- **Gmail**, par exemple, déploie des centaines de fois par jour. Cette pratique, affinée avec le temps, permet de développer une réelle confiance dans la fiabilité de l'infrastructure.
- Ansible permet précisément de gérer ces déploiements de manière fiable et reproductible.
### Erreurs à éviter lors d'un déploiement :

![[Pasted image 20260730120222.png|169]]
- Éviter tout déploiement manuel : une intervention manuelle ouvre systématiquement la porte à des erreurs humaines.

- Éviter les longs intervalles entre deux déploiements. Lorsqu'on souhaite mettre à jour uniquement le load balancer ou uniquement une partie du code, il faut pouvoir ne déployer que ce changement précis, sans embarquer d'autres modifications.

- Éviter de déployer le week-end ou la nuit. C'est en réalité le pire moment possible : c'est un jour non travaillé, et la nuit ouvre la porte à des problèmes qui ne seront détectés que tardivement.

- À l'inverse, déployer en semaine permet à toute l'équipe d'être présente pour vérifier que tout fonctionne correctement, et de profiter d'une journée complète où chacun est disponible et attentif.

### Règles à suivre


![[Pasted image 20260730120249.png|110]]


- Aucune opération manuelle, sous aucun prétexte. Lorsque tout est automatisé, un correctif se règle une bonne fois pour toutes dans le playbook, puis se redéploie partout de façon identique.

- Toute correction doit être apportée directement dans les playbooks, jamais sur les serveurs eux-mêmes.

- Multiplier les petits changements versionnés dans Git garantit une traçabilité fine de chaque modification.

- Augmenter progressivement la fréquence des déploiements : par exemple passer d'un déploiement mensuel, à hebdomadaire, puis quotidien, afin de fluidifier le processus.

- Privilégier des déploiements aussi petits que possible : réalisés rapidement, avec un minimum de changements cumulés, ils sont plus faciles à valider et à corriger en cas de problème.

- Découper un déploiement en plusieurs étapes si nécessaire, par exemple mettre à jour la base de données avant l'application, en adaptant le calendrier au besoin (un changement le matin, un autre l'après-midi).

- Déployer en journée, lorsque tout le monde est présent, plutôt qu'à 4h du matin ou le week-end.

- Utiliser un déploiement de type **blue/green** pour éviter tout downtime technique : une nouvelle infrastructure est déployée avec la nouvelle version de l'application, puis le trafic est redirigé progressivement vers cette nouvelle infrastructure.
### 1. Mise en place du déploiement de l'application

L'objectif est de déployer une application "todo list" sans aucun impact pour les utilisateurs.

Création du playbook `deploy.yml` :

```yaml
---
- hosts: web
  roles:
    - deploy
```

Création du rôle correspondant avec `ansible-galaxy` :

```bash
ansible-galaxy init deploy
- Role deploy was created successfully
```

L'arborescence du dossier de travail se présente alors ainsi :

```bash
ubuntu@ansible-main:~/ansible-playbooks$ ll
total 32
drwxrwxr-x 3 ubuntu ubuntu 4096 Jul 23 23:18 ./
drwxr-x--- 7 ubuntu ubuntu 4096 Jul 23 18:01 ../
-rw-rw-r-- 1 ubuntu ubuntu   36 Jul 23 23:18 deploy.yml
-rwxr-xr-x 1 ubuntu ubuntu  192 Jul 23 15:06 inventory*
-rw-rw-r-- 1 ubuntu ubuntu   59 Jul 23 18:01 lamp.yaml
-rw-rw-r-- 1 ubuntu ubuntu   36 Jul 23 17:34 mysql.yml
drwxrwxr-x 5 ubuntu ubuntu 4096 Jul 23 23:18 roles/
-rw-rw-r-- 1 ubuntu ubuntu   29 Jul 23 17:32 web.yml
```

### 2. Récupération de l'application via le module `git`

L'application est récupérée depuis son dépôt GitLab (`https://gitlab.com/ttwthomas/app-example-php`) à l'aide du module `git`. La tâche est écrite dans le `main.yml` du rôle `deploy`, avec un mécanisme de contrôle permettant de cloner le dépôt et d'importer la base de données de façon idempotente, sans rejouer l'import inutilement.

```yaml
---
- name: installer dependences
  apt:
    name: [git, curl, python3-pymysql, mysql-client]
    state: present

- name: cloner le repo git
  git:
    repo: 'https://gitlab.com/ttwthomas/app-example-php.git'
    dest: /var/www/html/app
    force: yes
  # variable qui capture le résultat du clone git
  register: git_repo

- name: restaurer la base de donnees
  mysql_db:
    name: cocadmin
    state: import
    target: /var/www/html/app/creer_table.sql
    # seul hostname disponible dans ce contexte
    login_host: web
    login_user: cocadmin
    login_password: 1234
    # empêche l'import à chaque exécution du playbook
    run_once: true
  # ne réimporte que si le module git a détecté un changement
  when: git_repo.changed
```

### 3. Bug rencontré : résultat HTML tronqué

Le déploiement fonctionne, mais une requête HTTP de test retourne un résultat HTML tronqué sur les deux serveurs :

```bash
ubuntu@web-server-1:~$ curl localhost/app/index.php
<!--
   Copyright 2017 Vinzenz Feenstra, Red Hat, Inc.
   ...
-->
<?php
define('DB_USER', 'cocadmin');
[...]
```

```bash
ubuntu@web-server-2:~$ curl localhost/app/index.php
<!--
   Copyright 2017 Vinzenz Feenstra, Red Hat, Inc.
   ...
-->
<?php
define('DB_USER', 'cocadmin');
[...]
```

> Note : ce comportement (le code PHP n'est pas interprété et s'affiche tel quel) sera résolu plus loin, en corrigeant l'activation du module PHP dans Apache (voir section 6).

### 4. Mise en place d'un load balancer HAProxy

HAProxy est utilisé pour répartir efficacement la charge entre les deux serveurs web.

Le principe : le client envoie une requête au load balancer, qui répartit la charge entre les deux serveurs. Pendant un déploiement, tout le trafic est redirigé vers un seul serveur pour garantir la disponibilité du service pendant que l'autre est mis à jour. 

Le serveur en cours de déploiement est retiré du load balancer ; toutes les nouvelles requêtes sont alors envoyées vers l'autre serveur. Une fois la mise à jour vérifiée, ce serveur est remis dans le load balancer et l'autre en est retiré à son tour pour être mis à jour. 

Le trafic continue ainsi d'être servi sans interruption (sans downtime).

Plutôt que d'écrire une recette de zéro, un rôle existant est utilisé depuis Ansible Galaxy : [`geerlingguy.haproxy`](https://galaxy.ansible.com/ui/standalone/roles/geerlingguy/haproxy/).

On installe le rôle :

```bash
ubuntu@ansible-main:~/ansible-playbooks$ ansible-galaxy role install geerlingguy.haproxy

Starting galaxy role install process
- downloading role 'haproxy', owned by geerlingguy
- downloading role from https://github.com/geerlingguy/ansible-role-haproxy/archive/1.3.2.tar.gz
- extracting geerlingguy.haproxy to /home/ubuntu/.ansible/roles/geerlingguy.haproxy
- geerlingguy.haproxy (1.3.2) was installed successfully
```

> Note : par défaut, les rôles installés via Ansible Galaxy sont placés dans `/home/USER/.ansible/roles/` :

```bash
~/.ansible/roles$ ll
total 12
drwxrwxr-x 3 ubuntu ubuntu 4096 Jul 27 16:05 ./
drwxrwxr-x 6 ubuntu ubuntu 4096 Jul 27 16:05 ../
drwxrwxr-x 9 ubuntu ubuntu 4096 Jul 27 16:05 geerlingguy.haproxy/
```

Pour que chaque rôle installé via Ansible Galaxy soit rangé dans un dossier local au projet plutôt que dans le dossier utilisateur global, une variable est ajoutée dans `ansible.cfg` pour le spécifier :

```ini
[defaults]
host_key_checking = False
retry_files_enabled = False
roles_path = /home/ubuntu/ansible-playbooks/
```
### 5. Configuration du rôle HAProxy

Le rôle expose de nombreuses variables par défaut, consultables dans `roles/geerlingguy.haproxy/defaults/main.yml` :

```yaml
---
haproxy_socket: /var/lib/haproxy/stats
haproxy_user: haproxy
haproxy_group: haproxy

# Frontend settings.
haproxy_frontend_name: 'hafrontend'
haproxy_frontend_bind_address: '*'
haproxy_frontend_port: 80
haproxy_frontend_mode: 'http'

# Backend settings.
haproxy_backend_name: 'habackend'
haproxy_backend_mode: 'http'
haproxy_backend_balance_method: 'roundrobin'
haproxy_backend_httpchk: 'HEAD / HTTP/1.1\r\nHost:localhost'

# List of backend servers.
haproxy_backend_servers: []
# - name: app1
#   address: 192.168.0.1:80
# - name: app2
#   address: 192.168.0.2:80
```

Un playbook `haproxy.yaml` est créé pour surcharger ces variables avec les deux serveurs web réels :

```c
---
- hosts: lb-host
  vars:
    haproxy_backend_servers:
      - name: web-server-1
        address: web-server-1:80
      - name: web-server-2
        address: web-server-2:80

  roles:
    - geerlingguy.haproxy

  tasks:
    - name: retirer cookie pour éviter bug de session
      lineinfile:
        path: /etc/haproxy/haproxy.cfg
        regexp: ".*cookie SERVERID.*"
        state: absent
```

Ces variables sont ensuite injectées dans le template Jinja2 du rôle, situé dans `roles/geerlingguy.haproxy/templates/haproxy.cfg.j2` :

```jinja2
{% endif %}
    cookie SERVERID insert indirect
{% for backend in haproxy_backend_servers %}
```

Cette ligne fait qu'à la première connexion, le load balancer assigne le client à un serveur précis et lui fournit un cookie, afin qu'il reste toujours connecté au même serveur par la suite. Utile pour préserver les sessions PHP, mais ici problématique  : si le client envoie une requête à un autre serveur, il reçoit un cookie différent, ce qui génère des bugs difficiles à tester et à déboguer. 

Cette fonctionnalité est donc désactivée afin que les requêtes puissent être réparties librement entre les deux serveurs (sinon, le client serait systématiquement redirigé vers le même serveur).

> Note : la bonne pratique consisterait à gérer ce comportement directement dans le code applicatif, via une gestion de session partagée (pool de requêtes conditionnée), pour rendre le code réellement modulable. Faute de temps, une solution rapide a été retenue ici : supprimer la ligne du cookie via une regex dans le playbook.

### 6. Conflit de port 80 entre HAProxy et Apache

Le déploiement du playbook HAProxy échoue :

```bash
TASK [geerlingguy.haproxy : Ensure HAProxy is started and enabled on boot.] ***************
fatal: [web-server-1]: FAILED! => {"changed": false, "msg": "Unable to start service haproxy: Job for haproxy.service failed because the control process exited with error code.\nSee \"systemctl status haproxy.service\" and \"journalctl -xeu haproxy.service\" for details.\n"}
fatal: [web-server-2]: FAILED! => {"changed": false, "msg": "Unable to start service haproxy: Job for haproxy.service failed because the control process exited with error code.\nSee \"systemctl status haproxy.service\" and \"journalctl -xeu haproxy.service\" for details.\n"}
```

Les logs révèlent la cause :

```bash
journalctl -xeu haproxy.service --no-pager -n 50 | grep "cannot "

Jul 27 17:31:31 web-server-1 haproxy[3387]: [ALERT]    (3387) : Starting frontend hafrontend: cannot bind socket (Address already in use) [0.0.0.0:80]
```

On peut voir ici que HAProxy et Apache tentent tous deux d'écouter sur le port 80 de la même machine, ce qui est impossible. Apache est déjà présent et actif sur ce port :

```bash
ubuntu@web-server-1:~$ sudo ss -tulnp 
Netid   State     Recv-Q    Send-Q           Local Address:Port        Peer Address:Port   Process                                                       
tcp     LISTEN    0         511                          *:80                     *:*       users:(("apache2",pid=666,fd=4),("apache2",pid=665,fd=4),("apache2",pid=663,fd=4))        
tcp     LISTEN    0         128                       [::]:22                  [::]:*       users:(("sshd",pid=656,fd=4))            
```

Il serait possible de faire écouter HAProxy sur un autre port (par exemple 8080), ou bien changer le port du serveur web. Mais ça n'a pas de sens de placer HAProxy en frontal sur la même machine que les services qu'il équilibre. On va donc créer une VM dédiée pour HAProxy, plus simple et plus propre.

Une nouvelle VM est créée avec Multipass, de façon analogue aux autres VM créées précédemment :

```bash
multipass launch 22.04 -n lb-server -c 1 -m 3G
```

On ajoute ensuite la même configuration que pour l'étape 1 : autorisation de connexion SSH en root, configuration d'Ansible, ajout dans `/etc/hosts`, ajout du nom de la VM dans le script cron de synchronisation DHCP, etc...

L'inventory est ensuite mis à jour pour ajouter un groupe parent commun, `all_servers`, ce qui évite les problèmes de connexion à la nouvelle VM :

```ini
[web]
web-server-1
web-server-2

[lb-host]
lb-server

[all_servers:children]
web
lb-host

[all_servers:vars]
ansible_user=root
ansible_password=ansible
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
ansible_python_interpreter=/usr/bin/python3.10
```

### 7. Correction du rôle web

En relisant le `main.yml` du rôle web, j'ai remarqué plusieurs défauts :

- l'usage de `add-apt-repository` au lieu du module dédié `apt_repository` ;
- l'absence d'un `apt update` explicite avant l'installation des paquets ;
- un mélange incohérent entre les versions PHP 7.2 et 7.3 ;
- une activation implicite du module PHP dans Apache, jamais garantie explicitement, et nécessaire dans notre cas.

Le rôle est donc corrigé pour gagner en idempotence et harmoniser les versions de paquets :

```yaml
---
# tasks file for web

- name: Installer Apache
  apt:
    name: apache2
    state: present
    update_cache: yes

- name: Démarrer Apache
  service:
    name: apache2
    state: started
    enabled: yes

- name: Installer le support des dépôts additionnels
  apt:
    name: software-properties-common
    state: present

- name: Ajouter le dépôt PHP Ondřej
  apt_repository:
    repo: ppa:ondrej/php
    state: present
    update_cache: yes

- name: Installer PHP et ses modules
  apt:
    name:
      - php7.3
      - libapache2-mod-php7.3
      - php7.3-cli
      - php7.3-common
      - php7.3-mysql
      - php7.3-curl
      - php7.3-gd
      - php7.3-zip
      - php7.3-apcu
    state: present
  notify: restart apache

- name: Désactiver mpm_event
  command: a2dismod mpm_event
  args:
    removes: /etc/apache2/mods-enabled/mpm_event.load
  notify: restart apache

- name: Activer mpm_prefork
  command: a2enmod mpm_prefork
  args:
    creates: /etc/apache2/mods-enabled/mpm_prefork.load
  notify: restart apache

- name: Activer le module PHP
  command: a2enmod php7.3
  args:
    creates: /etc/apache2/mods-enabled/php7.3.load
  notify: restart apache
```

On résout le bug de troncature observé plus tôt : le code PHP est désormais correctement interprété par Apache plutôt que renvoyé tel quel au navigateur :)

### 8. Déploiement final du load balancer

Le playbook HAProxy peut désormais être exécuté avec succès :

```bash
ansible-playbook -i inventory haproxy.yaml

[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see
details

PLAY [lb-host] ****************************************************************************

TASK [Gathering Facts] ********************************************************************
ok: [lb-server]

TASK [geerlingguy.haproxy : Ensure HAProxy is installed.] *********************************
changed: [lb-server]

TASK [geerlingguy.haproxy : Ensure HAProxy is enabled (so init script will start it on Debian).] ***
changed: [lb-server]

TASK [geerlingguy.haproxy : Get HAProxy version.] *****************************************
ok: [lb-server]

TASK [geerlingguy.haproxy : Set HAProxy version.] *****************************************
ok: [lb-server]

TASK [geerlingguy.haproxy : Copy HAProxy configuration in place.] *************************
changed: [lb-server]

TASK [geerlingguy.haproxy : Ensure HAProxy is started and enabled on boot.] ***************
ok: [lb-server]

TASK [retirer cookie pour éviter bug de session] ******************************************
changed: [lb-server]

RUNNING HANDLER [geerlingguy.haproxy : restart haproxy] ***********************************
changed: [lb-server]

PLAY RECAP ********************************************************************************
lb-server                  : ok=9    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

L'application est désormais fonctionnelle sur les deux serveurs, avec le trafic correctement réparti par le load balancer.

![[Pasted image 20260727222340.png|460]]

![[Pasted image 20260727222415.png|462]]

Le contenu affiché diffère volontairement d'un serveur à l'autre, puisque chacun héberge sa propre base de données MySQL locale. Il serait possible d'unifier ce comportement en mettant la base sur une autre VM et accessible par les deux serveurs web, ou en mettant en place une réplication entre les deux bases.
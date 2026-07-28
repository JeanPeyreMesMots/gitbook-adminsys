Ansible => éliminer la peur du déploiement pour le week-end quand on veut etre trkl

+ de changements entre 2 déploiement, si un truc va mal entre many changements => + difficile de voir la source du problème
+ time 2 market : + long, si trop long entreprise ralentit 
+ gmail : par ex déploie 100aine de fois par jour raffiné avec le temps => confiant dans l'infra
+ netflix : monkeyscript pour s'amuser à test l'infra
+ ansible permet ainsi de gérer efficacement les déploiements
## erreurs à éviter :

- éviter des dep manuels, pas d'intervention manuel => potentiellement des erreurs
- longs intervals de déploiement => on veut maj les LB, ou du code, on ne veut maj que ça 
- éviter les deploiement le week-end ou la nuit => on veut le - d'impact, pire moment en vrai : déjà car c'est le we, le soir ça ouvre la porte aux soucis
- en semaine : tt le monde est là pour voir si todo bueno, + journée tout le monde est frais
## règles à suivre :

- PAS d'opération manuel, par pitié !
- car quand tout est auto : on règle dans le play puis on rebalance tout, une x pour toutes
- corriger DANS le playbooks
- + changements avec git : garanti d'un chgmt
- par ex : déploi tt les mois => tt les semaines => tt les jours pour fluidifer le truc
- déploiement les + petits possibles : si on le fait rapidement avec moins de changements cumulés
- en plusieurs fois également : changmeent de la bdd avant l'appli, à adapter en fonction, par ex 1 le matin, 1 l'aprem
- déploiement dans la journée qd tt le monde est présent, pas à 4h du mat ou le we xD)
- déploiement sans downtime technique du blue/green : déploi d'une autre infra w/ nouvelle version d'une appli, petit à petit rediriger le traffic vers une autre infra 
## mise en pratique

On va faire une appli todo list => déploiement et sans impact pour les users

on commence par créer le deploy.yml :

```yaml
---
- hosts: web
roles:
- deploy
```

Puis avec la commande suivante on créé ensuite le role qu'il faut :

```bash
ansible-galaxy init deploy
- Role deploy was created successfully
```

On a donc l'arbo suivante :

```bash
cat deploy.yml 
---
- hosts: web
  roles:
  - deployubuntu@ansible-main:~/ansible-playbooks$ 


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

puis on get l'appli ici : https://gitlab.com/ttwthomas/app-example-php avec le module git, pour récupérer le repo. On créé ensuite la tache suivante dans le main.yml qui se décompose ainsi, avec des mécanisme de controle pour run la tache de façon idempotente avec le repo git get, et l'import du sql, sans avoir à la run une deuxième fois.

Note : "**python-pymysql**" n'existe plus sur Ubuntu 22.02, il faut donc le remplacer par "**python3-pymysql**"

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

# var pour le repo git

register: git_repo

  

- name: restaurer la base de donnees

mysql_db:

name: cocadmin

state: import

target: /var/www/html/app/creer_table.sql

# seul hostname que j'ai

login_host: web

login_user: cocadmin

login_password: 1234

# empecher la casse a chaque jeu en jouant la tache 1 fois

run_once: true

# pas de run une 2eme fois si module git a fait changement

when: git_repo.changed
```

Tout va bien, mais le test d'une req http bloque sur le serveur me retourne un resultat html tronqué.

Test sur le premier serveur :

```bash
ubuntu@web-server-1:~$ curl localhost/app/index.php
<!--
   Copyright 2017 Vinzenz Feenstra, Red Hat, Inc.

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->
<?php
define('DB_USER', 'cocadmin');
[...]
```

Et sur le deuxième :

```bash
ubuntu@web-server-2:~$ curl localhost/app/index.php
<!--
   Copyright 2017 Vinzenz Feenstra, Red Hat, Inc.

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->
<?php
define('DB_USER', 'cocadmin');
[...]
```

## Mise en place d'un load balancer

Let's go avec haproxy, pour répartir la charge efficacement entre nos deux serveurs

car quand client -> req -> lb -> serveur. charge balancé sur nos deux serveurs. pdt déploiement, trafic redirigé vers 1 seul serveur, pour assurer la dispo, et déployer d'un côté 

serveur sur lequel on déploei -> on le retire du LB. tout autres req ira sur l'autre serveur du coup, celui sur lequel on est pas en train de déployer. on check ensuite que tout est correct, on remet ce serveur sur l'autre LB, et on retire l'autre. on maj, puis on le remettra. traffic send to serv, without downtime.

on va pas utiliser de recette, on va use role depuis ansible galaxy :

https://galaxy.ansible.com/ui/standalone/roles/geerlingguy/haproxy/

down count high

On l'install :

```bash
ubuntu@ansible-main:~/ansible-playbooks$ ansible-galaxy role install geerlingguy.haproxy


Starting galaxy role install process
- downloading role 'haproxy', owned by geerlingguy
- downloading role from https://github.com/geerlingguy/ansible-role-haproxy/archive/1.3.2.tar.gz
- extracting geerlingguy.haproxy to /home/ubuntu/.ansible/roles/geerlingguy.haproxy
- geerlingguy.haproxy (1.3.2) was installed successfully
```

Note : les roles installés finissent dans : /home/USER/.ansible/roles/

```bash
~/.ansible/roles$ ll
total 12
drwxrwxr-x 3 ubuntu ubuntu 4096 Jul 27 16:05 ./
drwxrwxr-x 6 ubuntu ubuntu 4096 Jul 27 16:05 ../
drwxrwxr-x 9 ubuntu ubuntu 4096 Jul 27 16:05 geerlingguy.haproxy/
```

Pour avoir un dir statique pour chaque install de role de ansible galaxy, on fout une var dans le fichier "/**home/ubuntu/ansible-playbooks/ansible.cfg**" :

```c
[defaults]
host_key_checking = False
retry_files_enabled = False
roles_path = /home/ubuntu/ansible-playbooks/

```

Bueno :

```bash
ansible-galaxy role install geerlingguy.haproxy
Starting galaxy role install process
- downloading role 'haproxy', owned by geerlingguy
- downloading role from https://github.com/geerlingguy/ansible-role-haproxy/archive/1.3.2.tar.gz
- extracting geerlingguy.haproxy to /home/ubuntu/ansible-playbooks/roles/geerlingguy.haproxy
- geerlingguy.haproxy (1.3.2) was installed successfully
```

Comme toute les vars sont spécifiées, on va changer celles-ci :

```yaml
cat roles/geerlingguy.haproxy/defaults/main.yml 
---
haproxy_socket: /var/lib/haproxy/stats
haproxy_chroot: /var/lib/haproxy
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

On créé un playbook haproxy.yaml avec les vars suivantes :

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

ces vars seront ensuite injectés dans le fichier de config de haproxy "**/home/ubuntu/ansible-playbooks/roles/geerlingguy.haproxy/templates/haproxy.cfg.j2**"
archi jinja2. 

Sauf que dedans :

```c
{% endif %}
    cookie SERVERID insert indirect
{% for backend in haproxy_backend_servers %}
```

cette ligne : qd client co à lb, assigné à server, server give cookie, pour se co tjrs au meme endroit. pratique pour session php, sauf que si client make req to another serv -> autre cookie donné, donne des bugs dans notre cas. we will disable cela pour que req be shufled between two servers, autrement on sera tt temps redirigé vers autre serveur. hard à tester et débugger.

la bonne manière de faire ça : faire une pool req sur le code avec condition dessus, pour rendre code modulable. la c'est juste qu'on a pas le temps et qu'on veut aller vite :)
d'où la regex que l'on a mise dans le playbook.

Problème quand on déploie :

```bash
TASK [geerlingguy.haproxy : Ensure HAProxy is started and enabled on boot.] ***************
fatal: [web-server-1]: FAILED! => {"changed": false, "msg": "Unable to start service haproxy: Job for haproxy.service failed because the control process exited with error code.\nSee \"systemctl status haproxy.service\" and \"journalctl -xeu haproxy.service\" for details.\n"}
fatal: [web-server-2]: FAILED! => {"changed": false, "msg": "Unable to start service haproxy: Job for haproxy.service failed because the control process exited with error code.\nSee \"systemctl status haproxy.service\" and \"journalctl -xeu haproxy.service\" for details.\n"}
```

En regardant les logs :

```bash
journalctl -xeu haproxy.service --no-pager -n 50 | grep "cannot "

Jul 27 17:31:31 web-server-1 haproxy[3387]: [ALERT]    (3387) : Starting frontend hafrontend: cannot bind socket (Address already in use) [0.0.0.0:80]
```

Cela est à cause du role apache conf précedemment. HAProxy et les serveurs web ne doivent **pas** être sur la même machine s'ils écoutent tous les deux sur le port 80. On a déjà un apache qui tourne dessus sur sun serveur web :

```bash
ubuntu@web-server-1:~$ sudo ss -tulnp 
Netid   State     Recv-Q    Send-Q           Local Address:Port        Peer Address:Port   Process                                                       
tcp     LISTEN    0         511                          *:80                     *:*       users:(("apache2",pid=666,fd=4),("apache2",pid=665,fd=4),("apache2",pid=663,fd=4))        
tcp     LISTEN    0         128                       [::]:22                  [::]:*       users:(("sshd",pid=656,fd=4))            
```

Il faut alors faire écouter HAProxy sur un autre port (ex: 8080) et faire pointer ton serveur web sur ce port, ou inversement changer le port du serveur web. Mais architecturalement, ça n'a pas vraiment de sens d'avoir HAProxy en frontal **et** le service qu'il équilibre sur la même machine. On va donc créer une autre vm multipass dédiée, + simple et moins compliqué.
## creation vm lb

je choisis de repartir d'une image ubuntu vierge comme on a fait, en thin provisioning :

```bash
multipass launch 22.04 -n lb-server -c 1 -m 3G
```

on reconf comme on avait fait à l'étape 1, avec autorisaiton pour la connexion en ssh via root, ansible, ajout de /etc/hosts, ajout du nom de la vm dans le script de cron pour synchro le DHCP, etc...

On change ensuite l'inventory pour ajouter un groupe parent commun, ici "**all_servers**", et ainsi éviter des problèmes de connexion à la VM et tout :

```c
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

Note : en revoyant le playbook main.yml des tasks de mon role, je me suis aperçu de plusieurs défauts, notamment :

- `add-apt-repository`
- pas de `apt update`
- mélange PHP 7.2 / 7.3
- activation implicite de PHP

J'ai donc corrigé la playbook pour avoir plus d'idempotence + harmonisation des paquets :

```c
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

Ce qui a corrigé de soucis d'affichage et d'interprétation de code PHP côté navigateur
Puis on peut lancer le playbook haproxy :

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

Tout bon ! On a désormais notre appli fonctionnelle sur les deux serveurs avec le traffic load balancé !

![[Pasted image 20260727222340.png|460]]

![[Pasted image 20260727222415.png|462]]

Bien sûr le contenu n'est pas le même, car chaque serveur dispose de sa propre base de données. Mais il est possible de changer ça en créant un seul serveur hébergeant la base MySQL ou un créant une réplication de la base de données.


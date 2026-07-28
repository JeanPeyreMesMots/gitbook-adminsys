Avoid co une par une sur les serveurs, ingérables.
MultiSSH : un peu mieux, mais peu scalable si ~100aine machines, mauvaise vu sur les serveurs
Scripts : mieux, tjrs actuel. Pas idéal non plus, pleins de cas particuliers qu'on va pas gérer, checkings de logs chiants...

Avoir de l'idempotence, ansible permet d'avoir 

# Test de module

Pour permettre d'id u groupe, => fichier ansible.cfg pour prendre l'inventory :

```bash
cat ansible.cfg
[defaults]
inventory = ./inventory
```

et assigner un groupe. - On peux aussi faire plusieurs inventory files et specifier le bon a chaque run (-i). ainsi si on test le module shell pour tester l'exec d'une commande :

```bash
ansible web -m shell -a "uname -a"
web-server-1 | CHANGED | rc=0 >>
Linux web-server-1 5.15.0-185-generic #195-Ubuntu SMP Fri Jun 19 17:11:50 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
web-server-2 | CHANGED | rc=0 >>
Linux web-server-2 5.15.0-185-generic #195-Ubuntu SMP Fri Jun 19 17:11:50 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
```

Retour pas dans le mm ordre, en fonction du first host qui repond
note : diff groupes, pour diff variables.
ici par ex : web:vars est un sous groupe

en fonction des use case -> use regex :

```bash
cat inventory
[web]
web-server-1
web-server-2

[backup-web]
web-server-2

[web:vars]
ansible_ssh_user=root
ansible_ssh_pass=ansible
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
ansible_python_interpreter=/usr/bin/python3.10

ubuntu@ansible-main:~/ansible$ ansible backup* -i inventory -a "uname -a"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
web-server-2 | CHANGED | rc=0 >>
Linux web-server-2 5.15.0-185-generic #195-Ubuntu SMP Fri Jun 19 17:11:50 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
```

or list :

```bash
ansible backup-web,web -i inventory -a "uname -a"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
web-server-1 | CHANGED | rc=0 >>
Linux web-server-1 5.15.0-185-generic #195-Ubuntu SMP Fri Jun 19 17:11:50 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
web-server-2 | CHANGED | rc=0 >>
Linux web-server-2 5.15.0-185-generic #195-Ubuntu SMP Fri Jun 19 17:11:50 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
```

Module apt par ex :

```bash
ansible web -m apt -a "name=nmap"
web-server-1 | CHANGED => {
    "cache_update_time": 1783169532,
    "cache_updated": false,
    "changed": true,
    "stderr": "",
    "stderr_lines": [],
    "stdout": "Reading package lists...\nBuilding dependency tree...\nReading state information...\nThe following additional packages will be installed:\n  libblas3 liblinear4 liblua5.3-0 lua-lpeg nmap-common\nSuggested packages:\n  liblinear-t

==>
web-server-2 | SUCCESS => {
    "cache_update_time": 1783169589,
    "cache_updated": false,
    "changed": false
}
web-server-1 | SUCCESS => {
    "cache_update_time": 1783169532,
    "cache_updated": false,
    "changed": false
}
```

Module de script :

```bash
# ex avec script tout simple
cat script-test.sh
uname -a
date

ubuntu@ansible-main:~/ansible$ ansible web -m script -a "./script-test.sh"
web-server-1 | CHANGED => {
    "changed": true,
    "rc": 0,
    "stderr": "Shared connection to web-server-1 closed.\r\n",
    "stderr_lines": [
        "Shared connection to web-server-1 closed."
    ],
    "stdout": "Linux web-server-1 5.15.0-185-generic #195-Ubuntu SMP Fri Jun 19 17:11:50 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux\r\nSat Jul  4 17:25:48 CEST 2026\r\n",
    "stdout_lines": [
        "Linux web-server-1 5.15.0-185-generic #195-Ubuntu SMP Fri Jun 19 17:11:50 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux",
        "Sat Jul  4 17:25:48 CEST 2026"
    ]
}
web-server-2 | CHANGED => {
    "changed": true,
    "rc": 0,
    "stderr": "Shared connection to web-server-2 closed.\r\n",
    "stderr_lines": [
        "Shared connection to web-server-2 closed."
    ],
    "stdout": "Linux web-server-2 5.15.0-185-generic #195-Ubuntu SMP Fri Jun 19 17:11:50 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux\r\nSat Jul  4 17:25:48 CEST 2026\r\n",
    "stdout_lines": [
        "Linux web-server-2 5.15.0-185-generic #195-Ubuntu SMP Fri Jun 19 17:11:50 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux",
        "Sat Jul  4 17:25:48 CEST 2026"
    ]
}
```

Un usecase : remplacer les scripts par des modules via roles, playbooks, etc...
## Yaml, playbook, hostvars

- eliminer la peur des infrastructures : peur du déploiement, peur des maj, peur de casse...
- mise en place lente => ansible résout ce soucis. chaque fois qu'un nouvel env se pointait et un new bug => pb infra
- ne jamais faire de changement manuel sur un serveur => changement dans le role/code de la conf => mise dans git => en expliquant pourquoi => chgn fait sur all serveur

- tjrs faire changement ds role/playbook => git
- utiliser au max les recettes existantes. forte commu donc pleins de roles

- module shell => le - possible pour avoir des recettees idempotentes 
- chaque module pour chaque actions

- env jetable car recréable en qques minutes

## Mise en place stack lamp & load balancer

un fichier yaml commence toujours pas 3 tirest : ---

```yaml
---
- hosts: web
  tasks:
  - name: installer apache2
    apt:
      name: apache2
```

Ainsi :

```bash
ubuntu@ansible-main:~/ansible/playbooks$ ansible-playbook -i ../inventory lamp.yml

PLAY [web] *******************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************
ok: [web-server-1]
ok: [web-server-2]

TASK [installer apache2] *****************************************************************************************************************
ok: [web-server-2]
ok: [web-server-1]

PLAY RECAP *******************************************************************************************************************************
web-server-1               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
web-server-2               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Si check sur l'un des serveurs, apache bien installé :

```bash
root@web-server-1:~# ll /etc/apache2/
total 88
drwxr-xr-x  8 root root  4096 Jul  4 17:23 ./
drwxr-xr-x 91 root root  4096 Jul  6 19:39 ../
-rw-r--r--  1 root root  7224 Jun  3 17:42 apache2.conf
drwxr-xr-x  2 root root  4096 Jul  4 17:23 conf-available/
drwxr-xr-x  2 root root  4096 Jul  4 17:23 conf-enabled/
-rw-r--r--  1 root root  1782 Mar 18  2024 envvars
-rw-r--r--  1 root root 31063 Mar 18  2024 magic
drwxr-xr-x  2 root root 12288 Jul  4 17:23 mods-available/
drwxr-xr-x  2 root root  4096 Jul  4 17:23 mods-enabled/
-rw-r--r--  1 root root   320 Mar 18  2024 ports.conf
drwxr-xr-x  2 root root  4096 Jul  4 17:23 sites-available/
drwxr-xr-x  2 root root  4096 Jul  4 17:23 sites-enabled/
```

Si on essaye de l'enlever :

```bash
root@web-server-1:~# sudo apt remove --purge apache2
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages were automatically installed and are no longer required:
  apache2-bin apache2-data apache2-utils libapr1 libaprutil1 libaprutil1-dbd-sqlite3 libaprutil1-ldap ssl-cert
Use 'sudo apt autoremove' to remove them.
The following packages will be REMOVED:
  apache2*
0 upgraded, 0 newly installed, 1 to remove and 1 not upgraded.
After this operation, 549 kB disk space will be freed.
Do you want to continue? [Y/n] Y
(Reading database ... 66272 files and directories currently installed.)
Removing apache2 (2.4.52-1ubuntu4.21) ...
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for ufw (0.36.1-4ubuntu0.1) ...
(Reading database ... 66222 files and directories currently installed.)
Purging configuration files for apache2 (2.4.52-1ubuntu4.21) ...
Processing triggers for ufw (0.36.1-4ubuntu0.1) ..
```

ça nous indique le changement, avec changed :

```bash
$ ansible-playbook -i ../inventory lamp.yml

PLAY [web] *******************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************
ok: [web-server-1]
ok: [web-server-2]

TASK [installer apache2] *****************************************************************************************************************
ok: [web-server-2]
changed: [web-server-1]

PLAY RECAP *******************************************************************************************************************************
web-server-1               : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
web-server-2               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

ex ici avec service to run :

```bash
---
- hosts: web
  tasks:
  - name: installer apache2
    apt:
      name: apache2

  - name: demarrer apache
    service:
      name: apache2
      state: started
      enabled: yes
```

on peut voir l'idempotence ici en action pour une install via sources.list de php :

```yaml
  - name: installer add-apt-repository
    apt:
      name: software-properties-common

  - name: ajouter repo php
    shell: "add-apt-repository -y ppa:ondrej/php"
    environment:
      LC_ALL: "C.UTF-8"
    args:
        creates: /etc/apt/sources.list.d/ondrej-ubuntu-php-xenial.list
```

verif d'ajout du repo dans le apt sources list

# keep pushing

playbook w/ paquet and handlers :

```yaml
- hosts: web
  tasks:
    - name: installer apache2
      apt:
        name: apache2

    - name: demarrer apache
      service:
        name: apache2
        state: started
        enabled: yes

    - name: installer add-apt-repository
      apt:
        name: software-properties-common

    - name: ajouter repo php
      shell: "add-apt-repository -y ppa:ondrej/php"
      environment:
        LC_ALL: "C.UTF-8"
      args:
        creates: /etc/apt/sources.list.d/ondrej-ubuntu-php-xenial.list

    - name: installer php et ses modules
      apt:
        name:
          - php7.3
          - php7.3-common
          - php7.3-cli
          - php7.2-json
          - php7.2-gd
          - php7.2-curl
          - php7.2-mysql
          - php7.2-zip
          - php7.2-apcu
          cache_valid_time: yes
      notify: restart apache

  handlers:
    - name: restart apache
      service:
        name: apache2
        state: restarted
```

"cache_valid_time: 3600" permet de faire un apt update si pas fait depuis 1h

```bash
TASK [ajouter repo php] ******************************************************************************************************************
changed: [web-server-2]
changed: [web-server-1]

TASK [installer php et ses modules] ******************************************************************************************************
ok: [web-server-1]
ok: [web-server-2]

PLAY RECAP *******************************************************************************************************************************
web-server-1               : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
web-server-2               : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

on a désormais notre 1er playbook fonctionnel :D
ansible s'adapte au OS en fonction des noms de paquets et tout, gg à lui

# Création d'un playbook Mysql

- handler only made at end of playbook -> permet de run tache si une tache make a changment
- 1 suel fois à la fois : reboot mysql
- sauf qu'il faut faire ttoues les taches
- expliquer choix cocadmin: raison obscure; mysql n'accepte plus les connexions via mdp donc on passe par une socket
- check de conf au début du playbook lamp.

On a donc notre playbook dirty qui nous permet de creer full setup bdd avec auth mdp en dur (car lab). passe par une socket car contournement du blocage d'aut par mdp dépreciée

```yaml
---
- hosts: web
  vars:
    nom_bdd: cocadmin

  tasks:
    - name: installer mysql server
      apt:
        name:
          - mysql-server
          - mysql-client
          - python3-pymysql
      notify: demarrer mysql

    - name: configurer mysql
      ini_file:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        section: mysqld
        option: bind-address
        value: "0.0.0.0"
      notify: demarrer mysql

    - name: creer une bdd "cocadmin"
      mysql_db:
        name: "{{ nom_bdd }}"
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: supprimer la bdd "test"
      mysql_db:
        name: test
        state: absent
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: creer user pour bdd "cocadmin" avec authentification par mot de passe
      mysql_user:
        name: "{{ nom_bdd }}"
        password: "1234"
        priv: "{{ nom_bdd }}.*:ALL"
        host: "%"
        plugin: mysql_native_password
        login_unix_socket: /var/run/mysqld/mysqld.sock

  handlers:
    - name: demarrer mysql
      service:
        name: mysql
        state: restarted
```

sur chaques serv on a notre bdd avec combo user/1234 :

```sql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| cocadmin           |
| information_schema |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)

mysql> select cocamdin
    -> ^C
mysql> select cocadmn
    -> ^C
mysql> select cocadmin
    -> ^C
mysql> select cocadmin;
ERROR 1054 (42S22): Unknown column 'cocadmin' in 'field list'
mysql> use cocadmin;
Database changed
mysql> show tables;
Empty set (0.00 sec)

mysql>
```

maintenant comme on veut tout faire une seule commande. Comme on many files et on veut que ça soit plus évolutif, plus modulaire, on va voir comment on peut "joindre" ces deux fichiers pour tout déployer une seul commande, pour une meilleure reproductibilité comme on le trouverait dans une vraie infrastructure => cas + réél
## Ansible galaxy

playbook : list de plays
nos deux playbooks lamp et mysql sont en fait une liste de plays

on pourrait fusionner les deux playbooks, mais ça va devenir des fichiers gros et pas séparable en many parties distinctes, je peux pas par ex lancer que la partie web. 1 playbook = tout mes plays dedans.

playbook = contains plays
plays = contains roles

on va donc utiliser ces roles afin de basculer vers une utilisation + modulaire des playbooks

on va donc assembler ces roles dans un playbook, un role web & un mysql :

```bash
ubuntu@ansible-main:~/ansible/playbooks$ ansible-galaxy init web
- Role web was created successfully
ubuntu@ansible-main:~/ansible/playbooks$ ansible-galaxy init mysql
- Role mysql was created successfully
ubuntu@ansible-main:~/ansible/playbooks$ ll
total 24
drwxrwxr-x  4 ubuntu ubuntu 4096 Jul  8 19:02 ./
drwxrwxr-x  3 ubuntu ubuntu 4096 Jul  7 19:52 ../
-rw-rw-r--  1 ubuntu ubuntu  937 Jul  7 20:14 lamp.yml
drwxrwxr-x 10 ubuntu ubuntu 4096 Jul  8 19:02 mysql/
-rw-rw-r--  1 ubuntu ubuntu 1117 Jul  8 18:18 mysql.yml
drwxrwxr-x 10 ubuntu ubuntu 4096 Jul  8 19:02 web/
ubuntu@ansible-main:~/ansible/playbooks$ ll mysql
total 44
drwxrwxr-x 10 ubuntu ubuntu 4096 Jul  8 19:02 ./
drwxrwxr-x  4 ubuntu ubuntu 4096 Jul  8 19:02 ../
-rw-rw-r--  1 ubuntu ubuntu 1328 Jul  8 19:02 README.md
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 defaults/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 files/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 handlers/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 meta/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 tasks/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 templates/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 tests/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 vars/
ubuntu@ansible-main:~/ansible/playbooks$ ll web/
total 44
drwxrwxr-x 10 ubuntu ubuntu 4096 Jul  8 19:02 ./
drwxrwxr-x  4 ubuntu ubuntu 4096 Jul  8 19:02 ../
-rw-rw-r--  1 ubuntu ubuntu 1328 Jul  8 19:02 README.md
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 defaults/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 files/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 handlers/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 meta/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 tasks/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 templates/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 tests/
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul  8 19:02 vars/
```

Puis on va venir copier/coller depuis nos playbook.yml : les tasks, les handlers et les vars dans chaque main.yml dans les dossiers tasks/, handlers/, et vars/ pour chaque role. Ainsi pour le role web, on va retrouver par ex dans le fichier suivant "**web/tasks/main.yml**" toute les taches du playbook initial :

```yaml
~/ansible-playbooks$ cat web/tasks/main.yml 
---
# tasks file for web
- name: installer apache2
  apt:
	name: apache2

- name: demarrer apache
  service:
	name: apache2
	state: started
	enabled: yes

- name: installer add-apt-repository
  apt:
	name: software-properties-common

- name: ajouter repo php
  shell: "add-apt-repository -y ppa:ondrej/php"
  environment:
	LC_ALL: "C.UTF-8"
  args:
	creates: /etc/apt/sources.list.d/ondrej-ubuntu-php-xenial.list

- name: installer php et ses modules
  apt:
	name:
	  - php7.3
	  - php7.3-common
	  - php7.3-cli
	  - php7.2-json
	  - php7.2-gd
	  - php7.2-curl
	  - php7.2-mysql
	  - php7.2-zip
	  - php7.2-apcu
	cache_valid_time: yes
  notify: restart apache
```

On fait pareille pour les handlers :

```yaml
cat web/handlers/main.yml 
---
# handlers file for web
- name: restart apache
  service:
    name: apache2
    state: restarted
```

Même chose pour mysql et ses taches, handlers et vars :

```bash
ubuntu@ansible-main:~/ansible-playbooks$ cat mysql/tasks/main.yml 
---
# tasks file for mysql
- name: installer mysql server
  apt:
    name:
      - mysql-server
      - mysql-client
      - python3-pymysql
  notify: demarrer mysql

- name: configurer mysql
  ini_file:
    path: /etc/mysql/mysql.conf.d/mysqld.cnf
    section: mysqld
    option: bind-address
    value: "0.0.0.0"
  notify: demarrer mysql

- name: creer une bdd "cocadmin"
  mysql_db:
    name: "{{ nom_bdd }}"
    login_unix_socket: /var/run/mysqld/mysqld.sock

- name: supprimer la bdd "test"
  mysql_db:
    name: test
    state: absent
    login_unix_socket: /var/run/mysqld/mysqld.sock

- name: creer user pour bdd "cocadmin" avec authentification par mot de passe
  mysql_user:
    name: "{{ nom_bdd }}"
    password: "1234"
    priv: "{{ nom_bdd }}.*:ALL"
    host: "%"
    plugin: mysql_native_password
    login_unix_socket: /var/run/mysqld/mysqld.sock
```

```yaml
~/ansible-playbooks$ cat mysql/handlers/main.yml 
---
# handlers file for mysql
- name: demarrer mysql
  service:
    name: mysql
    state: restarted
```

Et ses vars :

```bash
cat mysql/vars/main.yml 
---
# vars file for mysql
nom_bdd: cocadmin
```

On va ensuite créer un playbook "**lamp.yml**" englobant tout qui va appeler tout ce que l'on a mis avant, et qui va installer tout ce que l'on a défini dans les roles :

```yaml
---
- import_playbook: web
- import_playbook: mysql
```

Et on garde la modularité d'avoir nos deux playbooks. Cependant si on appel le fichier avec ansible on se heurte à une erreur :

```bash
ll
total 32
drwxrwxr-x  4 ubuntu ubuntu 4096 Jul 23 17:54 ./
drwxr-x---  7 ubuntu ubuntu 4096 Jul 23 17:00 ../
-rwxr-xr-x  1 ubuntu ubuntu  192 Jul 23 15:06 inventory*
-rw-rw-r--  1 ubuntu ubuntu   51 Jul 23 17:49 lamp.yaml
drwxrwxr-x 10 ubuntu ubuntu 4096 Jul 23 16:30 mysql/
-rw-rw-r--  1 ubuntu ubuntu   36 Jul 23 17:34 mysql.yaml
drwxrwxr-x 10 ubuntu ubuntu 4096 Jul 23 16:30 web/
-rw-rw-r--  1 ubuntu ubuntu   29 Jul 23 17:32 web.yaml


ubuntu@ansible-main:~/ansible-playbooks$ ansible-playbook lamp.yaml 
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the
implicit localhost does not match 'all'
ERROR! an error occurred while trying to read the file '/home/ubuntu/ansible-playbooks/web': [Errno 21] Is a directory: b'/home/ubuntu/ansible-playbooks/web'. [Errno 21] Is a directory: b'/home/ubuntu/ansible-playbooks/web'
```

Cela est du fait que les roles doivent être dans un dossier qui s'appel "**roles**". On va donc bouger les dossiers, les roles donc, dans un dossier appelé "**roles**" :

```bash
mkdir roles
ubuntu@ansible-main:~/ansible-playbooks$ ll
total 36
drwxrwxr-x  5 ubuntu ubuntu 4096 Jul 23 17:57 ./
drwxr-x---  7 ubuntu ubuntu 4096 Jul 23 17:00 ../
-rwxr-xr-x  1 ubuntu ubuntu  192 Jul 23 15:06 inventory*
-rw-rw-r--  1 ubuntu ubuntu   51 Jul 23 17:49 lamp.yaml
drwxrwxr-x 10 ubuntu ubuntu 4096 Jul 23 16:30 mysql/
-rw-rw-r--  1 ubuntu ubuntu   36 Jul 23 17:34 mysql.yaml
drwxrwxr-x  2 ubuntu ubuntu 4096 Jul 23 17:57 roles/
drwxrwxr-x 10 ubuntu ubuntu 4096 Jul 23 16:30 web/
-rw-rw-r--  1 ubuntu ubuntu   29 Jul 23 17:32 web.yaml
ubuntu@ansible-main:~/ansible-playbooks$ mv web/ roles/
ubuntu@ansible-main:~/ansible-playbooks$ mv mysql/ roles/
```

La structure étant bonne on peut lancer la commande pour balancer tout ça sur nos deux serveurs ;) :

```bash
ansible-playbook -i inventory lamp.yaml 

PLAY [web] ********************************************************************************

TASK [Gathering Facts] ********************************************************************
ok: [web-server-1]
ok: [web-server-2]

TASK [web : installer apache2] ************************************************************
ok: [web-server-2]
ok: [web-server-1]

TASK [web : demarrer apache] **************************************************************
ok: [web-server-2]
ok: [web-server-1]

[...]

PLAY RECAP ********************************************************************************
web-server-1               : ok=12   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
web-server-2               : ok=12   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

désomrias + modulaire car on peut lancer l'une des deux parties. dans une entreprise, idéalement, chaque role aura son repo git. Toujours est-il que l'on peut se baser sur ansible galaxy pour standardiser et réutiliser les rôles le plus possible 

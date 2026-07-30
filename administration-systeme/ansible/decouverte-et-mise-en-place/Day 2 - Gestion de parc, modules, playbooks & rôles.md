### Comparaison avec les autres méthodes

|Méthode|Limite|
|---|---|
|SSH une par une sur chaque serveur|Ingérable dès que le parc grandit|
|MultiSSH|Un peu mieux mais peu scalable (mauvaise expérience dès ~100 machines)|
|Scripts bash|Mieux, encore utilisé, mais pas idéal : nombreux cas particuliers non gérés, vérification des logs fastidieuse|
|**Ansible**|Apporte l'**idempotence** (une tâche rejouée sans changement ne modifie rien)|

### Pourquoi Ansible ?

Ansible vise à éliminer la peur de toucher à l'infrastructure : peur de déployer, de faire une mise à jour, ou de casser un service en production. Aucun changement ne doit jamais être effectué manuellement sur un serveur. 

Toute modification doit passer par le rôle ou le playbook correspondant, être commitée dans Git avec un message expliquant le "pourquoi", puis appliquée uniformément à tous les serveurs concernés.

Les environnements doivent être considérés comme jetables, puisqu'ils peuvent être recréés en quelques minutes grâce à Ansible. Chaque module correspond idéalement à une seule action précise. Le module `shell` reste utilisable, mais il casse l'idempotence s'il est employé sans précaution. 

Mieux vaut donc s'appuyer sur les rôles déjà développés par la communauté (=> Ansible Galaxy qui possède un écosystème riche) plutôt que de réinventer des recettes existantes.

### Structure logique à savoir

```
playbook = liste de plays
play     = contient des rôles
rôle     = contient tasks / handlers / vars / defaults / templates / files / meta
```

Pour définir l'inventory par défaut on choisit de créer fichier `ansible.cfg` :

```bash
cat ansible.cfg
[defaults]
inventory = ./inventory
```

> Il est également possible de disposer de plusieurs fichiers d'inventory et de préciser celui à utiliser au moment de l'exécution grâce à l'option `-i`.

Exemple d'inventory avec groupes, sous-groupe de variables (`web:vars`) :

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
```

On peut aussi cibler un groupe par regex :

```bash
ansible backup* -i inventory -a "uname -a"
```

Ou plusieurs groupes via une virgule :

```bash
ansible backup-web,web -i inventory -a "uname -a"
```

> Chaque groupe peut posséder ses propres variables : ici, `web:vars` est un sous-groupe qui définit les variables associées au groupe `web`. L'ordre dans lequel les serveurs répondent dépend du premier hôte à avoir traité la commande, et non de l'ordre dans lequel ils sont déclarés dans l'inventory.
### Test de modules en ad-hoc

Il existe une très longue liste pour tout types de modules : https://docs.ansible.com/projects/ansible/latest/collections/index_module.html parmi lesquelles :

Le Module `shell` pour exécuter une commande :

```bash
ansible web -m shell -a "uname -a"
web-server-1 | CHANGED | rc=0 >>
Linux web-server-1 5.15.0-185-generic ...
web-server-2 | CHANGED | rc=0 >>
Linux web-server-2 5.15.0-185-generic ...
```

Le Module `apt` pour installer un paquet :

```bash
ansible web -m apt -a "name=nmap"
```

Le Module `script` pour exécuter un script local sur les hôtes distants :

```bash
cat script-test.sh
uname -a
date

ansible web -m script -a "./script-test.sh"
```

> L'objectif à terme dans une infra réél serait de remplacer progressivement les scripts par des modules dédiés, appelés depuis des rôles ou des playbooks.
### Premier playbook YAML

Un fichier YAML doit toujours commencer par trois tirets (`---`). Ici un exemple de playbook que j'ai appelé "lamp.yml" qui installe Apache (pour l'instant):

```yaml
---
- hosts: web
  tasks:
    - name: installer apache2
      apt:
        name: apache2
```

Exécution :

```bash
ansible-playbook -i ../inventory lamp.yml

PLAY [web] ****************************************************
TASK [Gathering Facts] ****************************************
ok: [web-server-1]
ok: [web-server-2]

TASK [installer apache2] **************************************
ok: [web-server-2]
ok: [web-server-1]

PLAY RECAP *****************************************************
web-server-1  : ok=2  changed=0  unreachable=0  failed=0
web-server-2  : ok=2  changed=0  unreachable=0  failed=0
```

Vérification manuelle sur un serveur :

```bash
root@web-server-1:~# ll /etc/apache2/
total 88
drwxr-xr-x  8 root root  4096 Jul  4 17:23 ./
drwxr-xr-x 91 root root  4096 Jul  6 19:39 ../
-rw-r--r--  1 root root  7224 Jun  3 17:42 apache2.conf
drwxr-xr-x  2 root root  4096 Jul  4 17:23 conf-available/
drwxr-xr-x  2 root root  4096 Jul  4 17:23 conf-

[...]
```

**Démonstration de l'idempotence** : si Apache est supprimé manuellement puis que le playbook est relancé, Ansible détecte l'écart et réinstalle uniquement ce qui manque. Le compte-rendu affiche alors `changed=1`, alors qu'il aurait affiché `changed=0` si aucune modification n'avait été nécessaire :

```bash
sudo apt remove --purge apache2
```

```bash
ansible-playbook -i ../inventory lamp.yml
...
TASK [installer apache2] ****************************************
ok: [web-server-2]
changed: [web-server-1]

PLAY RECAP ********************************************************
web-server-1  : ok=2  changed=1  unreachable=0  failed=0
web-server-2  : ok=2  changed=0  unreachable=0  failed=0
```
### Enrichissement du playbook : service, handlers et idempotence sur un dépôt

On peut compléter le playbook pour démarrer et activer automatiquement le service Apache :

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
```

Un dépôt PHP tiers est ensuite ajouté. Certes via le module shell, mais la tâche reste idempotente grâce à l'argument `creates`, qui empêche son exécution si le fichier indiqué existe déjà :

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

Voici le playbook complet, incluant l'installation de PHP et un **handler**. A noter cependant, les paquets installés sont vieux, mais correspondent au contexte de la version des VM. 
Le handler n'est déclenché que si une tâche l'ayant notifié a produit un changement, et il ne s'exécute qu'une seule fois, à la fin du playbook :

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

> L'option `cache_valid_time: 3600` permet de ne déclencher un `apt update` que si le cache des paquets n'a pas été rafraîchi depuis plus d'une heure. Ansible adapte automatiquement les noms de paquets en fonction du système d'exploitation cible, ce qui est très pratique :
### Playbook MySQL

**Remarques (contexte de lab, pas de production)** :

Ici on écrit un Playbook pour installer le role MySQL, avec quelques spécificités : 

- Le mot de passe est volontairement écrit en dur (`1234`), ce qui n'est acceptable que dans un contexte de lab.
- La connexion passe par `login_unix_socket` car MySQL n'accepte plus, par défaut, l'authentification classique par mot de passe. L'utilisation d'une socket locale permet de contourner cette limitation.
- La base de données `test`, créée par défaut lors de l'installation, est supprimée.
- Même si plusieurs tâches notifient le handler `demarrer mysql`, celui-ci ne s'exécute qu'une seule fois, à la fin du playbook.

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
### Ansible Galaxy et passage aux rôles

On a nos playbooks fonctionnels et reproductibles à volonté. 

Cependant, si on veut fusionner l'ensemble des plays dans un seul et même playbook pour déployer à partir d'un seul fichier le rend rapidement illisible et empêche de ne déployer qu'une partie de l'infrastructure, comme uniquement la partie web. 

Un besoin de modularité serait alors appréciable, ce qui justifie le passage à une organisation en ce que l'on appel des rôles.

Un **rôle Ansible (Ansible Role)** est un **ensemble organisé de fichiers** permettant de regrouper toutes les ressources nécessaires à une même fonctionnalité.

Par exemple, un rôle `web` peut contenir :

- les tâches d'installation d'Apache et PHP (`tasks/`) ;
- les handlers pour redémarrer Apache (`handlers/`) ;
- les variables (`vars/`) ;
- les fichiers de configuration (`files/`) ;
- les templates Jinja2 (`templates/`).

Au lieu d'avoir un énorme playbook, on découpe l'infrastructure en **briques réutilisables**.

Les squelettes de rôles sont générés avec `ansible-galaxy` :

```bash
ansible-galaxy init web
- Role web was created successfully
ansible-galaxy init mysql
- Role mysql was created successfully
```

Chaque rôle génère automatiquement l'arborescence adéquate :

```
web/
├── README.md
├── defaults/
├── files/
├── handlers/
├── meta/
├── tasks/
├── templates/
├── tests/
└── vars/
```

Le contenu des playbooks initiaux est ensuite réparti dans les fichiers `main.yml` correspondants, à l'intérieur de chaque rôle :

**`web/tasks/main.yml`**

```yaml
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

**`web/handlers/main.yml`**

```yaml
---
# handlers file for web
- name: restart apache
  service:
    name: apache2
    state: restarted
```

**`mysql/tasks/main.yml`**

```yaml
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

**`mysql/handlers/main.yml`**

```yaml
---
# handlers file for mysql
- name: demarrer mysql
  service:
    name: mysql
    state: restarted
```

**`mysql/vars/main.yml`**

```yaml
---
# vars file for mysql
nom_bdd: cocadmin
```

Un playbook englobant est alors créé pour appeler ces deux rôles :

```yaml
---
- import_playbook: web
- import_playbook: mysql
```

### Erreur rencontrée et correction : dossier `roles/`

En lançant directement ce playbook englobant, l'exécution échoue : Ansible s'attend à trouver un fichier à l'emplacement indiqué, mais tombe sur un dossier :

```bash
ansible-playbook lamp.yaml
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available.
ERROR! an error occurred while trying to read the file '/home/ubuntu/ansible-playbooks/web':
[Errno 21] Is a directory: b'/home/ubuntu/ansible-playbooks/web'
```

Les rôles doivent impérativement se trouver dans un dossier nommé `roles/` pour être correctement résolus par `import_playbook`. Les dossiers de rôles sont déplacés dans un dossier `roles/` créé à cet effet.

```bash
mkdir roles
mv web/ roles/
mv mysql/ roles/
```

Une fois la structure corrigée, l'ensemble de l'infrastructure peut être déployé en une seule commande :

```bash
ansible-playbook -i inventory lamp.yaml

PLAY [web] *********************************************************
TASK [Gathering Facts] *********************************************
ok: [web-server-1]
ok: [web-server-2]

TASK [web : installer apache2] *************************************
ok: [web-server-2]
ok: [web-server-1]

TASK [web : demarrer apache] ****************************************
ok: [web-server-2]
ok: [web-server-1]
[...]

PLAY RECAP **********************************************************
web-server-1  : ok=12  changed=1  unreachable=0  failed=0
web-server-2  : ok=12  changed=1  unreachable=0  failed=0
```

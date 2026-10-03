# 2 - Fleet management, modules, playbooks & roles

#### Comparison with other methods

| Method                              | Limitation                                                                          |
| ----------------------------------- | ----------------------------------------------------------------------------------- |
| SSH into each server one by one     | Unmanageable as soon as the fleet grows                                             |
| MultiSSH                            | A bit better but doesn't scale well (painful from \~100 machines)                   |
| Bash scripts                        | Better, but still not ideal: unhandled use cases, logs tedious to check             |
| **Ansible**                         | Brings **idempotence** (a task replayed with nothing to change, won't changes anything)     |

#### Why Ansible?

The idea behind Ansible is to never be afraid of touching the infrastructure again: afraid of deploying, of running an update, of breaking a production service. So we follow one basic rule: nobody touches a server by hand anymore.

Every change goes through a **role** or a **playbook**, is committed to Git with a message explaining why, and is then applied the same way to every server concerned. This gives us real traceability of what was done.

Environments managed by Ansible are disposable: if a server breaks, it can be rebuilt in minutes from playbooks stored in a GitLab repository, for example.

As for modules, the ideal is to use one dedicated module per specific action. The `shell` module always works, but used carelessly it can break idempotence. So whenever possible, dedicated modules come first.

Finally, no need to reinvent the wheel: **Ansible Galaxy** offers a large set of community roles for most common use cases.

#### Logical structure to remember

```
playbook = liste de plays
play     = contient des rôles
rôle     = contient tasks / handlers / vars / defaults / templates / files / meta
```

To set the default inventory, we create an `ansible.cfg` file:

```bash
cat ansible.cfg
[defaults]
inventory = ./inventory
```

Example inventory with groups and group variables (`web:vars`):

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

We can also target groups with a pattern:

```bash
ansible backup* -i inventory -a "uname -a"
```

Or several groups separated by a comma:

```bash
ansible backup-web,web -i inventory -a "uname -a"
```

Each group can have its own variables: here, `web:vars` defines the variables for the `web` group.&#x20;

**Note: the order in which servers respond depends on which host finishes first, not on the order in which they are declared in the inventory.**

#### Testing modules with ad-hoc commands

There is a very long list of modules for all kinds of tasks:&#x20;

* https://docs.ansible.com/projects/ansible/latest/collections/index\_module.html, including:

The `shell` module to run a command:

```bash
ansible web -m shell -a "uname -a"
web-server-1 | CHANGED | rc=0 >>
Linux web-server-1 5.15.0-185-generic ...
web-server-2 | CHANGED | rc=0 >>
Linux web-server-2 5.15.0-185-generic ...
```

The `apt` module to install a package:

```bash
ansible web -m apt -a "name=nmap"
```

The `script` module to run a local script on remote hosts:

```bash
cat script-test.sh
uname -a
date

ansible web -m script -a "./script-test.sh"
```

> In a real infrastructure, the long-term goal is to gradually replace scripts with dedicated modules, called from roles or playbooks.

#### First YAML playbook

A YAML file should start with three dashes (`---`). Here is an example playbook I called "**lamp.yml**", which installs Apache (for now):

```yaml
---
- hosts: web
  tasks:
    - name: installer apache2
      apt:
        name: apache2
```

When run, it targets **web-server-1** and **2**. "**ok=2**" means both tasks succeeded, and "**changed=0**" means Apache was already installed, so nothing had to be changed:

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

Check on **web-server-1**:

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

**Idempotence in action**: if Apache is removed manually and the playbook is run again, Ansible detects the drift and reinstalls only what is missing:

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

The recap now shows `changed=1` for web-server-1, whereas it would show `changed=0` if nothing had needed changing.

#### Extending the playbook: services, handlers and an idempotent repository

We can extend the playbook to start and enable the Apache service automatically:

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

A third-party PHP repository is then added. It uses the shell module, but the task stays idempotent thanks to the `creates` argument, which skips it if the given file already exists:

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

Here is the full playbook, including the PHP installation and a **handler**. Note that the packages are old, but they match the VM versions used in the course.&#x20;

A handler only runs if a task that notified it actually made a change, and it runs only once, at the end of the playbook:

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

> The `cache_valid_time: 3600` option only triggers an `apt update` if the package cache hasn't been refreshed in the last hour.

#### MySQL playbook

Here we write a playbook that installs MySQL, with a few details:

* The password is deliberately hardcoded (`1234`), for a lab context only, obviously.
* The connection goes through `login_unix_socket` because, by default, MySQL's root account no longer accepts classic password authentication. Connecting through the local socket works around this.
* The `test` database, created by default during installation, is removed.
* Even though several tasks notify the `demarrer mysql` handler, it only runs once, at the end of the playbook.

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

#### Ansible Galaxy and moving to roles

Our playbooks work and can be replayed at will. However, merging everything into a single playbook to deploy from one file quickly makes it unreadable, and prevents deploying only part of the infrastructure, such as the web layer alone. We need modularity, which means moving to what Ansible calls roles.

An **Ansible role** is an **organized set of files** that groups everything needed for one piece of functionality.

For example, a `web` role can contain:

* the tasks that install Apache and PHP (`tasks/`);
* the handlers that restart Apache (`handlers/`);
* variables (`vars/`);
* configuration files (`files/`);
* Jinja2 templates (`templates/`).

Instead of one huge playbook, the infrastructure is split into **reusable building blocks**.

Role skeletons are generated with `ansible-galaxy`:

```bash
ansible-galaxy init web
- Role web was created successfully
ansible-galaxy init mysql
- Role mysql was created successfully
```

Each role automatically gets the appropriate structure:

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

The content of the original playbooks is then split into the matching `main.yml` files inside each role:

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

A top-level playbook is then created to call both roles:

```yaml
---
- import_playbook: web
- import_playbook: mysql
```

#### Error and fix: the `roles/` directory

Running this top-level playbook directly fails: Ansible expects a file at the given path, but finds a directory:

```bash
ansible-playbook lamp.yaml
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available.
ERROR! an error occurred while trying to read the file '/home/ubuntu/ansible-playbooks/web':
[Errno 21] Is a directory: b'/home/ubuntu/ansible-playbooks/web'
```

By default, Ansible looks for roles in a `roles/` directory next to the playbook, so we move them into a dedicated `roles/` folder:

```bash
mkdir roles
mv web/ roles/
mv mysql/ roles/
```

Once the structure is fixed, the whole infrastructure can be deployed with a single command:

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
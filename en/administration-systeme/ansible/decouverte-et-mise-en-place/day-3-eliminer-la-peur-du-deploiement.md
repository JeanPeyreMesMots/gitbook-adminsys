# 3 - Eliminating the fear of deployment

The goal is to be able to deploy with confidence, even on a Friday afternoon, without fear of breaking production.

* **The more changes pile up between two deployments, the harder it becomes to pinpoint the source of a problem** when something goes wrong.
* The longer the delay between developing a feature and shipping it to production, the less responsive the organization becomes. This delay is known as **time-to-market**.
* Some large companies, such as Google with **Gmail**, deploy very frequently, which builds real confidence in the reliability of their infrastructure.
* Ansible makes it possible to handle these deployments in a reliable and reproducible way.

#### Mistakes to avoid when deploying:

<div align="left"><figure><img src="../../../.gitbook/assets/image (31).png" alt="" width="175"><figcaption></figcaption></figure></div>

* Avoid any manual deployments, always opens the door to human error.
* Avoid long gaps between deployments. When you only want to update the load balancer, or only part of the code, you should be able to deploy just that change, without bundling other modifications.
* Avoid deploying at night or on weekends. It's the worst possible time: nobody is around, and problems may only be noticed much later.

#### Rules to follow

<div align="left"><figure><img src="../../../.gitbook/assets/image (34).png" alt="" width="125"><figcaption></figcaption></figure></div>

* Every change or fix is made once and for all in the playbook, which is then redeployed identically everywhere.
* Changes are versioned in Git, for detailed traceability of every modification.
* Gradually increase deployment frequency: for example, go from monthly to weekly, then daily deployments, to make the process smoother.
* Prefer small deployments that can be done quickly. They are easier to validate, or to fix if something goes wrong.
* Split a deployment into several steps if needed, for example updating the database before the application, and adapt the schedule accordingly (one change in the morning, another one in the afternoon).
* Deploy during working hours, when everyone is around to check that everything works, rather than at 4 a.m. or on a weekend.
* Prefer a **blue/green** deployment to avoid any technical downtime: a new infrastructure is deployed with the new version of the application, then traffic is gradually switched over to it.

#### 1. Setting up the application deployment

The goal is to deploy a "todo list" web application with no impact on users.

Creating the `deploy.yml` playbook:

```yaml
---
- hosts: web
  roles:
    - deploy
```

Creating the matching role with `ansible-galaxy`:

```bash
ansible-galaxy init deploy
- Role deploy was created successfully
```

The working directory now looks like this:

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

#### 2. Fetching the application with the `git` module

The example application used in the course is fetched from its GitLab repository (`https://gitlab.com/ttwthomas/app-example-php`) with the `git` module.&#x20;

The task goes in the `main.yml` of the `deploy` role, with a check that clones the repository and imports the database idempotently, without replaying the import for nothing.

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

#### 3. Bug: raw source code instead of the rendered page

The deployment works, but a curl on the test page returns the raw source on both servers:

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

> Note: this behavior (the PHP code is not interpreted and is returned as is) is fixed later by enabling the PHP module in Apache (see section 7).

#### 4. Setting up an HAProxy load balancer

**HAProxy** is used to spread the load efficiently across the two web servers.

The principle: the client sends a request to the load balancer, which distributes the load between the two servers. During a deployment, HAProxy sends traffic to a single server so the service stays available while the other one is being updated.

The server being deployed is removed from the load balancer, and all new requests go to the other server. Once the update is verified, that server is put back into the load balancer, and the other one is removed in turn to be updated. Traffic keeps being served with no downtime.

Rather than writing everything from scratch, we use an existing role from Ansible Galaxy, by Jeff Geerling: [`geerlingguy.haproxy`](https://galaxy.ansible.com/ui/standalone/roles/geerlingguy/haproxy/).

We install it:

```bash
ubuntu@ansible-main:~/ansible-playbooks$ ansible-galaxy role install geerlingguy.haproxy

Starting galaxy role install process
- downloading role 'haproxy', owned by geerlingguy
- downloading role from https://github.com/geerlingguy/ansible-role-haproxy/archive/1.3.2.tar.gz
- extracting geerlingguy.haproxy to /home/ubuntu/.ansible/roles/geerlingguy.haproxy
- geerlingguy.haproxy (1.3.2) was installed successfully
```

> Note: by default, roles installed with Ansible Galaxy go to `/home/USER/.ansible/roles/`:

```bash
~/.ansible/roles$ ll
total 12
drwxrwxr-x 3 ubuntu ubuntu 4096 Jul 27 16:05 ./
drwxrwxr-x 6 ubuntu ubuntu 4096 Jul 27 16:05 ../
drwxrwxr-x 9 ubuntu ubuntu 4096 Jul 27 16:05 geerlingguy.haproxy/
```

We then set `roles_path` in `ansible.cfg`, pointing to a local folder where every role installed from Ansible Galaxy will live:

```ini
[defaults]
host_key_checking = False
retry_files_enabled = False
roles_path = /home/ubuntu/ansible-playbooks/
```

#### 5. Configuring the HAProxy role

The role exposes many default variables, listed in `roles/geerlingguy.haproxy/defaults/main.yml`:

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

A `haproxy.yaml` playbook is created to override these variables with the two real web servers:

```yaml
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

However, one thing bothers us. These variables are injected into the role's Jinja2 template, located in `roles/geerlingguy.haproxy/templates/haproxy.cfg.j2`:

```jinja2
{% endif %}
    cookie SERVERID insert indirect
{% for backend in haproxy_backend_servers %}
```

This line makes the load balancer pin each client to a given server on the first connection and hand it a cookie, so the client keeps hitting the same server afterwards (sticky sessions). That gets annoying here: if a request ends up on another server, the client gets a different cookie, which causes bugs that are hard to reproduce and debug.

So this feature is disabled, hence the "**state: absent**" in the playbook, so that requests can be spread freely across both servers (otherwise, a client would always be sent to the same server).

> Note: the proper approach would be to handle this in the application itself, with shared session storage, so the code doesn't depend on which server handles the request. For lack of time, removing the line with a regex was chosen here.

#### 6. Port conflict between HAProxy and Apache

Running the HAProxy playbook fails:

```bash
TASK [geerlingguy.haproxy : Ensure HAProxy is started and enabled on boot.] ***************
fatal: [web-server-1]: FAILED! => {"changed": false, "msg": "Unable to start service haproxy: Job for haproxy.service failed because the control process exited with error code.\nSee \"systemctl status haproxy.service\" and \"journalctl -xeu haproxy.service\" for details.\n"}
fatal: [web-server-2]: FAILED! => {"changed": false, "msg": "Unable to start service haproxy: Job for haproxy.service failed because the control process exited with error code.\nSee \"systemctl status haproxy.service\" and \"journalctl -xeu haproxy.service\" for details.\n"}
```

The logs reveal the cause:

```bash
journalctl -xeu haproxy.service --no-pager -n 50 | grep "cannot "

Jul 27 17:31:31 web-server-1 haproxy[3387]: [ALERT]    (3387) : Starting frontend hafrontend: cannot bind socket (Address already in use) [0.0.0.0:80]
```

HAProxy and Apache are both trying to listen on port 80 on the same machine, which is impossible. Apache is already running on that port:

```bash
ubuntu@web-server-1:~$ sudo ss -tulnp 
Netid   State     Recv-Q    Send-Q           Local Address:Port        Peer Address:Port   Process                                                       
tcp     LISTEN    0         511                          *:80                     *:*       users:(("apache2",pid=666,fd=4),("apache2",pid=665,fd=4),("apache2",pid=663,fd=4))        
tcp     LISTEN    0         128                       [::]:22                  [::]:*       users:(("sshd",pid=656,fd=4))            
```

We could make HAProxy listen on another port (for example 8080), or change the web server's port. But it makes no sense to put HAProxy in front on the same machine as the services it balances. So we create a dedicated VM for HAProxy, which is simpler and cleaner.

A new VM is created with Multipass, just like the previous ones:

```bash
multipass launch 22.04 -n lb-server -c 1 -m 3G
```

We then repeat the same setup as in step 1: allowing root SSH login, configuring Ansible access, adding it to `/etc/hosts`, adding the VM name to the DHCP sync cron script, etc.

The inventory is then updated with a common `all_servers` group, which avoids connection problems with the new VM:

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

#### 7. Fixing the web role

Rereading the `main.yml` of the web role, I noticed several issues:

* using `add-apt-repository` instead of the dedicated `apt_repository` module;
* no `apt update` before installing packages;
* an inconsistent mix of PHP 7.2 and 7.3 packages;
* the PHP module was never enabled in Apache by the playbook, which is required in our case.

The role is fixed to be more idempotent and to align package versions:

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

This also fixes the bug seen earlier: the PHP code is now interpreted by Apache instead of being sent as is to the browser :)

#### 8. Final load balancer deployment

The HAProxy playbook now runs successfully:

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

The application now works on both servers, with traffic properly distributed by the load balancer.

<figure><img src="../../../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

The content shown differs from one server to the other on purpose, since each one hosts its own local MySQL database. This could be unified by moving the database to a separate VM reachable by both web servers, or by setting up replication between the two databases.
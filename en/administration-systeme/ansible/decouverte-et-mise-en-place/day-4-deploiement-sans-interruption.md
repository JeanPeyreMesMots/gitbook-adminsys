# 4 - Zero-downtime deployment

We now move on to a truly zero-downtime deployment: each web server is removed from the load balancer, updated, fixed and verified, then put back into the load balancer, one server at a time.

We use the `community.general.haproxy` module, which drives HAProxy through its API, following the order of tasks in the playbook.

#### How it works

The idea is obviously never to update all servers at once. As described before, one server is removed from the load balancer, updated and verified while the other keeps serving all the traffic. It is then put back before moving on to the next server.

What helps us here is that HAProxy exposes an interface to dynamically add or remove a server from its pool without restarting the service. Ansible drives this interface through a dedicated module rather than a manual command, which is a big plus.

We follow this logical order for the whole workflow:

```
cloning the repo → 
configuration fixing (DB_HOST) → 
(if needed) restauration of DB → 
checking the page → 
reintegration into the LB
```

#### 1. First version: removing the server before the update

We start by removing a server from the load balancer with the `community.general.haproxy` module, added to the `main.yml` of the `deploy` role:

```yaml
- name: retirer serveur web du load balancer
  community.general.haproxy:
    state: disabled
    host: '{{ inventory_hostname }}'
    backend: habackend
    socket: /var/lib/haproxy/stats
    fail_on_not_found: yes
    shutdown_sessions: yes
  # hôte auquel la tâche est déléguée (le load balancer)
  delegate_to: ansible_lb_1

- name: cloner le repo git
  git:
    repo: 'https://gitlab.com/ttwthomas/app-example-php.git'
    dest: /var/www/html/app
    force: yes
  # variable qui capture le résultat du clone git
  register: git_repo

- name: vérifier que la page est up
  ansible.builtin.uri:
    # toujours accessible via le load balancer, grâce à une variable dynamique
    url: http://localhost/app/index.php
    return_content: true
  register: this
  failed_when: "ansible_hostname not in this.content"
```

A few details:

* The backend name, `habackend`, matches the name set earlier in the HAProxy configuration. It's the role's default, which I kept.
* In the `failed_when` condition, `ansible_hostname` is the hostname of the server the playbook is currently running on.
* `delegate_to` targets the load balancer, since that's the host that must receive the command to enable/disable the backend server.

To make sure only one server at a time is updated (and therefore unavailable), `serial: 1` is added to the deployment playbook:

```yaml
---
- hosts: web
  serial: 1
  roles:
    - deploy
```

#### 2. Fixing the task order

On review, a problem stands out: the task fixing `DB_HOST` was placed too late in the playbook, after the page check. As a result, the check always failed, since the configuration hadn't been fixed yet at that point.

The corrected playbook moves the `DB_HOST` fix right after the repository clone, and before the page check: 

```yaml
---
- name: installer dependences
  apt:
    name: [git, curl, python3-pymysql, mysql-client]
    state: present

- name: retirer serveur web du load balancer
  community.general.haproxy:
    state: disabled
    host: '{{ inventory_hostname }}'
    backend: habackend
    socket: /var/lib/haproxy/stats
    fail_on_not_found: yes
    shutdown_sessions: yes
  delegate_to: lb-server

- pause:
    seconds: 20

- name: cloner le repo git
  git:
    repo: 'https://gitlab.com/ttwthomas/app-example-php.git'
    dest: /var/www/html/app
    force: yes
  register: git_repo

- name: corriger le DB_HOST dans le fichier de config
  replace:
    path: /var/www/html/app/index.php
    regexp: "define\\('DB_HOST', '.*'\\);"
    replace: "define('DB_HOST', 'localhost');"

- name: restaurer la base de donnees
  mysql_db:
    name: cocadmin
    state: import
    target: /var/www/html/app/creer_table.sql
    login_user: cocadmin
    login_password: 1234
    login_unix_socket: /var/run/mysqld/mysqld.sock
  when: git_repo.changed

- name: vérifier que la page est up
  ansible.builtin.uri:
    url: http://localhost/app/index.php
    return_content: true
  register: this
  failed_when: "ansible_hostname not in this.content"

- name: remettre serveur web du load balancer
  community.general.haproxy:
    state: enabled
    host: '{{ inventory_hostname }}'
    backend: habackend
    socket: /var/lib/haproxy/stats
    fail_on_not_found: yes
    shutdown_sessions: yes
  delegate_to: lb-server
```

> A 20-second pause is added after removing the server from the load balancer, to let in-flight connections finish cleanly before continuing.

When running the playbook, we can see that `web-server-1` becomes temporarily unavailable while `web-server-2` keeps responding normally:

```bash
TASK [deploy : retirer serveur web du load balancer] **************************************
changed: [web-server-1 -> lb-server]

TASK [deploy : pause] *********************************************************************
Pausing for 20 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
```

From any machine, even the load balancer itself, we can see that `web-server-2` takes over while `web-server-1` is being updated:

```bash
ubuntu@lb-server:~$ for i in {1..10}; do curl -s http://lb-server/app/index.php | grep -oP '(?<=TODO )[^<]+'; done
App 
web-server-2
entry #1
entry #2
entry #3
App 
web-server-2
entry #1
entry #2
entry #3
App 
web-server-2
entry #1
entry #2
entry #3
```

#### 3. Final result

Once the playbook has fully run and both servers are back up, a new series of requests shows that the two servers correctly alternate in round robin, each answering in turn:

```bash
ubuntu@web-server-1:~$ for i in {1..10}; do curl -s http://lb-server/app/index.php | grep -oP '(?<=TODO )[^<]+'; done
App 
web-server-1
entry #1
entry #2
entry #3
App 
web-server-2
entry #1
entry #2
entry #3
App 
web-server-1
entry #1
entry #2
entry #3
App 
web-server-2
entry #1
entry #2
entry #3
```

### Conclusion

No downtime was observed, not even during the deployment itself. It is now possible to deploy in the middle of the day, calmly, with no service interruption for users.

Good to know: in a real infrastructure, modules exist to automatically notify the end of a deployment, for example by sending an e-mail or a Slack message once the deployment has completed successfully.

Before going further, it's also worth checking Ansible Galaxy for existing roles that could simplify managing the HAProxy configuration even more.
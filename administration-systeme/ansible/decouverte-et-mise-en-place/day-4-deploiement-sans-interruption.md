# 4 - Déploiement sans interruption

Nous passons désormais à la mise en place d'un déploiement réellement sans interruption de service : chaque serveur web est retiré du load balancer avant d'être mis à jour, corrigé, vérifié, puis remis dans le load balancer, sur un serveur à la fois.

On utilisera le module `community.general.haproxy` piloté via son API, en suivant l'ordre des tâches dans le playbook.

#### Principe

L'idée est évidemment de ne jamais mettre à jour tous les serveurs en même temps. Comme on l'a spécifié avant, l'un est retiré du load balancer, mis à jour et vérifié, pendant que l'autre continue de servir tout le trafic. Puis il est réintégré avant de passer au serveur suivant.

Ce qui va nous aider, c'est que HAProxy expose une interface permettant d'ajouter ou de retirer dynamiquement un serveur de son pool, sans redémarrer le service. Ansible pilote cette interface via un module dédié plutôt que par une commande manuelle, ce qui est ici un gros point fort.

On suivra l'ordre logique correct pour l'ensemble du workflow :

```
clone du dépôt → 
correction de la configuration (DB_HOST) → 
(si besoin) restauration de la BDD → 
vérification de la page → 
réintégration dans le load balancer
```

#### 1. Première version : retirer le serveur avant la mise à jour

On commence par retirer un serveur du load balancer via le module `community.general.haproxy`, rajouté dans le `main.yml` du rôle `deploy` :

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

Quelques précisions :

* Le nom du backend, `habackend`, correspond au nom choisi précédemment dans la configuration HAProxy. Il s'agit de celui par défaut, que j'ai laissé.
* Pour la condition `failed_when`, `ansible_hostname` désigne le hostname du serveur courant sur lequel se lance le playbook
* `delegate_to` cible bien le load balancer, puisque c'est cet hôte qui doit recevoir la commande d'activation/désactivation du backend.

Pour garantir qu'un seul serveur à la fois est mis à jour (et donc indisponible), l'option `serial: 1` est ajoutée au playbook de déploiement :

```yaml
---
- hosts: web
  serial: 1
  roles:
    - deploy
```

#### 2. Correction de l'ordre des tâches

Après relecture, un problème est identifié : la tâche de correction du `DB_HOST` était placée trop bas dans le playbook. Après la vérification de la page, ce qui faisait échouer systématiquement cette vérification puisque la configuration n'était pas encore corrigée à ce stade.

Le playbook corrigé replace la correction du `DB_HOST` juste après le clone du dépôt, et avant la vérification de la page :

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

> Une pause de 20 secondes est ajoutée après le retrait du serveur du load balancer, afin de laisser le temps aux connexions en cours de se terminer proprement avant de poursuivre.

Lors du lancement du playbook, on observe bien que le serveur `web-server-1` devient temporairement indisponible pendant que `web-server-2` continue de répondre normalement :

```bash
TASK [deploy : retirer serveur web du load balancer] **************************************
changed: [web-server-1 -> lb-server]

TASK [deploy : pause] *********************************************************************
Pausing for 20 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
```

Depuis n'importe quelle machine, voir le load balancer lui-même, on constate bien que c'est `web-server-2` qui prend le relais pendant que `web-server-1` est mis à jour :

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

#### 3. Résultat final

Une fois que le playbook est entièrement finalisé et que les serveurs sont sur roues, une nouvelle série de requêtes montre que les deux serveurs alternent correctement en round robin, chacun répondant à tour de rôle :

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

Aucun downtime n'a été constaté, y compris pendant le déploiement lui-même. Il est désormais possible de déployer en pleine journée, sereinement, sans interruption de service pour les utilisateurs.

À savoir quand dans le contexte d'une infra réelle, des modules existent pour notifier automatiquement la fin d'un déploiement. Par exemple un module d'envoi d'e-mail ou un module d'envoi de message Slack une fois le déploiement terminé avec succès.

Il est recommandé de vérifier avant de déployer sur Ansible Galaxy si des rôles existants permettent de simplifier davantage la gestion de la configuration HAProxy.

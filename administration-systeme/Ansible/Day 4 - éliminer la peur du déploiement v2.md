Passons maintenant au déploiement sans interruption : retirer un des serveurs du LB, maj l'autre, verif que tout est ok, le remettre dans le LB, puis check si tout est ok

Manière de faire ça : HAproxy goit api vers send command to add/remove server. on va use un module. 

Dans le main.yml de notre role deploy, on va donc ajouter ce module pour remove server before upgrading :

```c
- name: retirer serveur web du load balancer
  community.general.haproxy:
    state: disabled
    host: '{{ inventory_hostname }}'
    backend: habackend
    socket: /var/lib/haproxy/stats
    fail_on_not_found: yes
    shutdown_sessions: yes
  # à quel serveur déleguer
  delegate_to: ansible_lb_1

- name: cloner le repo git
  git:
    repo: 'https://gitlab.com/ttwthomas/app-example-php.git'
    dest: /var/www/html/app
    force: yes
  # var pour le repo git
  register: git_repo

- name: vérifier que la page est up
  ansible.builtin.uri:
    # on est toujours dans le load balancer car variable dynamique
    url: http://localhost/app/index.php
    return_content: true
  register: this
  failed_when: "ansible.hostname not in this.content"
```

"inventory_hostname" -> https://docs.ansible.com/projects/ansible/latest/reference_appendices/special_variables.html#term-inventory_hostname hostname sur lequel on est en train d'itérer. pratique car hote sur lequel on fait le changement. pour le nom du backend -> "habackend" nom que j'ai choisi

pour failed when : vu que j'ai que un seul serveur de bdd, je choisi de garder ansible.hostname

delegate to : pour le lb 

deploiement un par un pour chaque serveur :

Puis dans le deploy, on rajoute un serial : 1 pour dire que l'on veut faire un déploiement sur un serveur à la fois :

```c
---
- hosts: web
	serial: 1
	roles:
	- deploy
```

Enfin, on remarque un problème : "- name: corriger le DB_HOST dans le fichier de config" et **sa position dans le playbook**. Actuellement tu vérifies que la page fonctionne (`vérifier que la page est up`) **avant** d'avoir corrigé le `DB_HOST` — donc le check échoue systématiquement, avant même d'arriver à ta tâche de correction placée tout en bas.

Il faut corriger le fichier **juste après le clone**, et **avant** de vérifier que la page répond :

```c
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

l'ordre du jour suit alors la logique suivante : 

```
clone → corriger config → (si besoin) restaurer BDD → vérifier la page → remettre dans le LB
```

On peut alors lancer et constater lors du lancement que le serveur web-1 n'est plus accessible alors que le 2eme si :

```bash
TASK [deploy : retirer serveur web du load balancer] **************************************
changed: [web-server-1 -> lb-server]

TASK [deploy : pause] *********************************************************************
Pausing for 20 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
```

Ainsi depuis n'imp quel machine ou depuis le LB lui meme, on peut voir que le serv web 2 prend le relais :

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

Une fois le playbook finalisé, si on test cette fois depuis un serv web, on peut voir alterner les hostnames de chaques serveur à chaque fois en round robin :

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

On a pas eu de downtime meme durant le déploiement :D On peut désormais faire des déploiements dans la journée, dans le calme, sans interruptions.

Il existe pleins de modules : 
- mail pour envoyer un mail qd deploiement se finit
- slack, pour envoyer un message slack quand tout s'est bien passé

Checker dans ansible galaxy si des roles peuvent nous aider, pour changer la conf d'haproxy

Extrait de la page de cocadmin :

# Déployer une nouvelle version de notre application avec Ansible.

- Créer un rôle de déploiement (git module/ web download / untar)
    
- Register, conditions et run_once
    
- Ajouter haproxy role depuis galaxy
    

ansible-galaxy install geerlingguy.haproxy

  
Le rôle est dans _~/.ansible/roles_

On peu ajouter la variable **role_path** dans _/etc/ansible/ansible.cfg_

On retire la persistence du LB (cookie SERVERID)  
  

- Tester le bon déploiement (wait_for page de status ou homepage, test unitaire etc)
    
- Mettre à jour le haproxy dans le rôle de déploiement (delegate_to)
    
- Déployer la nouvelle version un par un (serial deploy)
    
- Vérifier que la page est up.
    
- Annoncer la bonne nouvelle (post_task email/slack)
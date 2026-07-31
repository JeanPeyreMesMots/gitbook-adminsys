## Bekasi
**Point de départ :** impossible de redémarrer nginx ou de vérifier sa conf normalement.

```bash
nginx -t
# -bash: nginx: command not found
systemctl stop nginx
# Failed to stop nginx.service: Access denied
systemctl start nginx
# Failed to start nginx.service: Access denied
```

Pas de binaire `nginx` en accès direct, et `systemctl` refuse même de le stopper sans sudo — réflexe à corriger côté commandes utilisées, mais le service tourne déjà :

```bash
systemctl status nginx.service
# Active: active (running) since Tue 2026-03-24 17:38:55 UTC
# ...
# Failed to parse PID from file /run/nginx.pid: Invalid argument
```

Une erreur apparaît dans les logs systemd malgré le service "actif" — mise de côté pour plus tard (`vamos a ver lo mas tarde`), pas bloquant dans l'immédiat.

### Explorer la conf nginx

Conf principale banale, rien d'anormal. Le vhost par défaut est plus intéressant :

```nginx
server {
    server_name bekasi;
    listen 443 ssl default_server;
    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    location /static {
        autoindex on;
        alias /srv/www/assets;
    }

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/admin/bekasi/bekasi.sock;
    }
}
```

Deux pistes à vérifier : le dossier `/srv/www/assets` référencé pour le statique, et le socket uWSGI `/home/admin/bekasi/bekasi.sock` pour le reste.

Check du dossier statique :

```bash
ll /srv/
# total 8.0K, rien dedans à part . et ..
```

Vide — pas normal, mais pas le blocage principal (l'appli dynamique passe par uWSGI, pas par ce dossier).

### Direction supervisor

La solution mentionnait `supervisorctl` — outil que je ne connaissais pas avant ce chall ([doc utilisée](http://blog.stephane-robert.info/docs/services/processus/supervisor/)), un gestionnaire de process qui peut superviser et relancer des applis (ici l'appli Python/uWSGI).

```bash
cat /etc/supervisor/supervisord.conf
# files = /etc/supervisor/conf.d/*.conf
```

Vérification des logs supervisor :

```bash
cat /var/log/supervisor/supervisord.log
# CRIT Supervisor is running as root...
# WARN No file matches via include "/etc/supervisor/conf.d/*.conf"   (avant reconfig)
# INFO Included extra file "/etc/supervisor/conf.d/uwsgi.conf"
# INFO spawned: 'bekasi' with pid 8938
# INFO success: bekasi entered RUNNING state
# INFO stopped: bekasi (exit status 0)
```

Le process `bekasi` démarre, tourne un peu, puis s'arrête — comportement instable plutôt qu'un crash immédiat.
### Lancer uWSGI à la main pour isoler le problème

Test du binaire directement, sans passer par supervisor :

```bash
./uwsgi
# The -s/--socket option is missing and stdin is not a socket.
```

Besoin explicite de préciser le socket :

```bash
./uwsgi -s ../bekasi.sock
# uwsgi socket 0 bound to UNIX address ../bekasi.sock fd 3
# *** no app loaded. going in full dynamic mode ***
# spawned uWSGI worker 1 (and the only) (pid: 1590, cores: 1)
```

Ça tourne. Passage en arrière-plan pour tester en parallèle :

```bash
# Ctrl+Z puis :
bg
curl -k https://bekasi
# 502 Bad Gateway
```

Toujours 502 malgré uWSGI apparemment lancé — donc soit nginx ne pointe pas où il faut, soit uWSGI ne répond pas correctement à ce socket précis.

### Comparer lancement manuel vs supervisor

Via `supervisorctl`, le process tourne aussi et log un comportement proche, mais avec plus de détails (mode preforking, plusieurs workers) :

```bash
sudo supervisorctl
# bekasi   RUNNING   pid 1210, uptime 0:20:21
supervisor> tail -f bekasi
# *** Operational MODE: preforking ***
# WSGI app 0 (mountpoint='') ready in 0 seconds ...
# spawned uWSGI master process (pid: 1210)
# spawned uWSGI worker 1..5
```

En comparant la sortie du lancement manuel et celle via supervisor, différence notable dans les toutes premières lignes du lancement manuel :

```text
!!! no internal routing support, rebuild with pcre support !!!
*** WARNING: you are running uWSGI without its master process manager ***
```

Ces lignes n'apparaissent pas dans la version supervisor — indice que l'environnement d'exécution diffère entre les deux lancements, pas juste la présence d'un master process.

### La vraie cause : variables d'environnement manquantes

La piste de solution suggérait de vérifier `~/.bashrc` :

```bash
tail -f ~/.bashrc
export BEKASI_SERVER=bekasi.sadservers.com
export BEKASI_USER=admin
```

Deux variables d'environnement définies uniquement dans le shell interactif de l'utilisateur — donc présentes quand on lance `uwsgi` à la main depuis ce shell, mais absentes du contexte dans lequel `supervisord` lance ses process (qui ne source pas `~/.bashrc`).

Vérification de la conf supervisor existante :

```bash
cat /etc/supervisor/conf.d/uwsgi.conf
[program:bekasi]
autorestart=true
command=/home/admin/bekasi/bin/uwsgi --ini /home/admin/bekasi/bekasi.ini
directory=/home/admin/bekasi
redirect_stderr=true
stdout_logfile=/var/log/bekasi.log
user=admin
```

Aucune variable d'environnement définie ici. Ajout de la ligne `environment` avec les deux variables nécessaires :

```ini
[program:bekasi]
autorestart=true
command=/home/admin/bekasi/bin/uwsgi --ini /home/admin/bekasi/bekasi.ini
directory=/home/admin/bekasi
redirect_stderr=true
stdout_logfile=/var/log/bekasi.log
user=admin
environment=BEKASI_SERVER="bekasi.sadservers.com",BEKASI_USER="admin"
```

### Rechargement et validation

```bash
sudo supervisorctl
supervisor> reread
# bekasi: changed
supervisor> update
# bekasi: stopped
# bekasi: updated process group
```

Test final :

```bash
curl -k https://bekasi
# Hello SadServers!
```

Résolu.
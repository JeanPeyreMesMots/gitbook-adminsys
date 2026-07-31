## Kortenberg
**Le problème :** impossible de créer quoi que ce soit dans le home, ni fichier ni dossier accessible.

```bash
mkdir test/
ll
# d--------- 2 admin admin 4.0K Mar 11 10:07 test
```

Le dossier se crée mais avec zéro permission. Confirmation :

```bash
ll test/
# ls: cannot open directory 'test/': Permission denied
touch file && echo "hey" > file
# -bash: file: Permission denied
```

Même un fichier tout frais est injoignable :

```bash
ll
# ---------- 1 admin admin 0 Mar 11 10:07 file
```

Réflexe : check l'umask.

```bash
umask
# 0777
```

Là, tout s'explique — un umask à 0777 retire _toutes_ les permissions à chaque nouveau fichier/dossier créé.

### Comprendre l'umask

Contrairement à Windows, Linux n'hérite pas des permissions du répertoire parent : les permissions des nouveaux fichiers sont déterminées par l'umask, qui **retire** des permissions par rapport au maximum possible.

```bash
umask
# 0022 (valeur typique)

# Calcul :
# Fichiers   : 666 - 022 = 644 (rw-r--r--)
# Répertoires: 777 - 022 = 755 (rwxr-xr-x)
```

|umask|Fichiers|Répertoires|Usage|
|---|---|---|---|
|022|644|755|Standard|
|027|640|750|Plus restrictif|
|077|600|700|Privé|
|002|664|775|Collaboratif|

### Correction

On veut du 755 pour les dossiers → umask 022. Test d'abord en session courante :

```bash
umask 022
mkdir test4/ && touch test4/test.txt
ll
# drwxr-xr-x 2 admin admin 4.0K Mar 11 10:12 test4
echo "hey" > test4/test.txt
cat test4/test.txt
# hey
```

Ça fonctionne. Reste à corriger la source du problème pour que ce soit permanent — check dans `/etc/profile` :

```bash
cat /etc/profile | grep "umask"
# umask 777
```

Voilà le coupable, réglé en dur à 777 pour toutes les sessions. Correction :

```bash
sudo sed -i 's/^umask[[:space:]]\+[0-7]\{3\}/umask 022/' /etc/profile
source .bashrc
```

Résolu.

## Manhattan
**Le problème :** PostgreSQL ne se connecte plus, locales qui semblent cassées au passage.

```bash
sudo -u postgres psql
# perl: warning: Setting locale failed.
# ...
# psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

### Fausse piste : les locales

Le warning sur les locales m'a fait partir sur cette piste en premier :

```bash
locale-gen fr_FR.UTF-8
# -bash: locale-gen: command not found
sudo locale-gen fr_FR.UTF-8
# Generating locales... Generation complete.
sudo dpkg-reconfigure locales
# fr_FR.UTF-8... done
```

Locales régénérées, mais le problème persiste identique :

```bash
sudo -u postgres psql
# psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

Donc ce n'était pas ça — juste un warning cosmétique, pas la vraie cause.

### La vraie piste

Recherche Google sur l'erreur de socket, tombé sur un article mentionnant qu'il rencontrait ce problème en restaurant une base de prod trop lourde, faute d'espace disque suffisant. Le mot "espace" m'a fait tilt, d'autant que le challenge portait le tag "disk volumes". Vérification immédiate :

```bash
df -h
# /dev/nvme0n1     8.0G  8.0G   28K 100% /opt/pgdata
```

Le volume `/opt/pgdata` est plein à 100%. Voilà la vraie cause.

En regardant dedans, un fichier prend quasi toute la place :

```bash
ll
# -rw-r--r--  1 root  root  7.0G May 21  2022 file1.bk
# -rw-r--r--  1 root  root  923M May 21  2022 file2.bk
# -rw-r--r--  1 root  root  488K May 21  2022 file3.bk
```

Suppression du plus gros :

```bash
sudo rm file1.bk
df -h
# /dev/nvme0n1     8.0G 1014M  7.1G  13% /opt/pgdata
```

Espace libéré. Mais l'erreur persiste malgré tout à ce stade :

```bash
sudo psql -U postgres
# psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

### Redémarrer le cluster proprement

Plutôt que de passer par `systemctl`, direction les outils PostgreSQL dédiés — lister le cluster puis consulter son log pour confirmer la cause :

```bash
pg_lsclusters
# 14  main  5432 down  postgres /opt/pgdata/main ...

cat /var/log/postgresql/postgresql-14-main.log
# FATAL: could not create lock file "postmaster.pid": No space left on device
```

Confirmation que PostgreSQL s'est arrêté à cause du manque d'espace disque (avant le nettoyage), il fallait donc le relancer explicitement :

```bash
sudo pg_ctlcluster 14 main restart
```

Vérification finale :

```bash
sudo -u postgres psql
# \l  →  liste des bases OK
sudo -u postgres psql -c "insert into persons(name) values ('jane smith');" -d dt
# INSERT 0 1
```

Résolu.
## Cape Town
**Le problème :** nginx down avec une erreur de syntaxe, puis une deuxième panne différente une fois la première réglée.

```bash
curl -I 127.0.0.1:80
# curl: (7) Failed to connect to 127.0.0.1 port 80: Connection refused

sudo systemctl status nginx
# Active: failed (Result: exit-code)
# nginx[573]: nginx: [emerg] unexpected ";" in /etc/nginx/sites-enabled/default:1
```

### Round 1 : erreur de syntaxe

Le fichier `/etc/nginx/sites-enabled/default` contient une erreur de syntaxe en première ligne :

```bash
head /etc/nginx/sites-enabled/default
# ; -> NONE
```

Un point-virgule orphelin en tout début de fichier. Correction du fichier, puis :

```bash
sudo systemctl restart nginx
sudo systemctl status nginx
# Active: active (running)
```

Ça semble reparti...

### Round 2 : mais 500 Internal Server Error

```bash
curl -I 127.0.0.1:80
# HTTP/1.1 500 Internal Server Error
```

Toujours cassé, autrement. Logs :

```bash
cat /var/log/nginx/error.log
# ... unexpected ";" (anciennes entrées, déjà corrigé)
# [alert] socketpair() failed while spawning "worker process" (24: Too many open files)
# [emerg] eventfd() failed (24: Too many open files)
# [crit] open() "/var/www/html/index.nginx-debian.html" failed (24: Too many open files)
```

Nouveau problème complètement différent : limite de file descriptors atteinte.

Recherche rapide qui confirme la piste — nginx qui manque de file descriptors, avec une méthode de vérification via `ulimit` sur l'utilisateur du service :

```bash
sudo su - www-data
# This account is currently not available.
```

Le compte `www-data` n'a pas de shell de login. Contournement :

```bash
sudo runuser -u www-data -- bash
```

Ça marche, on peut checker les limites :

```bash
ulimit -Hn
# 1048576
ulimit -Sn
# 1024
```

La limite soft (1024) est effectivement basse pour un serveur web.

### Correction : augmenter la limite via systemd

Deux méthodes possibles selon l'init system (avec ou sans systemd) — ici systemd, donc méthode native :

```bash
sudo systemctl edit nginx
# Ajout de :
# LimitNOFILE=65535
```

```bash
sudo systemctl daemon-reload
sudo nginx -t
# syntax ok, test successful
sudo systemctl restart nginx
sudo systemctl status nginx
# Active: active (running)
```

Complément : ajuster aussi la limite côté worker nginx directement dans la conf, pour que le process lui-même puisse ouvrir davantage de descripteurs (choix de 30000) :

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
worker_rlimit_nofile 30000;
include /etc/nginx/modules-enabled/*.conf;
```

Redémarrage final :

```bash
sudo systemctl restart nginx
curl -I 127.0.0.1:80
# HTTP/1.1 200 OK
```

Résolu, deux pannes différentes réglées l'une après l'autre.
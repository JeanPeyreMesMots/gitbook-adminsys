# 3 - Kortenberg, Manhattan & Cape Town

### Kortenberg

**Le problème :** impossible de créer quoi que ce soit dans le home, ni fichier ni dossier accessible.

```bash
mkdir test/
ll
# d--------- 2 admin admin 4.0K Mar 11 10:07 test
```

Le dossier se crée mais avec zéro permission :

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

umask serait le coupable ?

```bash
umask
# 0777
```

Anéfé : un umask à 0777 retire _toutes_ les permissions à chaque nouveau fichier/dossier créé.

Histoire de comprende l'umask : contrairement à Windows, Linux n'hérite pas des permissions du répertoire parent. Les permissions des nouveaux fichiers sont déterminées par l'umask, qui **retire** des permissions par rapport au maximum possible.

```bash
umask
# 0022 (valeur typique)

# Calcul :
# Fichiers   : 666 - 022 = 644 (rw-r--r--)
# Répertoires: 777 - 022 = 755 (rwxr-xr-x)
```

| umask | Fichiers | Répertoires | Usage           |
| ----- | -------- | ----------- | --------------- |
| 022   | 644      | 755         | Standard        |
| 027   | 640      | 750         | Plus restrictif |
| 077   | 600      | 700         | Privé           |
| 002   | 664      | 775         | Collaboratif    |

Nous ce qu'on veut c'est du 755 pour les dossiers, donc un umask en 022. On test ça d'abord dans la session courante :

```bash
umask 022
mkdir test4/ && touch test4/test.txt
ll
# drwxr-xr-x 2 admin admin 4.0K Mar 11 10:12 test4
echo "hey" > test4/test.txt
cat test4/test.txt
# hey
```

Super ça fonctionne. Reste à appliquer au niveau système wide dans `/etc/profile` :

```bash
cat /etc/profile | grep "umask"
# umask 777
```

Voilà le coupable, réglé en dur à 777 pour toutes les sessions :

```bash
sudo sed -i 's/^umask[[:space:]]\+[0-7]\{3\}/umask 022/' /etc/profile
source .bashrc
```

Résolu.

### Manhattan

**Le problème :** PostgreSQL ne se connecte plus, locales qui semblent cassées au passage.

```bash
sudo -u postgres psql
# perl: warning: Setting locale failed.
# ...
# psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

#### Fausse piste : les locales

Le warning sur les locales m'a fait partir sur cette piste en premier :

```bash
locale-gen fr_FR.UTF-8
# -bash: locale-gen: command not found
sudo locale-gen fr_FR.UTF-8
# Generating locales... Generation complete.
sudo dpkg-reconfigure locales
# fr_FR.UTF-8... done
```

Je régénére les locales, mais le problème persiste :

```bash
sudo -u postgres psql
# psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

#### La vraie piste

J'ai fait une petite recherche Google sur l'erreur de socket, puis suis tombé sur un article mentionnant qu'il rencontrait ce problème en restaurant une base de prod trop lourde, faute d'espace disque suffisant. Le mot "**espace**" justement m'a fait tilt, d'autant que le challenge portait le tag "**disk volumes**". On prend le poul de l'espace :

```bash
df -h
# /dev/nvme0n1     8.0G  8.0G   28K 100% /opt/pgdata
```

Et voilà, comme dirait Fred de Carglass. Le volume `/opt/pgdata` est plein à 100%. Dedans un fichier prend quasi toute la place :

```bash
ll
# -rw-r--r--  1 root  root  7.0G May 21  2022 file1.bk
# -rw-r--r--  1 root  root  923M May 21  2022 file2.bk
# -rw-r--r--  1 root  root  488K May 21  2022 file3.bk
```

On supprime le plus gros :

```bash
sudo rm file1.bk
df -h
# /dev/nvme0n1     8.0G 1014M  7.1G  13% /opt/pgdata
```

Ca libère de l'espace mais... :

```bash
sudo psql -U postgres
# psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

Une solution : redémarrer le cluster proprement. Sauf que plutôt que de passer par `systemctl` pour le faire, mieux vaut utiliser les outils PostgreSQL dédiés. En l'occurence "**pg\_lsclusters**" permet de lister le cluster, nous donnant l'occasion de consulter son log pour en confirmer la cause :

```bash
pg_lsclusters
# 14  main  5432 down  postgres /opt/pgdata/main ...

cat /var/log/postgresql/postgresql-14-main.log
# FATAL: could not create lock file "postmaster.pid": No space left on device
```

Ça nous confirme que PostgreSQL s'est arrêté à cause du manque d'espace disque (avant le nettoyage). Or on a déjà libéré, on va donc le relancer avec l'outil de Postgres "**pg\_ctlcluster**" :

```bash
sudo pg_ctlcluster 14 main restart
```

Check finale :

```bash
sudo -u postgres psql
# \l  →  liste des bases OK
sudo -u postgres psql -c "insert into persons(name) values ('jane smith');" -d dt
# INSERT 0 1
```

Résolu.

### Cape Town

**Le problème :** nginx down avec une erreur de syntaxe, puis une deuxième panne différente une fois la première réglée.

```bash
curl -I 127.0.0.1:80
# curl: (7) Failed to connect to 127.0.0.1 port 80: Connection refused

sudo systemctl status nginx
# Active: failed (Result: exit-code)
# nginx[573]: nginx: [emerg] unexpected ";" in /etc/nginx/sites-enabled/default:1
```

Le fichier `/etc/nginx/sites-enabled/default` contient une erreur de syntaxe en première ligne :

```bash
head /etc/nginx/sites-enabled/default
# ; -> NONE
```

Un point-virgule laissé en tout début de fichier. On corrige puis on redémarre :

```bash
sudo systemctl restart nginx
sudo systemctl status nginx
# Active: active (running)
```

Ça semble reparti mais on se prend on 500 :

```bash
curl -I 127.0.0.1:80
# HTTP/1.1 500 Internal Server Error
```

Les logs illicos :

```bash
cat /var/log/nginx/error.log
# ... unexpected ";" (anciennes entrées, déjà corrigé)
# [alert] socketpair() failed while spawning "worker process" (24: Too many open files)
# [emerg] eventfd() failed (24: Too many open files)
# [crit] open() "/var/www/html/index.nginx-debian.html" failed (24: Too many open files)
```

Arf, une limite de file descriptors atteinte, car si on se log sur le user de service pour nginx, c'est pas permis :

```bash
sudo su - www-data
# This account is currently not available.
```

Le compte `www-data` n'a pas de shell de login. On peut néamoins spawn un shell avec le binaire en sudo :

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

1024, c'est effectivement bas pour un serveur web.

Pour augmenter la limite, on peut passer par systemd, je préfère utiliser les outils natifs :

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

Puis on ajuste la limite côté worker directement dans la conf nginx, pour que le process lui-même puisse ouvrir davantage de descripteurs. Ici, on met le choix à 30000, ce qui sera large :

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
worker_rlimit_nofile 30000;
include /etc/nginx/modules-enabled/*.conf;
```

On restart nginx puis on arrive à taper en local :

```bash
sudo systemctl restart nginx
curl -I 127.0.0.1:80
# HTTP/1.1 200 OK
```


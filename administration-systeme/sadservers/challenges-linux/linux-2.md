# 2 - Rio de Janeiro, Nuuk, Cairo & Alexandria

## Rio de Janeiro

Il faut aussi débug du Jenkins. Le service ne voulait pas démarrer correctement, direction `systemctl status` pour comprendre ⇒ status en stopped.

Jenkins 2.516.3 refusait de démarrer proprement car il tournait avec une version de Java non recommandée pour cette version (Java 8 au départ, puis Java 25 signalé "not fully supported"). L'objectif est donc de basculer Jenkins sur Java 21, déjà installé sur la machine mais pas utilisé par défaut.

On regarde quelles versions sont utilisées par défaut :

```bash
java -version
javac -version
```

`java` pointait sur une vieille version (Java 8) tandis que `javac` pointait sur une version plus récente. Incohérence à corriger.

Sur Ubuntu/Debian, la gestion de plusieurs JVM se fait avec `update-alternatives`. On liste les versions installées :

```bash
sudo update-alternatives --config java
```

```
There are 3 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                         Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/java-25-openjdk-amd64/bin/java   2511      auto mode
  1            /usr/lib/jvm/java-21-openjdk-amd64/bin/java   2111      manual mode
  2            /usr/lib/jvm/java-25-openjdk-amd64/bin/java   2511      manual mode
  3            /usr/lib/jvm/temurin-8-jdk-amd64/bin/java     1081      manual mode
```

Java 8, 21 et 25 sont installés. Java 25 est sélectionné automatiquement mais pas pleinement supporté par cette version de Jenkins.

On force Java 21 (entrée `1`) :

```bash
sudo update-alternatives --config java
# Press <enter> to keep the current choice[*], or type selection number: 1
# update-alternatives: using /usr/lib/jvm/java-21-openjdk-amd64/bin/java to provide /usr/bin/java (java) in manual mode
```

Même manip pour `javac`. Ensuite `java -version` et `javac -version` renvoient bien Java 21.

On relance le service :

```bash
sudo systemctl status jenkins.service
sudo systemctl start jenkins.service
```

```bash
● jenkins.service - Jenkins Continuous Integration Server
     Active: active (running) since Mon 2026-03-09 18:39:17 UTC; 6s ago
     Main PID: 4169 (java)
     ...
Mar 09 18:39:17 i-0899387007d013833 jenkins[4169]: ... Jenkins is fully up and running
```

Puis on check que le port 8888 répond :

```bash
curl -s localhost:8888/login | grep Jenkins | head -n1
# <title>Sign in - Jenkins</title>...
```

On nous renvoi un "**Sign in - Jenkins**", l'interface web est donc accessible. Challenge résolu.

## Nuuk

Le titre du chall me rappel **SSHNuke**, référence à Matrix :P

**Objectif :** SSH ne fonctionnait pas en local sur la machine, malgré la présence des bonnes clés dans `~/.ssh/authorized_keys`.

Du gâteau, un simple coup d'œil sur le `.ssh` suffit à voir que les permissions étaient mauvaises :

```bash
ll
# d--------- 2 admin admin 4.0K Oct 21 17:27 .ssh
```

Aucune permission sur le dossier. On corrige :

```bash
sudo chmod 755 .ssh/
# drwxr-xr-x 2 admin admin 4.0K Oct 21 17:27 .ssh
```

Puis on se connecte en local sur la machine :

```bash
ssh 127.0.0.1
# The authenticity of host '127.0.0.1 (127.0.0.1)' can't be established.
# ED25519 key fingerprint is SHA256:SXwnOE3G0MEubcNLJQIryCk1URSsUStlsnc2dP2zj9s.
# Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
admin@i-0a31a7ab947fad896:~$
```

Connexion réussie.

## Cairo

**Contexte :** un script de health check critique (`/opt/scripts/health.sh`) est censé tourner toutes les 10 secondes via un timer systemd :

```bash
#!/bin/bash
# This script logs status and exits with 0 on success, 1 on failure
if curl -s --max-time 2 http://localhost | grep -q "Welcome to nginx"; then
  echo "$(date): STATUS: OK" >> /var/log/health.log
  exit 0
else
  echo "$(date): STATUS: FAILED" >> /var/log/health.log
  exit 1
fi
```

Comme vous allez le voir, celui-là m'a fait tourner en rond un bon moment.

En testant manuellement la commande dans le script :

```bash
curl http://localhost
^C
```

Rien n'arrive, ça reste bloqué à "Trying" :

```bash
curl -v http://localhost
* Host localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
*   Trying 127.0.0.1:80...
```

```bash
systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Active: active (running) since Tue 2026-03-10 18:59:27 UTC; 12min ago
     Main PID: 779 (nginx)
```

On check les logs :

```bash
cat /var/log/nginx/error.log
2025/11/22 16:01:28 [notice] 1600#1600: using inherited sockets from "5;6;"
```

Une seule ligne, un simple "notice", rien qui saute aux yeux. La syntaxe est pourtant bonne :

```bash
sudo nginx -t
# nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Rien d'anormal dans la conf non plus. Serait-ce le début d'un rabbit hole ?



```bash
sudo nginx
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] bind() to [::]:80 failed (98: Address already in use)
nginx: [emerg] still could not bind()
```

Problème de bind, donc avec les ports. Le port 80 est déjà utilisé mais par quoi ?

```bash
sudo ss -tuln | grep :80
tcp   LISTEN 0      511            0.0.0.0:80         0.0.0.0:*
tcp   LISTEN 0      511               [::]:80            [::]:*
tcp   LISTEN 0      4096                 *:8080             *:*
```

Arf, par nginx lui-même qui écoute déjà sur le port 80 (le process `systemctl status` tournait bien d'ailleurs) :

```bash
sudo systemctl stop nginx
sudo ss -tuln | grep :80
# tcp   LISTEN 0      4096                 *:8080             *:*
```

Après arrêt, le port 80 se libère bien. Donc nginx tourne correctement et écoute bien sur le bon port. Le problème est donc ailleurs.

J'ai fini par demander à l'ami Claude d'analyser ce log :

```bash
2025/11/22 16:01:28 [notice] 1600#1600: using inherited sockets from "5;6;"
```

Réponse : "théorie de sockets "zombies" hérités d'un redémarrage de conteneur, qui bloqueraient le port 80 malgré l'absence de process visible dans `ss`." Rien que ça. La solution proposée est de tuer tous les process nginx avec `pkill -9`, vérifier le port, relancer, et en dernier recours `fuser -k 80/tcp`.

Est-ce que l'IA avait raison ?

```bash
sudo pkill -9 nginx
sudo systemctl stop nginx
sudo ss -tunap | grep :80
# tcp   LISTEN 0      4096   *:8080   *:*   users:(("gotty",pid=704,fd=6))
sudo nginx -t
sudo systemctl start nginx
sudo ss -tunap | grep :80    # nginx bien présent, PIDs visibles
curl http://localhost        # toujours aucune réponse
```

<figure><img src="../../../.gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure>

Nope. Même après avoir fait ça, toujours pas de réponse contenu avec curl en localhost.

Allons faire un tour dans les règles iptables ?

```bash
sudo iptables -L -n
Chain INPUT (policy ACCEPT)
...
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DROP       tcp  --  0.0.0.0/0            127.0.0.1            tcp dpt:80 /* The hidden problem (IPv4) */
```

Vwelàaaaa, une règle DROP explicitement commentée "The hidden problem". Tu m'as eu hein saligots !

Hop on enlève ça :

```bash
sudo iptables -D OUTPUT 1
```

Et ça répond enfin !

```bash
curl http://localhost | grep "Welcome to"
<title>Welcome to nginx!</title>
<h1>Welcome to nginx!</h1>
```

On test le script directement :

```bash
bash /opt/scripts/health.sh
# /opt/scripts/health.sh: line 4: /var/log/health.log: Permission denied
```

Arf. Avec sudo peut-être ?

```bash
sudo bash /opt/scripts/health.sh
head /var/log/health.log
# Tue Mar 10 19:41:48 UTC 2026: STATUS: OK
```

Parfait. On vérifie si le timer systemd censé lancer ce script toutes les 10 secondes existe :

```bash
sudo systemctl list-unit-files --type=timer
# ...
# health.timer                 disabled enabled
```

Il existe mais est désactivé. On l'active :

```bash
sudo systemctl enable --now health.timer
# Created symlink '/etc/systemd/system/timers.target.wants/health.timer' → '/etc/systemd/system/health.timer'.
```

```bash
agent/check.sh
# OK
```

Résolu !

## Alexandria

**Contexte :** un job de backup cron mal configuré.

```bash
crontab -l
# no crontab for admin
sudo crontab -l
MAILTO="broken@nonexistent.local"
# DO NOT EDIT THIS FILE - edit the master and reinstall.
#Ansible: daily backup job
*/5 * * * * /opt/backup/old_backup.sh > /dev/null 2>&1
```

Deux problèmes visibles direct : le script appelé est `old_backup.sh`, et il tourne toutes les 5 minutes au lieu de la fréquence attendue. On fix :

```bash
*/10 * * * * /opt/backup/backup.sh > /dev/null 2>&1
```

Puis test du script :

```bash
./backup.sh
# Error: Backup already running (lock file exists)
```

Un fichier de lock bloque l'exécution. On le sort :

```bash
sudo rm backup.lock
./backup.sh
# touch: cannot touch '/opt/backup/backup.lock': Permission denied
# tar (child): /var/backups/daily/backup_20260311_090428.tar.gz: Cannot open: Permission denied
# Backup failed!
```

En sudo ça sera mieux ;) :

```bash
sudo ./backup.sh
# Backup successful: /var/backups/daily/backup_20260311_090432.tar.gz
```

Résolu.

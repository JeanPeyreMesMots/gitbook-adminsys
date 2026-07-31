# SadServers – Linux Challenges (2)

Suite des challenges de troubleshooting Linux sur [SadServers](https://sadservers.com/).
## <mark>Rio de Janeiro</mark>
**Découverte de Jenkins.** Le service ne voulait pas démarrer correctement, direction `systemctl status` pour comprendre.

### Contexte

Jenkins 2.516.3 refusait de démarrer proprement car il tournait avec une version de Java non recommandée pour cette version (Java 8 au départ, puis Java 25 signalé "not fully supported"). Objectif : basculer Jenkins sur Java 21, déjà installé sur la machine mais pas utilisé par défaut.

### Diagnostic

On regarde quelles versions sont utilisées par défaut :

```bash
java -version
javac -version
```

`java` pointait sur une vieille version (Java 8) tandis que `javac` pointait sur une version plus récente. Incohérence à corriger.

### Procédure

Sur Ubuntu/Debian, la gestion de plusieurs JVM se fait avec `update-alternatives`. On liste les versions installées :

```bash
sudo update-alternatives --config java
```

```text
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

Vérification de la version Jenkins :

```bash
jenkins --version
# 2.516.3
```

Relance du service :

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

Vérification que le port HTTP (8888) répond :

```bash
curl -s localhost:8888/login | grep Jenkins | head -n1
# <title>Sign in - Jenkins</title>...
```

Titre "Sign in - Jenkins" confirmé, l'interface web est accessible, challenge résolu.
## <mark>Nuuk</mark>
SSHNuke, référence à Matrix :P

**Objectif :** SSH ne fonctionnait pas en local sur la machine, malgré la présence des bonnes clés dans `~/.ssh/authorized_keys`.

Du gâteau — un simple coup d'œil sur le `.ssh` suffit à voir que les permissions sont mauvaises :

```bash
ll
# d--------- 2 admin admin 4.0K Oct 21 17:27 .ssh
```

Aucune permission sur le dossier. Correction :

```bash
sudo chmod 755 .ssh/
# drwxr-xr-x 2 admin admin 4.0K Oct 21 17:27 .ssh
```

Test :

```bash
ssh 127.0.0.1
# The authenticity of host '127.0.0.1 (127.0.0.1)' can't be established.
# ED25519 key fingerprint is SHA256:SXwnOE3G0MEubcNLJQIryCk1URSsUStlsnc2dP2zj9s.
# Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
admin@i-0a31a7ab947fad896:~$
```

Connexion réussie.
## <mark>Cairo</mark>
**Contexte :** un script de health check critique (`/opt/scripts/health.sh`) est censé tourner toutes les 10 secondes via un timer systemd. Le script :

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

Celui-là m'a fait tourner en rond un bon moment. Voici le fil complet, échecs compris.

### Round 1 : curl qui ne répond pas

Test manuel de la commande dans le script :

```bash
curl http://localhost
^C
```

Rien. Aucune réponse, obligé de couper avec Ctrl+C. En mode verbose :

```bash
curl -v http://localhost
* Host localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
*   Trying 127.0.0.1:80...
```

Ça reste bloqué à "Trying" — connexion qui ne s'établit jamais.

### Round 2 : nginx a l'air en bonne santé

```bash
systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Active: active (running) since Tue 2026-03-10 18:59:27 UTC; 12min ago
     Main PID: 779 (nginx)
```

Service actif, rien d'anormal en apparence. Logs :

```bash
cat /var/log/nginx/error.log
2025/11/22 16:01:28 [notice] 1600#1600: using inherited sockets from "5;6;"
```

Une seule ligne, un simple "notice", rien qui saute aux yeux. Config testée, syntaxe OK :

```bash
sudo nginx -t
# nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Rien d'anormal dans la conf non plus. À ce stade, je pensais que ça devait être un faux problème — spoiler : non, c'est le début d'un vrai rabbit hole.

### Round 3 : tenter de relancer nginx directement

```bash
sudo nginx
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] bind() to [::]:80 failed (98: Address already in use)
nginx: [emerg] still could not bind()
```

Port 80 déjà utilisé — mais par quoi ? Vérification :

```bash
sudo ss -tuln | grep :80
tcp   LISTEN 0      511            0.0.0.0:80         0.0.0.0:*
tcp   LISTEN 0      511               [::]:80            [::]:*
tcp   LISTEN 0      4096                 *:8080             *:*
```

En fait c'est nginx lui-même qui écoute déjà sur le port 80 (le process `systemctl status` tournait bien) :

```bash
sudo systemctl stop nginx
sudo ss -tuln | grep :80
# tcp   LISTEN 0      4096                 *:8080             *:*
```

Après arrêt, le port 80 se libère bien. Donc nginx tourne correctement, écoute bien sur le bon port — le problème est ailleurs.

### Round 4 : demander de l'aide à une IA, et se faire balader

J'ai fini par demander à un assistant IA d'analyser ce log :

```bash
2025/11/22 16:01:28 [notice] 1600#1600: using inherited sockets from "5;6;"
```

Réponse : théorie de sockets "zombies" hérités d'un redémarrage de conteneur, qui bloqueraient le port 80 malgré l'absence de process visible dans `ss`. Solution proposée : tuer tous les process nginx avec `pkill -9`, vérifier le port, relancer, et en dernier recours `fuser -k 80/tcp`.

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

![[Pasted image 20260731214155.png]]

Même après avoir tout tué et forcé le port avec `fuser -k`, `curl` restait muet. La théorie des sockets fantômes ne tenait pas — le vrai problème était ailleurs.

### Round 5 : le vrai coupable — un firewall caché

Je me suis dit qu'il fallait checker les règles iptables :

```bash
sudo iptables -L -n
Chain INPUT (policy ACCEPT)
...
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DROP       tcp  --  0.0.0.0/0            127.0.0.1            tcp dpt:80 /* The hidden problem (IPv4) */
```

Et voilà — une règle DROP explicitement commentée "The hidden problem". Bien joué, ils m'ont bien eu.

Suppression de la règle :

```bash
sudo iptables -D OUTPUT 1
```

Nouveau test :

```bash
curl http://localhost | grep "Welcome to"
<title>Welcome to nginx!</title>
<h1>Welcome to nginx!</h1>
```

Ça répond enfin.

### Finalisation

Test du script directement :

```bash
bash /opt/scripts/health.sh
# /opt/scripts/health.sh: line 4: /var/log/health.log: Permission denied
```

Nouveau problème de permission sur le fichier de log — résolu en lançant avec sudo :

```bash
sudo bash /opt/scripts/health.sh
head /var/log/health.log
# Tue Mar 10 19:41:48 UTC 2026: STATUS: OK
```

Reste à vérifier si le timer systemd censé lancer ce script toutes les 10 secondes existe :

```bash
sudo systemctl list-unit-files --type=timer
# ...
# health.timer                 disabled enabled
```

Il existe mais est désactivé. Activation :

```bash
sudo systemctl enable --now health.timer
# Created symlink '/etc/systemd/system/timers.target.wants/health.timer' → '/etc/systemd/system/health.timer'.
```

Vérification finale :

```bash
agent/check.sh
# OK
```

Résolu.
## <mark>Alexandria</mark>
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

Deux problèmes visibles direct : le script appelé est `old_backup.sh` (probablement obsolète), et il tourne toutes les 5 minutes au lieu de la fréquence attendue. Correction de la ligne cron :

```bash
*/10 * * * * /opt/backup/backup.sh > /dev/null 2>&1
```

Test manuel du script pour vérifier qu'il tourne correctement :

```bash
./backup.sh
# Error: Backup already running (lock file exists)
```

Un fichier de lock bloque l'exécution. Suppression :

```bash
sudo rm backup.lock
./backup.sh
# touch: cannot touch '/opt/backup/backup.lock': Permission denied
# tar (child): /var/backups/daily/backup_20260311_090428.tar.gz: Cannot open: Permission denied
# Backup failed!
```

Le script a aussi besoin des droits root pour écrire le lock file et l'archive. Avec sudo :

```bash
sudo ./backup.sh
# Backup successful: /var/backups/daily/backup_20260311_090432.tar.gz
```

Résolu.
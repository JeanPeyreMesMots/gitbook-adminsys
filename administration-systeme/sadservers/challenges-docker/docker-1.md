Challenges de troubleshooting Docker sur [SadServers](https://sadservers.com/scenarios/topic/docker).
## ==Salta==

**1.** Premier réflexe : lister les images et conteneurs existants :

```bash
sudo docker images
sudo docker ps -a
```

**2.** On consulte aussi les logs du conteneur concerné :

```bash
docker logs container_name
```

**3.** La cause : une coquille dans le `Dockerfile`, ligne `CMD` — `serve.js` au lieu de `server.js`. Correction et rebuild depuis `/home/admin/app` :

```bash
docker build -t app .
```

(image locale `node:15.7-alpine` fournie, pas d'accès Internet pour en tirer d'autres). Alternative sans rebuild :

```bash
docker run -d app node server.js
```

**4.** En voulant vérifier le port exposé attendu par le conteneur, j'ai remarqué que le serveur nginx tournait déjà sur le même port sur l'hôte, il faut l'arrêter avant de relancer le conteneur.

**5.** Dernier ajustement : dans le `Dockerfile`, la ligne `EXPOSE` déclarait le port `8880` au lieu de `8888`. On corrige puis on rebuild, puis :

```bash
docker run -d -p 8888:8888 app
```

Ou, sans toucher à l'image : `docker run -d -p 8888:8888 app node server.js`.

Ici le port 8888 du serveur est mappé vers le port 8888 du conteneur, donc accessible depuis l'extérieur via `http://notre-ip:8888`. Classique pour les serveurs web (Jupyter, Node.js, etc.).

```bash
docker run -d -p :8888 app
```

Ici, aucun port hôte n'est mappé. Le conteneur écoute sur son port 8888 en interne uniquement, ce qui nous permet alors de taper dessus, et de récupérer le flag en faisait un "**curl localhost:8888**".
## ==Venice==

Pas de panne à corriger ici, plutôt un exercice d'identification : déterminer si l'environnement tourne dans un vrai conteneur Docker ou autre chose.

Une ressource utile pour ce chall sur le sujet micro-VM vs conteneur : [some-natalie.dev](https://some-natalie.dev/blog/microvm-or-container/). Elle indique une méthode pour vérifier ce type de détail : inspecter l'environnement du process PID 1 à la recherche d'une variable `container` :

```bash
cat /proc/1/environ | tr "\0" "\n" | grep container
```

Normalement `container=podman` dans ce cas précis (la valeur avait été modifiée dans le challenge pour corser l'exercice).

Autre indicateur possible : l'absence de kernel threads (`[kthreadd]` par exemple) dans la liste des process, signe qu'on n'est pas dans un environnement avec son propre noyau complet. 

## ==Tarifa==

Un challenge dont je n'ai pas eu le temps de finir de noter la soluce malheureseument. 

1er coup d'oeil dans les logs comme toujours :

```bash
docker logs nginx_1
```

```text
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: can not modify /etc/nginx/conf.d/default.conf (read-only file system?)
```

Le script d'entrypoint nginx tente de modifier un fichier de configuration mais le système de fichiers est monté en lecture seule à cet endroit. Il s'agit d'un volume Docker monté en `:ro` (read-only) là où l'image nginx attend de pouvoir écrire.

[à compléter plus tard]
## ==Helsingør==

**Contexte :** ce chall expose une réplication PostgreSQL primaire/replica via Docker Compose, où le replica refuse de démarrer.

```bash
docker compose ps
```

```text
postgres-db-master    Up 2 minutes (healthy)
postgres-db-replica   Restarting (1) 35 seconds ago
```

Le replica boucle en restart. Hop dans les logs :

```bash
docker compose logs postgres-db-replica
```

```text
FATAL: recovery aborted because of insufficient parameter settings
DETAIL: max_connections = 80 is a lower setting than on the primary server, where its value was 100.
```

PostgreSQL refuse le démarrage du replica car certains paramètres de configuration sont inférieurs à ceux du primaire — une contrainte stricte de la réplication physique PostgreSQL.
### Première tentative : corriger max_connections

Un recherche Google qui confirme la piste (fichier `postgresql.conf`) :

```bash
grep "max_co*" postgres.conf
# max_connections = 100  # (change requires restart)
```

On ajuste la valeur de max connections comme indiqué, puis :

```bash
docker compose down
docker compose up -d
```

Toujours en échec, mais avec une **erreur différente** cette fois :

```text
DETAIL: max_worker_processes = 4 is a lower setting than on the primary server, where its value was 8.
```

Pas de quoi se décourager. Après plusieurs cycles de `down`/`up`, trois paramètres gérant les connexions devaient être alignés sur (ou au-dessus) des valeurs du primaire, chacun révélé un par un après correction du précédent :

- `max_connections` → 100
- `max_worker_processes` → 10 (le primaire était à 8)
- `max_wal_senders` → 10 (le primaire était à 10)
- `max_locks_per_transaction` → 64 (le primaire était à 64)

Puis le compose a pu relancer le service correctement sans autre erreur. 
Ce qui résout le challenge.
## ==Bharuch==

On à affaire ici à un conteneur qui boucle en erreur immédiatement au lancement.

```bash
docker logs web-server
# exec /bin/sh: exec format error
# (répété en boucle)
```

On tente de retrouver et lire directement le code applicatif depuis le système de fichiers de l'image, en cherchant le fichier `app.py` :

```bash
sudo find / -name app.py
```

Trouvé, mais accès refusé sans sudo, puis erreur différente une fois en sudo :

```bash
sudo python3 /var/lib/docker/overlay2/.../app/app.py
# ModuleNotFoundError: No module named 'flask'
```
### La vraie cause : mismatch d'architecture CPU

Une recherche Google indique que `exec /bin/sh: exec format error` correspond à un problème d'architecture. En inspectant l'image avec un grep :

```bash
docker inspect web-server:latest | grep -i "archi"
# "Architecture": "arm64",
uname -a
# ... x86_64 GNU/Linux
```

L'image a été buildée pour ARM64, alors que l'hôte tourne en x86_64 : le binaire ne peut tout simplement pas s'exécuter nativement.
### Solution

Plutôt que de reconstruire l'image pour la bonne architecture (plus long), utilisation de l'émulation QEMU pour permettre l'exécution multi-architecture sur l'hôte :

```bash
docker run --rm -d --privileged multiarch/qemu-user-static --reset -p yes
```

Ce qui résout le challenge.
## ==Quito==

**Objectif :** piloter le conteneur `nginx` (le démarrer) depuis un autre conteneur, `docker-access`.

```bash
docker ps -a
```

```text
nginx           Exited (137) 10 months ago
docker-access    Exited (137) 14 seconds ago
```

On démarrage le conteneur pilote "docker-access" puis on rentre dedans :

```bash
docker start docker-access
docker exec -ti docker-access sh
```

### Premier blocage : pas d'accès au démon Docker depuis l'intérieur

```bash
docker ps

Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

Le conteneur n'a par défaut aucun accès au Docker de l'hôte. Il faut pour cela que :

1. Le démon Docker tourne sur l'hôte.
2. Le socket `/var/run/docker.sock` de l'hôte doit être monté dans le conteneur, avec les bonnes permissions.

On check côté hôte que le démon tourne bien :

```bash
systemctl status docker
# Active: active (running)
```

Puis on lance le conteneur avec le socket monté :

```bash
docker run -it -v /var/run/docker.sock:/var/run/docker.sock --name docker-access docker-access
```

Résolu : le conteneur `docker-access` peut désormais piloter Docker sur l'hôte via le socket monté.

```bash
docker images
docker ps -a
# nginx présent, Exited
docker start nginx
docker ps
# nginx bien Up
```

### Point de vigilance : lancer en root

Le conteneur avait été lancé en root pour que ça fonctionne, ce qui n'est pas idéal côté sécurité (accès complet au démon Docker de l'hôte depuis un conteneur root équivaut quasiment à un accès root sur l'hôte lui-même). Meilleure approche à privilégier la prochaine fois plutôt que `--user root` :

```bash
docker run -it --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/bin/docker:/usr/bin/docker \
  --group-add $(getent group docker | cut -d: -f3) \
  ton_image
```

Cette approche ajoute le GID du groupe `docker` de l'hôte au conteneur, permettant d'utiliser le socket sans donner un accès root complet.
## ==Atlantis==

**Objectif :** builder et lancer un conteneur "app" à partir d'un Dockerfile multi-stage :

```dockerfile
# STAGE 1
FROM debian:13 AS builder
RUN apt-get update && apt-get install -y gcc
WORKDIR /src
COPY hello.c .
RUN gcc -o hello hello.c

# STAGE 2
FROM alpine:3.20
COPY --from=builder /src/hello /usr/local/bin/hello
CMD ["/usr/local/bin/hello"]
```

Le build réussit, mais le run échoue :

```bash
docker build -t app:latest . && docker run app
# ... build terminé avec succès ...
exec /usr/local/bin/hello: no such file or directory
```

Ce message est trompeur : le fichier existe pourtant bel et bien dans l'image (copié depuis le stage builder). En y regardant de plus près, le stage 1 (compilation) utilise `debian:13`, basé sur **glibc**, tandis que le stage 2 (exécution finale) utilise `alpine:3.20`, basé sur **musl**. 

Le binaire compilé et linké dynamiquement contre glibc dans le premier stage ne peut pas s'exécuter dans un environnement du second. D'où un message d'erreur qui semble indiquer un fichier manquant, alors qu'il s'agit en réalité d'un binaire incompatible avec l'environnement d'exécution.

Aligner les deux stages sur la même base pour garantir la compatibilité libc — suggestion (avec l'aide d'une IA) d'utiliser `debian:13-slim` pour les deux étages plutôt que de mixer Debian et Alpine, permet d'avoir un multistage réparé :

```dockerfile
# STAGE 1
FROM debian:13-slim AS builder
RUN apt-get update && apt-get install -y gcc
WORKDIR /src
COPY hello.c .
RUN gcc -o hello hello.c

# STAGE 2
FROM debian:13-slim
COPY --from=builder /src/hello /usr/local/bin/hello
CMD ["/usr/local/bin/hello"]
```

Avec un build et run réussis dans la foulée :

```bash
docker build -t app . && docker run app
# ... build OK ...
# sortie du programme, résolu
```

## ==Auderghem==

**Objectif :** un reverse proxy nginx doit rediriger le trafic vers deux conteneurs `statichtml1` et `statichtml2`.

On inspecte chaque conteneur pour ses IP :

```text
statichtml1 --> 172.172.0.11
statichtml2 --> 172.172.0.12
nginx       --> 172.17.0.2
```

La conf nginx (`/home/admin/app/default.conf`) référence les hostnames `statichtml1.sadservers.local` et `statichtml2.sadservers.local`. Le ping fonctionne sur les 3 IP directement, mais pas sur les hostnames.

### Round 1 : problème réseau — nginx n'est pas sur le bon réseau Docker

Répartition observée : `statichtml1`/`statichtml2` sont sur un réseau bridge dédié nommé `static-net`, alors que `nginx` reste sur le réseau `bridge` par défaut.

Les logs nginx confirment le symptôme :

```text
upstream timed out (110: Connection timed out) while connecting to upstream ... http://172.172.0.11:80/
```

Vérification du réseau `static-net` :

```bash
docker network inspect cc3e04c023f1
# static-net, bridge, subnet 172.172.0.0/24
# statichtml1 -> 172.172.0.11, statichtml2 -> 172.172.0.12
```

Depuis l'intérieur du conteneur nginx, le ping par IP fonctionne mais pas par hostname. Logique, puisque nginx n'est même pas dans ce réseau (confirmé via `docker inspect nginx`, qui ne montre que le réseau `bridge` par défaut).

Une solution serait de connecter nginx au réseau `static-net` :

```bash
docker network connect static-net nginx
```

`docker inspect nginx` montre bien la nouvelle entrée réseau avec résolution DNS (`DNSNames`). Le ping par hostname fonctionne enfin depuis l'intérieur du conteneur nginx (après installation de `iputils-ping`, absent par défaut de l'image).

### Round 2 : toujours en échec depuis l'hôte, cette fois un problème de port

```bash
curl http://localhost/1
# 502 Bad Gateway
```

Je redémarrage tous les conteneurs par précaution mais sans effet. En observant `docker ps`, les conteneurs `statichtml1`/`statichtml2` écoutent en fait sur le port **3000**, pas 80.

Or, dans la conf nginx, le `proxy_pass` ne précisait aucun port :

```nginx
location /1 {
    proxy_pass http://statichtml1.sadservers.local;
}
```

Sans port explicite, nginx redirige par défaut vers le port 80 du backend, qui n'écoute pas dessus. Correction :

```nginx
proxy_pass http://statichtml1.sadservers.local:3000;
proxy_pass http://statichtml2.sadservers.local:3000;
```

On reboot les conteneurs, ce qui résout le chall ensuite.
## ==Woluwe==

**Contexte :** un pipeline a généré plusieurs images Docker locales pour une même appli web ; toutes sauf une contiennent une typo introduite par un développeur (`index.htmlz` au lieu de `index.html`). Objectif : retrouver la bonne image, la tagger `prod`, et la déployer sur le port 3000.

Script (avec l'aide de Perplexity) pour scanner l'historique de chaque image à la recherche de la typo :

```bash
for img in $(docker images --format '{{.ID}}'); do
  if ! docker history --no-trunc "$img" 2>/dev/null | grep -q 'index.htmlz'; then
    echo "Image sans index.htmlz : $img"
  fi
done
```

Deux résultats. Le premier (`dd15126afe8d`) s'avère être une image générique de base, sans rapport direct avec l'app (probablement présente pour brouiller les pistes du chall) :

```bash
docker history dd15126afe8d --no-trunc
# CMD busybox httpd ... rien de spécifique à l'app
```

Le second (`3f8befa65f01`) est le bon candidat. Son historique de layers montre bien la commande correcte :

```bash
docker history 3f8befa65f01 --no-trunc
# RUN ... echo "HelloWorld;$HW" > index.html
```

On tag puis on déploie :

```bash
docker tag 3f8befa65f01 prod
docker run -d --name prod -p 3000:3000 prod
curl http://localhost:3000
# HelloWorld;529
```

Résolu.
## ==Torino==

**Objectif :** réduire la taille d'une image Node.js qui pèse environ 1 Go.

```bash
docker images
# torino       latest   916MB
# node         16       909MB
# node         16-alpine 118MB
```

Le poids vient directement de l'image de base utilisée dans le Dockerfile :

```dockerfile
FROM node:16
WORKDIR /app
COPY package.json .
COPY app.js .
RUN npm install
EXPOSE 3000
CMD ["node", "app.js"]
```

`node:16` (basée sur Debian complet) pèse près d'1 Go, contre ~118 Mo pour `node:16-alpine`. On peut basculer vers l'Alpine, après sauvegarde du Dockerfile d'origine par précaution :

```bash
cp Dockerfile Dockerfile_OLD
```

```dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package.json .
COPY app.js .
COPY node_modules .
EXPOSE 3000
CMD ["node", "app.js"]
```

On ajoute un `COPY node_modules .` pour que les dépendances déjà installées soient bien présentes dans l'image, puis on build avec le tag attendu par le challenge :

```bash
docker build -t torino:latest .
docker images
# torino latest 120MB
```

On passe ainsi de 916 Mo à 120 Mo. Test final :

```bash
nohup node app.js > app.log 2>&1 &
curl localhost:3000
# {"message":"Hello from Torino!"}
```

Résolu.

### Bonus : et si on demandait à l'IA ?

En demandant à ChatGPT d'optimiser encore plus le Dockerfile et le contenu du dossier en contexte, il nous sort un **build multi-stage** :

```dockerfile
# Étape de build : installation des dépendances
FROM node:16-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production --no-package-lock && \
    npm cache clean --force

# Étape de runtime : image finale ultra-légère
FROM node:16-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY app.js .
EXPOSE 3000
CMD ["node", "app.js"]
```

Cette version sépare l'installation des dépendances (étape build) du runtime final, en ne copiant que le strict nécessaire (`node_modules` déjà installés + code applicatif) dans l'image finale. Cela évite d'embarquer le cache npm, les fichiers de lock, et les outils de build dans l'image livrée.
## ==San-juan==

**Objectif :** un Traefik dockerisé qui route vers plusieurs conteneurs `whoami`, mais ne répond correctement qu'une fois sur trois.

```bash
curl -s app.sadserver | head -n1
# tantôt Hostname: xxx, tantôt "Bad Gateway", tantôt rien du tout
```

Vérification que tous les conteneurs sont up :

```bash
docker ps
# traefik + 4 conteneurs whoami (app01 à app04), tous "Up"
```

Logs du conteneur Traefik principal, filtrés sur les erreurs :

```bash
docker logs a2f3f16b0928 | grep "error"
# 502 Bad Gateway error="dial tcp 172.19.0.3:81: connect: connection refused"
```

Voilà : Traefik essaie de joindre un des conteneurs sur le **port 81**. Ce port n'existe évidemment pas côté conteneur `whoami` (qui écoute en 80).

Dans le `docker-compose.yml`, le conteneur fautif `app02` déclare explicitement ce mauvais port dans son label Traefik :

```yaml
app02:
  image: traefik/whoami
  labels:
    traefik.http.services.app.loadbalancer.server.port: "81"
```

On sauvegarde le fichier par précaution, puis on corrige :

```bash
cp docker-compose.yml docker-compose.yml_OLD
sed -i 's/"81"/"80"/g' docker-compose.yml
docker compose up -d
```

Test après correction : les 4 conteneurs répondent désormais correctement à tour de rôle, sans erreur ni interruption :

```bash
curl -s app.sadserver | head -n1
# Hostname: xxx (à chaque fois, load-balancing normal entre les 4 whoami)
```
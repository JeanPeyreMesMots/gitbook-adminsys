# 2 - Auderghem, Woluwe, Torino & San-juan

### <mark style="color:$warning;">Auderghem</mark>

**Goal:** an nginx reverse proxy must forward traffic to two containers, `statichtml1` and `statichtml2`.

We inspect each container to get its IP:

```
statichtml1 --> 172.172.0.11
statichtml2 --> 172.172.0.12
nginx       --> 172.17.0.2
```

The nginx configuration (`/home/admin/app/default.conf`) references the hostnames `statichtml1.sadservers.local` and `statichtml2.sadservers.local`. Pinging the 3 IPs directly works, but not the hostnames.

The catch: `statichtml1`/`statichtml2` are on a dedicated bridge network, `static-net`, while `nginx` is on the default `bridge` network.

The nginx logs confirm it:

```
upstream timed out (110: Connection timed out) while connecting to upstream ... http://172.172.0.11:80/
```

We check the `static-net` network:

```bash
docker network inspect cc3e04c023f1
# static-net, bridge, subnet 172.172.0.0/24
# statichtml1 -> 172.172.0.11, statichtml2 -> 172.172.0.12
```

From inside the nginx container, pinging by IP works but not by hostname. That makes sense, since nginx isn't even on that network, and Docker's internal DNS only resolves container names on user-defined networks the container belongs to.

One solution is to connect nginx to the `static-net` network:

```bash
docker network connect static-net nginx
```

`docker inspect nginx` now shows the network configuration, and pinging by hostname works from inside the nginx container (after installing `iputils-ping`).

However, when we hit the machine itself:

```bash
curl http://localhost/1
# 502 Bad Gateway
```

I restart all the containers just in case, with no effect. Looking at `docker ps`, the `statichtml1`/`statichtml2` containers actually listen on port **3000**, not 80.

And in the nginx configuration, `proxy_pass` didn't specify any port:

```nginx
location /1 {
    proxy_pass http://statichtml1.sadservers.local;
}
```

Without an explicit port, nginx forwards to port 80 on the backend, where nothing is listening. We fix it:

```nginx
proxy_pass http://statichtml1.sadservers.local:3000;
proxy_pass http://statichtml2.sadservers.local:3000;
```

After restarting the containers, the challenge is solved.

### <mark style="color:$warning;">Woluwe</mark>

**Context:** a pipeline generated several local Docker images for the same web app; all but one contain a typo introduced by a developer (`index.htmlz` instead of `index.html`). Goal: find the right image, tag it `prod`, and deploy it on port 3000.

A script (written with Perplexity's help) to scan each image's history for the typo:

```bash
for img in $(docker images --format '{{.ID}}'); do
  if ! docker history --no-trunc "$img" 2>/dev/null | grep -q 'index.htmlz'; then
    echo "Image sans index.htmlz : $img"
  fi
done
```

Two results. The first one (`dd15126afe8d`) turns out to be a generic base image, unrelated to the app (probably there as a decoy):

```bash
docker history dd15126afe8d --no-trunc
# CMD busybox httpd ... rien de spécifique à l'app
```

The second one (`3f8befa65f01`) is the right candidate. Its layer history shows the correct command:

```bash
docker history 3f8befa65f01 --no-trunc
# RUN ... echo "HelloWorld;$HW" > index.html
```

We tag it and deploy it:

```bash
docker tag 3f8befa65f01 prod
docker run -d --name prod -p 3000:3000 prod
curl http://localhost:3000
# HelloWorld;529
```

Solved.

### <mark style="color:$warning;">Torino</mark>

**Goal:** reduce the size of a Node.js image weighing around 1 GB.

```bash
docker images
# torino       latest   916MB
# node         16       909MB
# node         16-alpine 118MB
```

The size comes directly from the base image used in the Dockerfile:

```dockerfile
FROM node:16
WORKDIR /app
COPY package.json .
COPY app.js .
RUN npm install
EXPOSE 3000
CMD ["node", "app.js"]
```

`node:16` (based on full Debian) weighs almost 1 GB, versus \~118 MB for `node:16-alpine`. We can switch to Alpine, after backing up the original Dockerfile just in case:

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

We add a `COPY node_modules .` so that the dependencies already installed locally end up in the image (since the challenge environment has no Internet access for `npm install`), then build with the tag expected by the challenge:

```bash
docker build -t torino:latest .
docker images
# torino latest 120MB
```

We go from 916 MB down to 120 MB. Final test:

```bash
nohup node app.js > app.log 2>&1 &
curl localhost:3000
# {"message":"Hello from Torino!"}
```

Solved.

#### Bonus: what if we asked AI?

Asking ChatGPT to optimize the Dockerfile even further, with the folder contents as context, it suggests a **multi-stage build**:

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

This version separates dependency installation (build stage) from the final runtime, copying only what's strictly needed (installed `node_modules` + application code) into the final image. This avoids shipping the npm cache and build tooling in the delivered image. As always with AI-generated code, it should be reviewed and tested before use.

### <mark style="color:$warning;">San-Juan</mark>

**Goal:** a dockerized Traefik routes to several `whoami` containers, but only responds correctly some of the time.

```bash
curl -s app.sadserver | head -n1
# tantôt Hostname: xxx, tantôt "Bad Gateway", tantôt rien du tout
```

We check that all the containers are up:

```bash
docker ps
# traefik + 4 conteneurs whoami (app01 à app04), tous "Up"
```

Errors in the Traefik container's logs:

```bash
docker logs a2f3f16b0928 | grep "error"
# 502 Bad Gateway error="dial tcp 172.19.0.3:81: connect: connection refused"
```

There it is: Traefik tries to reach one of the containers on **port 81**, which the `whoami` container doesn't listen on (it listens on 80).

In `docker-compose.yml`, the faulty container, `app02`, explicitly declares that wrong port in its Traefik label:

```yaml
app02:
  image: traefik/whoami
  labels:
    traefik.http.services.app.loadbalancer.server.port: "81"
```

We back up the file just in case, then fix it:

```bash
cp docker-compose.yml docker-compose.yml_OLD
sed -i 's/"81"/"80"/g' docker-compose.yml
docker compose up -d
```

Test after the fix: all 4 containers now answer in turn, with no errors or interruptions:

```bash
curl -s app.sadserver | head -n1
# Hostname: xxx (à chaque fois, load-balancing normal entre les 4 whoami)
```
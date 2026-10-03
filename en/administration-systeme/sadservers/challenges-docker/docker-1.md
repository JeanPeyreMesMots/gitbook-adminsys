# 1 - Salta, Venice, Tarifa, Helsingør, Bharuch, Quito & Atlantis

Docker troubleshooting challenges from [SadServers](https://sadservers.com/scenarios/topic/docker).

### <mark style="color:$warning;">Salta</mark>

First reflex: list the existing images and containers:

```bash
sudo docker images
sudo docker ps -a
```

We also check the logs of the container involved:

```bash
docker logs container_name
```

The cause: a typo in the `Dockerfile`, on the `CMD` line: `serve.js` instead of `server.js`. We fix it and rebuild from `/home/admin/app`:

```bash
docker build -t app .
```

(the `node:15.7-alpine` image is provided locally, with no Internet access to pull others). Alternative without rebuilding, by overriding the command:

```bash
docker run -d app node server.js
```

While checking which port the container was expected to expose, I noticed that an nginx server was already running on the same port on the host. It has to be stopped before starting the container again.

Last fix: in the `Dockerfile`, the `EXPOSE` line declared port `8880` instead of `8888`. We fix it, rebuild, then:

```bash
docker run -d -p 8888:8888 app
```

Or, without touching the image: `docker run -d -p 8888:8888 app node server.js`.

Here, port 8888 on the host is mapped to port 8888 in the container, so the app is reachable from outside at `http://our-ip:8888`, and locally with "**curl localhost:8888**", which returns the expected response and solves the challenge. A classic setup for web servers (Jupyter, Node.js, etc.).

```bash
docker run -d -p :8888 app
```

For comparison, with this syntax no host port is specified: Docker publishes the container's port 8888 on a random host port (which `docker port <container>` shows).

### <mark style="color:$warning;">Venice</mark>

No outage to fix here, rather an identification exercise: determine whether the environment runs in a real Docker container or something else.

A useful resource on micro-VMs vs containers for this challenge: [some-natalie.dev](https://some-natalie.dev/blog/microvm-or-container/). It describes a way to check this: inspect the environment of PID 1, looking for a `container` variable:

```bash
cat /proc/1/environ | tr "\0" "\n" | grep container
```

In this case it should be `container=podman` (the value had been altered in the challenge to make it harder).

Another possible clue: the absence of kernel threads (such as `[kthreadd]`) in the process list, a sign that the environment doesn't have its own kernel, unlike a VM or micro-VM.

### <mark style="color:$warning;">Tarifa</mark>

A challenge whose solution I unfortunately didn't have time to write up completely.

First look at the logs, as always:

```bash
docker logs nginx_1
```

```
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: can not modify /etc/nginx/conf.d/default.conf (read-only file system?)
```

The nginx entrypoint script tries to modify a configuration file, but the file system is read-only at that location: a Docker volume mounted as `:ro` (read-only) where the nginx image expects to be able to write.

\[to be completed later]

### <mark style="color:$warning;">Helsingør</mark>

**Context:** this challenge sets up PostgreSQL primary/replica replication with Docker Compose, and the replica refuses to start.

```bash
docker compose ps
```

```
postgres-db-master    Up 2 minutes (healthy)
postgres-db-replica   Restarting (1) 35 seconds ago
```

The replica is stuck in a restart loop. Let's check the logs:

```bash
docker compose logs postgres-db-replica
```

```
FATAL: recovery aborted because of insufficient parameter settings
DETAIL: max_connections = 80 is a lower setting than on the primary server, where its value was 100.
```

PostgreSQL refuses to start the replica because its `max_connections` is lower than on the primary. A quick search confirms where to fix it (the `postgresql.conf` file):

```bash
grep "max_co*" postgres.conf
# max_connections = 100  # (change requires restart)
```

We adjust `max_connections` as indicated and bring the containers back up:

```bash
docker compose down
docker compose up -d
```

Still failing, but with a **different error** this time:

```
DETAIL: max_worker_processes = 4 is a lower setting than on the primary server, where its value was 8.
```

No reason to give up. After several `down`/`up` cycles, it turned out that several parameters had to be equal to or higher than the primary's values:

* `max_connections` → 100
* `max_worker_processes` → 10 (the primary was at 8)
* `max_wal_senders` → 10 (the primary was at 10)
* `max_locks_per_transaction` → 64 (the primary was at 64)

Compose was then able to start the service without any other error, which solved the challenge.

### <mark style="color:$warning;">Bharuch</mark>

Here we have a container that fails immediately on startup, in a loop:

```bash
docker logs web-server
# exec /bin/sh: exec format error
# (répété en boucle)
```

Looking for the application code, we find an `app.py`:

```bash
sudo find / -name app.py
```

Found, but access is denied without sudo, and we get a different error with sudo:

```bash
sudo python3 /var/lib/docker/overlay2/.../app/app.py
# ModuleNotFoundError: No module named 'flask'
```

A quick search shows that `exec /bin/sh: exec format error` points to an architecture mismatch. Inspecting the image with a grep:

```bash
docker inspect web-server:latest | grep -i "archi"
# "Architecture": "arm64",
uname -a
# ... x86_64 GNU/Linux
```

The image was built for ARM64, while the host runs x86\_64: the binaries simply can't run natively.

Rather than rebuilding the image for the right architecture (which takes longer), we register QEMU emulation on the host to allow multi-architecture execution:

```bash
docker run --rm -d --privileged multiarch/qemu-user-static --reset -p yes
```

Once the container is restarted, it runs under emulation, which solves the challenge.

### <mark style="color:$warning;">Quito</mark>

**Goal:** start the `nginx` container from inside the `docker-access` container.

```bash
docker ps -a
```

```
nginx           Exited (137) 10 months ago
docker-access    Exited (137) 14 seconds ago
```

We start the "**docker-access**" container and get a shell inside it:

```bash
docker start docker-access
docker exec -ti docker-access sh
```

However, it can't reach the Docker daemon from inside:

```bash
docker ps

Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

By default, a container has no access to the host's Docker. For that:

* the Docker daemon must be running on the host;
* the host's `/var/run/docker.sock` socket must be mounted into the container, with the right permissions.

So we check on the host that the daemon is running:

```bash
systemctl status docker
# Active: active (running)
```

Then we run the container with the socket mounted:

```bash
docker run -it -v /var/run/docker.sock:/var/run/docker.sock --name docker-access docker-access
```

Solved: the `docker-access` container can now control the host's Docker through the mounted socket.

```bash
docker images
docker ps -a
# nginx présent, Exited
docker start nginx
docker ps
# nginx bien Up
```

#### A word of caution about running as root

The container was run as root to make this work, which isn't ideal security-wise. An alternative to `--user root` is to add the host's `docker` group GID to the container:

```bash
docker run -it --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/bin/docker:/usr/bin/docker \
  --group-add $(getent group docker | cut -d: -f3) \
  ton_image
```

This lets a non-root user inside the container use the socket. However, keep in mind that any access to the Docker socket, root or not, is practically equivalent to root access on the host: whoever can talk to the daemon can start a privileged container. Mounting the socket should therefore be reserved for trusted containers.

### <mark style="color:$warning;">Atlantis</mark>

**Goal:** build and run an "app" container from a multi-stage Dockerfile:

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

The build succeeds, but the run fails:

```bash
docker build -t app:latest . && docker run app
# ... build terminé avec succès ...
exec /usr/local/bin/hello: no such file or directory
```

This message is misleading: the file does exist in the image (copied from the builder stage). Looking closer, stage 1 compiles on `debian:13`, based on **glibc**, while stage 2 runs on `alpine:3.20`, based on **musl**.

The binary, dynamically linked against glibc in the first stage, can't run in the second stage's environment: the "missing file" is actually the glibc dynamic loader, which doesn't exist on Alpine. Hence an error about a missing file, when the real issue is a binary incompatible with the runtime environment.

Both stages therefore need to share the same C library. We change the images so that both stages use `debian:13-slim` (another option would be to compile a static binary with `-static`):

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

The build and run then succeed:

```bash
docker build -t app . && docker run app
# ... build OK ...
# sortie du programme, résolu
```
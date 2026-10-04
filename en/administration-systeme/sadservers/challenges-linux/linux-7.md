# 7 - Bekasi

**Starting point:** impossible to restart nginx or check its config normally.

```bash
nginx -t
# -bash: nginx: command not found
systemctl stop nginx
# Failed to stop nginx.service: Access denied
systemctl start nginx
# Failed to start nginx.service: Access denied
```

No `nginx` binary directly accessible, and `systemctl` even refuses to stop it without sudo. Yet the service is already running:

```bash
systemctl status nginx.service
# Active: active (running) since Tue 2026-03-24 17:38:55 UTC
# ...
# Failed to parse PID from file /run/nginx.pid: Invalid argument
```

Looking at the nginx config, I notice something with the vhost:

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

Two things to check: the `/srv/www/assets` directory referenced for static content, and the uWSGI socket `/home/admin/bekasi/bekasi.sock` for the rest.

Checking the static folder:

```bash
ll /srv/
# total 8.0K, rien dedans à part . et ..
```

Empty, which isn't normal, but it's not the main blocker (the dynamic app goes through uWSGI, not this folder).

After a while, I check the solution, which mentions `supervisorctl`, a tool I didn't know before this challenge ([doc used](http://blog.stephane-robert.info/docs/services/processus/supervisor/)): a process manager that can supervise and restart apps (here the Python/uWSGI app).

```bash
cat /etc/supervisor/supervisord.conf
# files = /etc/supervisor/conf.d/*.conf
```

We look at the supervisor logs:

```bash
cat /var/log/supervisor/supervisord.log
# CRIT Supervisor is running as root...
# WARN No file matches via include "/etc/supervisor/conf.d/*.conf"   (avant reconfig)
# INFO Included extra file "/etc/supervisor/conf.d/uwsgi.conf"
# INFO spawned: 'bekasi' with pid 8938
# INFO success: bekasi entered RUNNING state
# INFO stopped: bekasi (exit status 0)
```

We notice the `bekasi` process starts, runs for a bit, then stops.

We run the binary by hand first to see what happens, without going through supervisor:

```bash
./uwsgi
# The -s/--socket option is missing and stdin is not a socket.
```

Right, we need to specify the socket:

```bash
./uwsgi -s ../bekasi.sock
# uwsgi socket 0 bound to UNIX address ../bekasi.sock fd 3
# *** no app loaded. going in full dynamic mode ***
# spawned uWSGI worker 1 (and the only) (pid: 1590, cores: 1)
```

It runs. We send it to the background to test in parallel:

```bash
# Ctrl+Z puis :
bg
curl -k https://bekasi
# 502 Bad Gateway
```

But still a 502 despite uWSGI apparently running. Either nginx isn't pointing to the right place, or uWSGI isn't responding correctly on that specific socket.

However, via `supervisorctl`, we notice the process also runs and logs details:

```bash
sudo supervisorctl
# bekasi   RUNNING   pid 1210, uptime 0:20:21
supervisor> tail -f bekasi
# *** Operational MODE: preforking ***
# WSGI app 0 (mountpoint='') ready in 0 seconds ...
# spawned uWSGI master process (pid: 1210)
# spawned uWSGI worker 1..5
```

Comparing the manual launch output with the supervisor one, there's a notable difference in the very first lines of the manual launch:

```
!!! no internal routing support, rebuild with pcre support !!!
*** WARNING: you are running uWSGI without its master process manager ***
```

These lines don't appear in the supervisor version. So the execution environment differs between the two launches. That's why the solution hint suggested checking `~/.bashrc`:

```bash
tail -f ~/.bashrc
export BEKASI_SERVER=bekasi.sadservers.com
export BEKASI_USER=admin
```

We find two environment variables defined only in the user's interactive shell. So they're present when we run `uwsgi` by hand from that shell, but absent from the context in which `supervisord` launches its processes (which doesn't source `~/.bashrc`).

Checking the existing supervisor config:

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

No environment variables defined here, so we add the two needed ones:

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

Then we reload everything:

```bash
sudo supervisorctl
supervisor> reread
# bekasi: changed
supervisor> update
# bekasi: stopped
# bekasi: updated process group
```

Final test:

```bash
curl -k https://bekasi
# Hello SadServers!
```

Solved.
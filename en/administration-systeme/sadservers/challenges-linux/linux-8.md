# 8 - Batumi

A Caddy server to debug here:

```bash
ps faux | grep "caddy"
# /usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
```

Its config is simple, a reverse proxy to a local backend:

```caddyfile
:80 {
    reverse_proxy localhost:5050
}
```

Except it responds with a 500:

```bash
curl -vv -I http://localhost:5050
# HTTP/1.1 500 Internal Server Error
# Content-Length: 158 (zero-length body malgré le header)
```

We check the Caddy service logs via `journalctl`; the config seems loaded fine, and the service is running:

```bash
journalctl -u caddy.service -e
# using config from file /etc/caddy/Caddyfile
# server running, protocols h1/h2/h3
```

There's a warning in the Caddyfile (`caddy fmt --overwrite`). On the systemd side, a second unit exists but is disabled:

```bash
caddy-api.service   disabled  enabled
caddy.service       enabled   enabled
```

We enable it just in case:

```bash
sudo systemctl enable caddy-api.service
```

Then we re-check the `caddy.service` logs, but nothing abnormal there. So Caddy isn't at fault.

As in the previous challenges, we check the rules to see whether they've added one that drops things to give us trouble :D :

```bash
sudo iptables -L
Chain INPUT (policy ACCEPT)
DROP  tcp  --  anywhere  anywhere  tcp dpt:http
```

And indeed, a DROP rule blocks all incoming traffic on the web port, again. We remove it:

```bash
sudo iptables -D INPUT -p tcp --dport 80 -j DROP
```

But it still won't work... only now we get an error on the PostgreSQL port:

```bash
curl http://localhost
# could not connect to server: Connection refused
# Is the server running on host "127.0.0.1" and accepting TCP/IP connections on port 5433?
```

We take a look at the backend's Python script, which does query a PostgreSQL database:

```python
conn = psycopg2.connect(**db_params)
cursor.execute("SELECT secret FROM secrets WHERE id=1;")
```

Usual checkup:

```bash
systemctl status postgresql.service
# Active: inactive (dead)
sudo systemctl start postgresql.service
sudo systemctl status postgresql.service
# Active: active (exited) — ExecStart=/bin/true
```

The service "starts" but via `/bin/true`, which I find strange. Just to be sure, I check what's actually listening:

```bash
sudo netstat -tunalp | grep postgres
# tcp  127.0.0.1:5432  LISTEN  1580/postgres
```

PostgreSQL is already running on port **5432** (the standard port), while the app's `.env` points to **5433**. Would a service restart help?

```bash
sudo systemctl restart postgresql.service
# toujours Active: active (exited) via /bin/true, rien de concret
curl http://localhost
# même erreur, port 5433 introuvable
```

<figure><img src="../../../.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

In the end, the solution pointed out that a dedicated service, `db_connector`, had to be restarted: it acts as the real bridge between the web backend and PostgreSQL:

```bash
systemctl list-unit-files | grep db_connector
# db_connector.service   enabled   enabled
systemctl restart db_connector
```

Solved.
# 8 - Batumi

Un serveur caddy est ici à débugger :

```bash
ps faux | grep "caddy"
# /usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
```

Sa conf reste simple, un reverse proxy vers un backend local :

```caddyfile
:80 {
    reverse_proxy localhost:5050
}
```

Sauf qu'il répond en 500 :

```bash
curl -vv -I http://localhost:5050
# HTTP/1.1 500 Internal Server Error
# Content-Length: 158 (zero-length body malgré le header)
```

On check les logs du service Caddy via `journalctl` et la conf semble bien chargée, de même pour l'état en running :

```bash
journalctl -u caddy.service -e
# using config from file /etc/caddy/Caddyfile
# server running, protocols h1/h2/h3
```

Un warning est présent dans le Caddyfile (`caddy fmt --overwrite`). Côté systemd, une deuxième existe mais est désactivée :

```bash
caddy-api.service   disabled  enabled
caddy.service       enabled   enabled
```

On active par précaution :

```bash
sudo systemctl enable caddy-api.service
```

Puis on recheck les logs de `caddy.service` mais rien d'anormal dans les logs. Donc Caddy n'est pas fautif.

Comme dans les challs précédents on check les règles pour voir si ils se sont pas amusés à en mettre une qui dropperait des trucs pour nous faire galérer :D  :

```bash
sudo iptables -L
Chain INPUT (policy ACCEPT)
DROP  tcp  --  anywhere  anywhere  tcp dpt:http
```

Et effectivement une règle DROP bloque tout le trafic entrant sur le port web, décidément. On l'enlève :

```bash
sudo iptables -D INPUT -p tcp --dport 80 -j DROP
```

Mais ça veut toujours pas... seulement on a une erreur sur le porte de PostegreSQL :

```bash
curl http://localhost
# could not connect to server: Connection refused
# Is the server running on host "127.0.0.1" and accepting TCP/IP connections on port 5433?
```

On jette un coup d'œil dans le script Python du backend, qui interroge bien une base PostgreSQL :

```python
conn = psycopg2.connect(**db_params)
cursor.execute("SELECT secret FROM secrets WHERE id=1;")
```

Checkup habituel :

```bash
systemctl status postgresql.service
# Active: inactive (dead)
sudo systemctl start postgresql.service
sudo systemctl status postgresql.service
# Active: active (exited) — ExecStart=/bin/true
```

Le service "démarre" mais via `/bin/true`, ce que je trouve étrange. Par acquis de conscience je choisis quand même de voir ce qui écoute réellement :

```bash
sudo netstat -tunalp | grep postgres
# tcp  127.0.0.1:5432  LISTEN  1580/postgres
```

PostgreSQL tourne déjà sur le port **5432** (port standard), alors que le `.env` de l'appli, lui, pointe sur **5433**. Un restart du service nous aiderait ?

```bash
sudo systemctl restart postgresql.service
# toujours Active: active (exited) via /bin/true, rien de concret
curl http://localhost
# même erreur, port 5433 introuvable
```

<figure><img src="../../../.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

Finalement la solution indiquait qu'il fallait relancer un service dédié, `db_connector`, qui fait office de vrai pont entre le backend web et PostgreSQL :

```bash
systemctl list-unit-files | grep db_connector
# db_connector.service   enabled   enabled
systemctl restart db_connector
```

Résolu.

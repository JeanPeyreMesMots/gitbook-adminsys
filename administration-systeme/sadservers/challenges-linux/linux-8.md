## ==Batumi==

### Reconnaissance

```bash
ps faux | grep "caddy"
# /usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
```

Conf Caddy simple, en reverse proxy vers un backend local :

```caddyfile
:80 {
    reverse_proxy localhost:5050
}
```

### Round 1 : le backend répond en 500

```bash
curl -vv -I http://localhost:5050
# HTTP/1.1 500 Internal Server Error
# Content-Length: 158 (zero-length body malgré le header)
```

Le backend sur le port 5050 répond mais en erreur. Check des logs du service Caddy via `journalctl` — la conf est bien chargée, tout semble démarrer normalement côté Caddy lui-même :

```bash
journalctl -u caddy.service -e
# using config from file /etc/caddy/Caddyfile
# server running, protocols h1/h2/h3
```

Un warning apparaît sur le formatage du Caddyfile (`caddy fmt --overwrite`), sans gravité apparente. Côté unités systemd, une deuxième unité liée existe mais est désactivée :

```bash
caddy-api.service   disabled  enabled
caddy.service       enabled   enabled
```

Activation par précaution :

```bash
sudo systemctl enable caddy-api.service
```

Statut de `caddy.service` re-vérifié : actif, rien d'anormal dans les logs. Donc Caddy lui-même n'est pas le problème — le souci vient bien du backend qu'il proxifie.

### Round 2 : le port 80 bloqué par iptables

```bash
sudo iptables -L
Chain INPUT (policy ACCEPT)
DROP  tcp  --  anywhere  anywhere  tcp dpt:http
```

Une règle DROP explicite bloque tout le trafic entrant sur le port 80. Suppression :

```bash
sudo iptables -D INPUT -p tcp --dport 80 -j DROP
```

### Round 3 : mauvais port pour PostgreSQL

Nouveau test, nouvelle erreur — cette fois côté base de données :

```bash
curl http://localhost
# could not connect to server: Connection refused
# Is the server running on host "127.0.0.1" and accepting TCP/IP connections on port 5433?
```

Coup d'œil dans le script Python du backend, qui interroge bien une base PostgreSQL :

```python
conn = psycopg2.connect(**db_params)
cursor.execute("SELECT secret FROM secrets WHERE id=1;")
```

Vérification du service PostgreSQL :

```bash
systemctl status postgresql.service
# Active: inactive (dead)
sudo systemctl start postgresql.service
sudo systemctl status postgresql.service
# Active: active (exited) — ExecStart=/bin/true
```

Le service "démarre" mais via `/bin/true`, donc ne fait littéralement rien — pas un vrai lancement de PostgreSQL. Malgré ça, l'erreur persiste. Vérification de ce qui écoute réellement :

```bash
sudo netstat -tunalp | grep postgres
# tcp  127.0.0.1:5432  LISTEN  1580/postgres
```

PostgreSQL tourne en fait déjà, mais sur le port **5432** (port standard), alors que le `.env` de l'appli pointe sur **5433**. Mismatch de configuration entre l'appli et la vraie base.

Tentative de redémarrage du service PostgreSQL pour voir si ça change quelque chose :

```bash
sudo systemctl restart postgresql.service
# toujours Active: active (exited) via /bin/true, rien de concret
curl http://localhost
# même erreur, port 5433 introuvable
```

Le redémarrage du service systemd (qui ne fait rien de réel, on l'a vu) ne pouvait évidemment rien changer.

### La vraie dernière étape : relancer le connecteur applicatif

La solution indiquait qu'il fallait relancer un service dédié, `db_connector`, apparemment le vrai pont entre le backend web et PostgreSQL :

```bash
systemctl list-unit-files | grep db_connector
# db_connector.service   enabled   enabled
systemctl restart db_connector
```

Résolu.
# 1 - Geneva, Tokyo, Marseille, Paris

### <mark style="color:$warning;">Geneva</mark>

**Objectif :** renouveler un certificat SSL nginx expiré.

On vérifie la validité du certificat en place :

```bash
echo | openssl s_client -connect localhost:443 2>/dev/null | openssl x509 -noout -dates
# notBefore=Feb 28 16:52:24 2025 GMT
# notAfter=Feb 29 16:52:24 2024 GMT
```

La date d'expiration est antérieure à la date du début (`notAfter` 2024 alors que `notBefore` est 2025). Le certificat est clairement expiré.

Dans la conf nginx les certificats sont stockés ici :

```nginx
ssl_certificate /etc/nginx/ssl/nginx.crt;
ssl_certificate_key /etc/nginx/ssl/nginx.key;
```

On génère alors nouveau certificat auto-signé, avec la commande suivante en ciblant les bons chemins :

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx.key \
  -out /etc/nginx/ssl/nginx.crt \
  -subj "/C=CH/ST=Geneva/L=Geneva/O=Acme/OU=IT Department/CN=localhost"
```

Ce qui résout le chall :)

### <mark style="color:$warning;">Tokyo</mark>

**Contexte :** Apache tourne, semble en bonne santé, mais reste injoignable.

```bash
sudo systemctl status apache2
# Active: active (running)
```

On déroule la méthodologie suivante, de la mine d'or qu'est le blog de Stephane Robert :

* https://blog.stephane-robert.info/docs/services/web/apache/#d%C3%A9pannage-express

On vérifie les permissions du fichier servi par précaution :

```bash
chmod 755 /var/www/html/index.html
```

Le process écoute bien sur le port 80 :

```bash
ss -tunap | grep "apache"
# tcp LISTEN *:80 ... apache2
```

Et la syntaxe est valide :

```bash
sudo apache2ctl configtest
# Syntax OK
```

Et les logs ne remontent rien d'anormal.

La vraie cause demeure en fait toujours le même truc que **SadServers** répète parfois, ce qui est un peu répétitif : iptables et une règle DROP qui foire tout

```bash
iptables -L
Chain INPUT (policy ACCEPT)
DROP  tcp  --  anywhere  anywhere  tcp dpt:http
```

On supprime :

```bash
sudo iptables -D INPUT 1
```

Résolu.

### <mark style="color:$warning;">Marseille</mark>

**Contexte :** une stack LAMP (Apache + PHP-FPM) sous Rocky Linux, qui échoue à traiter les requêtes PHP.

Premiers logs consultés :

```bash
cat /var/log/httpd/error_log
# [proxy:error] (13)Permission denied: AH00957: FCGI: attempt to connect to 127.0.0.1:9001 failed
# [proxy_fcgi:error] AH01079: failed to make connection to backend: 127.0.0.1
```

Deux pistes possibles à ce stade : SELinux (mentionné dans les logs de démarrage : _"SELinux policy enabled"_) ou un problème de configuration réseau/port entre Apache et PHP-FPM.

On vérifie le port réellement utilisé par PHP-FPM :

```bash
sudo grep -E "listen.*=" /etc/php-fpm.d/*.conf
# listen = 127.0.0.1:9000
```

Puis on compare avec la conf Apache :

```apache
<VirtualHost *:80>
    <FilesMatch \.php$>
        SetHandler "proxy:fcgi://127.0.0.1:9001"
    </FilesMatch>
</VirtualHost>
```

Apache tente de joindre PHP-FPM sur le port **9001**, alors que PHP-FPM écoute réellement sur **9000**. On corrige alors le port dans la conf Apache (9001 → 9000), puis :

```bash
sudo systemctl reload httpd
curl localhost | head -n1
```

Si le port est corrigé, la requête retourne mal la réponse. On peut déjà tester en désactivant temporairement SELinux pour juste confirmer le diagnostic, pour déjà comprendre et corriger la policy plutôt que de désactiver la protection :

```bash
sudo setenforce 0
curl localhost | head -n1
# "SadServers - LAMP Stack" (réponse attendue !)
```

Cela confirme que SELinux était bien en cause. On recherche la règle exacte bloquée plutôt que de laisser SELinux désactivé :

```bash
sudo ausearch -m avc -ts recent | grep denied
```

```
avc: denied { name_connect } for pid=1970 comm="httpd" dest=9000
scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:http_port_t:s0 tclass=tcp_socket
```

La policy SELinux par défaut interdit à Apache (`httpd_t`) les connexions sortantes vers d'autres process (`name_connect`), en l'occurrence ici vers PHP-FPM en écoute sur le 9000. Une soluce propre est d'activer le booléen SELinux dédié, plutôt qu'en désactivant SELinux globalement :

```bash
sudo setsebool -P httpd_can_network_connect on
sudo systemctl reload httpd
curl localhost | head -n1
# SadServers - LAMP Stack
```

Résolu, SELinux est réactivé et laissé en enforcing.

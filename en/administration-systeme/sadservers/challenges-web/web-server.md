# 1 - Geneva, Tokyo, Marseille, Paris

### <mark style="color:$warning;">Geneva</mark>

**Goal:** renew an expired nginx SSL certificate.

We check the validity of the current certificate:

```bash
echo | openssl s_client -connect localhost:443 2>/dev/null | openssl x509 -noout -dates
# notBefore=Feb 28 16:52:24 2025 GMT
# notAfter=Feb 29 16:52:24 2024 GMT
```

The expiry date is earlier than the start date (`notAfter` in 2024 while `notBefore` is in 2025), and is already in the past: the certificate is clearly invalid.

In the nginx config, the certificates are stored here:

```nginx
ssl_certificate /etc/nginx/ssl/nginx.crt;
ssl_certificate_key /etc/nginx/ssl/nginx.key;
```

We then generate a new self-signed certificate with the following command, targeting the right paths:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx.key \
  -out /etc/nginx/ssl/nginx.crt \
  -subj "/C=CH/ST=Geneva/L=Geneva/O=Acme/OU=IT Department/CN=localhost"
```

After reloading nginx so it picks up the new certificate, the challenge is solved :)

### <mark style="color:$warning;">Tokyo</mark>

**Context:** Apache is running and looks healthy, but can't be reached.

```bash
sudo systemctl status apache2
# Active: active (running)
```

We follow the troubleshooting method from Stéphane Robert's blog, a real gold mine (in French):

* https://blog.stephane-robert.info/docs/services/web/apache/#d%C3%A9pannage-express

We check the permissions of the served file just in case:

```bash
chmod 755 /var/www/html/index.html
```

The process does listen on port 80:

```bash
ss -tunap | grep "apache"
# tcp LISTEN *:80 ... apache2
```

And the syntax is valid:

```bash
sudo apache2ctl configtest
# Syntax OK
```

The logs don't show anything abnormal either.

The real cause turns out to be the same thing SadServers keeps reusing, which gets a bit repetitive: an iptables DROP rule breaking everything:

```bash
iptables -L
Chain INPUT (policy ACCEPT)
DROP  tcp  --  anywhere  anywhere  tcp dpt:http
```

We remove it:

```bash
sudo iptables -D INPUT 1
```

Solved.

### <mark style="color:$warning;">Marseille</mark>

**Context:** a LAMP stack (Apache + PHP-FPM) on Rocky Linux that fails to process PHP requests.

First logs checked:

```bash
cat /var/log/httpd/error_log
# [proxy:error] (13)Permission denied: AH00957: FCGI: attempt to connect to 127.0.0.1:9001 failed
# [proxy_fcgi:error] AH01079: failed to make connection to backend: 127.0.0.1
```

Two possible leads at this point: SELinux (mentioned in the startup logs: _"SELinux policy enabled"_), or a network/port configuration problem between Apache and PHP-FPM.

We check the port PHP-FPM actually uses:

```bash
sudo grep -E "listen.*=" /etc/php-fpm.d/*.conf
# listen = 127.0.0.1:9000
```

Then compare with the Apache config:

```apache
<VirtualHost *:80>
    <FilesMatch \.php$>
        SetHandler "proxy:fcgi://127.0.0.1:9001"
    </FilesMatch>
</VirtualHost>
```

Apache tries to reach PHP-FPM on port **9001**, while PHP-FPM actually listens on **9000**. We fix the port in the Apache config (9001 → 9000), then:

```bash
sudo systemctl reload httpd
curl localhost | head -n1
```

Even with the port fixed, the request still fails. To confirm the diagnosis, we temporarily switch SELinux to permissive mode, only to understand the problem and then fix the policy, rather than leaving the protection disabled:

```bash
sudo setenforce 0
curl localhost | head -n1
# "SadServers - LAMP Stack" (réponse attendue !)
```

This confirms SELinux was the culprit. We look for the exact denial rather than leaving SELinux off:

```bash
sudo ausearch -m avc -ts recent | grep denied
```

```
avc: denied { name_connect } for pid=1970 comm="httpd" dest=9000
scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:http_port_t:s0 tclass=tcp_socket
```

SELinux's default policy prevents Apache (`httpd_t`) from making outbound network connections (`name_connect`), here to PHP-FPM listening on 9000. A clean fix is to enable the dedicated SELinux boolean, rather than disabling SELinux globally:

```bash
sudo setsebool -P httpd_can_network_connect on
sudo systemctl reload httpd
curl localhost | head -n1
# SadServers - LAMP Stack
```

Solved, with SELinux switched back on and left in enforcing mode.
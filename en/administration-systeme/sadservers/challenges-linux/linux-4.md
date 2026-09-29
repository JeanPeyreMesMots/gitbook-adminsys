# 4 - Oaxaca, Melbourne & Lisbon

### <mark style="color:$warning;">Oaxaca</mark>

**Objectif :** fermer un fichier ouvert par un process, sans tuer ce process.

Direction Google direct : _"close a file without killing its process"_, qui mène à [ce thread superuser](https://superuser.com/questions/963612/closing-open-file-without-killing-the-process).

On regarde le fichier ouvert et le process qui le tient :

```bash
ll /home/admin/somefile
# -rw-r--r-- 1 admin admin 0 Mar 12 16:16 /home/admin/somefile

lsof /home/admin/somefile
# COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
# bash    1037 admin   77w   REG  259,1        0 272875 /home/admin/somefile
```

Le fichier est ouvert par `bash` (PID 1037) sur le descripteur `77w` (écriture).

On le voit avec `lsof -p` :

```bash
lsof -p 1037
# bash    1037 admin   77w   REG  259,1        0 272875 /home/admin/somefile
```

Si on le ferme :

```bash
exec 77w>&-
# -bash: exec: 77w: not found
```

Erreur de syntaxe, le `w` ne fait pas partie du numéro de descripteur, c'est juste un indicatif dans la sortie de `lsof`. Comme ça c'est mieux :

```bash
exec 77>&-
lsof -p 1037
# (plus rien listé sur ce fichier)
```

Le descripteur est fermé, avec le process bash toujours vivant. Plus simple que prévu au final.

### <mark style="color:$warning;">Melbourne</mark>

**Contexte :** une appli Python WSGI (`/home/admin/wsgi.py`) est censée sortir "**Hello, world!**", derrière Gunicorn, lui-même derrière nginx. Chaîne attendue : `curl → nginx → Gunicorn → wsgi.py`. Objectif : que `curl localhost` renvoie bien "Hello, world!".

Nginx éteint, on le rallume :

```bash
sudo systemctl status nginx
# Active: inactive (dead)
sudo systemctl start nginx
sudo systemctl status nginx
# Active: active (running)
```

Conf testée, syntaxe OK :

```bash
sudo nginx -t
# syntax ok, test successful
```

Mais toujours pas bon. Ça marchait avant mais plus maintenant :P :

```bash
curl http://localhost
# 502 Bad Gateway
```

Intéressons nous au fichier wsgi en question :

```python
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html'), ('Content-Length', '0'), ])
    return [b'Hello, world!']
```

Si on essaye de le lancer en background :

```bash
gunicorn wsgi:application --daemon
```

On se prend 502. Que disent les saints logs de nginx ?

```bash
cat /var/log/nginx/error.log
# connect() to unix:/run/gunicorn.socket failed (2: No such file or directory)
```

Ich, nginx qui cherche à joindre un socket qui n'existe pas. SAUF QUE, en regardant le status de Gunicorn, il était arrêté :

```bash
sudo systemctl status gunicorn
# Active: inactive (dead)
sudo systemctl start gunicorn
sudo systemctl status gunicorn
# Active: active (running)
```

Toujours 502 pourtant. Je m'oriente vers la piste du socket fantôme, mais en regardant de plus près :

```bash
ls -la /run/gunicorn.socket
# ls: cannot access '/run/gunicorn.socket': No such file or directory
ll /run/gunicorn.sock
# srw-rw-rw- 1 root root 0 Mar 12 18:17 /run/gunicorn.sock
```

En fait le socket réel s'appelle `gunicorn.sock` (sans le "**et**" final), pas `gunicorn.socket`. Cette coquille est visible dans la conf nginx :

```nginx
server {
    listen 80;
    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.socket;  # --> à corriger en "gunicorn.sock"
    }
}
```

On corrige, puis on redémarre les services concernés. Et là :

```bash
curl -I http://localhost
# HTTP/1.1 200 OK
# Content-Length: 0
```

Les headers passent, mais content-length à 0, donc rien en réponse.

Dans le **wsgi.py** lui-même le fichier déclare `Content-Length: 0` en dur alors qu'il retourne bien `b'Hello, world!'` d'où l'incohérence entre le header annoncé et le corps réel. J'ai quand même demandé à une IA de review le code, ce qui donne à la fin :

```python
def application(environ, start_response):
    status = '200 OK'
    output = b'Hello, world!'
    headers = [('Content-Type', 'text/html'), ('Content-Length', str(len(output)))]
    start_response(status, headers)
    return [output]
```

Un des changements appliqués dedans modifie `Content-Length` de façon à ce qu'il calcule dynamiquement la taille à partir du contenu retourné. On tape en locale :

```bash
curl http://localhost
# Hello, world!
```

### <mark style="color:$warning;">Lisbon</mark>

**Contexte :** serveur etcd avec, en apparence, un problème de certificat SSL.

```bash
ps faux | grep etcd
# /usr/bin/etcd --cert-file /etc/ssl/certs/localhost.crt --key-file /etc/ssl/certs/localhost.key --advertise-client-urls=https://localhost:2379 --listen-client-urls=https://localhost:2379
```

Je suis parti sur l'idée qu'il fallait renouveler le certificat SSL en fonction de la date système. J'ai enchaîné plusieurs tutos, tous orientés nginx / Let's Encrypt / certbot mais rien n'y faisait.

En changeant la date système à une date antérieure (1er janvier 2023), l'erreur de certificat disparaissait bien... mais une autre est apparue à la place :

```bash
sudo date -s 01/03/2023
etcdctl get foo
# Error: client: response is invalid json. The endpoint is probably not valid etcd cluster endpoint.
```

Donc le certificat n'était pas la root cause, juste un symptôme lié à la date, pas la cause racine.

On test direct les endpoints etcd en HTTPS :

```bash
curl https://localhost:2379/v2/keys/foo
# 404 Not Found (nginx)
curl https://localhost:2379/v2/
# 404 Not Found (nginx)
curl https://localhost:2379/
# Testing SSL
```

Toujours bloque en "Testing SSL...". Vérification de la conf nginx : rien d'anormal en apparence (écoute sur 443, config syntaxiquement valide) :

```bash
sudo nginx -t
# syntax ok, test successful
```

Direction les règles iptables, notamment la table NAT :

```bash
sudo iptables -t nat -L
```

```
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
REDIRECT   tcp  --  anywhere             anywhere             tcp dpt:2379 redir ports 443
```

Trouvé : **tout** le trafic TCP à destination du port 2379 (etcd) est forwardé vers le port 443 par iptables. C'est cette règle qui causait les 404 nginx à la place des réponses etcd.&#x20;

On vire les règles de redirection sur la chaîne OUTPUT de la table NAT :

```bash
sudo iptables -t nat -F OUTPUT
```

Puis on regarde si on a du neuf :

```bash
curl https://localhost:2379/v2/keys/foo
# {"action":"get","node":{"key":"/foo","value":"bar","modifiedIndex":4,"createdIndex":4}}
```

Résolu.

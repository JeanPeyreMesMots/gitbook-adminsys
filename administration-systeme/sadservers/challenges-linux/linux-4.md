## <mark>Oaxaca</mark>
**Objectif :** fermer un fichier ouvert par un process, sans tuer ce process.

Direction Google direct : _"close a file without killing its process"_, qui mène à [ce thread superuser](https://superuser.com/questions/963612/closing-open-file-without-killing-the-process).

Identification du fichier ouvert et du process qui le tient :

```bash
ll /home/admin/somefile
# -rw-r--r-- 1 admin admin 0 Mar 12 16:16 /home/admin/somefile

lsof /home/admin/somefile
# COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
# bash    1037 admin   77w   REG  259,1        0 272875 /home/admin/somefile
```

Le fichier est ouvert par `bash` (PID 1037) sur le descripteur `77w` (écriture).

Confirmation avec `lsof -p` :

```bash
lsof -p 1037
# bash    1037 admin   77w   REG  259,1        0 272875 /home/admin/somefile
```

Tentative de fermeture du descripteur directement depuis le shell :

```bash
exec 77w>&-
# -bash: exec: 77w: not found
```

Erreur de syntaxe — le `w` ne fait pas partie du numéro de descripteur, juste indicatif dans la sortie de `lsof`. Bonne syntaxe :

```bash
exec 77>&-
lsof -p 1037
# (plus rien listé sur ce fichier)
```

Descripteur fermé, process bash toujours vivant. Plus simple que prévu au final.
## <mark>Melbourne</mark>
**Contexte :** une appli Python WSGI (`/home/admin/wsgi.py`) censée servir "Hello, world!", derrière Gunicorn, lui-même derrière nginx. Chaîne attendue : `curl → nginx → Gunicorn → wsgi.py`. Objectif : que `curl localhost` renvoie bien "Hello, world!".

### Étape 1 : nginx éteint

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

Mais toujours pas bon :

```bash
curl http://localhost
# 502 Bad Gateway
```

### Étape 2 : Gunicorn pas lancé

Lecture du fichier wsgi :

```python
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html'), ('Content-Length', '0'), ])
    return [b'Hello, world!']
```

Lancement manuel de Gunicorn en arrière-plan :

```bash
gunicorn wsgi:application --daemon
```

Toujours 502. Check des logs nginx pour comprendre :

```bash
cat /var/log/nginx/error.log
# connect() to unix:/run/gunicorn.socket failed (2: No such file or directory)
```

nginx cherche à joindre un socket qui n'existe pas.

### Étape 3 : service Gunicorn down

En fait un vrai service systemd Gunicorn existe déjà, mais était arrêté :

```bash
sudo systemctl status gunicorn
# Active: inactive (dead)
sudo systemctl start gunicorn
sudo systemctl status gunicorn
# Active: active (running)
```

Toujours 502 pourtant. J'ai demandé à une IA à ce stade, qui a confirmé qu'on cherchait à joindre `/run/gunicorn.socket` — mais en listant moi-même le dossier, j'ai trouvé le vrai problème :

```bash
ls -la /run/gunicorn.socket
# ls: cannot access '/run/gunicorn.socket': No such file or directory
ll /run/gunicorn.sock
# srw-rw-rw- 1 root root 0 Mar 12 18:17 /run/gunicorn.sock
```

Le socket réel s'appelle `gunicorn.sock` (sans le "et" final), pas `gunicorn.socket`. Coquille dans la conf nginx :

```nginx
server {
    listen 80;
    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.socket;  # --> à corriger en "gunicorn.sock"
    }
}
```

Correction, puis redémarrage des services concernés. Nouveau test :

```bash
curl -I http://localhost
# HTTP/1.1 200 OK
# Content-Length: 0
```

Les headers passent, mais content-length à 0 — donc pas de corps de réponse.

### Étape 4 : le wsgi.py lui-même

Le fichier déclare `Content-Length: 0` en dur alors qu'il retourne bien `b'Hello, world!'` — incohérence entre le header annoncé et le corps réel. Sans connaître les détails de l'API WSGI, demandé à une IA de relire le fichier :

```python
def application(environ, start_response):
    status = '200 OK'
    output = b'Hello, world!'
    headers = [('Content-Type', 'text/html'), ('Content-Length', str(len(output)))]
    start_response(status, headers)
    return [output]
```

La correction calcule dynamiquement `Content-Length` à partir de la taille réelle du corps, plutôt que de le figer à 0. Rien de dangereux à appliquer même en prod. Test final :

```bash
curl http://localhost
# Hello, world!
```

Résolu : 4 couches de la stack (nginx down, Gunicorn down, mauvais nom de socket, bug applicatif) à débloquer une par une.
## <mark>Lisbon</mark>
**Contexte :** serveur etcd avec, en apparence, un problème de certificat SSL.

```bash
ps faux | grep etcd
# /usr/bin/etcd --cert-file /etc/ssl/certs/localhost.crt --key-file /etc/ssl/certs/localhost.key --advertise-client-urls=https://localhost:2379 --listen-client-urls=https://localhost:2379
```

### Fausse piste : renouveler le certificat

Je suis parti sur l'idée qu'il fallait renouveler le certificat SSL en fonction de la date système. J'ai enchaîné plusieurs tutos, tous orientés nginx / Let's Encrypt / certbot — impossible d'ailleurs d'installer certbot sur la machine.

En changeant la date système à une date antérieure (1er janvier 2023), l'erreur de certificat disparaissait effectivement... mais une autre est apparue à la place :

```bash
sudo date -s 01/03/2023
etcdctl get foo
# Error: client: response is invalid json. The endpoint is probably not valid etcd cluster endpoint.
```

Donc le certificat n'était pas vraiment le problème central — juste un symptôme lié à la date, pas la cause racine.

### La vraie piste : du routage, pas du SSL

Test direct des endpoints etcd en HTTPS :

```bash
curl https://localhost:2379/v2/keys/foo
# 404 Not Found (nginx)
curl https://localhost:2379/v2/
# 404 Not Found (nginx)
curl https://localhost:2379/
# Testing SSL
```

La racine répond ("Testing SSL"), mais pas les endpoints etcd attendus — et surtout, c'est **nginx** qui répond, pas etcd. Suspect : le port 2379 (port standard etcd) semble en fait être intercepté par autre chose.

Vérification de la conf nginx : rien d'anormal en apparence (écoute sur 443, config syntaxiquement valide) :

```bash
sudo nginx -t
# syntax ok, test successful
```

Direction les règles iptables, notamment la table NAT :

```bash
sudo iptables -t nat -L
```

```text
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
REDIRECT   tcp  --  anywhere             anywhere             tcp dpt:2379 redir ports 443
```

Trouvé : **tout** le trafic TCP à destination du port 2379 (etcd) est redirigé vers le port 443 par iptables. C'est cette règle, et non un souci de certificat, qui causait les 404 nginx à la place des réponses etcd.

### Correction

Suppression des règles de redirection sur la chaîne OUTPUT de la table NAT :

```bash
sudo iptables -t nat -F OUTPUT
```

Vérification que la règle a bien disparu, puis nouveau test :

```bash
curl https://localhost:2379/v2/keys/foo
# {"action":"get","node":{"key":"/foo","value":"bar","modifiedIndex":4,"createdIndex":4}}
```

Résolu — le certificat n'avait jamais été le vrai problème.
# 6 - Paris, Manado & Moyogalpa

### <mark style="color:$warning;">Paris</mark>

Un peu de hacking, comme au bon vieux temps :D

#### Reconnaissance

Process en cours :

```bash
ps -faux | grep "python"
# root  693  ... /usr/bin/python3 /home/admin/webserver.py
```

Test de l'endpoint :

```bash
curl -v http://localhost:5000
# HTTP/1.1 200 OK
# Server: Werkzeug/3.1.4 Python/3.13.5
# ...
Unauthorized
```

Réponse 200 mais contenu "Unauthorized" — donc pas une vraie 401, l'appli gère l'auth elle-même dans le corps de la réponse.

Petit script bash pour tester une liste de couples login/mdp courants (admin:admin, root:root, guest:guest, etc.) :

```bash
for cred in "${creds[@]}"; do
  username="${cred%%:*}"
  password="${cred#*:}"
  response=$(curl -s -u "$username:$password" "$URL")
  if [[ ! "$response" =~ Unauthorized ]]; then
    echo "SUCCESS! avec $username:$password"
    exit 0
  fi
done
```

Aucun succès. Recherche d'exploits connus sur le stack utilisé : rien de concluant non plus.

#### La faille : header User-Agent absent

Idée testée un peu au hasard : retirer complètement le header `User-Agent` de la requête.

```bash
curl -v -u admin:admin -H "User-Agent:" http://localhost:5000
# HTTP/1.1 200 OK
# Content-Length: 35
Welcome! Password is FDZPmh5AX3oiJt
```

Bingo, sans User-Agent l'appli laisse passer un mot de passe en clair dans la réponse.

On spray le mdp avec différents logins (admin, root, sad, sadservers, guest...) : à chaque fois la même réponse "**Welcome**!" revient, peu importe le login utilisé.

Donc ce n'était pas un vrai identifiant de connexion, mais très probablement directement la solution du challenge :

```bash
echo "FDZPmh5AX3oiJt" > ~/mysolution
```

Confirmé :)

### <mark style="color:$warning;">Manado</mark>

_(Note prise rapidement, à détailler plus tard)_

Exercice autour de la commande `sort` et de la compression `xz` :

```bash
sort names > names_COPY
ll
# -rw-r--r-- 1 root  root  35147 Mar  2  2024 names
# -rw-r--r-- 1 admin admin 35148 Mar 23 16:43 names_COPY
```

On compresse :

```bash
xz -k names_COPY
# crée names_COPY.xz (9328 octets), garde l'original grâce à -k
rm names_COPY.xz
```

Tentative avec le niveau de compression maximal :

```bash
xz -9 names_COPY
# xz: names_COPY: Cannot allocate memory
```

Échec, pas assez d'espace disque. Essayons avec le niveau 5 :

```bash
xz -5 names_COPY
ll
# names_COPY.xz  9336 octets
```

Ça fonctionne, on copie le résultat dans le dossier solution  :

```bash
cp names_COPY.xz solution/
```

### <mark style="color:$warning;">Moyogalpa</mark>

**Contexte :** une appli en Go sécurisée par John et Mike, cassée par eux. Le chall nous donne un cahier des charges :

* communication uniquement en HTTPS ;
* accès limité aux seuls fichiers nécessaires (certificats + fichiers statiques) ;
* rate limiting à 10 requêtes/seconde ;
* exécution sous utilisateur non-root.

Au début, pourquoi j'ai galéré car l'appli n'était pas lancée à la main (via un `go run ...`), mais gérée comme service systemd. J'ai mis un moment à me décider qu'il fallait regarder les logs via `journalctl -u webapp` plutôt que chercher un process lancé manuellement.

```bash
sudo journalctl -u webapp
# open /home/webapp/pki/server.crt: permission denied
# open /home/webapp/pki/server.pem: permission denied
# can not access certificate/key file. sleeping for 10s and will retry
```

Déjà les permissions sur les certificats sont pas ok, on commence par les corriger :

```bash
ll /home/webapp/pki/
# ls: cannot open directory '/home/webapp/pki/': Permission denied
ll /home/webapp/
# drwx------ 2 root root 4096 Apr 10 2024 pki
```

```bash
sudo chmod -R 755 pki/
sudo chown -R admin: pki/
```

Mais ça veut pas :

```bash
open /home/webapp/pki/server.pem: permission denied
```

Avec une boucle openssh on peut tester la validité de chaque chaque fichier de certificat :

```bash
for cert in *.crt *.pem; do
    openssl x509 -in "$cert" -noout -dates 2>/dev/null || echo "Pas un certificat"
done
# CA.crt : OK
# server.crt : OK
# server.pem : Pas un certificat
```

`server.pem` n'est pas reconnu comme certificat. En regardant son contenu :

```bash
head server.pem
# -----BEGIN RSA PRIVATE KEY-----
```

Logique : il s'agit d'une clé privée RSA, pas d'un certificat, donc `openssl x509` ne peut pas le lire. On vérifie quand même que la clé est valide et qu'elle correspond bien au certificat :

```bash
openssl rsa -in server.pem -check -noout
# RSA key ok

openssl rsa -noout -modulus -in server.pem | openssl md5
openssl x509 -noout -modulus -in server.crt | openssl md5
# mêmes hash des deux côtés → la paire clé/certificat est cohérente
```

Les fichiers eux-mêmes sont sains. J'ai ajusté les permissions sur les fichiers statiques par précaution :

```bash
sudo chmod -R 755 static-files/
```

Mais sans effets sur le problème principal.

Après avoir remis les fichiers au bon proprio (`webapp:webapp` plutôt qu'`admin`), et cherché la solution en ligne, j'ai découvert qu'il fallait mettre le certificat dans le dossier système " _**/usr/local/share/ca-certificates/**_" pour éviter d'avoir à utiliser `--cacert` à chaque requête :

```bash
sudo cp /home/webapp/pki/CA.crt /usr/local/share/ca-certificates/webappCA.crt
sudo chmod 644 /usr/local/share/ca-certificates/webappCA.crt
sudo update-ca-certificates
```

On yest :

```bash
curl https://webapp:7000
# curl: (6) Could not resolve host: webapp
```

On va y arriver :D Le nom d'hôte `webapp` se résout pas car pas présent dans /etc/hosts, on l'ajoute dedans :

```bash
echo "127.0.0.1 webapp" | sudo tee --append /etc/hosts
```

Nouvelle erreur, différente cette fois-ci :

```bash
curl https://webapp:7000
# Forbidden
```

Hop là les logs :

```bash
open /home/webapp/static-files/users.html: permission denied
```

Et là, le vraie coupable se montre. En cherchant le message "**permission denied**" sur Google, l'application est confinée par un profil AppArmor (`/etc/apparmor.d/usr.local.bin.webapp`), qui autorisait l'accès aux certificats mais pas aux fichiers statiques. Le profil se présente comme suit :

```
/home/webapp/pki/ r,
/home/webapp/pki/server.pem r,
/home/webapp/pki/server.crt r,
# manquant : accès à static-files
```

On ajoute les lignes nécessaires :

```
/home/webapp/static-files/ r,
/home/webapp/static-files/* r,
```

Et on reload :

```bash
apparmor_parser -r /etc/apparmor.d/usr.local.bin.webapp
```

Test final :

```bash
curl https://webapp:7000/users.html
# <p>From Users Page</p>
```

Résolu. Dans l'ordre on a réglé les permissions classiques, vérifié les certificats, la DNS locale, et enfin le AppArmor.


## ==Paris==

Un peu de hacking, comme au bon vieux temps :D

### Reconnaissance

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

### Tentative brute-force sur des creds classiques

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

### La faille : header User-Agent absent

Idée testée un peu au hasard : retirer complètement le header `User-Agent` de la requête.

```bash
curl -v -u admin:admin -H "User-Agent:" http://localhost:5000
# HTTP/1.1 200 OK
# Content-Length: 35
Welcome! Password is FDZPmh5AX3oiJt
```

Bingo — sans User-Agent, l'appli laisse fuiter un mot de passe en clair dans la réponse. Manifestement une vérification de sécurité mal branchée qui dépend (à tort) du User-Agent plutôt que des vraies credentials.

Tentatives de spray le mdp avec différents logins (admin, root, sad, sadservers, guest...) : à chaque fois la même réponse "**Welcome**!" revient, peu importe le login utilisé. 

Donc ce n'était pas un vrai identifiant de connexion, mais très probablement directement la solution du challenge :

```bash
echo "FDZPmh5AX3oiJt" > ~/mysolution
```

Confirmé :) :

![[Pasted image 20260731221201.png|390]]
## ==Manado==

_(Note prise rapidement, à détailler plus tard)_

Exercice autour de la commande `sort` et de la compression `xz` :

```bash
sort names > names_COPY
ll
# -rw-r--r-- 1 root  root  35147 Mar  2  2024 names
# -rw-r--r-- 1 admin admin 35148 Mar 23 16:43 names_COPY
```

Compression standard pour test :

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

Échec — pas assez de mémoire disponible pour ce niveau de compression sur cette machine. Repli sur un niveau intermédiaire :

```bash
xz -5 names_COPY
ll
# names_COPY.xz  9336 octets
```

Ça fonctionne. Copie du résultat dans le dossier solution attendu :

```bash
cp names_COPY.xz solution/
```

## ==Moyogalpa==

**Contexte (via `/home/README.txt`) :** une application Golang sécurisée par John et Mike, cassée par leurs propres mesures de sécurité. Contraintes à respecter :

- communication uniquement en HTTPS ;
- accès limité aux seuls fichiers nécessaires (certificats + fichiers statiques) ;
- rate limiting à 10 requêtes/seconde ;
- exécution sous utilisateur non-root.

### Pourquoi j'ai galéré au début

L'appli n'était pas lancée à la main (`go run ...`), mais gérée comme service systemd — j'ai mis un moment à réaliser qu'il fallait donc regarder les logs via `journalctl -u webapp` plutôt que chercher un process lancé manuellement.

```bash
sudo journalctl -u webapp
# open /home/webapp/pki/server.crt: permission denied
# open /home/webapp/pki/server.pem: permission denied
# can not access certificate/key file. sleeping for 10s and will retry
```

### Round 1 : permissions sur les certificats

```bash
ll /home/webapp/pki/
# ls: cannot open directory '/home/webapp/pki/': Permission denied
ll /home/webapp/
# drwx------ 2 root root 4096 Apr 10 2024 pki
```

Correction des permissions et du propriétaire :

```bash
sudo chmod -R 755 pki/
sudo chown -R admin: pki/
```

Toujours en échec derrière :

```bash
open /home/webapp/pki/server.pem: permission denied
```

### Round 2 : vérifier les certificats eux-mêmes

Test de validité avec openssl sur chaque fichier :

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

Logique — c'est une clé privée RSA, pas un certificat, donc `openssl x509` ne peut pas le lire (mauvais outil pour ce type de fichier). Vérification que la clé est valide et correspond bien au certificat :

```bash
openssl rsa -in server.pem -check -noout
# RSA key ok

openssl rsa -noout -modulus -in server.pem | openssl md5
openssl x509 -noout -modulus -in server.crt | openssl md5
# mêmes hash des deux côtés → la paire clé/certificat est cohérente
```

Donc les fichiers eux-mêmes étaient sains — le souci devait venir d'ailleurs, probablement du code Go de l'application. Permissions ajustées sur les fichiers statiques par précaution également, sans effet sur le problème principal :

```bash
sudo chmod -R 755 static-files/
```

### Round 3 : propriétaire correct + DNS local

Après avoir remis les fichiers au bon propriétaire (`webapp:webapp` plutôt qu'`admin`), et cherché la solution en ligne, découverte qu'il fallait enregistrer le certificat CA dans le magasin système pour éviter d'avoir à utiliser `--cacert` à chaque requête :

```bash
sudo cp /home/webapp/pki/CA.crt /usr/local/share/ca-certificates/webappCA.crt
sudo chmod 644 /usr/local/share/ca-certificates/webappCA.crt
sudo update-ca-certificates
```

Test :

```bash
curl https://webapp:7000
# curl: (6) Could not resolve host: webapp
```

Le nom d'hôte `webapp` n'était tout simplement pas résolu — ajout dans `/etc/hosts` :

```bash
echo "127.0.0.1 webapp" | sudo tee --append /etc/hosts
```

Nouvelle erreur, différente :

```bash
curl https://webapp:7000
# Forbidden
```

Logs :

```bash
open /home/webapp/static-files/users.html: permission denied
```

### Round 4 : AppArmor, la vraie cause finale

Aucune idée que ça pouvait venir de là — trouvé en cherchant la solution que l'application est confinée par un profil AppArmor (`/etc/apparmor.d/usr.local.bin.webapp`), qui autorisait explicitement l'accès aux certificats mais pas aux fichiers statiques. Le profil :

```text
/home/webapp/pki/ r,
/home/webapp/pki/server.pem r,
/home/webapp/pki/server.crt r,
# manquant : accès à static-files
```

Ajout des lignes nécessaires :

```text
/home/webapp/static-files/ r,
/home/webapp/static-files/* r,
```

Rechargement du profil :

```bash
apparmor_parser -r /etc/apparmor.d/usr.local.bin.webapp
```

Test final :

```bash
curl https://webapp:7000/users.html
# <p>From Users Page</p>
```

Résolu — après permissions classiques, vérification des certificats, résolution DNS locale, et enfin un mécanisme de confinement (AppArmor) totalement absent des premières hypothèses.
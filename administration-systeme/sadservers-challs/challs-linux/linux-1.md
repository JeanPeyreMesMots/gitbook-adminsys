# SadServers – Linux Challenges

Solutions à des challenges de troubleshooting Linux sur la plateforme [SadServers](https://sadservers.com/) — accès SSH à une VM avec un problème réel à résoudre : permissions, services cassés, scripts, etc.

---

## Apia

**Objectif :** un mot a été ajouté dans un fichier parmi une centaine, il faut le retrouver et fournir la solution sous forme de hash MD5.

On commence par lister tous les fichiers :

```bash
ls -l
total 404
-rw-r--r-- 1 admin admin 1046 Feb 25  2024 file0.txt
-rw-r--r-- 1 admin admin 1046 Feb 25  2024 file1.txt
-rw-r--r-- 1 admin admin 1046 Feb 25  2024 file10.txt
```

Puisqu'on sait qu'un seul mot a été ajouté, un seul fichier doit avoir sa taille changée. En triant par taille décroissante, le fichier modifié ressort direct en première ligne :

```bash
ls -lS
-rw-r--r-- 1 admin admin 1054 Feb 25  2024 file76.txt
```

Pour repérer le mot ajouté, `vimdiff` avec n'importe quel autre fichier comme référence (ici `file0.txt`) :

```bash
vimdiff file76.txt file0.txt
```

**eureka**, le mot est trouvé. Plus qu'à le mettre dans la solution :

```bash
echo "eureka" > /home/admin/solution
```

Et vérifier que le hash correspond à celui attendu par le chall :

```bash
md5sum /home/admin/solution
55aba155290288b58e9b778c8f616560  /home/admin/solution
```

C'est bien le cas.

### Concepts clés

- `ls -lS` : tri par taille décroissante, permet d'isoler rapidement un fichier modifié dans un lot homogène.
- `vimdiff` : diff visuel entre deux fichiers, pratique pour spotter une différence ligne à ligne.

### À retenir

- Trier par taille est une méthode rapide et bête pour isoler un fichier modifié dans un ensemble de fichiers similaires — pas besoin de tout comparer un par un.

---

## Tokamachi

**Objectif :** un writer doit envoyer des messages en continu dans un named pipe (`/home/admin/namedpipe`), un reader doit les capturer avec timestamp dans `/home/admin/reader.log`.

Le reader tourne déjà, avec un délai de 2 secondes entre chaque lecture :

```bash
nohup /bin/bash -c 'while true; do
  if read line < /home/admin/namedpipe; then
    echo "$(date) Received: $line" >> /home/admin/reader.log
  fi
  sleep 2
done' &>/dev/null &
```

Le writer proposé par défaut par SadServers n'a aucun délai :

```bash
/bin/bash -c 'while true; do echo "this is a test message being sent to the pipe" > /home/admin/namedpipe; done' &
```

### Essais ratés

J'ai d'abord essayé de corriger la syntaxe du writer comme suggéré, avec un `sleep 3` (délai plus grand que le sleep 2 du reader) :

```bash
/bin/bash -c 'while true; do if read line < /home/admin/namedpipe; then echo "$(date) Received: $line" >> /home/admin/reader.log; fi; sleep 3; done'
```

Ça ne marchait pas non plus.

Puis avec indentation et `2>/dev/null` :

```bash
/bin/bash -c 'while true; do if read line < /home/admin/namedpipe 2>/dev/null; then echo "$(date) Received: $line" >> /home/admin/reader.log; fi; sleep 2; done'
```

Log vide, et SadServers n'acceptait pas la solution.

### Ce qui a marché

Après avoir tué l'ancien processus writer (`ps` + `grep "pipe"` pour trouver le PID, puis `kill`), la commande qui a fonctionné utilise `nohup` :

```bash
nohup /bin/bash -c 'while true; do echo "this is a test message being sent to the pipe" > /home/admin/namedpipe; sleep 2; done' &
```

`nohup` garde le processus en vie après déconnexion, c'est là toute la différence.

> `nohup` (« no hang up ») exécute une commande ou un script de manière à ce qu'il continue de s'exécuter même si la session terminal qui l'a lancé est fermée ou interrompue. Il détache la commande de la session courante et la place dans un processus indépendant, garantissant la persistance de son exécution. — [zonetuto.fr](https://zonetuto.fr/shell-bash/nohup-lancer-un-script-en-arriere-plan-sur-un-serveur-linux/)

### Concepts clés

- **nohup** : détache un processus de la session terminal, il survit à la fermeture du terminal.
- Un named pipe demande une synchro cohérente entre writer et reader (délais compatibles), sinon ça part en vrille silencieusement.

### Erreurs fréquentes

- Writer sans délai → écriture trop rapide, désynchro avec le reader.
- Oublier de tuer l'ancien processus writer avant d'en relancer un autre → conflit sur le pipe, résultats incohérents.

### À retenir

- Face à un service censé tourner en arrière-plan indéfiniment, `nohup` est souvent le chaînon manquant, pas juste un détail de confort.

---

## Yokohama

**Objectif :** gestion de permissions pour 4 users (abe, betty, carlos, debora) :

- chacun modifie son propre fichier ;
- chacun ne peut pas _lire_ les fichiers des autres, mais peut y _écrire_ ;
- tous peuvent modifier le contenu de `shared/project_ALL`, mais pas la première ligne.

On a les accès root, donc on peut ajuster les droits librement.

État initial :

```bash
ls -l /home/admin/shared
total 20
-rw-r--r-- 1 root   admin  38 Feb  2  2025 ALL
-rw-r----- 1 abe    abe    27 Feb  2  2025 project_abe
-rw-r----- 1 betty  betty  29 Feb  2  2025 project_betty
-rw-r----- 1 carlos carlos 30 Feb  2  2025 project_carlos
-rw-r----- 1 debora debora 30 Feb  2  2025 project_debora
```

Test en se connectant avec un autre user (debora) : ni lecture ni écriture possible sur les fichiers des autres.

```bash
sudo su - debora
cd /home/admin/shared/
cat *
First line in the shared project file
cat: project_abe: Permission denied
cat: project_betty: Permission denied
cat: project_carlos: Permission denied
"This is debora's project file"
```

### Procédure

Pour gagner du temps sur le calcul des octales, outil pratique : [chmod-calculator.com](https://chmod-calculator.com/).

Pour les fichiers `project_USER`, le bit d'exécution n'est pas nécessaire — on ajoute read + write pour le groupe qui porte le même nom que le user. Connexion avec chaque user pour appliquer les permissions correspondantes, exemple avec debora :

```bash
cd /home/admin/shared/
ls -l
total 20
-rw-r--r-- 1 root   admin  38 Feb  2  2025 ALL
-rw-rw-r-- 1 abe    abe    27 Feb  2  2025 project_abe
-rw-rw-r-- 1 betty  betty  29 Feb  2  2025 project_betty
-rw-rw-r-- 1 carlos carlos 30 Feb  2  2025 project_carlos
-rw-r----- 1 debora debora 30 Feb  2  2025 project_debora

chmod 664 project_debora

ls -l
-rw-rw-r-- 1 debora debora 30 Feb  2  2025 project_debora
```

Répété pour chaque user. Cette approche fichier par fichier n'est pas à favoriser — mieux vaut passer par un groupe partagé :

```bash
# 1. Créer un groupe commun pour tous les utilisateurs
sudo groupadd projectusers

# 2. Ajouter TOUS les utilisateurs au groupe
sudo usermod -aG projectusers abe
sudo usermod -aG projectusers debora
# ... (idem betty, carlos)

# 3. Appliquer les permissions via le groupe
sudo chown :projectusers /shared/project_ALL
sudo chmod 664 /shared/project_ALL        # rw pour owner+group
```

Pour permettre à tous les users d'ajouter du contenu à `/home/admin/shared/project_ALL`, il faut un accès lecture/écriture pour tout le monde → 666 :

```bash
-rw-rw-rw- 1 root   admin  38 Feb  2  2025 ALL
```

Et enfin, `chattr +a` (« append ») pour ne permettre que l'ajout de contenu dans `ALL`, sans possibilité de modifier ce qui existe déjà (dont la première ligne) :

```bash
lsattr *
-----a--------e------- ALL
--------------e------- project_abe
--------------e------- project_betty
--------------e------- project_carlos
--------------e------- project_debora
```

Done.

### Concepts clés

- **Groupe partagé** : plus scalable que d'ajuster chaque fichier individuellement dès qu'on a plusieurs users à gérer.
- **`chattr +a`** : attribut système qui restreint à l'ajout de contenu (append-only), empêche la modification ou suppression du contenu existant — indépendant des permissions rwx classiques.

### Erreurs fréquentes / point à surveiller

- Les permissions `664` (rw pour le groupe) autorisent aussi la _lecture_ par le groupe, ce qui ne colle pas exactement à la contrainte initiale ("ne pas lire, mais pouvoir écrire"). Le chmod classique ne permet pas nativement d'accorder l'écriture sans la lecture pour un groupe — ça demanderait des ACL (`setfacl`) pour restreindre plus finement. À garder en tête si la consigne exacte est réellement "write-only".

### À retenir

- `chattr` complète bien les permissions Unix classiques quand on veut un contrôle plus fin (append-only, immutable, etc.) que rwx seul.

---

## Fukuoka

**Objectif :** un serveur nginx renvoie une 404 par défaut au lieu de servir un fichier avec le message _"Welcome to the Real Site!"_.

```bash
curl localhost
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.18.0</center>
</body>
</html>
```

Un rapide coup d'œil dans la conf nginx pour savoir où chercher les logs :

```bash
cat /etc/nginx/nginx.conf
[...]
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;
}
```

Boom, une erreur sur `/var/www/html`, le répertoire qui sert les fichiers :

```bash
cat /var/log/nginx/error.log
2026/03/06 21:04:42 [crit] 618#618: *1 stat() "/var/www/html/" failed (13: Permission denied), client: 127.0.0.1, server: _, request: "GET / HTTP/1.1", host: "localhost"
```

En vérifiant, on n'a effectivement pas les droits, et le répertoire a l'air cassé :

```bash
ll /var/www/html
ls: cannot access '/var/www/html': Permission denied
ll /var/www/
total 0
d????????? ? ? ? ?            ? html
```

`/var/www/html` est censé appartenir au groupe `www-data`, utilisé par nginx pour servir ses fichiers. On corrige :

```bash
sudo chown -R www-data:www-data www/
sudo chmod -R 755 www/
```

```bash
ll
drwxr-xr-x  3 www-data www-data 4096 Jul 21  2025 www
```

Reload + restart du service :

```bash
sudo systemctl reload nginx
sudo systemctl restart nginx
```

Pour autant nginx ne veut toujours pas. On a maintenant un 403 à la place d'un 404 — toujours une erreur d'accès, mais sur le fichier html principal cette fois :

```bash
curl localhost
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.18.0</center>
</body>
</html>
```

Et pour cause : le lien symbolique `index.html` pointe vers un fichier qui n'a pas les bonnes permissions :

```bash
ll
lrwxrwxrwx 1 www-data www-data  33 Jul 21  2025 index.html -> /opt/site-content/real_index.html
-rwxr-xr-x 1 www-data www-data 612 Jul 21  2025 index.nginx-debian.html

ll /opt/site-content/real_index.html
-rw-r----- 1 root root 34 Jul 21  2025 /opt/site-content/real_index.html
```

Confirmé dans les logs :

```bash
cat /var/log/nginx/error.log
2026/03/06 21:12:25 [error] 928#928: *1 open() "/var/www/html/index.html" failed (13: Permission denied), client: 127.0.0.1, server: _, request: "GET / HTTP/1.1", host: "localhost"
```

Correction des permissions du fichier cible réel :

```bash
sudo chown root:www-data /opt/site-content/real_index.html
sudo chmod 640 /opt/site-content/real_index.html
```

```bash
ll /opt/site-content/real_index.html
-rw-r----- 1 root www-data 34 Jul 21 2025 /opt/site-content/real_index.html
```

nginx (`www-data`) ne doit que _lire_ le fichier. Seul `root` peut l'éditer, `www-data` y accède en lecture via le groupe.

Et pour finir :

```bash
curl localhost
# Welcome to the Real Site!
```

### Concepts clés

- **www-data** : user/groupe système dédié à nginx, doit avoir accès (au moins en lecture) aux fichiers servis.
- Un lien symbolique hérite de ses propres permissions, mais l'accès final dépend de la **cible réelle** — deux points de vérification distincts.
- Principe de moindre privilège : nginx n'a besoin que d'un accès lecture sur les fichiers servis, pas d'écriture.

### Erreurs fréquentes

- Corriger les permissions du répertoire sans vérifier celles du fichier pointé par le lien symbolique (d'où le passage 404 → 403 plutôt qu'une résolution directe).
- Ne pas consulter `error.log` à chaque étape, alors qu'il donne la cause exacte (permission denied + chemin concerné) à chaque fois.

### À retenir

- Toujours croiser `curl` (symptôme) et `error.log` (cause exacte) pour diagnostiquer un problème nginx — le code HTTP seul ne suffit pas à localiser le problème.
- Un `Permission denied` sur un lien symbolique peut venir du lien ou de sa cible : vérifier les deux séparément.

---

## Points à vérifier

- **Yokohama :** les permissions `664` appliquées autorisent la lecture par le groupe, ce qui contredit partiellement la contrainte "pas de lecture, écriture seulement". Si cette contrainte était stricte, des ACL (`setfacl`) auraient été nécessaires plutôt qu'un chmod classique.
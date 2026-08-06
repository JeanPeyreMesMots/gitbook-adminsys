# 1 - Apia, Tokamachi, Yokohama & Fukuoka

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

On passe ensuite les deux fichiers à `vimdiff` :

```bash
vimdiff file76.txt file0.txt
```

Le mot est bien trouvé, il s'agit de **"eureka"**. Plus qu'à le mettre dans la solution :

```bash
echo "eureka" > /home/admin/solution
```

Et vérifier que le hash correspond à celui attendu par le chall :

```bash
md5sum /home/admin/solution
55aba155290288b58e9b778c8f616560  /home/admin/solution
```

Ce qui est bien le cas.

## Tokamachi

**Objectif :** un writer doit envoyer des messages en continu dans un named pipe situé à (`/home/admin/namedpipe`), puis un reader les capture avec logs horodatés dans `/home/admin/reader.log`.

Le reader tourne déjà, avec un délai de 2 secondes entre chaque lecture :

```bash
nohup /bin/bash -c 'while true; do
  if read line < /home/admin/namedpipe; then
    echo "$(date) Received: $line" >> /home/admin/reader.log
  fi
  sleep 2
done' &>/dev/null &
```

Le writer proposé par défaut par SadServers n'a lui aucun délai :

```bash
/bin/bash -c 'while true; do echo "this is a test message being sent to the pipe" > /home/admin/namedpipe; done' &
```

### Essais ratés

J'ai d'abord essayé de corriger la syntaxe du writer comme suggéré, avec un `sleep 3` :

```bash
/bin/bash -c 'while true; do if read line < /home/admin/namedpipe; then echo "$(date) Received: $line" >> /home/admin/reader.log; fi; sleep 3; done'
```

Mais ça ne marchait pas. On peut tenter une indentation et un `2>/dev/null` :

```bash
/bin/bash -c 'while true; do if read line < /home/admin/namedpipe 2>/dev/null; then echo "$(date) Received: $line" >> /home/admin/reader.log; fi; sleep 2; done'
```

Mais ça rend le log vide, et SadServers n'acceptait pas la solution.

### Ce qui a marché

Après avoir tué l'ancien processus writer (`ps` + `grep "pipe"` pour trouver le PID, puis `kill`), la commande qui a fonctionné utilise `nohup` :

```bash
nohup /bin/bash -c 'while true; do echo "this is a test message being sent to the pipe" > /home/admin/namedpipe; sleep 2; done' &
```

`nohup` (pour « no hang up ») lance une commande de manière à ce qu'il continue de s'exécuter même si la session terminal qui l'a lancé est fermée ou interrompue. Il détache la commande de la session courante et la place dans un processus indépendant, garantissant la persistance de son exécution.

source : [zonetuto.fr](https://zonetuto.fr/shell-bash/nohup-lancer-un-script-en-arriere-plan-sur-un-serveur-linux/)

## Yokohama

**Objectif :** gestion de permissions pour 4 users (**abe**, **betty**, **carlos**, **debora**) :

* chacun modifie son propre fichier ;
* personne ne peut _lire_ les fichiers des autres, mais peut y _écrire_ ;
* tous peuvent modifier le contenu de `shared/project_ALL`, sauf sa première ligne.

On a les accès root, donc on peut ajuster les droits librement.

Au début on a ça :

```bash
ls -l /home/admin/shared
total 20
-rw-r--r-- 1 root   admin  38 Feb  2  2025 ALL
-rw-r----- 1 abe    abe    27 Feb  2  2025 project_abe
-rw-r----- 1 betty  betty  29 Feb  2  2025 project_betty
-rw-r----- 1 carlos carlos 30 Feb  2  2025 project_carlos
-rw-r----- 1 debora debora 30 Feb  2  2025 project_debora
```

On se loge avec un autre user comme **debora**, mais il n'a ni lecture ni écriture possible sur les fichiers des autres :

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

Pour gagner du temps sur le calcul des octales des permissions, un outil fort pratique : [chmod-calculator.com](https://chmod-calculator.com/).

Pour les fichiers `project_USER`, le bit d'exécution n'est pas nécessaire, on a juste besoin de read + write pour le groupe qui porte le même nom que le user. Puis on se connecte avec chaque user pour appliquer les permissions correspondantes, exemple avec debora :

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

Sauf qu'il faut répèter le process pour chaque user, un peu fastidieux et pas dans une bonne approche. On va passer par un groupe partagé :

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

Pour permettre à tous les users d'ajouter du contenu à `/home/admin/shared/project_ALL`, il faut un accès lecture/écriture pour tout le monde → 666 :&#x20;

```bash
-rw-rw-rw- 1 root   admin  38 Feb  2  2025 ALL
```

Et enfin, `chattr +a` (« append ») pour ne permettre que l'ajout de contenu dans `ALL`, sans pouvoir modifier ce qui existe déjà, et donc la première ligne :

```bash
lsattr *
-----a--------e------- ALL
--------------e------- project_abe
--------------e------- project_betty
--------------e------- project_carlos
--------------e------- project_debora
```

## Fukuoka

**Objectif :** un serveur nginx renvoie une 404 par défaut au lieu de servir un fichier avec le message _"Welcome to the Real Site!"_.

```html
curl localhost
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.18.0</center>
</body>
</html>
```

Un rapide coup d'œil dans la conf nginx nous indique où chercher les logs :

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

Mais nginx ne veut toujours pas, sacré nginx. On se prend maintenant un 403 à la place d'un 404, donc toujours une erreur d'accès, mais sur le fichier html principal cette fois :

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

Quand on regarde de plus près, le lien symbolique `index.html` pointe vers un fichier qui n'a pas les bonnes permissions :

```bash
ll
lrwxrwxrwx 1 www-data www-data  33 Jul 21  2025 index.html -> /opt/site-content/real_index.html
-rwxr-xr-x 1 www-data www-data 612 Jul 21  2025 index.nginx-debian.html

ll /opt/site-content/real_index.html
-rw-r----- 1 root root 34 Jul 21  2025 /opt/site-content/real_index.html
```

On le voit d'ailleurs dans les logs :

```bash
cat /var/log/nginx/error.log
2026/03/06 21:12:25 [error] 928#928: *1 open() "/var/www/html/index.html" failed (13: Permission denied), client: 127.0.0.1, server: _, request: "GET / HTTP/1.1", host: "localhost"
```

On corrige donc tout ça avec les bonnes permissions du fichier cible :

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

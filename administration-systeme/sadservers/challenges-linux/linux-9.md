## Budapest
**Objectif :** créer un compte pour chaque utilisateur listé dans `user_list.txt` (format `user;password`), avec le mot de passe correspondant.

```bash
cat user_list.txt
# alexsmith;Yt7kE9wq
# emilyjones;Fg3vU1pz
# ...
```

Script demandé à Perplexity pour parcourir le fichier et créer/mettre à jour chaque compte :

```bash
#!/bin/bash
INPUT="user_list.txt"

while IFS=';' read -r user pass; do
    [ -z "$user" ] && continue
    echo "Création de l'utilisateur $user"

    if id "$user" >/dev/null 2>&1; then
        echo "  -> $user existe déjà, mise à jour du mot de passe"
    else
        useradd -m "$user"
    fi

    echo "${user}:${pass}" | chpasswd
done < "$INPUT"
```

## Tukaani
**Contexte :** un service (`jobapp`) charge une version malveillante d'une librairie système (`liblzma.so.5`) via une variable d'environnement, à la place de la vraie librairie.

### Comprendre le mécanisme

Rappel utile avant de creuser : une librairie statique est intégrée à la compilation, une librairie partagée/dynamique est chargée au runtime par le _dynamic linker/loader_ (`ld.so`). C'est ce chargement dynamique qui peut être détourné.

La bonne version de la librairie, présente sur le système :

```bash
ll /usr/lib/x86_64-linux-gnu/liblzma.so.5
# lrwxrwxrwx 1 root root 16 Apr 11 2022 liblzma.so.5 -> liblzma.so.5.2.5
```

### Repérer les services concernés

Deux services web tournent sur la machine (`jobapp.service`, `webapp.service`). Inspection de leurs unités systemd :

```bash
cat /etc/systemd/system/multi-user.target.wants/webapp.service
[Service]
ExecStart = /opt/webapp/webapp.py
Environment="LD_LIBRARY_PATH=/opt/.trash/"
```

```bash
cat /etc/systemd/system/multi-user.target.wants/jobapp.service
[Service]
ExecStart = /opt/job-app/jobapp.py
EnvironmentFile=/opt/.trash/.jobapp.env
```

Les deux pointent vers un dossier suspect, `/opt/.trash/` — nom qui sent déjà le mauvais signe.

### Le coupable

```bash
sudo ls /opt/.trash/
# liblzma.so.5
```

Une deuxième copie de la librairie existe dans ce dossier, en dehors de l'emplacement système standard. Confirmation avec une recherche globale :

```bash
sudo find / -name "liblzma*" 2>/dev/null
# /usr/lib/x86_64-linux-gnu/liblzma.so.5      <- la vraie
# /opt/.trash/liblzma.so.5                     <- la suspecte
```

Contenu du fichier d'environnement de `jobapp` :

```bash
cat /opt/.trash/.jobapp.env
APP_CONFIG_DB_NAME="jobapp"
APP_CONFIG_USER="dev"
LD_PRELOAD="/opt/.trash/liblzma.so.5"
DB_CONFIG_PRELOAD="true"
```

`LD_PRELOAD` force le chargement de cette librairie avant toute autre — c'est la technique classique pour injecter du code malveillant dans un process légitime, en interceptant les appels à des fonctions de la vraie librairie.

### Correction

Pour `webapp`, suppression de la variable d'environnement empoisonnée directement via un override systemd :

```bash
sudo systemctl edit webapp
# [Service]
# Environment=
sudo systemctl daemon-reload
```

Pour neutraliser la librairie malveillante elle-même, remplacement par un lien vers la vraie version système :

```bash
# Suppression du fichier malveillant (optionnel)
sudo rm /opt/.trash/liblzma.so.5

# Lien symbolique vers la bonne librairie
sudo ln -s /usr/lib/x86_64-linux-gnu/liblzma.so.5.2.5 /opt/.trash/liblzma.so.5
```

Ainsi, même si un service référence encore ce chemin, il pointe désormais vers la librairie légitime.
## Tokelau
**Problème :** vider des entrées spécifiques de l'historique bash (contenant "foo") sans que la session courante ne les fasse réapparaître.

Point clé compris après coup : `history -r` ne **remplace pas** l'historique de la session en cours, il **ajoute** le contenu du fichier à la liste déjà chargée en mémoire. Du coup, supprimer une ligne du fichier `.bash_history` avec `sed` ne suffit pas si l'historique en mémoire de la session contient encore les commandes visées.

Solution en une ligne :

```bash
sed -i '/foo/d' ~/.bash_history && history -c && history -r
```

- `sed -i '/foo/d' ~/.bash_history` : supprime les lignes contenant "foo" du fichier.
- `history -c` : vide l'historique de la session courante (en mémoire).
- `history -r` : recharge l'historique depuis le fichier (désormais propre).
## Hanoi
**Objectif :** croiser deux fichiers — `users.txt` (liste d'utilisateurs) et `groups.txt` (format `group_name:user1,user2,user3`) — pour lister les utilisateurs appartenant à plus d'un groupe, triés alphabétiquement.

Prompt envoyé à Perplexity pour générer le script :

> _Génère une commande bash triant les utilisateurs de users.txt appartenant à plus d'1 groupe dans groups.txt, triés alphabétiquement, un nom par ligne._

Script obtenu :

```bash
#!/bin/bash

# Charge users.txt dans un tableau associatif pour lookup rapide
declare -A valid_users
while IFS= read -r user; do
    [[ -n "$user" ]] && valid_users["$user"]=1
done < users.txt

# Compte les appartenances aux groupes pour chaque utilisateur valide
declare -A user_counts
while IFS=: read -r group users_list; do
    [[ -z "$group" || -z "$users_list" ]] && continue
    IFS=',' read -ra user_array <<< "$users_list"
    for user in "${user_array[@]}"; do
        [[ -n "$user" && ${valid_users[$user]+isset} ]] || continue
        ((user_counts[$user]++))
    done
done < groups.txt

# Sortie triée des utilisateurs avec plus d'1 groupe
for user in "${!user_counts[@]}"; do
    [[ ${user_counts[$user]} -gt 1 ]] && echo "$user"
done | sort > /home/admin/multi-group-users.txt
```

Fonctionne du premier coup.
## Kampala
**Contexte :** des scripts de déploiement qui refusent de s'exécuter, sans erreur de permission ni de syntaxe visible au premier abord.

### Premiers checks, rien d'anormal

```bash
ll deploy/
# -rwxr-xr-x 1 admin admin 293 Sep 29 14:07 deploy.sh
```

Permissions correctes, contenu du script lu — rien de suspect syntaxiquement :

```bash
cat deploy.sh
#!/bin/bash
echo "Starting deployment process..."
# ...
```

### L'erreur mystère

Copie du script pour tester sans risque, changement des perms, exécution :

```bash
chmod 755 deploy_2.sh
./deploy_2.sh
# -bash: ./deploy_2.sh: cannot execute: required file not found
sudo ./deploy_2.sh
# sudo: unable to execute ./deploy_2.sh: No such file or directory
```

Message trompeur — le fichier existe pourtant bel et bien. En le lançant explicitement avec `bash` plutôt qu'en exécution directe, le vrai problème apparaît :

```bash
sudo bash deploy_2.sh
# deploy_2.sh: line 2: $'\r': command not found
# deploy_2.sh: line 14: syntax error: unexpected end of file
```

`$'\r'` — un caractère de retour chariot invisible en fin de ligne, typique des fichiers édités/créés sous Windows (fins de ligne CRLF au lieu de LF Unix).

Recherche Google sur le message d'erreur original (`cannot execute: required file not found`) qui confirme cette piste : c'est un classique des scripts avec fins de ligne Windows, l'interpréteur indiqué par le shebang (`#!/bin/bash\r`) n'est pas trouvé tel quel à cause du `\r` parasite.

### Correction

```bash
dos2unix deploy_2.sh
# converting file deploy_2.sh to Unix format...
```

Appliqué à tous les scripts du dossier par précaution :

```bash
dos2unix *
# converting file backup.sh to Unix format...
# converting file deploy.sh to Unix format...
# converting file setup.sh to Unix format...
```

Test après correction :

```bash
./setup.sh
# Setting up application environment...
# mkdir: cannot create directory '/opt/app/logs': Permission denied
# Environment setup completed!
```

Le script s'exécute enfin (reste une erreur de permission distincte sur `/opt/app`, problème séparé et hors scope du diagnostic initial).
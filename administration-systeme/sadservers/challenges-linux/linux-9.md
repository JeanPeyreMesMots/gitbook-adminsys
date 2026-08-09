# 9 - Budapest, Tukaani, Tokelau, Hanoi & Kampala

### <mark style="color:$warning;">Budapest</mark>

**L'objectif est ici tout simple :** créer un compte pour chaque user listé dans `user_list.txt` (format `user;password`), avec le mot de passe correspondant.

```bash
cat user_list.txt
# alexsmith;Yt7kE9wq
# emilyjones;Fg3vU1pz
# ...
```

Là pour le coup l'IA nous aide grandement. N'importe laquelle peut pour parcourir le fichier et créer/mettre à jour chaque compte :

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

### <mark style="color:$warning;">Tukaani</mark>

**Contexte de chall de la backdoor XZ Utils (CVE-2024-3094)**, découverte en mars 2024. L'un des incidents de supply-chain les plus sérieux de ces dernières années dans l'écosystème Linux **:** un service `jobapp` charge une version malveillante d'une librairie système (`liblzma.so.5`) via une variable d'environnement, à la place de la vraie librairie.

Rappel utile avant de creuser : une librairie statique est intégrée à la compilation, une librairie partagée/dynamique est chargée au runtime par le _dynamic linker/loader_ (`ld.so`). C'est ce chargement dynamique qui peut être détourné.

La bonne version de la librairie, présente sur le système est ici :

```bash
ll /usr/lib/x86_64-linux-gnu/liblzma.so.5
# lrwxrwxrwx 1 root root 16 Apr 11 2022 liblzma.so.5 -> liblzma.so.5.2.5
```

Ici, deux services web tournent sur la machine (`jobapp.service`, `webapp.service`). On inspecte leurs unités systemd :

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

Les deux pointent vers un dossier suspect, `/opt/.trash/` ça sent déjà le mauvais signe.

```bash
sudo ls /opt/.trash/
# liblzma.so.5
```

Une deuxième copie de la librairie existe dans ce dossier, en dehors de l'emplacement système standard. On va quand même rechercher partout, et on tombe sur le mauvais éléments :

```bash
sudo find / -name "liblzma*" 2>/dev/null
# /usr/lib/x86_64-linux-gnu/liblzma.so.5      <- la vraie
# /opt/.trash/liblzma.so.5                     <- la suspecte
```

Cette dernière est d'ailleurs référencé dans le fichier `/opt/.trash/.jobapp.env` :

```bash
cat /opt/.trash/.jobapp.env
APP_CONFIG_DB_NAME="jobapp"
APP_CONFIG_USER="dev"
LD_PRELOAD="/opt/.trash/liblzma.so.5"
DB_CONFIG_PRELOAD="true"
```

`LD_PRELOAD` force le chargement de la librairie malveillante avant toute autre.&#x20;

On va donc la supprimer via systemctl :

```bash
sudo systemctl edit webapp
# [Service]
# Environment=
sudo systemctl daemon-reload
```

Puis on supprime le fichier de librairie et remplace par un lien vers la vraie version système, en gardant par contre le nom originale pour pas casser une app/service. Ainsi même si un service référence encore ce chemin, il pointe désormais vers la librairie légitime.

```bash
# Suppression du fichier malveillant (optionnel)
sudo rm /opt/.trash/liblzma.so.5

# Lien symbolique vers la bonne librairie
sudo ln -s /usr/lib/x86_64-linux-gnu/liblzma.so.5.2.5 /opt/.trash/liblzma.so.5
```

### <mark style="color:$warning;">Tokelau</mark>

**Objectif :** vider les lignes de l'historiques contenant "foo", sans que la session courante ne les fasse réapparaître.

Ce chall m'a permis de voir un truc : `history -r` ne **remplace pas** l'historique de la session en cours, il **ajoute** le contenu du fichier de l'historique bash à la liste déjà chargée en mémoire. Du coup, supprimer une ligne du fichier `.bash_history` avec `sed` ne suffit pas si l'historique en mémoire de la session contient encore des commandes.

La solution pour pallier à cela tiens en une ligne :

```bash
sed -i '/foo/d' ~/.bash_history && history -c && history -r
```

* `sed -i '/foo/d' ~/.bash_history` : supprime les lignes contenant "foo" du fichier.
* `history -c` : vide l'historique de la session courante en mémoire.
* `history -r` : recharge l'historique depuis le fichier bash\_history.

### <mark style="color:$warning;">Hanoi</mark>

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

### <mark style="color:$warning;">Kampala</mark>

**Contexte :** le serveurs contient des scripts de déploiement qui refusent de s'exécuter

Premiers checks, rien d'anormal

```bash
ll deploy/
# -rwxr-xr-x 1 admin admin 293 Sep 29 14:07 deploy.sh
```

Les permissions sont correctes, tout comme le contenu contenu du script deploy.sh :

```bash
cat deploy.sh
#!/bin/bash
echo "Starting deployment process..."
# ...
```

Au cas où, on fait une copie si on aura besoin de modifier des trucs par précaution. Puis on change les perms et test en lançant pour voir :

```bash
chmod 755 deploy_2.sh
./deploy_2.sh
# -bash: ./deploy_2.sh: cannot execute: required file not found
sudo ./deploy_2.sh
# sudo: unable to execute ./deploy_2.sh: No such file or directory
```

Le "./" placé avant un fichier pour lance un script direct avec le shell par défaut ne passe pas. En le lançant directement avec `bash`, le vrai problème apparaît :

```bash
sudo bash deploy_2.sh
# deploy_2.sh: line 2: $'\r': command not found
# deploy_2.sh: line 14: syntax error: unexpected end of file
```

Le script contient un caractère de retour chariot `$'\r'` invisible en fin de ligne, typique des fichiers édités/créés sous Windows (fins de ligne CRLF au lieu de LF Unix).

Une recherche Google sur le message d'erreur original (`cannot execute: required file not found`) qui confirme cette piste : c'est un classique des scripts avec fins de ligne Windows, l'interpréteur indiqué par le shebang (`#!/bin/bash\r`) n'est pas trouvé tel quel à cause du `\r` parasite.

Pour convertir le fichier on va utiliser **dos2unix**. C'est un outil en ligne de commande qui convertit les fins de ligne des fichiers texte du format DOS/Windows (CRLF) vers le format Unix/Linux (LF). Il supprime les retours chariot superflus pour rendre les scripts et les fichiers compatibles avec les systèmes de type Unix :

```bash
dos2unix deploy_2.sh
# converting file deploy_2.sh to Unix format...
```

On appliqué à tous les scripts du dossier par précaution :

```bash
dos2unix *
# converting file backup.sh to Unix format...
# converting file deploy.sh to Unix format...
# converting file setup.sh to Unix format...
```

Puis :

```bash
./setup.sh
# Setting up application environment...
# mkdir: cannot create directory '/opt/app/logs': Permission denied
# Environment setup completed!
```

Le script s'exécute enfin (reste une erreur de permission distincte sur `/opt/app`, problème séparé car hors scope).

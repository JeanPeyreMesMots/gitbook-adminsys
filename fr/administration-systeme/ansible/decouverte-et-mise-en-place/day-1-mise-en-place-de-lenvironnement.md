# 1 - Mise en place de l'environnement

### Pourquoi Ansible ?

<div align="left"><img src="../../../.gitbook/assets/image (10).png" alt="" width="224"></div>

Ansible est un outil d'automatisation permettant de gérer un grand nombre de serveurs sans devoir intervenir manuellement sur chacun d'eux.

Il permet notamment :

* d'automatiser la configuration des serveurs ;
* de standardiser les installations ;
* de déployer des applications ;
* d'exécuter des tâches sur plusieurs machines simultanément.

En effet lorsque le nombre de serveurs augmente, les tâches manuelles dessus deviennent longues. Certains serveurs deviennent des **flocons de neiges** (machines uniques et fragiles) qui provoquent de l’appréhension pour les déploiements. Dans plusieurs infras on assiste ainsi à l'apparition de plusieurs scripts d'automatisations, difficiles à maintenir.

L'objectif d'Ansible est d'éviter ça, en rendant les déploiements reproductibles, fiables et automatisés, et surtout, **idempotent**.

La formation sera organisée en trois grandes parties :

### 1. Maintenance

Comment réaliser rapidement des actions de maintenance sur plusieurs machines, par exemple pour :

* récupérer une version d'application ;
* lancer une commande sur plusieurs serveurs.

> Cette pratique reste exceptionnelle : les actions de masse doivent être automatisées via des playbooks.

### 2. Installation des serveurs

Cette partie se concentre sur la création de playbooks et de rôles Ansible pour standardiser et automatiser le déploiement de serveurs.

### 3. Déploiement applicatif

Cette dernière partie traite du déploiement applicatif sur une infra déjà en place, ayant comme objectifs de :

* déployer automatiquement une nouvelle version ;
* réduire les risques humains ;
* fiabiliser les mises en production.

## Mise en place du lab

Comme indiqué au tuto précédent, on va passer par Multipass pour monter des VM légères rapidement et facilement. Il s'agit d'un gestionnaire léger de machines virtuelles développé par Canonical. Il permet la création très rapide de VM, avec faible consommation de ressources, disponible sous Windows, Linux et macOS. Un outil fort pratique que je vous recommande pour déployer instant des VM rapidement !

Une fois Multipass installé, on créé les machines avec les commandes suivantes :

```bash
multipass launch 22.04 -n ansible-main -c 2 -m 1G

multipass launch 22.04 -n web-server-1 -c 1 -m 1G

multipass launch 22.04 -n web-server-2 -c 1 -m 1G

multipass launch 22.04 -n lb-server -c 1 -m 1G
```

**Note : la VM "lb-server" servira pour plus tard pour du load balancing ;)**

Je me suis heurté à un soucis sur ma VM Ubuntu avec Multipass dessus, qui attribue les adresses IP via DHCP. Ce qui fait que les IP changent régulièrement (donc pas d'IP fixes), et les noms d'hôtes ne sont plus résolus correctement.

La solution est donc de mettre à jour automatiquement le fichier `/etc/hosts` à partir de la sortie de `multipass list` avec un script comme ça :

```bash
sudo tee /usr/local/bin/sync-multipass-hosts.sh <<'EOF'
#!/bin/bash
set -e

MARKER_START="# BEGIN MULTIPASS"
MARKER_END="# END MULTIPASS"
HOSTS_FILE="/etc/hosts"

BLOCK=$(multipass list --format csv | tail -n +2 | while IFS=',' read -r name state ip image release; do
  ip_clean=$(echo "$ip" | cut -d';' -f1)

  if [ -n "$ip_clean" ] && [ "$state" = "Running" ]; then
    echo "$ip_clean  $name"
  fi
done)

sudo sed -i "/$MARKER_START/,/$MARKER_END/d" "$HOSTS_FILE"

{
  echo "$MARKER_START"
  echo "$BLOCK"
  echo "$MARKER_END"
} | sudo tee -a "$HOSTS_FILE" > /dev/null

echo "✅ /etc/hosts synchronisé :"
echo "$BLOCK"
EOF

sudo chmod +x /usr/local/bin/sync-multipass-hosts.sh
```

Pour maintenir le fichier hosts à jour, on peut faire une crontab de la sorte pour permettre au script est exécuté toutes les cinq minutes :

```bash
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/local/bin/sync-multipass-hosts.sh > /dev/null 2>&1") | crontab -
```

Ainsi les hosts que l'on a ici :

```
$ multipass list

Name              State     IPv4
ansible-main      Running   10.3.241.40
web-server-1      Running   10.3.241.212
web-server-2      Running   10.3.241.176
```

Se retrouvent dans le fichier `/etc/hosts` suivant :

```
# BEGIN MULTIPASS
10.3.241.179 ansible-main
10.3.241.212 web-server-1
10.3.241.176 web-server-2
# END MULTIPASS
```

## Préparation des VM pour Ansible

Les VM doivent autoriser la connexion SSH, l'authentification par mot de passe et la connexion du compte root. À éviter bien sûr en prod mais là on est dans un lab locale donc c'est à des fins de démonstrations 🙂 :

```bash
multipass exec web-server-1 -- sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf

multipass exec web-server-1 -- sh -c "echo 'root:ansible' | sudo chpasswd"

multipass exec web-server-1 -- sudo systemctl restart ssh
```

Même procédure pour les deux autres serveurs "**web-server-2**" et le serveur "**lb-server**" qui servira plus tard pour le load balacing.

Cependant, si ma VM hôte Ubuntu connaît les IP et les hostnames Multipass, ma VM `ansible-main` ne les connaît pas automatiquement. Un second script génère un fichier contenant les adresses IP puis les copie dans la VM :

```bash
#!/bin/bash
set -e

TARGET_VM="ansible-main"
TMP_FILE="$HOME/hostfile-multipass"

multipass list --format csv | tail -n +2 | while IFS=',' read -r name state ip image release; do
  ip_clean=$(echo "$ip" | cut -d';' -f1)

  if [ -n "$ip_clean" ] && [ "$state" = "Running" ]; then
    echo "$ip_clean  $name"
  fi
done > "$TMP_FILE"

multipass transfer "$TMP_FILE" "$TARGET_VM":/home/ubuntu/hostfile

multipass exec "$TARGET_VM" -- bash -c '
sudo sed -i "/# BEGIN MULTIPASS/,/# END MULTIPASS/d" /etc/hosts
{
  echo "# BEGIN MULTIPASS"
  cat /home/ubuntu/hostfile
  echo "# END MULTIPASS"
} | sudo tee -a /etc/hosts > /dev/null
'

rm "$TMP_FILE"
```

**Note : Ce script ne stabilise pas les IP. Il ne fait que maintenir la résolution des hostnames.**

Puis on se créé un fichier d'inventaire "**inventory.ini**". Bien sûr pas de mot de passe en dur en prod 🙂 :

```ini
[web]
web-server-1
web-server-2

[lb-host]
lb-server

[web:vars]
ansible_ssh_user=root
ansible_ssh_pass=ansible
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
ansible_python_interpreter=/usr/bin/python3.10
```

Puis sur la VM "**ansible-main**" on installe **python3-pip** + **ansible** + ajout dans le PATH :

```bash
export PATH=$PATH:~/.local/bin
```

## Première erreur rencontrée

Quand on veut ping les hôtes :

```bash
ansible -m ping -i inventory all
```

Une erreur indique que le paquet `sshpass` est obligatoire dans ce cas.

```
FAILED!

to use the 'ssh' connection type with passwords
you must install the sshpass program
```

On règle ça + update pour la forme  :

```bash
sudo apt update

sudo apt install -y sshpass
```

## Vérification du fonctionnement

On ping tout les hosts qui se trouvent dans **hosts.ini** :

```bash
ansible -m ping -i hosts.ini all
```

On a donc en sortie 😉 :

```
web-server-1 | SUCCESS
web-server-2 | SUCCESS
```

Sortie :

```
{
    "changed": false,
    "ping": "pong"
}
```

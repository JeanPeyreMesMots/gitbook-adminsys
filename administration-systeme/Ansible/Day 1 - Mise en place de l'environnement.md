
## Explications :

Ansible -> Outil de gestion de conf pour éviter de devoir gérer plusieurs serveurs à la main
Rôles seront testés par Ansible
Déploiement continue
--> éliminer la peur du déploiement

À l’issue de cette formation, vous serez capable de gérer plus facilement un grand nombre de serveurs, ainsi que de déployer et d’utiliser Ansible au sein de votre entreprise.

La gestion d’un parc de serveurs devient en effet de plus en plus complexe :

- il est difficile d’agir sur plusieurs machines en même temps ;
- certains serveurs deviennent de véritables “flocons de neige”, c’est-à-dire qu’ils sont uniques et fragiles dès qu’on les touche ;
- les déploiements peuvent susciter de l’appréhension ;
- et l’on souhaite éviter de multiplier les systèmes d’automatisation non maîtrisés.

La formation est organisée en trois grandes parties :
## 1. Maintenance

Cette première partie présente comment réaliser rapidement des actions de maintenance sur plusieurs machines, par exemple :

- récupérer une version d’application ;
- exécuter rapidement une commande sur un grand nombre de serveurs, même si cette pratique doit rester exceptionnelle.
## 2. Installation des serveurs

Cette partie se concentre sur la création de playbooks et de rôles Ansible afin de standardiser et automatiser l’installation des serveurs.
## 3. Déploiement applicatif

Enfin, la dernière partie traite du déploiement applicatif sur une infrastructure déjà en place, avec pour objectif :

- déployer automatiquement une nouvelle version d’application ;
- fiabiliser les mises en production ;
- et réduire les risques liés aux déploiements manuels.

# Mise en place d'un serveur de tests

==> Multipass est un gestionnaire de VM léger conçu pour que les développeurs puissent lancer un nouvel environnement avec une seule commande. **Multipass** est rapide, efficace et peut fonctionner on all OS

```bash
multipass launch 22.04 -n ansible-main -c 2 -m 1G
multipass launch 22.04 -n web-server-1 -c 1 -m 1G
multipass launch 22.04 -n web-server-2 -c 1 -m 1G
```

Modif d'IP impossible car DHCP box => Vbox et ip interne au driver de multipass. Donc script + cron au démarrgae pour automatiser complètement :

```bash
sudo tee /usr/local/bin/sync-multipass-hosts.sh <<'EOF'
#!/bin/bash
set -e

MARKER_START="# BEGIN MULTIPASS"
MARKER_END="# END MULTIPASS"
HOSTS_FILE="/etc/hosts"

# Génère le bloc à partir de multipass list
BLOCK=$(multipass list --format csv | tail -n +2 | while IFS=',' read -r name state ip image release; do
  # ignore les VM sans IP (arrêtées) ou avec plusieurs IP (garde la 1ere)
  ip_clean=$(echo "$ip" | cut -d';' -f1)
  if [ -n "$ip_clean" ] && [ "$state" = "Running" ]; then
    echo "$ip_clean  $name"
  fi
done)

# Supprime l'ancien bloc s'il existe, puis rajoute le nouveau
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

```bash
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/local/bin/sync-multipass-hosts.sh > /dev/null 2>&1") | crontab -
```

Ainsi :

```powershell
$ multipass list
Name                    State             IPv4             Image
ansible-main            Running           10.3.241.40      Ubuntu 22.04 LTS
web-server-1            Running           10.3.241.212     Ubuntu 22.04 LTS
web-server-2            Running           10.3.241.176     Ubuntu 22.04 LTS

```

```bash
bash /usr/local/bin/sync-multipass-hosts.sh
✅ /etc/hosts synchronisé :
10.3.241.179  ansible-main
10.3.241.212  web-server-1
10.3.241.176  web-server-2
jpmm@kos-boss:~/Documents/ansible-formation/conf-hosts$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 kos-boss

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
127.0.0.1 metrics.ubuntu.com
127.0.0.1 www.metrics.ubuntu.com
127.0.0.1 popcon.ubuntui.com
127.0.0.1 www.popcon.ubuntu.com
10.0.2.15 registry.leoni.local

# route traefik
127.0.0.1 xavki.localhost
# BEGIN MULTIPASS
10.3.241.179  ansible-main
10.3.241.212  web-server-1
10.3.241.176  web-server-2
# END MULTIPASS
```

Autorise SSH avec root + pass ansible :

```bash
multipass exec web-server-1 -- sh -c "sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config ; sudo sed -i 's/c no/PasswordAuthentication yes/' /etc/ssh/sshd_config ;echo 'root:ansible' | sudo chpasswd ; sudo service ssh restart"

multipass exec web-server-2 -- sh -c "sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config ; sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config ;echo 'root:ansible' | sudo chpasswd ; sudo service ssh restart"

multipass exec lb-server -- sh -c "sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config ; sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config ;echo 'root:ansible' | sudo chpasswd ; sudo service ssh restart"
```

Puis generate hostfile avec DHCP¨maj dans la machine ubuntu, on s'emmerde pas avec, fait à l'arrache, pour reach les vm via leur hostname

```bash
cat /usr/local/bin/sync-multipass-hosts.sh
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

echo "📄 Hostfile généré :"
cat "$TMP_FILE"

multipass transfer "$TMP_FILE" "$TARGET_VM":/home/ubuntu/hostfile

multipass exec "$TARGET_VM" -- bash -c '
sudo sed -i "/# BEGIN MULTIPASS/,/# END MULTIPASS/d" /etc/hosts
{
  echo "# BEGIN MULTIPASS"
  cat /home/ubuntu/hostfile
  echo "# END MULTIPASS"
} | sudo tee -a /etc/hosts > /dev/null
'

echo "✅ /etc/hosts de $TARGET_VM synchronisé."
rm "$TMP_FILE"
```

Le script résout le problème de la synchro, mais pas celui de la stabilité des IP elles-mêmes, d'où le cron précédent :

```bash
multipass exec ansible-main -- sh -c "cat /home/ubuntu/hostfile"
10.3.241.179  ansible-main
10.3.241.212  web-server-1
10.3.241.176  web-server-2
```

Pusi fichier d'inventaire. Bien sur pas de mot de passe en dur en prod :D :

```yaml
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

Puis insall python3-pip + ansible + add to path :

```bash
ubuntu@ansible-main:~$ ll .local/
bin/ lib/
ubuntu@ansible-main:~$ ll .local/bin/ansible
-rwxrwxr-x 1 ubuntu ubuntu 216 Jul  4 15:16 .local/bin/ansible*
ubuntu@ansible-main:~$ export PATH=$PATH:~/.local/bin
```

BUT :

```bash
ubuntu@ansible-main:~/ansible$ ansible -m ping -i inventory all
web-server-1 | FAILED! => {
    "msg": "to use the 'ssh' connection type with passwords or pkcs11_provider, you must install the sshpass program"
}
web-server-2 | FAILED! => {
    "msg": "to use the 'ssh' connection type with passwords or pkcs11_provider, you must install the sshpass program"
}
```

==> 

```bash
multipass exec web-server-1 -- sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
multipass exec web-server-1 -- sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
multipass exec web-server-1 -- sh -c "echo 'root:ansible' | sudo chpasswd"
multipass exec web-server-1 -- sudo systemctl restart ssh
```

```bash
multipass exec web-server-2 -- sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
multipass exec web-server-2 -- sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
multipass exec web-server-2 -- sh -c "echo 'root:ansible' | sudo chpasswd"
multipass exec web-server-2 -- sudo systemctl restart ssh
```

Pour haproxy après :

```bash
multipass exec lb-server -- sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
multipass exec lb-server -- sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
multipass exec lb-server -- sh -c "echo 'root:ansible' | sudo chpasswd"
multipass exec lb-server -- sudo systemctl restart ssh
```

Pour être sûr mdr. Enfin, install de sshpass + inventory pingable sur la vm ansible-main :

```bash
sudo apt update && sudo apt install -y sshpass

$ ansible -m ping -i hosts.ini all
web-server-1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
web-server-2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```


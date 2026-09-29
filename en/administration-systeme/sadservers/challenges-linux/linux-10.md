# 10 - Bizerte, Kampot & Valladolid

### <mark style="color:$warning;">Kampot</mark>

**Contexte :** une appli Python tourne sur le port 20280, gérée par supervisor, et ne peut pas être reconfigurée pour changer de port. L'objectif est de rendre le service accessible localement sur le port 80 sans toucher à sa config.

Comme on à eu affaire à ça sur les challs précédents, je tilte sur la redirection de port, en m'aidant [cet article](https://blog.cloudfrancois.fr/2016-05-17-iptables-rule-forward-local/).

On vérifie d'abord que le forwarding IP est actif :

```bash
cat /proc/sys/net/ipv4/ip_forward
# 1
```

Puis on créé une règle de redirection sur l'interface loopback, car la requête part de la machine elle-même :

```bash
sudo iptables -t nat -A OUTPUT -o lo -p tcp --dport 80 -j REDIRECT --to-port 20280
```

Puis on test :

```bash
curl localhost:80/accounts
# [{"id":1,"name":"Alice","type":"Checking"}, ...]
```

Et ça marche.

### <mark style="color:$warning;">Valladolid</mark>

**Contexte :** un service `log-cleaner` est censé nettoyer les vieux logs mais ne fait pas ce qu'il devrait.

#### Diagnostic initial

```bash
sudo systemctl status log-cleaner
# Active: inactive (dead)
# ... Starting Cleanup... / Cleanup finished. / Deactivated successfully.
```

Le service n'est pas actif, mais semble précedemment s'être lancé. Il a loggé "**Starting/Finished Cleanup**", puis se termine. On regarde qu'aucun process ne bloque les fichiers concernés :

```bash
lsof /var/log/app/old_data.log
lsof /var/log/app/recent_data.log
# rien de bloquant
```

Mais rien d'anormal. Le vrai problème : le service n'est pas activé :

```bash
systemctl is-enabled log-cleaner
# static
systemctl is-active log-cleaner
# inactive
```

Lorsqu'on essaye de l'activer :

```bash
sudo systemctl enable log-cleaner
# The unit files have no installation config (WantedBy=, RequiredBy=, ...)
# This means they are not meant to be enabled or disabled using systemctl.
```

Le service n'a pas de section `[Install]` donc on peut pas le faire démarrer automatiquement. On peut d'ailleurs voir cela en lisant l'unité :

```bash
systemctl cat log-cleaner
[Unit]
Description=Daily Log Cleaner
[Service]
Type=oneshot
ExecStart=/bin/bash /opt/scripts/log-cleaner.sh
WorkingDirectory=/root
```

Effectivement, aucune section `[Install]`. Et dans le script lui-même :

```bash
cat /opt/scripts/log-cleaner.sh
#!/bin/bash
LOG_DIR="/var/log/app"
DAYS=7

echo "Starting Cleanup..."
find . -maxdepth 1 -name "*.log" -type f -mtime -7 -print -delete
echo "Cleanup finished."
```

On peut y voir 3 coquilles :

1. Les variables `LOG_DIR` et `DAYS` sont déclarées mais jamais utilisées. Ce qui fait que le `find` cherche dans `.` (le `WorkingDirectory`, `/root`) au lieu du vrai dossier de logs.
2. `-mtime -7` sélectionne les fichiers modifiés il y a **moins** de 7 jours, donc les fichiers récents, alors que nous ce qu'on veut c'est les fichiers **plus vieux** que N jours (`+7`).
3. Aucun contrôle pour éviter de toucher au fichier de log actif (`recent_data.log`) du coup haha, qui pourrait donc se faire supprimer aussi.

On corrige tout ça dans un premier temps :

```bash
#!/bin/bash
LOG_DIR="/var/log/app"
DAYS=7

echo "Starting Cleanup..."
find "$LOG_DIR" -maxdepth 1 -name "*.log" -type f -not -name "recent_data.log" -mtime +$DAYS -print -delete
echo "Cleanup finished."
```

Puis on corrige l'unité systemd. On y ajoute la section `[Install]` manquante. Le service étant censé être lancé manuellement (pas de timer ni cron associé dans ce contexte), on utilise `multi-user.target` propre à n'importe quelle système Unix :

```ini
[Unit]
Description=Daily Log Cleaner

[Service]
Type=oneshot
ExecStart=/bin/bash /opt/scripts/log-cleaner.sh
WorkingDirectory=/root

[Install]
WantedBy=multi-user.target
```

On reload et puis on active :

```bash
sudo systemctl daemon-reload
sudo systemctl enable log-cleaner
# Created symlink '/etc/systemd/system/multi-user.target.wants/log-cleaner.service' → ...
```

Et on lance le script de reset des logs de test, avant de relancer manuellement le service :

```bash
./reset_logs.sh
# Logs reset.
sudo systemctl restart log-cleaner
cat /var/log/app/recent_data.log
# contenu récent toujours présent, intact
cat /var/log/app/old_data.log
# cat: /var/log/app/old_data.log: No such file or directory
```

Le fichier récent est conservé, l'ancien a bien été supprimé, le soucis est résolu.

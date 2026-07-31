## ==Kampot==

**Contexte :** une appli Python tourne sur le port 20280, gérée par supervisor, et ne peut pas être reconfigurée pour changer de port (contrainte de sécurité/legacy imposée). Objectif : rendre le service accessible localement sur le port 80 sans toucher à sa config.

Réflexion partie sur iptables pour faire une redirection de port, confirmée par [cet article](https://blog.cloudfrancois.fr/2016-05-17-iptables-rule-forward-local/).

Vérification préalable que le forwarding IP est actif (nécessaire pour ce type de règle) :

```bash
cat /proc/sys/net/ipv4/ip_forward
# 1
```

Création de la règle de redirection sur l'interface loopback, puisque la requête part de la machine elle-même :

```bash
sudo iptables -t nat -A OUTPUT -o lo -p tcp --dport 80 -j REDIRECT --to-port 20280
```

Test :

```bash
curl localhost:80/accounts
# [{"id":1,"name":"Alice","type":"Checking"}, ...]
```

Fonctionne du premier coup.
## ==Valladolid==

**Contexte :** un service `log-cleaner` censé nettoyer les vieux logs, apparemment "fonctionnel" mais qui ne fait pas ce qu'il devrait.

### Diagnostic initial

```bash
sudo systemctl status log-cleaner
# Active: inactive (dead)
# ... Starting Cleanup... / Cleanup finished. / Deactivated successfully.
```

Le service se lance, log "Starting/Finished Cleanup", et se termine — comportement normal pour un service `oneshot`. Vérification qu'aucun process ne bloque les fichiers concernés :

```bash
lsof /var/log/app/old_data.log
lsof /var/log/app/recent_data.log
# rien de bloquant
```

Logs `journalctl` cohérents avec plusieurs exécutions passées, rien d'anormal à première vue.

### Le vrai problème : le service n'est pas activable normalement

```bash
systemctl is-enabled log-cleaner
# static
systemctl is-active log-cleaner
# inactive
```

Tentative de l'activer :

```bash
sudo systemctl enable log-cleaner
# The unit files have no installation config (WantedBy=, RequiredBy=, ...)
# This means they are not meant to be enabled or disabled using systemctl.
```

L'unité n'a pas de section `[Install]` — donc pas de mécanisme standard pour la faire démarrer automatiquement (au boot, via un target, etc.). Confirmation en lisant l'unité :

```bash
systemctl cat log-cleaner
[Unit]
Description=Daily Log Cleaner
[Service]
Type=oneshot
ExecStart=/bin/bash /opt/scripts/log-cleaner.sh
WorkingDirectory=/root
```

Effectivement, aucune section `[Install]`.

### Bug dans le script lui-même

```bash
cat /opt/scripts/log-cleaner.sh
#!/bin/bash
LOG_DIR="/var/log/app"
DAYS=7

echo "Starting Cleanup..."
find . -maxdepth 1 -name "*.log" -type f -mtime -7 -print -delete
echo "Cleanup finished."
```

Trois problèmes identifiés d'un coup :

1. Les variables `LOG_DIR` et `DAYS` sont déclarées mais jamais utilisées — le `find` cherche dans `.` (le `WorkingDirectory`, `/root`) au lieu du vrai dossier de logs.
2. `-mtime -7` sélectionne les fichiers modifiés il y a **moins** de 7 jours (donc les fichiers récents), l'inverse de l'intention — un nettoyage de logs doit cibler les fichiers **plus vieux** que N jours (`+7`).
3. Aucune exception pour épargner le fichier de log actif (`recent_data.log`), qui pourrait donc se faire supprimer aussi.

Correction :

```bash
#!/bin/bash
LOG_DIR="/var/log/app"
DAYS=7

echo "Starting Cleanup..."
find "$LOG_DIR" -maxdepth 1 -name "*.log" -type f -not -name "recent_data.log" -mtime +$DAYS -print -delete
echo "Cleanup finished."
```

### Correction de l'unité systemd

Ajout de la section `[Install]` manquante. Le service étant censé être lancé manuellement (pas de timer ni cron associé dans ce contexte), `multi-user.target` est le choix standard pour un système multi-utilisateur classique :

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

Rechargement et activation :

```bash
sudo systemctl daemon-reload
sudo systemctl enable log-cleaner
# Created symlink '/etc/systemd/system/multi-user.target.wants/log-cleaner.service' → ...
```

### Validation finale

Reset des logs de test, puis relance manuelle du service :

```bash
./reset_logs.sh
# Logs reset.
sudo systemctl restart log-cleaner
cat /var/log/app/recent_data.log
# contenu récent toujours présent, intact
cat /var/log/app/old_data.log
# cat: /var/log/app/old_data.log: No such file or directory
```

Le fichier récent est conservé, l'ancien a bien été supprimé — comportement désormais correct.
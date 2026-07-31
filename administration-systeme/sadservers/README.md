# 🛠️ SadServers

---
description: Troubleshooting Linux sur SadServers
---

[SadServers](https://sadservers.com/) est une plateforme qui met à disposition une VM Linux réelle, accessible en SSH, sur laquelle un problème concret a été délibérément introduit : service cassé, permissions mal configurées, réseau mal routé, script buggé, etc. Contrairement à un lab classique où l'on suit une procédure, l'objectif ici est de diagnostiquer soi-même la panne à partir de zéro, avec les mêmes outils et réflexes qu'en conditions réelles d'administration système — logs, `systemctl`, `journalctl`, `iptables`, permissions, et ainsi de suite.

Voici mes prises de notes au fil de ces challenges : le raisonnement suivi, les pistes explorées (bonnes comme mauvaises), et les solutions retenues pour chaque cas.
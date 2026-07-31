# 🛠️ SadServers

[SadServers](https://sadservers.com/) est une plateforme orientée CTF-Like qui met à disposition une VM Linux sur laquelle un problème concret a été délibérément introduit : service cassé, permissions mal configurées, réseau mal routé, script buggé, BDD à restaurer, etc. La plateforme couvre de nombreux topics comme Linux, Docker, BDD, Monitoring, etc. L'objectif ici est de diagnostiquer soi-même le soucis comme on le ferait en tant qu'admin-sys avec des outils `systemctl`, `journalctl`, `iptables`, permissions, et ainsi de suite.

Voici mes prises de notes au fil de ces challenges : le raisonnement suivi, les pistes explorées (bonnes comme mauvaises), et les solutions retenues pour chaque cas.
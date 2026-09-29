---
description: Découverte de Ansible
---

# ☁️ Découverte et mise en place

Dans le cadre de ma reconversion en tant que sysadmin, je publierais ici mes notes prises durant la formation de Cocadmin sur Ansible. Une formation que m'a été hyper utile pour me remettre en jambes sur cet outil que j'avais déjà utilisé durant mon alternance de 2022 pour déployer et provisioner une VM servant à copier des fichiers d'un vieux serveur de prod vers un autre.

La première étape de ma formation consiste à mettre en place un environnement de test permettant d'apprendre et d'utiliser Ansible.

Les objectifs sont divisés dans l'ordre suivant :

* Déployer plusieurs machines virtuelles de test avec Multipass.
* Faciliter la résolution de noms malgré les changements d'IP dû au DHCP.
* Préparer les serveurs pour les connexions SSH d'Ansible.
* Installer Ansible et vérifier le bon fonctionnement de l'inventaire.

Pour information, j'utilise la config suivant : VM Multipass KVM, dans une VM Ubuntu, elle même sur VirtualBox, sur Windows 11.


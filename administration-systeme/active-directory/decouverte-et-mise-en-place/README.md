---
description: Mes premiers pas sur Active Directory :)
---

# 💻 Découverte et mise en place

Au départ, je ne connaissais pas du tout Active Directory. Je savais que c'était un serveur d'authentification LDAP pour les machines Windows en entreprise mais je n'avais jamais touché à quoi que ce soit dessus.&#x20;

J'ai donc sollicité l'aide d'un administrateur système travaillant dans le public sur Discord, afin qu'il puisse me donner un TP couvrant les bases de la mise en place d'un serveur Active Directory.

Il était question de mettre en place un serveur pour une entreprise nommée "MesMots" (en référence à mon pseudo :D) avec différents comptes appartenant à un groupe. Ces derniers auraient des permissions pour accéder à différents dossiers hébergés sur un partage SMB. Le contexte donné était le suivant :

"_**MesMots est une entreprise fictive spécialisée dans l'enseignement des lettres et de la littérature. Elle compte 24 collaborateurs répartis en quatre services distincts. Ce TP s'appuie sur cette structure pour mettre en pratique la gestion des identités et des accès sous Windows Server 2019.**_"

Les objectifs du TP étaient les suivants :

* Créer et organiser l'Active Directory (OU, Groupes, Utilisateurs)
* Configurer un partage SMB avec lecteur réseau mappé sur X:
* Appliquer des permissions NTFS granulaires par service
* Comprendre la ségrégation des droits d'accès en entreprise
* Mettre en œuvre les bonnes pratiques de sécurité Windows Server

L’environnement repose sur un serveur **Windows Server 2019** choisi car encore largement présent en entreprise au moment où j'écris ces lignes. Il sera configuré en contrôleur de domaine, avec une gestion des utilisateurs appliquées, des groupes, des unités d’organisation et des partages de fichiers.

Je devais me baser sur l'organigramme des services suivants :

<figure><img src="../../../.gitbook/assets/image (71).png" alt=""><figcaption></figcaption></figure>

Sans plus attendre commençons !


































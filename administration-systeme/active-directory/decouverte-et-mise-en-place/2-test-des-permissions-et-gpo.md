# 2 - Test des permissions et GPO

Comme spécifié dans la nomenclature, l'un des membres du groupes "**GG\_GRP\_ADMIN**" doit pouvoir accéder au dossier "**Administratif**" et aux sous dossiers en lecture et écriture sans soucis dessus. On va donc ouvrir une session avec un utilisateur faisant partie du groupe mentionné, en l'occurrence "claurent", suivi d'un mot de passe que l'utilisateur devra changer :&#x20;

<figure><img src="../../../.gitbook/assets/image (99).png" alt=""><figcaption></figcaption></figure>

On va donc monter le partage avec la commande "**net use**" sur la lettre **X:**, car Windows s'authentifie directement avec la session en cours :

```powershell
 net use X: \\mesmots.local\ADMINISTRATIF
```

Le partage apparait alors :

<figure><img src="../../../.gitbook/assets/image (100).png" alt=""><figcaption></figcaption></figure>

Et on peut donc écrire dessus, avec les bons droits comme demandés. Par exemple ici, dans le dossier "ADMINISTRATIF/RH" du partage :&#x20;

<figure><img src="../../../.gitbook/assets/image (101).png" alt=""><figcaption></figcaption></figure>

A l'inverse quand on essaye d'accéder à un autre dossier interdit, comme "DIRECTION", on ne peut pas :

<figure><img src="../../../.gitbook/assets/image (102).png" alt=""><figcaption></figcaption></figure>

C'est là où on voit que les permissions NTFS vont bien leurs effets ! Cependant, un utilisateur va trouver rébarbatif à chaque fois d'ouvrir l'explorateur de fichiers pour se connecter sur le partage. Et pas question de passer par une commande CMD ou PowerShell.

On va donc passer par la création d'une GPO pour permettre d'avoir le lecteur qui soit monté directement à l'ouverture de la session.

### Création de la GPO

On créé donc une GPO "**U - Connecter - Lecteur - Réseau**" où le lecteur réseau sera mappé en tant que "**X:**" comme demandé sur le TP. Je ne savais pas faire ça avant, heureusement l'excellent blog It-Connect en a publié la procédure au moment où j'écris ces lignes :&#x20;

{% embed url="https://www.it-connect.fr/windows-comment-ajouter-des-emplacements-reseau-par-gpo/" %}

En suivant le tuto j'ai donc ma GPO qui se présente comme ceci :

<figure><img src="../../../.gitbook/assets/image (107).png" alt=""><figcaption></figcaption></figure>

Pour comprendre, le premier partage "P:" correspond au profil perso, voulu dans le cahier des charges. Il s'agit en fait du dossier personnel par utilisateur qui prend la variable de l'utilisateur en propriété. (ex: **X:\PROFIL\_PERSO\Jean.MARTIN**). Il est configuré comme suit :&#x20;

<figure><img src="../../../.gitbook/assets/image (108).png" alt=""><figcaption></figcaption></figure>

De façon analogue, pour le dossier de Partages, qui lui prendra la lettre "**X:**" :&#x20;

<figure><img src="../../../.gitbook/assets/image (109).png" alt=""><figcaption></figcaption></figure>

On y inclus ensuite tout les membres des groupes, avec les utilisateurs authentifiés :&#x20;

<figure><img src="../../../.gitbook/assets/image (110).png" alt=""><figcaption></figcaption></figure>

Pour valider le bon fonctionnement de la GPO, on se connecte sur un poste Windows avec un compte ciblé par la GPO (appartenant au bon groupe de sécurité si le ciblage a été configuré). En l'occurence, on peut toujours rester sur "claurent".

On exécute un `gpupdate /force` pour forcer l'actualisation des stratégies et récupérer la configuration depuis le contrôleur de domaine.

Dans l'Explorateur de fichiers, sous **Ce PC** > **Emplacements réseau**, le raccourci nommé "**Partage**" apparaît bien, tout comme notre "**PROFIL\_PERSO**", confirmant que la GPO est appliquée correctement :&#x20;

<figure><img src="../../../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

### Définition d'un quota sur le partage :

Pour éviter que le partage arrive à saturation, il est bon ton de définir un quota à ne pas dépasser pour les utilisateurs. Il est possible sur Windows Server de mettre une limite sur le dossier&#x20;

On commence par installer le rôle FSRM :

<figure><img src="../../../.gitbook/assets/image (112).png" alt=""><figcaption></figcaption></figure>

On choisit une limite de 100 Mo pour le partage que j'applique sur les profils persos. Même si c'est pas assez (car le partage fait 10Go/24users, donc peu ^^), ça permet de voir ce que ça donne et d'adopter une approche vraie de ce que l'on trouve en entreprise.

<figure><img src="../../../.gitbook/assets/image (113).png" alt=""><figcaption></figcaption></figure>

Résultat lorsqu'on est connecté sur une session, la limite change pour passer à 100Mo :

<figure><img src="../../../.gitbook/assets/image (114).png" alt=""><figcaption></figcaption></figure>

Mettre un fichier de + de 100Mo dans son partage est donc impossible :

<figure><img src="../../../.gitbook/assets/image (115).png" alt=""><figcaption></figcaption></figure>

Pour finir, on active le bureau à distance :

<figure><img src="../../../.gitbook/assets/image (116).png" alt=""><figcaption></figcaption></figure>

Avec le port 3389 par défaut à changer quand même.

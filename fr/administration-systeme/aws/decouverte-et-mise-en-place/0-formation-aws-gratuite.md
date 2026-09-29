# 0 - Concepts & Fondamentaux

#### Compte root vs utilisateur IAM

* Le compte **root** est le compte principal créé à l'ouverture d'un compte AWS. Certaines actions ne peuvent être réalisées qu'avec ce compte. Il ne doit **jamais** être utilisé en production, car il dispose d'un accès total et sans restriction à l'ensemble des ressources du compte. Et évidemment protégé par mot de passe fort + 2FA.
* Si le compte root est hacké, catastrophe : il faut démontrer/justifier de nombreux éléments, sans pouvoir identifier la personne à l'origine de la création du compte (si la personne est un ancien employé qui est parti par exemple). Il ne doit jamais être utilisé au quotidien, ni en production. Une fois le compte AWS mis en place, les utilisateurs IAM prennent le relais.

#### IAM (Identity and Access Management)

Sur un compte AWS, IAM est le système qui gère qui a le droit de faire quoi sur les ressources du compte. Il utilise l'authentification et les autorisations pour gérer **utilisateurs**, **groupes**, les **rôles** et **politiques** afin de contrôler précisément l'accès aux différents services AWS (S3, EC2, VPC, etc.).

**Par exemple :**

* Donner à un développeur un accès en lecture seule à un bucket S3.
* Autoriser une instance EC2 à écrire des données dans CloudWatch.
* Permettre à un administrateur de gérer l'ensemble des ressources, sans pour autant utiliser le compte root.

**Note :** Tout ce qui peut être réalisé depuis l'interface graphique de la console AWS peut également être réalisé via l'API. La console n'est donc qu'une façon parmi d'autres d'interagir avec les mêmes fonctionnalités sous-jacentes. Pour les besoins de la découverte de AWS, les deux parties seront abordés.

#### Types de comptes et support

AWS propose différents types de compte :

* **Free Tier**, pour découvrir les services avec un usage gratuit limité.
* Un autre orienté **développeur**.
* Un autre orienté **business**, incluant un support proposé par AWS.

Le support demeure tout de même utile en production, en cas de souci de facturation ou de problème technique nécessitant l'aide direct d'un personnel AWS.

#### Account ID

Il s'agit d'un identifiant à 12 chiffres partagé par le compte root, ainsi que par tous les utilisateurs et tous les rôles IAM créés au sein de ce compte.

#### Organisation multi-comptes (Identity Center)

AWS Identity Center permet de créer des comptes en fonction de différents usecases : un pour la facturation globale, un pour le développement, un pour hoster une VM spécifique, etc...

#### Permissions et tags

Les permissions AWS sont des groupes de permissions prêts à l'emploi. Toute ressource créée dans AWS est une ressource à part entière, à laquelle des règles et des permissions peuvent s'appliquer.

**Note : La politique `AdministratorAccess` accorde des droits d'administration complets, à l'exception de la gestion de la facturation.**

Les **tags** quand à eux servent à étiqueter une ressource. Ils demeurent utiles pour distinguer les environnements (développement, staging, production) et pour filtrer les coûts associés.

### Bonnes pratiques concernant les clés d'accès

* Ne jamais en stocker une en clair, que ce soit dans un dépôt de code ou dans le code source. Si cette dernière venait à être publié accidentellement dans un repo GitHub/GitLab, même le temps d'une 1 minute, partir du principe que cette dernière est compromise et qu'il faut la révoquer.
* La désactiver ou la supprimer une fois qu'elle n'est plus nécessaire.
* Appliquer le principe du moindre privilège lors de l'attribution des permissions.
* Assurer une rotation régulière des clés d'accès, pour limiter les risques en cas de fuite.

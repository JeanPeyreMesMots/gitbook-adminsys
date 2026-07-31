## Résumé rapide

Passons maintenant à l'installation de l'AWS CLI sur une VM Ubuntu, suivi de la création d'un utilisateur IAM via CLI (avec gen d'une access key), puis configuration de ce dernier (`aws configure`).
### 1. Installation de l'AWS CLI

Réalisée sur une VM Ubuntu, en suivant la [documentation officielle d'installation](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

L'installation crée un dossier de configuration dédié :

```bash
jpmm@kos-boss:~/.aws$ ll
total 16
drwxrwxr-x  2 jpmm jpmm 4096 mai   29 20:03 ./
drwxr-x--- 22 jpmm jpmm 4096 mai   29 20:03 ../
-rw-------  1 jpmm jpmm   28 mai   29 20:03 config
-rw-------  1 jpmm jpmm  116 mai   29 20:03 credentials
```

- `config` contient la configuration générale (région, format de sortie, etc.).
- `credentials` contient les identifiants d'accès (access key/secret key).

L'[autocomplétion](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-completion.html) peut également être activée pour faciliter l'usage de la CLI.

### 2. Création d'un utilisateur IAM (démo console)

Étapes suivies dans la console AWS :

1. Aller dans **IAM** → **Users**.
2. Cliquer sur **Add user**.
3. Activer l'accès à la console (**access to console**) pour cet utilisateur IAM.
4. Définir un mot de passe personnalisé (**custom password**), sans exiger de réinitialisation à la première connexion.
5. Attacher les politiques de permissions existantes nécessaires (**Attach existing policies**). Ne pas ajouter de tags à cette étape.
6. Retourner à la liste des utilisateurs, et sélectionner l'utilisateur créé, puis aller dans l'onglet **Security credentials**.
7. Dans la section liée à la CLI, valider la prise de connaissance des conditions (**"CLI, I understand"**).
8. Créer une clé d'accès (**Create access key**), qui servira à configurer l'AWS CLI.

### 3. Configuration de l'AWS CLI

Une fois la clé d'accès générée, la CLI est configurée avec la commande :

```bash
aws configure
```

Les informations suivantes sont demandées :

- **Access key ID** et **Secret access key**, récupérées à l'étape précédente.
- **Région par défaut** (default region) : point important, car elle détermine la région AWS utilisée par défaut pour toutes les commandes CLI qui n'en précisent pas explicitement une autre. J'ai gardé us-east-1 car fuseau horaire indiqué par Cocadmin.
- **Format de sortie** : `json` par défaut.

=> Ces informations sont enregistrées dans le fichier `~/.aws/config`.

### 4. Première commande de test

```bash
aws s3 ls
```

Cette commande liste les buckets S3 accessibles avec les identifiants configurés, et permet de vérifier que la CLI est correctement configurée.

# 3 - Hardening du serveur

Étant intéressé par la cybersécurité, je me suis intéressé par la sécurité de l'AD déjà déployé.. On va utiliser Ping Castle va nous permettre. Un scan Ping Castle lancé sur le domaine directement nous donne un mauvais score :

<figure><img src="../../../.gitbook/assets/image (117).png" alt=""><figcaption></figcaption></figure>

Ce qui est courant sur une conf par défaut quand on installe. De nombreux points d'améliorations sont évoqués :

| Risk model                       | Score | Raison affichée dans le rapport                                                                                                                                                                                                                                                                                                                                   |
| -------------------------------- | ----- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Anomalies**                    | 70    | Le spooler est accessible à distance depuis 1 DC, et LAPS n’est pas installé ; d’autres mitigations ASR ne sont pas toutes activées [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                               |
| **Privileged accounts**          | 40    | Présence de comptes admin non protégés par l’option “sensitive and cannot be delegated” [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                                           |
| **Stale Objects**                | 31    | Un seul DC, dernière sauvegarde AD il y a 20 jours, et d’autres règles sur les objets “stale” [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                                     |
| **Pass-the-credential**          | 25    | Le spooler est accessible à distance depuis 1 DC, ce qui augmente le risque de vol/réutilisation d’identifiants [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                   |
| **Account take over**            | 20    | Présence de comptes admin non sensibles à la délégation [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                                                                           |
| **Backup**                       | 20    | Nombre insuffisant de DC pour la redondance, et dernière sauvegarde AD trop ancienne [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                                              |
| **Irreversible change**          | 20    | OU sans protection contre la suppression accidentelle, Schema Admins non vide, Recycle Bin non activée [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                            |
| **Old authentication protocols** | 15    | Le paramètre LAN Manager autorise NTLMv1 ou LM [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                                                                                    |
| **Audit**                        | 10    | L’audit PowerShell n’est pas complètement activé, et l’audit des DC ne collecte pas certains événements clés [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                      |
| **Local group vulnerability**    | 10    | Les utilisateurs non-admin peuvent ajouter jusqu’à 10 ordinateurs au domaine [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                                                      |
| **Provisioning**                 | 10    | Le mot de passe de certains comptes n’expire jamais, et le niveau de sécurité des mots de passe est insuffisant [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                   |
| **Weak password**                | 10    | Pas de politique de mot de passe pour les comptes de service, et au moins une politique avec longueur inférieure à 8 caractères [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                   |
| **Network topography**           | 5     | Risque d’exécution de scripts Internet depuis les DC, et un subnet DC manque dans la déclaration [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                                  |
| **Network sniffing**             | 5     | Hardened Paths ont été abaissés, et LLMNR n’est pas désactivé partout [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                                                                                                             |
| **Object configuration**         | 1     | Defender ASR n’est pas entièrement en mode Block/Warn, certaines options de fichiers sont risquées, Kerberos Armoring est à vérifier, et Terminal Services GPO n’est pas conforme [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html). |
| **Reconnaissance**               | 0     | DsHeuristics ne mitige pas CVE-2021-42291, NetCease n’est pas trouvé, PreWin2000 contient “Authenticated Users”, et Anonymous Binding au rootDSE est activé [ad\_hc\_mesmots.local.html](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/344793290/e55608b6-bc95-4ce6-a6f1-f30b86f093ad/ad_hc_mesmots.local.html).                       |

On va donc régler tout ça ;)

### Désactiver NTLM :

NTLM n'a plus besoin d'être présenté comme ayant besoin d'être désactivé, il est obsolète et facilement utilisable dans les attaques NTLM Relay. On peut désactiver l'authentification NTLM via la GPO suivante :&#x20;

<figure><img src="../../../.gitbook/assets/image (118).png" alt=""><figcaption></figcaption></figure>

Il faut aussi appliquer ce paramètres, pour que NTMLv2 ne soit utilisé que au niveau LAN Manager :

<figure><img src="../../../.gitbook/assets/image (119).png" alt=""><figcaption></figcaption></figure>

On vérifie dans l'état de la GPO :

<figure><img src="../../../.gitbook/assets/image (120).png" alt=""><figcaption></figcaption></figure>

Le stale object passe ensuite de 31/100 à :

<figure><img src="../../../.gitbook/assets/image (121).png" alt=""><figcaption></figcaption></figure>

C'est déjà mieux ! Mais continuons.

### Activer le flag anti délégation pour le compte admin :

<figure><img src="../../../.gitbook/assets/image (122).png" alt=""><figcaption></figcaption></figure>

Cela permet d'éviter qu'un service compromis vole le TGT Kerberos de l'Admin. Le stale object en question a diminué ainsi :&#x20;

<figure><img src="../../../.gitbook/assets/image (123).png" alt=""><figcaption></figcaption></figure>

### Protection contre la suppression accidentelle des OU :

Les OU suivantes n'ont pas le flag anti delete activés :

<figure><img src="../../../.gitbook/assets/image (124).png" alt=""><figcaption></figcaption></figure>

On l'active donc pour toutes les OU :

```powershell
Get-ADOrganizationalUnit -Filter * | Set-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $true
```

### Schema Admins non vide (1 compte)

Le groupe **Schema Admins** contient 1 compte au lieu d'être vide : le compte "**Administrateur**"

```powershell
Remove-ADGroupMember "Administrateurs du schéma" -Members "Administrateur"
```

Ce groupe a des privilèges sur le schéma AD (irréversible). Il doit être vide en production, mais dans notre cas je choisis de le laisser pour éviter de ne plus pouvoir être connecté sur l'AD en Admin justement.

### Corbeille non activée :

```powershell
# Active la corbeille AD
Enable-ADOptionalFeature -Identity 'Recycle Bin Feature' -Scope ForestOrConfigurationSet -Target 'mesmots.local'
```

Cela permet de **récupérer n'importe quel objet supprimé** pendant 180 jours au lieu de le perdre définitivement.

Priv Accounts sur le rapport tombe désormais à 0 :&#x20;

<figure><img src="../../../.gitbook/assets/image (125).png" alt=""><figcaption></figcaption></figure>

### Absence de backup :

<figure><img src="../../../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

Très important, mais pas utile dans notre contexte car lab locale évidemment.

### Spooler accessible à distance

Le service **Print Spooler** sur ton DC est accessible via RPC (MS-RPRN), ce qui expose à **PrintNightmare** et autres attaques RCE sur les DC. Aucune raison que les DC impriment donc à désactiver en powershell :

```powershell
# Sur ton DC, désactive complètement le spooler distant
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Spooler" -Name "DependOnService" -Value @("RPCSS")
Stop-Service Spooler -Force
Set-Service Spooler -StartupType Disabled
```

### Configuration du LAPS

LAPS permet de gérer la rotation des mots de passes admin des machines connectées au domaine : Windows LAPS va **générer un mot de passe robuste et unique pour le compte administrateur local de chaque machine** qu'il gère, tout en effectuant **une rotation automatique de ces mots de passe**. Ensuite, les sésames seront chiffrés et stockés dans l'Active Directory ou l'Azure Active Directory, selon la configuration mise en place.

Sur l'AD, on peut donc utiliser la commande PowerShell suivante, qui va donner les droits d'écriture pour l'OU des machines à stocker l'objet dans les machines :

```powershell
Set-LapsADComputerSelfPermission -Identity "OU=Ordinateurs,OU=MesMots,DC=mesmots,DC=local"
```

Ensuite avec une GPO > Configuration ordinateur > Stratégies > Modèles d'administration > Système > LAPS :

<figure><img src="../../../.gitbook/assets/image (127).png" alt=""><figcaption></figcaption></figure>

On garde au moins un mot de passe en historique en cas de lockout :

<figure><img src="../../../.gitbook/assets/image (128).png" alt=""><figcaption></figcaption></figure>

On se retrouve donc avec ça :

<figure><img src="../../../.gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

On passe ensuite à la conf sur le PC client, successivement :

```powershell
gpupdate /force

# Reboot pour appliquer la conf
Restart-Computer
```

On rouvre une session sur le PC client, puis on remarque que sur le journal d'événements de l'AD :

**Journaux des applications et des services > Microsoft > Windows > LAPS > Operational**

<figure><img src="../../../.gitbook/assets/image (130).png" alt=""><figcaption></figcaption></figure>

Le LAPS s'est bien appliqué.

On récupère ensuite le mot de passe assigné pour le PC en question via la commande Powershell :

```powershell
PS C:\Users\Administrateur> Get-LapsADPassword "WIN11-HOME-LAB" -AsPlainText

ComputerName        : WIN11-HOME-LAB
DistinguishedName   : CN=WIN11-HOME-LAB,OU=Ordinateurs,OU=MesMots,DC=mesmots,DC=local
Account             : Administrateur
Password            : XXXXXXXXXXXXXXX
PasswordUpdateTime  : 09/05/2026 23:17:40
ExpirationTimestamp : 08/06/2026 23:17:40
Source              : EncryptedPassword
DecryptionStatus    : Success
AuthorizedDecryptor : MESMOTS0\Admins du domaine
```

On peut désormais lancer un programme en tant qu'admin sur le PC client en spécifiant bien dans l'UAC qui s'affiche :

* ".\Administrateur" comme username (j'oubliais le .\ pour spécifier que c'est un compte local au début ce qui fait que j'ai été bloqué pour rien 1h haha)
* Et le mot de passe généré par LAPS

<figure><img src="../../../.gitbook/assets/image (131).png" alt=""><figcaption></figcaption></figure>

On a maintenant un shell "System32" sur le PC ce qui veut dire que le compte admin passe bien !

<figure><img src="../../../.gitbook/assets/image (132).png" alt=""><figcaption></figcaption></figure>

Et le score PingCastle baisse à nouveau :

<figure><img src="../../../.gitbook/assets/image (133).png" alt=""><figcaption></figcaption></figure>

### Bloquer internet pour les moteur de scripts malveillants

Les moteurs de scripts malveillants (certains sont des LOLbins) suivants :

**wscript.exe, cscript.exe, mshta.exe, conhost.exe, runScriptHelper.exe**

peuvent se connecter directement à Internet pour exfiltrer des données ou télécharger des charges. Par défaut, le pare-feu Windows ne les bloque pas en sortie.

**Solution via GPO** :

* Édite une GPO liée aux DCs et postes (`Ordinateur > Configuration > Paramètres Windows > Pare-feu Windows avec sécurité avancée`) que j'ai appelé "**Sécurité - Bloquer moteur de scripts /= internet**" :

<figure><img src="../../../.gitbook/assets/image (134).png" alt=""><figcaption></figcaption></figure>

```
Programme : %SystemRoot%\System32\wscript.exe (et les autres)
```

Action :

* Bloquer la connexion

On se retrouve donc avec les programmes suivants bloqués :

<figure><img src="../../../.gitbook/assets/image (135).png" alt=""><figcaption></figcaption></figure>

### Sous réseau DC

Bien que l'AD a une IP fixe, je choisis quand même de créer un sous réseau dédié pour que l'AD puisse optimiser les connexions clientes :

<figure><img src="../../../.gitbook/assets/image (136).png" alt=""><figcaption></figcaption></figure>

Le site affiché et ensuite correct :

```powershell
PS C:\Users\Administrateur> nltest /dsgetsite
Default-First-Site-Name
La commande a été correctement exécutée
```

Avec ces deux règles Stale Objects est passé de 31 à **11/100** (S-OldNtlm corrigé via ta GPO "**Sécurité - Désactiver NTLM**") :

<figure><img src="../../../.gitbook/assets/image (137).png" alt=""><figcaption></figcaption></figure>

PingCastle continue à flagger ça :

<figure><img src="../../../.gitbook/assets/image (139).png" alt=""><figcaption></figcaption></figure>

Mais normal car **Runscripthelper.exe** était présent uniquement sur **Windows 10 build 16299**. L'AD tourne sur **Windows Server 2019**, où ce binaire n'existe tout simplement plus — Microsoft l'a retiré dans les versions suivantes.

Il reste encore beaucoup à faire pour arriver à un score à 0 (moins probable en production). Ce qui fait que le score reste très élevé est dû au fait que l'AD ne dispose pas d'un backup, ce qui est normale car l'AD est en locale sur mon PC.

Tout les autres points sont des **"informative rules" (0 point)** - typiques labo solo.


































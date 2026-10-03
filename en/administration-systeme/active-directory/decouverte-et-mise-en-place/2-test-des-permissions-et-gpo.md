# 2 - Permissions testing and GPOs

As specified in the naming scheme, any member of the "**GG\_GRP\_ADMIN**" group must be able to access the "**Administratif**" folder and its subfolders with read and write permissions without any issue. So we open a session with a user who belongs to that group, in this case "**claurent**", followed by the password that the user will have to change:

<figure><img src="../../../.gitbook/assets/image (99).png" alt=""><figcaption></figcaption></figure>

We then mount the share with the "**net use**" command on drive letter **X:**, thanks to Windows authenticating with the current session name:

```powershell
 net use X: \\mesmots.local\ADMINISTRATIF
```

The share then appears:

<figure><img src="../../../.gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>

And we can write to it, with the right permissions as requested. For example here, in the "**ADMINISTRATIF/RH**" folder of the share:

<figure><img src="../../../.gitbook/assets/image (86).png" alt=""><figcaption></figcaption></figure>

At contrary, when we try to access another forbidden folder such as "**DIRECTION**", we can't:

<figure><img src="../../../.gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

This is where we can see that the NTFS permissions are doing their job! However, a user will find it tedious to open File Explorer every time to connect to the share, and using a CMD or PowerShell command is out of the question.

We then must create a GPO so that the drive is mounted directly when the session opens.

### Creating the GPO

We create a GPO named "**U - Connecter - Lecteur - Réseau**" where the network drive will be mapped as "**X:**", as requested in the lab. I didn't know how to do this beforehand, fortunately the excellent blog [IT-Connect](https://www.it-connect.fr/) (in French) published the procedure at the time of writing:

{% embed url="https://www.it-connect.fr/windows-comment-ajouter-des-emplacements-reseau-par-gpo/" %}

Following the tutorial, my GPO looks like this:

<figure><img src="../../../.gitbook/assets/image (107).png" alt=""><figcaption></figcaption></figure>

To explain: the first share, "**P:**", corresponds to the personal profile required in the specifications. It is a personal folder per user, which uses the user variable as its path (e.g. **X:\PROFIL\_PERSO\Jean.MARTIN**). It is configured as follows:

<figure><img src="../../../.gitbook/assets/image (108).png" alt=""><figcaption></figcaption></figure>

Similarly, for the Shares folder, which takes the letter "**X:**":

<figure><img src="../../../.gitbook/assets/image (109).png" alt=""><figcaption></figcaption></figure>

We then include all the members of the groups, along with authenticated users:

<figure><img src="../../../.gitbook/assets/image (110).png" alt=""><figcaption></figcaption></figure>

To validate that the GPO works, we log in on a Windows workstation with an account targeted by the GPO (belonging to the right security group if targeting has been configured). Here, we can keep using "**claurent**".

We run `gpupdate /force` to force a policy refresh and fetch the configuration from the domain controller.

In File Explorer, under **This PC** > **Network locations**, the shortcut named "**Partage**" does appear, so does our "**PROFIL\_PERSO**", confirming that the GPO is applied correctly:

<figure><img src="../../../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

### Setting a quota on the share:

To prevent the share from becoming full, it is good practice to define a quota that users must not exceed. On Windows Server, it is possible to set a limit on a folder.

We start by installing the FSRM role:

<figure><img src="../../../.gitbook/assets/image (112).png" alt=""><figcaption></figcaption></figure>

We choose a limit of 100 MB for the share, which I apply to the personal profiles. Even if it's not enough (the share is 10 GB for 24 users, not so much ^^), it lets us see how it behaves and adopt an approach close to what you find in a company.

<figure><img src="../../../.gitbook/assets/image (113).png" alt=""><figcaption></figcaption></figure>

Result: when logged in to a session, the limit changes to 100 MB:

<figure><img src="../../../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

Putting a file larger than 100 MB in your share is therefore impossible:

<figure><img src="../../../.gitbook/assets/image (115).png" alt=""><figcaption></figcaption></figure>

Finally, we enable Remote Desktop:

<figure><img src="../../../.gitbook/assets/image (116).png" alt=""><figcaption></figcaption></figure>

The default port 3389 still # 2 - Permissions testing and GPOs

As specified in the naming scheme, any member of the "**GG\_GRP\_ADMIN**" group must be able to access the "**Administratif**" folder and its subfolders with read and write permissions without any issue. So we open a session with a user who belongs to that group, in this case "**claurent**", followed by a password that the user will have to change:

<figure><img src="../../../.gitbook/assets/image (99).png" alt=""><figcaption></figcaption></figure>

We then mount the share with the "**net use**" command on drive letter **X:**, since Windows authenticates directly with the current session:

```powershell
 net use X: \\mesmots.local\ADMINISTRATIF
```

The share then appears:

<figure><img src="../../../.gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>

And we can write to it, with the right permissions as requested. For example here, in the "**ADMINISTRATIF/RH**" folder of the share:

<figure><img src="../../../.gitbook/assets/image (86).png" alt=""><figcaption></figcaption></figure>

Conversely, when we try to access another forbidden folder such as "**DIRECTION**", we can't:

<figure><img src="../../../.gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

This is where we can see that the NTFS permissions are doing their job! However, a user will find it tedious to open File Explorer every time to connect to the share, and using a CMD or PowerShell command is out of the question.

So we will create a GPO so that the drive is mounted directly when the session opens.

### Creating the GPO

We create a GPO named "**U - Connecter - Lecteur - Réseau**" where the network drive will be mapped as "**X:**", as requested in the lab. I didn't know how to do this beforehand, fortunately the excellent blog [IT-Connect](https://www.it-connect.fr/) (in French) published the procedure at the time of writing:

{% embed url="https://www.it-connect.fr/windows-comment-ajouter-des-emplacements-reseau-par-gpo/" %}

Following the tutorial, my GPO looks like this:

<figure><img src="../../../.gitbook/assets/image (107).png" alt=""><figcaption></figcaption></figure>

To explain: the first share, "**P:**", corresponds to the personal profile required in the specifications. It is a personal folder per user, which uses the user variable as its path (e.g. **X:\PROFIL\_PERSO\Jean.MARTIN**). It is configured as follows:

<figure><img src="../../../.gitbook/assets/image (108).png" alt=""><figcaption></figcaption></figure>

Similarly, for the Shares folder, which takes the letter "**X:**":

<figure><img src="../../../.gitbook/assets/image (109).png" alt=""><figcaption></figcaption></figure>

We then include all the members of the groups, along with authenticated users:

<figure><img src="../../../.gitbook/assets/image (110).png" alt=""><figcaption></figcaption></figure>

To validate that the GPO works, we log in on a Windows workstation with an account targeted by the GPO (belonging to the right security group if targeting has been configured). Here, we can keep using "**claurent**".

We run `gpupdate /force` to force a policy refresh and fetch the configuration from the domain controller.

In File Explorer, under **This PC** > **Network locations**, the shortcut named "**Partage**" does appear, as does our "**PROFIL\_PERSO**", confirming that the GPO is applied correctly:

<figure><img src="../../../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

### Setting a quota on the share:

To prevent the share from becoming full, it is good practice to define a quota that users must not exceed. On Windows Server, it is possible to set a limit on a folder.

We start by installing the FSRM role:

<figure><img src="../../../.gitbook/assets/image (112).png" alt=""><figcaption></figcaption></figure>

We choose a limit of 100 MB for the share, which I apply to the personal profiles. Even if it's not enough (the share is 10 GB for 24 users, so not much ^^), it lets us see how it behaves and adopt an approach close to what you find in a company.

<figure><img src="../../../.gitbook/assets/image (113).png" alt=""><figcaption></figcaption></figure>

Result: when logged in to a session, the limit changes to 100 MB:

<figure><img src="../../../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

Putting a file larger than 100 MB in your share is therefore impossible:

<figure><img src="../../../.gitbook/assets/image (115).png" alt=""><figcaption></figcaption></figure>

Finally, we enable Remote Desktop:

<figure><img src="../../../.gitbook/assets/image (116).png" alt=""><figcaption></figcaption></figure>

The default port 3389 should be changed, though.
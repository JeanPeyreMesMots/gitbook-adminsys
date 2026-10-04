# 2 - Permissions testing and GPOs

As defined in the requirements, members of the "**GG\_GRP\_ADMIN**" group must have read and write access to the "**Administratif**" folder and its subfolders. To test this, we log in as a member of that group, "**claurent**", using a temporary password that must be changed at first logon:

<figure><img src="../../../.gitbook/assets/image (99).png" alt=""><figcaption></figcaption></figure>

We mount the share on **X:** with the "**net use**" command. No credentials are needed, since Windows uses the current session:

```powershell
 net use X: \\mesmots.local\ADMINISTRATIF
```

The share then appears:

<figure><img src="../../../.gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>

And we can write to it, as expected. For example, in the share's "**ADMINISTRATIF/RH**" folder:

<figure><img src="../../../.gitbook/assets/image (86).png" alt=""><figcaption></figcaption></figure>

Conversely, access to a forbidden folder such as "**DIRECTION**" is denied:

<figure><img src="../../../.gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

The NTFS permissions are doing their job! However, users won't want to connect to the share manually every time, and asking them to type a command is out of the question.

So we'll create a GPO that maps the drive automatically at logon.

### Creating the GPO

We create a GPO named "**U - Connecter - Lecteur - Réseau**" where the network drive will be mapped as "**X:**", as requested in the lab. I didn't know how to do this yet, but luckily the excellent blog [IT-Connect](https://www.it-connect.fr/) (in French) had just published the procedure:

{% embed url="https://www.it-connect.fr/windows-comment-ajouter-des-emplacements-reseau-par-gpo/" %}

Following the tutorial, my GPO looks like this:

<figure><img src="../../../.gitbook/assets/image (107).png" alt=""><figcaption></figcaption></figure>

The first drive, "**P:**", is the personal folder required by the specifications: one folder per user, whose path is built from the username variable (e.g. **X:\PROFIL\_PERSO\Jean.MARTIN**). It is configured as follows:

<figure><img src="../../../.gitbook/assets/image (108).png" alt=""><figcaption></figcaption></figure>

Same thing for the "**Partages**" folder, mapped to "**X:**":

<figure><img src="../../../.gitbook/assets/image (109).png" alt=""><figcaption></figcaption></figure>

We then target all group members, along with Authenticated Users:

<figure><img src="../../../.gitbook/assets/image (110).png" alt=""><figcaption></figcaption></figure>

To validate the GPO, we log in on a Windows workstation with an account it targets (a member of the right security group, if item-level targeting is configured). We can keep using "**claurent**".

We run `gpupdate /force` to force a policy refresh and fetch the configuration from the domain controller.

In File Explorer, under **This PC** > **Network locations**, both the "**Partage**" and "**PROFIL\_PERSO**" shortcuts appear, confirming that the GPO is applied:

<figure><img src="../../../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

### Setting a quota on the share:

To keep the share from filling up, it's good practice to set a quota per user. Windows Server can enforce a size limit on a folder.

We start by installing the FSRM role:

<figure><img src="../../../.gitbook/assets/image (112).png" alt=""><figcaption></figcaption></figure>

I set a 100 MB limit on the personal folders. It's tight (the share is only 10 GB for 24 users ^^), but it shows how quotas behave and mirrors what you'd find in a real company.

<figure><img src="../../../.gitbook/assets/image (113).png" alt=""><figcaption></figcaption></figure>

Result: once logged in, the user sees a 100 MB limit:

<figure><img src="../../../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

Copying a file larger than 100 MB to the personal folder fails:

<figure><img src="../../../.gitbook/assets/image (115).png" alt=""><figcaption></figcaption></figure>

Finally, we enable Remote Desktop:

<figure><img src="../../../.gitbook/assets/image (116).png" alt=""><figcaption></figcaption></figure>

In a real environment, the default port 3389 should be changed.
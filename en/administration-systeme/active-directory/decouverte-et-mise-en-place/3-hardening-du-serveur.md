# 3 - Hardening the server

Since I'm into cybersecurity, I wanted to assess how secure my freshly deployed AD was, using PingCastle. A first scan of the domain gives a poor score:

<figure><img src="../../../.gitbook/assets/image (117).png" alt=""><figcaption></figcaption></figure>

That's typical of a default installation. The report raises many points:

| Risk model                       | Score | Reason shown in the report                                                                                                                                              |
| -------------------------------- | ----- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Anomalies**                    | 70    | The spooler is reachable remotely from 1 DC, LAPS is not installed, and some other ASR mitigations are not enabled.                                                     |
| **Privileged accounts**          | 40    | Admin accounts are not protected by the "sensitive and cannot be delegated" option.                                                                                     |
| **Stale Objects**                | 31    | Only one DC, last AD backup 20 days ago, and other rules on "stale" objects.                                                                                            |
| **Pass-the-credential**          | 25    | The spooler is reachable remotely from 1 DC, which increases the risk of credential theft/reuse.                                                                        |
| **Account take over**            | 20    | Admin accounts are not marked as sensitive to delegation.                                                                                                               |
| **Backup**                       | 20    | Not enough DCs for redundancy, and the last AD backup is too old.                                                                                                       |
| **Irreversible change**          | 20    | OUs without protection against accidental deletion, Schema Admins not empty, Recycle Bin not enabled.                                                                   |
| **Old authentication protocols** | 15    | The LAN Manager setting allows NTLMv1 or LM.                                                                                                                            |
| **Audit**                        | 10    | PowerShell auditing is not fully enabled, and DC auditing does not collect some key events.                                                                             |
| **Local group vulnerability**    | 10    | Non-admin users can add up to 10 computers to the domain.                                                                                                               |
| **Provisioning**                 | 10    | The password of some accounts never expires, and the password security level is insufficient.                                                                           |
| **Weak password**                | 10    | No password policy for service accounts, and at least one policy with a length under 8 characters.                                                                      |
| **Network topography**           | 5     | Risk of Internet scripts running from DCs, and a DC subnet is missing from the declaration.                                                                             |
| **Network sniffing**             | 5     | Hardened Paths have been lowered, and LLMNR is not disabled everywhere.                                                                                                 |
| **Object configuration**         | 1     | Defender ASR is not fully in Block/Warn mode, some file options are risky, Kerberos Armoring needs checking, and the Terminal Services GPO is not compliant.             |
| **Reconnaissance**               | 0     | DsHeuristics does not mitigate CVE-2021-42291, NetCease is not found, PreWin2000 contains "Authenticated Users", and Anonymous Binding to the rootDSE is enabled.        |

So let's fix all of that 😎

### Disabling NTLM:

Disabling NTLM is a well-known recommendation: it is obsolete and easily abused in NTLM Relay attacks. We can disable NTLM authentication with the following GPO:

<figure><img src="../../../.gitbook/assets/image (118).png" alt=""><figcaption></figcaption></figure>

We also set the LAN Manager authentication level so that only NTLMv2 is accepted:

<figure><img src="../../../.gitbook/assets/image (119).png" alt=""><figcaption></figcaption></figure>

We check that the GPO is applied:

<figure><img src="../../../.gitbook/assets/image (120).png" alt=""><figcaption></figcaption></figure>

The Stale Objects score then goes from 31/100 to:

<figure><img src="../../../.gitbook/assets/image (121).png" alt=""><figcaption></figcaption></figure>

Better already! Let's keep going.

### Enabling the anti-delegation flag for the admin account:

<figure><img src="../../../.gitbook/assets/image (122).png" alt=""><figcaption></figcaption></figure>

This prevents a compromised service from stealing the Admin's Kerberos TGT through delegation. The related score drops as follows:

<figure><img src="../../../.gitbook/assets/image (123).png" alt=""><figcaption></figcaption></figure>

### Protection against accidental deletion of OUs:

The following OUs do not have the anti-delete flag enabled:

<figure><img src="../../../.gitbook/assets/image (124).png" alt=""><figcaption></figcaption></figure>

So we enable it for all OUs:

```powershell
Get-ADOrganizationalUnit -Filter * | Set-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $true
```

### Schema Admins not empty (1 account)

The **Schema Admins** group contains 1 account instead of being empty: the "**Administrateur**" account.

```powershell
Remove-ADGroupMember "Administrateurs du schéma" -Members "Administrateur"
```

This group can modify the AD schema, and schema changes are irreversible. It should be empty in production; in this lab, I chose not to run the command above, to avoid any risk of locking myself out of the Admin account.

### Recycle Bin not enabled:

```powershell
# Active la corbeille AD
Enable-ADOptionalFeature -Identity 'Recycle Bin Feature' -Scope ForestOrConfigurationSet -Target 'mesmots.local'
```

This makes it possible to **recover any deleted object** for 180 days instead of losing it permanently.

Privileged Accounts in the report now drops to 0:

<figure><img src="../../../.gitbook/assets/image (125).png" alt=""><figcaption></figcaption></figure>

### No backup:

<figure><img src="../../../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

Critical in production, but out of scope here since this is a local lab.

### Spooler reachable remotely

The **Print Spooler** service on the DC is reachable via RPC (MS-RPRN), which exposes it to **PrintNightmare** and other RCE attacks on DCs. A DC has no reason to print, so we disable the service with PowerShell:

```powershell
# Sur ton DC, désactive complètement le spooler distant
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Spooler" -Name "DependOnService" -Value @("RPCSS")
Stop-Service Spooler -Force
Set-Service Spooler -StartupType Disabled
```

### Configuring LAPS

LAPS manages the local admin passwords of domain-joined machines: Windows LAPS **generates a strong, unique password for the local administrator account of each machine it manages**, while **automatically rotating these passwords**. The passwords are then encrypted and stored in Active Directory or Azure Active Directory, depending on the configuration.

On the DC, the following PowerShell command allows computers in the OU to write their own LAPS password attributes:

```powershell
Set-LapsADComputerSelfPermission -Identity "OU=Ordinateurs,OU=MesMots,DC=mesmots,DC=local"
```

Then with a GPO > Computer Configuration > Policies > Administrative Templates > System > LAPS:

<figure><img src="../../../.gitbook/assets/image (127).png" alt=""><figcaption></figcaption></figure>

We keep at least one password in history in case of a lockout:

<figure><img src="../../../.gitbook/assets/image (128).png" alt=""><figcaption></figcaption></figure>

We end up with this:

<figure><img src="../../../.gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

We then move on to the configuration on the client PC, running in order:

```powershell
gpupdate /force

# Reboot pour appliquer la conf
Restart-Computer
```

After logging back in on the client PC, we check its event log:

**Applications and Services Logs > Microsoft > Windows > LAPS > Operational**

<figure><img src="../../../.gitbook/assets/image (130).png" alt=""><figcaption></figcaption></figure>

LAPS was applied successfully. We then retrieve the password assigned to that PC with the following PowerShell command:

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

We can now run a program as admin on the client PC, entering in the UAC prompt:

* "**.\Administrateur**" as the username (I forgot the **.\\** that indicates a local account at first, which got me stuck for almost an hour 😄)
* And the password generated by LAPS:

<figure><img src="../../../.gitbook/assets/image (131).png" alt=""><figcaption></figcaption></figure>

We get an elevated shell in "**System32**", so the LAPS-managed admin account works!

<figure><img src="../../../.gitbook/assets/image (132).png" alt=""><figcaption></figcaption></figure>

And the PingCastle score drops again:

<figure><img src="../../../.gitbook/assets/image (133).png" alt=""><figcaption></figcaption></figure>

### Blocking Internet access for malicious script engines

The following script engines (some of them LOLBins):

**wscript.exe, cscript.exe, mshta.exe, conhost.exe, runScriptHelper.exe**

can connect directly to the Internet to exfiltrate data or download payloads. By default, the Windows firewall does not block them outbound.

**Solution via GPO**:

* Edit a GPO linked to the DCs and workstations (`Computer > Configuration > Windows Settings > Windows Firewall with Advanced Security`), which I named "**Sécurité - Bloquer moteur de scripts /= internet**":

<figure><img src="../../../.gitbook/assets/image (134).png" alt=""><figcaption></figcaption></figure>

```
Programme : %SystemRoot%\System32\wscript.exe (et les autres)
```

Action:

* Block the connection

We end up with the following programs blocked:

<figure><img src="../../../.gitbook/assets/image (135).png" alt=""><figcaption></figcaption></figure>

### DC subnet

Even though the DC has a static IP, I declared its subnet in Sites and Services so that clients are mapped to the right site:

<figure><img src="../../../.gitbook/assets/image (136).png" alt=""><figcaption></figcaption></figure>

The displayed site is then correct:

```powershell
PS C:\Users\Administrateur> nltest /dsgetsite
Default-First-Site-Name
La commande a été correctement exécutée
```

With these two rules, Stale Objects went from 31 to **11/100** (S-OldNtlm fixed via the "**Sécurité - Désactiver NTLM**" GPO):

<figure><img src="../../../.gitbook/assets/image (137).png" alt=""><figcaption></figcaption></figure>

PingCastle still flags this:

<figure><img src="../../../.gitbook/assets/image (138).png" alt=""><figcaption></figcaption></figure>

But that is expected, because **Runscripthelper.exe** only existed on **Windows 10 build 16299**. The AD runs on **Windows Server 2019**, where this binary simply no longer exists: Microsoft removed it in later versions.

There is still work to do to reach a score of 0 (rarely achieved in production anyway). The main thing keeping the score high is the missing backup, which is expected for an AD running locally on my PC.

All the other points are **"informative rules" (0 points)**, typical of a solo lab.
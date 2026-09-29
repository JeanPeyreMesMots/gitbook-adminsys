---
description: >-
  Mise en place d'un lab avec VM Multipass KVM hosted > une VM Ubuntu >
  VirtualBox > Windows 11.
---

# Mise en place du lab

J'avais une VM Ubuntu sur VirtualBox de déjà installé sur mon PC. Il faut d'abord l'arrêter complètement :

```bash
sudo poweroff
```

Tout d'abord il est nécessaire d'activer la virtualisation imbriquée. Cette étape est nécessaire puisque l'on va lancer des VM Multipass dans une VM Ubuntu. On ouvre donc un **PowerShell** en administrateur, puis on se rend dans le répertoire d'installation de VBox où se trouve les binaires :

```powershell
cd "C:\Program Files\Oracle\VirtualBox"
```

On active **Nested VT-x** :

```powershell
.\VBoxManage.exe modifyvm "Ubuntu-JPMM-CLONADO" --nested-hw-virt on
```

Et **Nested Paging** :

```powershell
.\VBoxManage.exe modifyvm "Ubuntu-JPMM-CLONADO" --nestedpaging on
```

```powershell
.\VBoxManage.exe showvminfo "Ubuntu-JPMM-CLONADO" | Select-String "Nested VT-x","Nested Paging","Hardware Virtualization"
```

Résultat attendu :

```
Nested VT-x/AMD-V: enabled
Nested Paging:      enabled
Hardware Virtualization: enabled
```

```
Ubuntu-JPMM-CLONADO
 └── Configuration
      └── Système
           ├── Mémoire : 8000 MB
           └── Processeur : 2 CPU
```

Puis activer les options suivante :

```
[x] Activer VT-x/AMD-V
[x] Pagination imbriquée
```

On démarre ensuite la VM de Ubuntu, puis on vérifie le module kvm qui doit être activé pour transformer Linux en hyperviseur et ainsi exécuter Multipass :

```bash
lsmod | grep kvm
```

On y aperçoit :

```
kvm
irqbypass
```

Puis on vérifie que KVM est exposé :

```bash
ls -l /dev/kvm
```

Ce qui est bien le cas :

```
crw-rw----+ 1 root kvm ... /dev/kvm
```

On peut aussi tester avec cpu-checker par exemple pour voir si on obtient une sortie avec marqué "**kvm-ok**" dedans &#x20;

```bash
sudo apt update
sudo apt install cpu-checker
kvm-ok
```

Résultat :

```
INFO: /dev/kvm exists
KVM acceleration can be used
```

On peut maintenant installer Multipass et déployer nos VM :

```bash
sudo snap install multipass
```

```bash
multipass version
```

```bash
multipass launch 22.04 \
-n ansible-main \
-c 2 \
-m 1G
```

```bash
multipass launch 22.04 \
-n web-server-1 \
-c 1 \
-m 1G
```

```bash
multipass launch 22.04 \
-n web-server-2 \
-c 1 \
-m 1G
```

On vérifie ensuite la liste des VM pour voir si elles obtiennent une IP et sont bien lancées :

```bash
multipass list
```

Exemple :

```
Name            State      IPv4
ansible-main    Running    10.3.241.40
web-server-1    Running    10.3.241.50
web-server-2    Running    10.3.241.60
```

### Note : si Multipass casse après un crash

Nettoyer toutes les VM :

```bash
multipass delete --all
multipass purge
```

Puis vérifier que tout est partie :

```bash
multipass list
```

Si on tombe sur cette erreur :

```
launch failed:
KVM support is not enabled on this machine
```

Cela vient du fait que la pagination : Nested VT-x est parfois désactivé.

Vérifier depuis Windows :

```powershell
.\VBoxManage.exe showvminfo "NOM_VM_VBOX" | Select-String "Nested"
```

Si on vois :

```
Nested VT-x/AMD-V: disabled
Nested Paging: disabled
```

Refaire comme plus haut :

```powershell
.\VBoxManage.exe modifyvm "NOM_VM_VBOX" --nested-hw-virt on
.\VBoxManage.exe modifyvm "NOM_VM_VBOX" --nestedpaging on
```

La bonne configuration finale doit être :

```
VirtualBox
 ├── VT-x/AMD-V          ON
 ├── Nested VT-x/AMD-V   ON
 └── Nested Paging       ON
```

On a maintenant un **mini cluster Ansible de labo dans une VM VirtualBox**, avec KVM accéléré au lieu d'une émulation lente. C'est le bon chemin, avec toutefois les paramètres de couche de virtualisation imbriquée à vérifier.

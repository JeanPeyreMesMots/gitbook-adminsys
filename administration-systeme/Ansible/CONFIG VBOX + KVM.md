😂 Oui, là on a trouvé le bon montage. Je te fais un **recap propre de A à Z** pour refaire la manip si un jour tu casses la config.

L'objectif final :

```text
Windows
│
└── VirtualBox
    │
    └── Ubuntu 24.04 (kos-boss)
        │
        └── Multipass + KVM
            │
            ├── ansible-main   (2 CPU / 1 Go RAM)
            ├── web-server-1   (1 CPU / 1 Go RAM)
            └── web-server-2   (1 CPU / 1 Go RAM)
```

---

# 1) Configuration VirtualBox côté Windows

## Arrêter complètement la VM

Dans Ubuntu :

```bash
sudo poweroff
```

---

## Activer la virtualisation imbriquée

Ouvrir **PowerShell administrateur**.

Se placer où tu veux, puis :

```powershell
cd "C:\Program Files\Oracle\VirtualBox"
```

Activer Nested VT-x :

```powershell
.\VBoxManage.exe modifyvm "Ubuntu-JPMM-CLONADO" --nested-hw-virt on
```

Activer Nested Paging :

```powershell
.\VBoxManage.exe modifyvm "Ubuntu-JPMM-CLONADO" --nestedpaging on
```

---

## Vérifier la configuration

```powershell
.\VBoxManage.exe showvminfo "Ubuntu-JPMM-CLONADO" | Select-String "Nested VT-x","Nested Paging","Hardware Virtualization"
```

Résultat attendu :

```
Nested VT-x/AMD-V: enabled
Nested Paging:      enabled
Hardware Virtualization: enabled
```

---

# 2) Configuration CPU/RAM VirtualBox

Dans VirtualBox :

```
Ubuntu-JPMM-CLONADO
 └── Configuration
      └── Système
           ├── Mémoire : 8000 MB
           └── Processeur : 2 CPU
```

Options :

```
[x] Activer VT-x/AMD-V
[x] Pagination imbriquée
```

---

# 3) Démarrer Ubuntu et vérifier KVM

Dans Ubuntu :

Vérifier le module :

```bash
lsmod | grep kvm
```

Tu dois voir :

```
kvm
irqbypass
```

---

Vérifier que KVM est exposé :

```bash
ls -l /dev/kvm
```

Résultat attendu :

```
crw-rw----+ 1 root kvm ... /dev/kvm
```

---

Tester :

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

---

# 4) Installer Multipass dans Ubuntu

Si besoin :

```bash
sudo snap install multipass
```

Tester :

```bash
multipass version
```

---

# 5) Créer le lab

## VM Ansible

```bash
multipass launch 22.04 \
-n ansible-main \
-c 2 \
-m 1G
```

Vérifier :

```bash
multipass info ansible-main
```

Entrer dedans :

```bash
multipass shell ansible-main
```

---

## VM Web 1

```bash
multipass launch 22.04 \
-n web-server-1 \
-c 1 \
-m 1G
```

---

## VM Web 2

```bash
multipass launch 22.04 \
-n web-server-2 \
-c 1 \
-m 1G
```

---

# 6) Vérifier le réseau

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

Tester :

```bash
ping 10.3.241.40
```

---

# 7) Si Multipass casse après un crash

Nettoyage :

```bash
multipass delete --all
multipass purge
```

Puis vérifier :

```bash
multipass list
```

---

# 8) Si tu retombes sur cette erreur

## Erreur :

```
launch failed:
KVM support is not enabled on this machine
```

Cause :

Nested VT-x désactivé.

Vérifier depuis Windows :

```powershell
.\VBoxManage.exe showvminfo "Ubuntu-JPMM-CLONADO" | Select-String "Nested"
```

Si tu vois :

```
Nested VT-x/AMD-V: disabled
Nested Paging: disabled
```

refaire :

```powershell
.\VBoxManage.exe modifyvm "Ubuntu-JPMM-CLONADO" --nested-hw-virt on
.\VBoxManage.exe modifyvm "Ubuntu-JPMM-CLONADO" --nestedpaging on
```

---

# 9) Le piège qu'on a rencontré

Au début on avait :

```
Nested VT-x: enabled
Nested Paging: enabled
```

mais VirtualBox crashait :

```
Guru Meditation
VERR_PGM_MAPPING_IPE
```

Puis on a désactivé trop loin :

```
Nested VT-x: disabled
Nested Paging: disabled
```

Résultat :

```
Multipass:
KVM support is not enabled
```

La bonne configuration finale est :

```
VirtualBox
 ├── VT-x/AMD-V          ON
 ├── Nested VT-x/AMD-V   ON
 └── Nested Paging       ON
```

---

# Commandes de diagnostic utiles à garder

### Windows PowerShell

```powershell
VBoxManage showvminfo "Ubuntu-JPMM-CLONADO"
```

ou :

```powershell
VBoxManage --version
```

---

### Ubuntu

```bash
lsmod | grep kvm

ls -l /dev/kvm

kvm-ok

multipass list
```

---

Franchement, le montage est assez propre : tu as maintenant un **mini cluster Kubernetes/Ansible de labo dans une VM VirtualBox**, avec KVM accéléré au lieu d'une émulation lente. C'était le bon chemin, le problème était juste la couche de virtualisation imbriquée. 😄
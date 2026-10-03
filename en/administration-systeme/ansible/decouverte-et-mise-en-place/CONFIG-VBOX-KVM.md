---
description: >-
  Setting up a lab with Multipass VMs (KVM) hosted > in an Ubuntu VM >
  on VirtualBox > on Windows 11.
---

# Setting up the lab

I already had an Ubuntu VM installed on VirtualBox on my PC. First, it needs to be fully shut down:

```bash
sudo poweroff
```

Next, nested virtualization must be enabled. This is required because we will run Multipass VMs inside an Ubuntu VM. We open **PowerShell** as administrator and go to the VirtualBox installation directory, where the binaries are located:

```powershell
cd "C:\Program Files\Oracle\VirtualBox"
```

We enable **Nested VT-x**:

```powershell
.\VBoxManage.exe modifyvm "Ubuntu-JPMM-CLONADO" --nested-hw-virt on
```

And **Nested Paging**:

```powershell
.\VBoxManage.exe modifyvm "Ubuntu-JPMM-CLONADO" --nestedpaging on
```

```powershell
.\VBoxManage.exe showvminfo "Ubuntu-JPMM-CLONADO" | Select-String "Nested VT-x","Nested Paging","Hardware Virtualization"
```

Expected result:

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

Then enable the following options in the VM settings:

```
[x] Activer VT-x/AMD-V
[x] Pagination imbriquée
```

We then start the Ubuntu VM and check that the KVM module is loaded, allows Linux to run into a hypervisor, which Multipass needs:

```bash
lsmod | grep kvm
```

We can see:

```
kvm
irqbypass
```

Then we check that KVM is exposed:

```bash
ls -l /dev/kvm
```

Which is indeed the case:

```
crw-rw----+ 1 root kvm ... /dev/kvm
```

We can also double-check with cpu-checker, which should confirm that KVM acceleration is available:

```bash
sudo apt update
sudo apt install cpu-checker
kvm-ok
```

Result:

```
INFO: /dev/kvm exists
KVM acceleration can be used
```

We can now install Multipass and deploy our VMs:

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

We then list the VMs to check that they are running and have an IP:

```bash
multipass list
```

Example:

```
Name            State      IPv4
ansible-main    Running    10.3.241.40
web-server-1    Running    10.3.241.50
web-server-2    Running    10.3.241.60
```

### Note: if Multipass breaks after a crash

Clean up all VMs:

```bash
multipass delete --all
multipass purge
```

Then check that everything is gone:

```bash
multipass list
```

If you get this error:

```
launch failed:
KVM support is not enabled on this machine
```

It's because nested virtualization (Nested VT-x) sometimes gets disabled.

Check from Windows:

```powershell
.\VBoxManage.exe showvminfo "NOM_VM_VBOX" | Select-String "Nested"
```

If you see:

```
Nested VT-x/AMD-V: disabled
Nested Paging: disabled
```

Run the same commands as above again:

```powershell
.\VBoxManage.exe modifyvm "NOM_VM_VBOX" --nested-hw-virt on
.\VBoxManage.exe modifyvm "NOM_VM_VBOX" --nestedpaging on
```

The correct final configuration should be:

```
VirtualBox
 ├── VT-x/AMD-V          ON
 ├── Nested VT-x/AMD-V   ON
 └── Nested Paging       ON
```

We now have a **mini Ansible lab cluster inside a VirtualBox VM**, with hardware-accelerated KVM instead of slow emulation. It works well, as long as you keep an eye on the nested virtualization settings.
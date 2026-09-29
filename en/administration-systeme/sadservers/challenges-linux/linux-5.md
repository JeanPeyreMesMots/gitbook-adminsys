# 5 - Kihei

**Objectif :** faire fonctionner `/home/admin/kihei` sans supprimer `/home/admin/datafile`. Le test de réussite : lancer le programme doit renvoyer `Done.`.

Point de départ, avec [ce guide](https://blog.stephane-robert.info/docs/admin-serveurs/linux/gestion-espace-disque/) en support :

```bash
df -hT -x tmpfs -x devtmpfs
# /dev/nvme0n1p1  ext4  7.7G  6.8G  486M  94% /
```

Le disque principale est à 94% d'utilisation, on regarde qui est le plus volumineux :

```bash
du -h --max-depth=1 /chemin | sort -h
```

```
428M	/var
1.3G	/usr
5.1G	/home
6.8G	/
```

`/home` est en très grande partie occupé par `datafile` lui-même (fichier à ne pas toucher), donc le nettoyage doit se faire sur `/var` et `/usr`.

```bash
du -h --max-depth=1 /var/cache/ | sort -h
# 231M  /var/cache/apt
```

Le cache apt est le gros morceau, on le vide peu :

```bash
sudo apt clean --dry-run
# Del /var/cache/apt/archives/* /var/cache/apt/archives/partial/*
# Del /var/lib/apt/lists/partial/*
# Del /var/cache/apt/pkgcache.bin /var/cache/apt/srcpkgcache.bin
sudo apt clean
```

Résultat : `/var/cache` passe de 234M à 3.5M. Puis on nettoie /var/lib et /apt/lists qui se trouve dedans :

```bash
130M	/var/lib/apt
```

```bash
sudo rm -rf /var/lib/apt/lists/*
```

`/var/lib/apt` passe alors de 130M à 36K.

Pareille pour les logs journaliers inférieurs à 7 jours :

```bash
49M	/var/log/journal
```

```bash
sudo journalctl --vacuum-time=7d
# Deleted archived journal ... (8.0M) x4
# Vacuuming done, freed 32.0M of archived journals
```

Est-ce suffisant ? Non mon général !

```bash
/home/admin/kihei
# panic: exit status 1
# goroutine 1 [running]: main.main() ./main.go:64 +0x47d
```

Toujours en échec, malgré environ 500M libérés au total :

```bash
df -h
# /dev/nvme0n1p1   7.7G  6.4G  878M  89% /
```

On peut quand même regarder `/usr/lib` :

```bash
du -h --max-depth=1 /usr | sort -h
# 454M  /usr/lib
# 1.3G  /usr
```

Mais après nettoyage de ce qui pouvait l'être, toujours pas assez d'espace.

En regardant les options de `kihei` :

```bash
./kihei -h
# -v  Verbose mode (print extra info)
./kihei -v
# Creating file /home/admin/data/newdatafile with size 1.5GB...
# panic: exit status 1
```

Le programme essaie de créer un fichier de **1.5 Go**. Avec 878M d'espace libre max, même un nettoyage parfait du disque root suffit pas. Si rien d'autres n'est retirable, on va donc créer un volume dédié, et donc un LVM.

Vérification des disques disponibles :

```bash
sudo lsblk
NAME         SIZE TYPE MOUNTPOINT
nvme0n1        8G disk
├─nvme0n1p1  7.9G part /
└─nvme0n1p15 124M part /boot/efi
nvme1n1        1G disk
nvme2n1        1G disk
```

Deux disques supplémentaires de 1G chacun, non utilisés. Deux bons candidats à des volumes LVM. (guide utilisé : [lien](https://blog.stephane-robert.info/docs/admin-serveurs/linux/lvm/)).

**1. Conversion en Physical Volumes :**

```bash
sudo pvcreate /dev/nvme1n1 /dev/nvme2n1
# Physical volume "/dev/nvme1n1" successfully created.
# Physical volume "/dev/nvme2n1" successfully created.
```

**2. Création du Volume Group (fusion des deux disques) :**

```bash
sudo vgcreate ChiMai /dev/sdb /dev/sdc
# Volume group "ChiMai" successfully created
sudo vgdisplay
# VG Size  1.99 GiB
```

Les deux disques de 1G sont maintenant regroupés en un seul groupe de \~2G.

**3. Création du Logical Volume :**

```bash
sudo lvcreate -L 1G -n ChiMaiData ChiMai
# Logical volume "ChiMaiData" created.
```

**4. Formatage en ext4 :**

```bash
sudo mkfs.ext4 /dev/ChiMai/ChiMaiData
```

**5. Montage :**

```bash
sudo mkdir -p /chimaidata
sudo mount /dev/ChiMai/ChiMaiData /chimaidata/
df -h
# /dev/mapper/ChiMai-ChiMaiData  2.0G   24K  1.9G   1% /chimaidata
```

Le nouveau volume est monté et dispose de 1.9G libres. Cependant, j'ai raté mon premier réflexe : vouloir déplacer `datafile` directement. Or le fichier fait 5 Go, largement plus que le volume disponible :

```bash
sudo mv /home/admin/datafile .
# mv: error writing './datafile': No space left on device
```

Donc plutôt que de déplacer les données existantes, on fait pointer le dossier où le programme écrit, c'est à dire (`/home/admin/data`), vers le nouveau volume via un lien symbolique.

```bash
sudo chown -R admin:admin chimaidata/
rm -rf /home/admin/data
ln -s /chimaidata /home/admin/data
```

On check :

```bash
ll /home/admin/
# -rw-r--r-- 1 root  root  5.0G Dec 14 04:29 datafile
# lrwxrwxrwx 1 admin admin   11 Mar 18 18:42 data -> /chimaidata
```

`datafile` reste intact à sa place, seul le dossier de destination `data` pointe maintenant vers le nouveau volume LVM.

Résultat :

```bash
/home/admin/kihei -v
# Creating file /home/admin/data/newdatafile with size 1.5GB...
# Done.
```

Résolu.

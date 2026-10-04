# 5 - Kihei

**Goal:** make `/home/admin/kihei` work without deleting `/home/admin/datafile`. The success test: running the program must return `Done.`.

Starting point, with [this guide](https://blog.stephane-robert.info/docs/admin-serveurs/linux/gestion-espace-disque/) as support:

```bash
df -hT -x tmpfs -x devtmpfs
# /dev/nvme0n1p1  ext4  7.7G  6.8G  486M  94% /
```

The main disk is at 94% usage; we look at what's taking the most space:

```bash
du -h --max-depth=1 /chemin | sort -h
```

```
428M	/var
1.3G	/usr
5.1G	/home
6.8G	/
```

`/home` is mostly taken up by `datafile` itself (the file we must not touch), so the cleanup has to happen in `/var` and `/usr`.

```bash
du -h --max-depth=1 /var/cache/ | sort -h
# 231M  /var/cache/apt
```

The apt cache is the big chunk; we clear it:

```bash
sudo apt clean --dry-run
# Del /var/cache/apt/archives/* /var/cache/apt/archives/partial/*
# Del /var/lib/apt/lists/partial/*
# Del /var/cache/apt/pkgcache.bin /var/cache/apt/srcpkgcache.bin
sudo apt clean
```

Result: `/var/cache` goes from 234M to 3.5M. Then we clean `/var/lib` and the `apt/lists` inside it:

```bash
130M	/var/lib/apt
```

```bash
sudo rm -rf /var/lib/apt/lists/*
```

`/var/lib/apt` then goes from 130M to 36K.

Same for journal logs older than 7 days:

```bash
49M	/var/log/journal
```

```bash
sudo journalctl --vacuum-time=7d
# Deleted archived journal ... (8.0M) x4
# Vacuuming done, freed 32.0M of archived journals
```

Is that enough? No sir!

```bash
/home/admin/kihei
# panic: exit status 1
# goroutine 1 [running]: main.main() ./main.go:64 +0x47d
```

Still failing, despite about 500M freed in total:

```bash
df -h
# /dev/nvme0n1p1   7.7G  6.4G  878M  89% /
```

We can still look at `/usr/lib`:

```bash
du -h --max-depth=1 /usr | sort -h
# 454M  /usr/lib
# 1.3G  /usr
```

But after cleaning what could be cleaned, still not enough space.

Looking at `kihei`'s options:

```bash
./kihei -h
# -v  Verbose mode (print extra info)
./kihei -v
# Creating file /home/admin/data/newdatafile with size 1.5GB...
# panic: exit status 1
```

The program tries to create a **1.5 GB** file. With 878M of free space at most, even a perfect cleanup of the root disk isn't enough. If nothing else can be removed, we create a dedicated volume, and therefore an LVM.

Checking the available disks:

```bash
sudo lsblk
NAME         SIZE TYPE MOUNTPOINT
nvme0n1        8G disk
├─nvme0n1p1  7.9G part /
└─nvme0n1p15 124M part /boot/efi
nvme1n1        1G disk
nvme2n1        1G disk
```

Two extra 1G disks, unused. Two good candidates for LVM volumes. (guide used: [link](https://blog.stephane-robert.info/docs/admin-serveurs/linux/lvm/)).

**1. Converting to Physical Volumes:**

```bash
sudo pvcreate /dev/nvme1n1 /dev/nvme2n1
# Physical volume "/dev/nvme1n1" successfully created.
# Physical volume "/dev/nvme2n1" successfully created.
```

**2. Creating the Volume Group (merging the two disks):**

```bash
sudo vgcreate ChiMai /dev/sdb /dev/sdc
# Volume group "ChiMai" successfully created
sudo vgdisplay
# VG Size  1.99 GiB
```

The two 1G disks are now combined into a single ~2G group.

**3. Creating the Logical Volume:**

```bash
sudo lvcreate -L 1G -n ChiMaiData ChiMai
# Logical volume "ChiMaiData" created.
```

**4. Formatting as ext4:**

```bash
sudo mkfs.ext4 /dev/ChiMai/ChiMaiData
```

**5. Mounting:**

```bash
sudo mkdir -p /chimaidata
sudo mount /dev/ChiMai/ChiMaiData /chimaidata/
df -h
# /dev/mapper/ChiMai-ChiMaiData  2.0G   24K  1.9G   1% /chimaidata
```

The new volume is mounted with 1.9G free. However, I got my first instinct wrong: wanting to move `datafile` directly. But the file is 5 GB, far more than the available volume:

```bash
sudo mv /home/admin/datafile .
# mv: error writing './datafile': No space left on device
```

So rather than moving the existing data, we make the folder the program writes to (`/home/admin/data`) point to the new volume via a symlink.

```bash
sudo chown -R admin:admin chimaidata/
rm -rf /home/admin/data
ln -s /chimaidata /home/admin/data
```

We check:

```bash
ll /home/admin/
# -rw-r--r-- 1 root  root  5.0G Dec 14 04:29 datafile
# lrwxrwxrwx 1 admin admin   11 Mar 18 18:42 data -> /chimaidata
```

`datafile` stays intact where it is; only the destination folder `data` now points to the new LVM volume.

Result:

```bash
/home/admin/kihei -v
# Creating file /home/admin/data/newdatafile with size 1.5GB...
# Done.
```

Solved.
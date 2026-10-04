# 3 - Kortenberg, Manhattan & Cape Town

### <mark style="color:$warning;">Kortenberg</mark>

**The problem:** impossible to create anything in the home directory, no accessible file or folder.

```bash
mkdir test/
ll
# d--------- 2 admin admin 4.0K Mar 11 10:07 test
```

The directory is created but with zero permissions:

```bash
ll test/
# ls: cannot open directory 'test/': Permission denied
touch file && echo "hey" > file
# -bash: file: Permission denied
```

Even a brand-new file is unreachable:

```bash
ll
# ---------- 1 admin admin 0 Mar 11 10:07 file
```

Could umask be the culprit?

```bash
umask
# 0777
```

Indeed: a umask of 0777 strips _all_ permissions from every new file/directory created.

To understand umask: unlike Windows, Linux doesn't inherit permissions from the parent directory. New files' permissions are determined by the umask, which **removes** permissions relative to the maximum possible.

```bash
umask
# 0022 (valeur typique)

# Calcul :
# Fichiers   : 666 - 022 = 644 (rw-r--r--)
# Répertoires: 777 - 022 = 755 (rwxr-xr-x)
```

| umask | Files | Directories | Use             |
| ----- | ----- | ----------- | --------------- |
| 022   | 644   | 755         | Standard        |
| 027   | 640   | 750         | More restrictive|
| 077   | 600   | 700         | Private         |
| 002   | 664   | 775         | Collaborative   |

What we want is 755 for directories, so a umask of 022. We test it first in the current session:

```bash
umask 022
mkdir test4/ && touch test4/test.txt
ll
# drwxr-xr-x 2 admin admin 4.0K Mar 11 10:12 test4
echo "hey" > test4/test.txt
cat test4/test.txt
# hey
```

Great, it works. Now to apply it system-wide in `/etc/profile`:

```bash
cat /etc/profile | grep "umask"
# umask 777
```

There's the culprit, hardcoded to 777 for all sessions:

```bash
sudo sed -i 's/^umask[[:space:]]\+[0-7]\{3\}/umask 022/' /etc/profile
source .bashrc
```

Solved.

### <mark style="color:$warning;">Manhattan</mark>

**The problem:** PostgreSQL no longer connects, with locales that seem broken too.

```bash
sudo -u postgres psql
# perl: warning: Setting locale failed.
# ...
# psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

The locale warning led me down that path first:

```bash
locale-gen fr_FR.UTF-8
# -bash: locale-gen: command not found
sudo locale-gen fr_FR.UTF-8
# Generating locales... Generation complete.
sudo dpkg-reconfigure locales
# fr_FR.UTF-8... done
```

I regenerate the locales, but the problem persists:

```bash
sudo -u postgres psql
# psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

I did a quick search on the socket error and came across an article where someone hit this problem while restoring a too-large production database, for lack of enough disk space. The word "**space**" rang a bell, especially since the challenge was tagged "**disk volumes**". We check the space:

```bash
df -h
# /dev/nvme0n1     8.0G  8.0G   28K 100% /opt/pgdata
```

And there we go. The `/opt/pgdata` volume is 100% full. Inside, one file takes up almost all the space:

```bash
ll
# -rw-r--r--  1 root  root  7.0G May 21  2022 file1.bk
# -rw-r--r--  1 root  root  923M May 21  2022 file2.bk
# -rw-r--r--  1 root  root  488K May 21  2022 file3.bk
```

We delete the biggest one:

```bash
sudo rm file1.bk
df -h
# /dev/nvme0n1     8.0G 1014M  7.1G  13% /opt/pgdata
```

That frees up space but... :

```bash
sudo psql -U postgres
# psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

One solution: restart the cluster cleanly. But rather than going through `systemctl`, it's better to use the dedicated PostgreSQL tools. Here, "**pg_lsclusters**" lists the cluster, giving us a chance to check its log to confirm the cause:

```bash
pg_lsclusters
# 14  main  5432 down  postgres /opt/pgdata/main ...

cat /var/log/postgresql/postgresql-14-main.log
# FATAL: could not create lock file "postmaster.pid": No space left on device
```

That confirms PostgreSQL stopped because of the lack of disk space (before the cleanup). Since we've already freed some, we restart it with PostgreSQL's "**pg_ctlcluster**" tool:

```bash
sudo pg_ctlcluster 14 main restart
```

Final check:

```bash
sudo -u postgres psql
# \l  →  liste des bases OK
sudo -u postgres psql -c "insert into persons(name) values ('jane smith');" -d dt
# INSERT 0 1
```

Solved.

### <mark style="color:$warning;">Cape Town</mark>

**The problem:** nginx down with a syntax error, then a second, different failure once the first was fixed.

```bash
curl -I 127.0.0.1:80
# curl: (7) Failed to connect to 127.0.0.1 port 80: Connection refused

sudo systemctl status nginx
# Active: failed (Result: exit-code)
# nginx[573]: nginx: [emerg] unexpected ";" in /etc/nginx/sites-enabled/default:1
```

The `/etc/nginx/sites-enabled/default` file has a syntax error on the first line:

```bash
head /etc/nginx/sites-enabled/default
# ; -> NONE
```

A semicolon left at the very start of the file. We fix it, then restart:

```bash
sudo systemctl restart nginx
sudo systemctl status nginx
# Active: active (running)
```

It seems back up, but we get a 500:

```bash
curl -I 127.0.0.1:80
# HTTP/1.1 500 Internal Server Error
```

Straight to the logs:

```bash
cat /var/log/nginx/error.log
# ... unexpected ";" (anciennes entrées, déjà corrigé)
# [alert] socketpair() failed while spawning "worker process" (24: Too many open files)
# [emerg] eventfd() failed (24: Too many open files)
# [crit] open() "/var/www/html/index.nginx-debian.html" failed (24: Too many open files)
```

A file-descriptor limit reached. If we log in as nginx's service user, it's not allowed:

```bash
sudo su - www-data
# This account is currently not available.
```

The `www-data` account has no login shell. We can still spawn a shell using the binary with sudo:

```bash
sudo runuser -u www-data -- bash
```

It works, and we can check the limits:

```bash
ulimit -Hn
# 1048576
ulimit -Sn
# 1024
```

1024 is indeed low for a web server.

To raise the limit, we can go through systemd (I prefer using the native tools):

```bash
sudo systemctl edit nginx
# Ajout de :
# LimitNOFILE=65535
```

```bash
sudo systemctl daemon-reload
sudo nginx -t
# syntax ok, test successful
sudo systemctl restart nginx
sudo systemctl status nginx
# Active: active (running)
```

Then we adjust the worker limit directly in the nginx config, so the process itself can open more descriptors. Here we set it to 30000, which is plenty:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
worker_rlimit_nofile 30000;
include /etc/nginx/modules-enabled/*.conf;
```

We restart nginx and can now reach it locally:

```bash
sudo systemctl restart nginx
curl -I 127.0.0.1:80
# HTTP/1.1 200 OK
```
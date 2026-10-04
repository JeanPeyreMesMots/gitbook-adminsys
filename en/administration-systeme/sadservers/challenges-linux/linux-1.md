# 1 - Apia, Tokamachi, Yokohama & Fukuoka

## <mark style="color:$warning;">Apia</mark>

**Goal:** a word was added to one file among a hundred; find it and provide the solution as an MD5 hash.

We start by listing all the files:

```bash
ls -l
total 404
-rw-r--r-- 1 admin admin 1046 Feb 25  2024 file0.txt
-rw-r--r-- 1 admin admin 1046 Feb 25  2024 file1.txt
-rw-r--r-- 1 admin admin 1046 Feb 25  2024 file10.txt
```

Since we know only one word was added, exactly one file should have a different size. Sorting by descending size, the modified file comes up right on the first line:

```bash
ls -lS
-rw-r--r-- 1 admin admin 1054 Feb 25  2024 file76.txt
```

We then diff the two files with `vimdiff`:

```bash
vimdiff file76.txt file0.txt
```

The word is found: it's **"eureka"**. We just put it in the solution file:

```bash
echo "eureka" > /home/admin/solution
```

And check that the hash matches the one the challenge expects:

```bash
md5sum /home/admin/solution
55aba155290288b58e9b778c8f616560  /home/admin/solution
```

Which it does.

## <mark style="color:$warning;">Tokamachi</mark>

**Goal:** a writer must continuously send messages to a named pipe (`/home/admin/namedpipe`), and a reader captures them with timestamped logs in `/home/admin/reader.log`.

The reader is already running, with a 2-second delay between reads:

```bash
nohup /bin/bash -c 'while true; do
  if read line < /home/admin/namedpipe; then
    echo "$(date) Received: $line" >> /home/admin/reader.log
  fi
  sleep 2
done' &>/dev/null &
```

The writer provided by default by SadServers has no delay:

```bash
/bin/bash -c 'while true; do echo "this is a test message being sent to the pipe" > /home/admin/namedpipe; done' &
```

I first tried fixing the writer's syntax as suggested, with a `sleep 3`:

```bash
/bin/bash -c 'while true; do if read line < /home/admin/namedpipe; then echo "$(date) Received: $line" >> /home/admin/reader.log; fi; sleep 3; done'
```

But it didn't work. We can also try adding indentation and a `2>/dev/null`:

```bash
/bin/bash -c 'while true; do if read line < /home/admin/namedpipe 2>/dev/null; then echo "$(date) Received: $line" >> /home/admin/reader.log; fi; sleep 2; done'
```

But that leaves the log empty, and SadServers didn't accept the solution.

After killing the old writer process (`ps` + `grep "pipe"` to find the PID, then `kill`), the command that worked uses `nohup`:

```bash
nohup /bin/bash -c 'while true; do echo "this is a test message being sent to the pipe" > /home/admin/namedpipe; sleep 2; done' &
```

`nohup` ("no hang up") runs a command so that it keeps running even if the terminal session that started it is closed or interrupted. It detaches the command from the current session and places it in an independent process, keeping it alive.

source: [zonetuto.fr](https://zonetuto.fr/shell-bash/nohup-lancer-un-script-en-arriere-plan-sur-un-serveur-linux/)

## <mark style="color:$warning;">Yokohama</mark>

**Goal:** manage permissions for 4 users (**abe**, **betty**, **carlos**, **debora**):

* each one can modify their own file;
* nobody can _read_ the others' files, but can _write_ to them;
* everyone can modify the contents of `shared/project_ALL`, except its first line.

We have root access, so we can adjust the permissions freely.

At the start we have this:

```bash
ls -l /home/admin/shared
total 20
-rw-r--r-- 1 root   admin  38 Feb  2  2025 ALL
-rw-r----- 1 abe    abe    27 Feb  2  2025 project_abe
-rw-r----- 1 betty  betty  29 Feb  2  2025 project_betty
-rw-r----- 1 carlos carlos 30 Feb  2  2025 project_carlos
-rw-r----- 1 debora debora 30 Feb  2  2025 project_debora
```

We log in as another user, such as **debora**, who can neither read nor write the others' files:

```bash
sudo su - debora
cd /home/admin/shared/
cat *
First line in the shared project file
cat: project_abe: Permission denied
cat: project_betty: Permission denied
cat: project_carlos: Permission denied
"This is debora's project file"
```

A handy tool to save time computing octal permissions: [chmod-calculator.com](https://chmod-calculator.com/).

For the `project_USER` files, the execute bit isn't needed; we just need read + write for the group that has the same name as the user. We then connect as each user to apply the matching permissions, for example with debora:

```bash
cd /home/admin/shared/
ls -l
total 20
-rw-r--r-- 1 root   admin  38 Feb  2  2025 ALL
-rw-rw-r-- 1 abe    abe    27 Feb  2  2025 project_abe
-rw-rw-r-- 1 betty  betty  29 Feb  2  2025 project_betty
-rw-rw-r-- 1 carlos carlos 30 Feb  2  2025 project_carlos
-rw-r----- 1 debora debora 30 Feb  2  2025 project_debora

chmod 664 project_debora

ls -l
-rw-rw-r-- 1 debora debora 30 Feb  2  2025 project_debora
```

But repeating the process for each user is tedious and not a clean approach. Let's create a shared group instead:

```bash
# 1. Créer un groupe commun pour tous les utilisateurs
sudo groupadd projectusers

# 2. Ajouter TOUS les utilisateurs au groupe
sudo usermod -aG projectusers abe
sudo usermod -aG projectusers debora
# ... (idem betty, carlos)

# 3. Appliquer les permissions via le groupe
sudo chown :projectusers /shared/project_ALL
sudo chmod 664 /shared/project_ALL        # rw pour owner+group
```

To let all users add content to `/home/admin/shared/project_ALL`, read/write access is needed for everyone → 666:&#x20;

```bash
-rw-rw-rw- 1 root   admin  38 Feb  2  2025 ALL
```

And finally, `chattr +a` ("append") so that `ALL` can only have content appended, without being able to modify what already exists, and therefore the first line:

```bash
lsattr *
-----a--------e------- ALL
--------------e------- project_abe
--------------e------- project_betty
--------------e------- project_carlos
--------------e------- project_debora
```

## <mark style="color:$warning;">Fukuoka</mark>

**Goal:** an nginx server returns a default 404 instead of serving a file with the message _"Welcome to the Real Site!"_.

```html
curl localhost
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.18.0</center>
</body>
</html>
```

A quick look at the nginx config tells us where the logs are:

```bash
cat /etc/nginx/nginx.conf
[...]
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;
}
```

There it is, an error on `/var/www/html`, the directory that serves the files:

```bash
cat /var/log/nginx/error.log
2026/03/06 21:04:42 [crit] 618#618: *1 stat() "/var/www/html/" failed (13: Permission denied), client: 127.0.0.1, server: _, request: "GET / HTTP/1.1", host: "localhost"
```

Checking, we indeed don't have the rights, and the directory looks broken:

```bash
ll /var/www/html
ls: cannot access '/var/www/html': Permission denied
ll /var/www/
total 0
d????????? ? ? ? ?            ? html
```

`/var/www/html` is supposed to belong to the `www-data` group, used by nginx to serve its files. We fix it:

```bash
sudo chown -R www-data:www-data www/
sudo chmod -R 755 www/
```

```bash
ll
drwxr-xr-x  3 www-data www-data 4096 Jul 21  2025 www
```

Reload + restart the service:

```bash
sudo systemctl reload nginx
sudo systemctl restart nginx
```

But nginx still won't cooperate. We now get a 403 instead of a 404, so still an access error, but on the main HTML file this time:

```bash
curl localhost
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.18.0</center>
</body>
</html>
```

Looking closer, the `index.html` symlink points to a file that doesn't have the right permissions:

```bash
ll
lrwxrwxrwx 1 www-data www-data  33 Jul 21  2025 index.html -> /opt/site-content/real_index.html
-rwxr-xr-x 1 www-data www-data 612 Jul 21  2025 index.nginx-debian.html

ll /opt/site-content/real_index.html
-rw-r----- 1 root root 34 Jul 21  2025 /opt/site-content/real_index.html
```

We can see it in the logs too:

```bash
cat /var/log/nginx/error.log
2026/03/06 21:12:25 [error] 928#928: *1 open() "/var/www/html/index.html" failed (13: Permission denied), client: 127.0.0.1, server: _, request: "GET / HTTP/1.1", host: "localhost"
```

So we fix the target file's permissions:

```bash
sudo chown root:www-data /opt/site-content/real_index.html
sudo chmod 640 /opt/site-content/real_index.html
```

```bash
ll /opt/site-content/real_index.html
-rw-r----- 1 root www-data 34 Jul 21 2025 /opt/site-content/real_index.html
```

nginx (`www-data`) only needs to _read_ the file. Only `root` can edit it; `www-data` accesses it read-only through the group.

And finally:

```bash
curl localhost
# Welcome to the Real Site!
```
# 2 - Rio de Janeiro, Nuuk, Cairo & Alexandria

## <mark style="color:$warning;">Rio de Janeiro</mark>

Some Jenkins debugging here. The service wouldn't start properly, so off to `systemctl status` to understand ⇒ status stopped.

Jenkins 2.516.3 refused to start cleanly because it was running on a Java version not recommended for it (Java 8 at first, then Java 25 flagged as "not fully supported"). The goal is therefore to switch Jenkins to Java 21, already installed on the machine but not used by default.

We check which versions are used by default:

```bash
java -version
javac -version
```

`java` pointed to an old version (Java 8) while `javac` pointed to a newer one. An inconsistency to fix.

On Ubuntu/Debian, managing several JVMs is done with `update-alternatives`. We list the installed versions:

```bash
sudo update-alternatives --config java
```

```
There are 3 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                         Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/java-25-openjdk-amd64/bin/java   2511      auto mode
  1            /usr/lib/jvm/java-21-openjdk-amd64/bin/java   2111      manual mode
  2            /usr/lib/jvm/java-25-openjdk-amd64/bin/java   2511      manual mode
  3            /usr/lib/jvm/temurin-8-jdk-amd64/bin/java     1081      manual mode
```

Java 8, 21 and 25 are installed. Java 25 is selected automatically but not fully supported by this Jenkins version.

We force Java 21 (entry `1`):

```bash
sudo update-alternatives --config java
# Press <enter> to keep the current choice[*], or type selection number: 1
# update-alternatives: using /usr/lib/jvm/java-21-openjdk-amd64/bin/java to provide /usr/bin/java (java) in manual mode
```

Same for `javac`. After that, `java -version` and `javac -version` both return Java 21.

We restart the service:

```bash
sudo systemctl status jenkins.service
sudo systemctl start jenkins.service
```

```bash
● jenkins.service - Jenkins Continuous Integration Server
     Active: active (running) since Mon 2026-03-09 18:39:17 UTC; 6s ago
     Main PID: 4169 (java)
     ...
Mar 09 18:39:17 i-0899387007d013833 jenkins[4169]: ... Jenkins is fully up and running
```

Then we check that port 8888 responds:

```bash
curl -s localhost:8888/login | grep Jenkins | head -n1
# <title>Sign in - Jenkins</title>...
```

It returns a "**Sign in - Jenkins**", so the web interface is reachable. Challenge solved.

## <mark style="color:$warning;">Nuuk</mark>

The challenge name reminds me of **SSHNuke**, a Matrix reference :P

**Goal:** SSH wasn't working locally on the machine, despite the right keys being present in `~/.ssh/authorized_keys`.

A piece of cake: a quick look at `.ssh` is enough to see the permissions were wrong:

```bash
ll
# d--------- 2 admin admin 4.0K Oct 21 17:27 .ssh
```

No permissions on the directory. We fix it:

```bash
sudo chmod 755 .ssh/
# drwxr-xr-x 2 admin admin 4.0K Oct 21 17:27 .ssh
```

Then we connect locally to the machine:

```bash
ssh 127.0.0.1
# The authenticity of host '127.0.0.1 (127.0.0.1)' can't be established.
# ED25519 key fingerprint is SHA256:SXwnOE3G0MEubcNLJQIryCk1URSsUStlsnc2dP2zj9s.
# Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
admin@i-0a31a7ab947fad896:~$
```

Connection successful.

## <mark style="color:$warning;">Cairo</mark>

**Context:** a critical health-check script (`/opt/scripts/health.sh`) is supposed to run every 10 seconds via a systemd timer:

```bash
#!/bin/bash
# This script logs status and exits with 0 on success, 1 on failure
if curl -s --max-time 2 http://localhost | grep -q "Welcome to nginx"; then
  echo "$(date): STATUS: OK" >> /var/log/health.log
  exit 0
else
  echo "$(date): STATUS: FAILED" >> /var/log/health.log
  exit 1
fi
```

As you'll see, this one had me going in circles for a good while.

Testing the command from the script manually:

```bash
curl http://localhost
^C
```

Nothing happens, it stays stuck at "Trying":

```bash
curl -v http://localhost
* Host localhost:80 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:80...
*   Trying 127.0.0.1:80...
```

```bash
systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Active: active (running) since Tue 2026-03-10 18:59:27 UTC; 12min ago
     Main PID: 779 (nginx)
```

We check the logs:

```bash
cat /var/log/nginx/error.log
2025/11/22 16:01:28 [notice] 1600#1600: using inherited sockets from "5;6;"
```

A single line, a simple "notice", nothing obvious. And the syntax is fine:

```bash
sudo nginx -t
# nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Nothing wrong in the config either. Could this be the start of a rabbit hole?

```bash
sudo nginx
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
nginx: [emerg] bind() to [::]:80 failed (98: Address already in use)
nginx: [emerg] still could not bind()
```

A bind problem, so a port issue. Port 80 is already in use, but by what?

```bash
sudo ss -tuln | grep :80
tcp   LISTEN 0      511            0.0.0.0:80         0.0.0.0:*
tcp   LISTEN 0      511               [::]:80            [::]:*
tcp   LISTEN 0      4096                 *:8080             *:*
```

By nginx itself, which is already listening on port 80 (the `systemctl status` process was indeed running):

```bash
sudo systemctl stop nginx
sudo ss -tuln | grep :80
# tcp   LISTEN 0      4096                 *:8080             *:*
```

After stopping it, port 80 is freed. So nginx runs fine and listens on the right port. The problem is elsewhere.

I ended up asking my friend Claude to analyze this log:

```bash
2025/11/22 16:01:28 [notice] 1600#1600: using inherited sockets from "5;6;"
```

Answer: a theory of "zombie" sockets inherited from a container restart, blocking port 80 despite no visible process in `ss`. Quite something. The suggested fix is to kill all nginx processes with `pkill -9`, check the port, restart, and as a last resort `fuser -k 80/tcp`.

Was the AI right?

```bash
sudo pkill -9 nginx
sudo systemctl stop nginx
sudo ss -tunap | grep :80
# tcp   LISTEN 0      4096   *:8080   *:*   users:(("gotty",pid=704,fd=6))
sudo nginx -t
sudo systemctl start nginx
sudo ss -tunap | grep :80    # nginx bien présent, PIDs visibles
curl http://localhost        # toujours aucune réponse
```

<figure><img src="../../../.gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure>

Nope. Even after that, still no response from a local curl.

Let's take a look at the iptables rules, the NAT table in particular?

```bash
sudo iptables -L -n
Chain INPUT (policy ACCEPT)
...
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DROP       tcp  --  0.0.0.0/0            127.0.0.1            tcp dpt:80 /* The hidden problem (IPv4) */
```

There it is, a DROP rule explicitly commented "The hidden problem". Got me, you rascals!

Let's remove it:

```bash
sudo iptables -D OUTPUT 1
```

And it finally responds!

```bash
curl http://localhost | grep "Welcome to"
<title>Welcome to nginx!</title>
<h1>Welcome to nginx!</h1>
```

We test the script directly:

```bash
bash /opt/scripts/health.sh
# /opt/scripts/health.sh: line 4: /var/log/health.log: Permission denied
```

Hmm. With sudo maybe?

```bash
sudo bash /opt/scripts/health.sh
head /var/log/health.log
# Tue Mar 10 19:41:48 UTC 2026: STATUS: OK
```

Perfect. We check whether the systemd timer meant to run this script every 10 seconds exists:

```bash
sudo systemctl list-unit-files --type=timer
# ...
# health.timer                 disabled enabled
```

It exists but is disabled. We enable it:

```bash
sudo systemctl enable --now health.timer
# Created symlink '/etc/systemd/system/timers.target.wants/health.timer' → '/etc/systemd/system/health.timer'.
```

```bash
agent/check.sh
# OK
```

Solved!

## <mark style="color:$warning;">Alexandria</mark>

**Context:** a misconfigured cron backup job.

```bash
crontab -l
# no crontab for admin
sudo crontab -l
MAILTO="broken@nonexistent.local"
# DO NOT EDIT THIS FILE - edit the master and reinstall.
#Ansible: daily backup job
*/5 * * * * /opt/backup/old_backup.sh > /dev/null 2>&1
```

Two problems visible right away: the script being called is `old_backup.sh`, and it runs every 5 minutes instead of the expected frequency. We fix it:

```bash
*/10 * * * * /opt/backup/backup.sh > /dev/null 2>&1
```

Then we test the script:

```bash
./backup.sh
# Error: Backup already running (lock file exists)
```

A lock file blocks execution. We remove it:

```bash
sudo rm backup.lock
./backup.sh
# touch: cannot touch '/opt/backup/backup.lock': Permission denied
# tar (child): /var/backups/daily/backup_20260311_090428.tar.gz: Cannot open: Permission denied
# Backup failed!
```

With sudo it'll be better ;) :

```bash
sudo ./backup.sh
# Backup successful: /var/backups/daily/backup_20260311_090432.tar.gz
```

Solved.
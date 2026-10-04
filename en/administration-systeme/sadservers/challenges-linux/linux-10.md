# 10 - Bizerte, Kampot & Valladolid

### <mark style="color:$warning;">Kampot</mark>

**Context:** a Python app runs on port 20280, managed by supervisor, and can't be reconfigured to change its port. The goal is to make the service reachable locally on port 80 without touching its config.

Since we dealt with this in previous challenges, I immediately think of port redirection, with the help of [this article](https://blog.cloudfrancois.fr/2016-05-17-iptables-rule-forward-local/).

We first check that IP forwarding is enabled:

```bash
cat /proc/sys/net/ipv4/ip_forward
# 1
```

Then we create a redirect rule on the loopback interface, since the request comes from the machine itself:

```bash
sudo iptables -t nat -A OUTPUT -o lo -p tcp --dport 80 -j REDIRECT --to-port 20280
```

Then we test:

```bash
curl localhost:80/accounts
# [{"id":1,"name":"Alice","type":"Checking"}, ...]
```

And it works.

### <mark style="color:$warning;">Valladolid</mark>

**Context:** a `log-cleaner` service is supposed to clean up old logs but doesn't do what it should.

#### Initial diagnosis

```bash
sudo systemctl status log-cleaner
# Active: inactive (dead)
# ... Starting Cleanup... / Cleanup finished. / Deactivated successfully.
```

The service isn't active, but seems to have run previously. It logged "**Starting/Finished Cleanup**", then exits. We check that no process is locking the files involved:

```bash
lsof /var/log/app/old_data.log
lsof /var/log/app/recent_data.log
# rien de bloquant
```

Nothing abnormal. The real problem: the service isn't enabled:

```bash
systemctl is-enabled log-cleaner
# static
systemctl is-active log-cleaner
# inactive
```

When we try to enable it:

```bash
sudo systemctl enable log-cleaner
# The unit files have no installation config (WantedBy=, RequiredBy=, ...)
# This means they are not meant to be enabled or disabled using systemctl.
```

The service has no `[Install]` section, so it can't be set to start automatically. We can see this by reading the unit:

```bash
systemctl cat log-cleaner
[Unit]
Description=Daily Log Cleaner
[Service]
Type=oneshot
ExecStart=/bin/bash /opt/scripts/log-cleaner.sh
WorkingDirectory=/root
```

Indeed, no `[Install]` section. And in the script itself:

```bash
cat /opt/scripts/log-cleaner.sh
#!/bin/bash
LOG_DIR="/var/log/app"
DAYS=7

echo "Starting Cleanup..."
find . -maxdepth 1 -name "*.log" -type f -mtime -7 -print -delete
echo "Cleanup finished."
```

We can spot 3 mistakes:

1. The `LOG_DIR` and `DAYS` variables are declared but never used. So the `find` searches in `.` (the `WorkingDirectory`, `/root`) instead of the real log folder.
2. `-mtime -7` selects files modified **less** than 7 days ago, i.e. recent files, whereas we want files **older** than N days (`+7`).
3. No safeguard to avoid touching the active log file (`recent_data.log`), which could therefore get deleted too.

We fix all of this first:

```bash
#!/bin/bash
LOG_DIR="/var/log/app"
DAYS=7

echo "Starting Cleanup..."
find "$LOG_DIR" -maxdepth 1 -name "*.log" -type f -not -name "recent_data.log" -mtime +$DAYS -print -delete
echo "Cleanup finished."
```

Then we fix the systemd unit, adding the missing `[Install]` section. Since the service is meant to be launched manually (no timer or cron associated in this context), we use `multi-user.target`, standard on any Unix-like system:

```ini
[Unit]
Description=Daily Log Cleaner

[Service]
Type=oneshot
ExecStart=/bin/bash /opt/scripts/log-cleaner.sh
WorkingDirectory=/root

[Install]
WantedBy=multi-user.target
```

We reload and then enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable log-cleaner
# Created symlink '/etc/systemd/system/multi-user.target.wants/log-cleaner.service' → ...
```

And we run the test-log reset script before restarting the service manually:

```bash
./reset_logs.sh
# Logs reset.
sudo systemctl restart log-cleaner
cat /var/log/app/recent_data.log
# contenu récent toujours présent, intact
cat /var/log/app/old_data.log
# cat: /var/log/app/old_data.log: No such file or directory
```

The recent file is kept, the old one has been deleted, and the problem is solved.
# 9 - Budapest, Tukaani, Tokelau, Hanoi & Kampala

### <mark style="color:$warning;">Budapest</mark>

**The goal here is simple:** create an account for each user listed in `user_list.txt` (format `user;password`), with the matching password.

```bash
cat user_list.txt
# alexsmith;Yt7kE9wq
# emilyjones;Fg3vU1pz
# ...
```

Here AI helps a lot. Any of them can loop through the file and create/update each account:

```bash
#!/bin/bash
INPUT="user_list.txt"

while IFS=';' read -r user pass; do
    [ -z "$user" ] && continue
    echo "Création de l'utilisateur $user"

    if id "$user" >/dev/null 2>&1; then
        echo "  -> $user existe déjà, mise à jour du mot de passe"
    else
        useradd -m "$user"
    fi

    echo "${user}:${pass}" | chpasswd
done < "$INPUT"
```

### <mark style="color:$warning;">Tukaani</mark>

**Context of the XZ Utils backdoor challenge (CVE-2024-3094)**, discovered in March 2024, one of the most serious supply-chain incidents in the Linux ecosystem in recent years**:** a `jobapp` service loads a malicious version of a system library (`liblzma.so.5`) via an environment variable, instead of the real library.

Useful reminder before digging in: a static library is baked in at compile time, while a shared/dynamic library is loaded at runtime by the _dynamic linker/loader_ (`ld.so`). It's this dynamic loading that can be hijacked.

The correct version of the library, present on the system, is here:

```bash
ll /usr/lib/x86_64-linux-gnu/liblzma.so.5
# lrwxrwxrwx 1 root root 16 Apr 11 2022 liblzma.so.5 -> liblzma.so.5.2.5
```

Here, two web services run on the machine (`jobapp.service`, `webapp.service`). We inspect their systemd units:

```bash
cat /etc/systemd/system/multi-user.target.wants/webapp.service
[Service]
ExecStart = /opt/webapp/webapp.py
Environment="LD_LIBRARY_PATH=/opt/.trash/"
```

```bash
cat /etc/systemd/system/multi-user.target.wants/jobapp.service
[Service]
ExecStart = /opt/job-app/jobapp.py
EnvironmentFile=/opt/.trash/.jobapp.env
```

Both point to a suspicious folder, `/opt/.trash/`, which already smells bad.

```bash
sudo ls /opt/.trash/
# liblzma.so.5
```

A second copy of the library exists in that folder, outside the standard system location. We search everywhere anyway, and find the bad element:

```bash
sudo find / -name "liblzma*" 2>/dev/null
# /usr/lib/x86_64-linux-gnu/liblzma.so.5      <- la vraie
# /opt/.trash/liblzma.so.5                     <- la suspecte
```

The latter is also referenced in `/opt/.trash/.jobapp.env`:

```bash
cat /opt/.trash/.jobapp.env
APP_CONFIG_DB_NAME="jobapp"
APP_CONFIG_USER="dev"
LD_PRELOAD="/opt/.trash/liblzma.so.5"
DB_CONFIG_PRELOAD="true"
```

`LD_PRELOAD` forces the malicious library to load before any other.&#x20;

So we remove it via systemctl:

```bash
sudo systemctl edit webapp
# [Service]
# Environment=
sudo systemctl daemon-reload
```

Then we delete the library file and replace it with a symlink to the real system version, keeping the original name so as not to break an app/service. That way, even if a service still references that path, it now points to the legitimate library.

```bash
# Suppression du fichier malveillant (optionnel)
sudo rm /opt/.trash/liblzma.so.5

# Lien symbolique vers la bonne librairie
sudo ln -s /usr/lib/x86_64-linux-gnu/liblzma.so.5.2.5 /opt/.trash/liblzma.so.5
```

### <mark style="color:$warning;">Tokelau</mark>

**Goal:** clear the history lines containing "foo", without the current session making them reappear.

This challenge taught me something: `history -r` does **not replace** the current session's history, it **appends** the content of the bash history file to the list already loaded in memory. So deleting a line from `.bash_history` with `sed` isn't enough if the session's in-memory history still contains those commands.

The fix for this fits on one line:

```bash
sed -i '/foo/d' ~/.bash_history && history -c && history -r
```

* `sed -i '/foo/d' ~/.bash_history`: removes the lines containing "foo" from the file.
* `history -c`: clears the current session's in-memory history.
* `history -r`: reloads the history from the bash_history file.

### <mark style="color:$warning;">Hanoi</mark>

**Goal:** cross-reference two files — `users.txt` (list of users) and `groups.txt` (format `group_name:user1,user2,user3`) — to list the users belonging to more than one group, sorted alphabetically.

Prompt sent to Perplexity to generate the script:

> _Generate a bash command that sorts the users of users.txt belonging to more than 1 group in groups.txt, sorted alphabetically, one name per line._

Script obtained:

```bash
#!/bin/bash

# Charge users.txt dans un tableau associatif pour lookup rapide
declare -A valid_users
while IFS= read -r user; do
    [[ -n "$user" ]] && valid_users["$user"]=1
done < users.txt

# Compte les appartenances aux groupes pour chaque utilisateur valide
declare -A user_counts
while IFS=: read -r group users_list; do
    [[ -z "$group" || -z "$users_list" ]] && continue
    IFS=',' read -ra user_array <<< "$users_list"
    for user in "${user_array[@]}"; do
        [[ -n "$user" && ${valid_users[$user]+isset} ]] || continue
        ((user_counts[$user]++))
    done
done < groups.txt

# Sortie triée des utilisateurs avec plus d'1 groupe
for user in "${!user_counts[@]}"; do
    [[ ${user_counts[$user]} -gt 1 ]] && echo "$user"
done | sort > /home/admin/multi-group-users.txt
```

Works on the first try.

### <mark style="color:$warning;">Kampala</mark>

**Context:** the server contains deployment scripts that refuse to run.

First checks, nothing abnormal:

```bash
ll deploy/
# -rwxr-xr-x 1 admin admin 293 Sep 29 14:07 deploy.sh
```

The permissions are correct, as is the content of `deploy.sh`:

```bash
cat deploy.sh
#!/bin/bash
echo "Starting deployment process..."
# ...
```

Just in case, we make a copy in case we need to modify things. Then we change the perms and test by running it:

```bash
chmod 755 deploy_2.sh
./deploy_2.sh
# -bash: ./deploy_2.sh: cannot execute: required file not found
sudo ./deploy_2.sh
# sudo: unable to execute ./deploy_2.sh: No such file or directory
```

Running a script directly with `./` through the default shell doesn't work. Running it explicitly with `bash`, the real problem appears:

```bash
sudo bash deploy_2.sh
# deploy_2.sh: line 2: $'\r': command not found
# deploy_2.sh: line 14: syntax error: unexpected end of file
```

The script contains an invisible carriage-return character `$'\r'` at the end of lines, typical of files edited/created under Windows (CRLF line endings instead of Unix LF).

A search on the original error message (`cannot execute: required file not found`) confirms this: it's a classic issue with Windows line endings, where the interpreter named in the shebang (`#!/bin/bash\r`) isn't found as-is because of the stray `\r`.

To convert the file, we use **dos2unix**. It's a command-line tool that converts text files' line endings from DOS/Windows format (CRLF) to Unix/Linux format (LF). It removes the extra carriage returns to make scripts and files compatible with Unix-like systems:

```bash
dos2unix deploy_2.sh
# converting file deploy_2.sh to Unix format...
```

We apply it to all the scripts in the folder just in case:

```bash
dos2unix *
# converting file backup.sh to Unix format...
# converting file deploy.sh to Unix format...
# converting file setup.sh to Unix format...
```

Then:

```bash
./setup.sh
# Setting up application environment...
# mkdir: cannot create directory '/opt/app/logs': Permission denied
# Environment setup completed!
```

The script finally runs (a separate permission error remains on `/opt/app`, which is a different problem, out of scope).
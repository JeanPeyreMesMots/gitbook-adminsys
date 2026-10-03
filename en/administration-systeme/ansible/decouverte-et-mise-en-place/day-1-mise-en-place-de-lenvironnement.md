# 1 - Setting up the environment

### Why Ansible?

<div align="left"><img src="../../../.gitbook/assets/image (10).png" alt="" width="224"></div>

Ansible is an automation tool that lets you manage a large number of servers without working on each of them by hand letting you:

* automate server configuration;
* standardize installations;
* deploy applications;
* run tasks on many machines at once.

As the number of servers grows, manual work on them becomes time-consuming. Some servers turn into **snowflakes** (unique, fragile machines) that make every deployment stressful. Many infrastructures also end up with a pile of automation scripts that are hard to maintain.

Ansible aims to avoid this by making deployments reproducible, reliable, automated and, above all, **idempotent**.

The course is organized in three main parts:

### 1. Maintenance

How to quickly perform maintenance actions on many machines, for example to:

* check an application version;
* run a command on several servers.

> This should remain the exception: bulk actions should be automated through playbooks.

### 2. Server provisioning

This part focuses on writing Ansible playbooks and roles to standardize and automate server deployment.

### 3. Application deployment

This last part covers deploying an application on existing infrastructure, with the following goals:

* deploy new versions automatically;
* reduce human error;
* make production releases more reliable.

## Setting up the lab

As described in the previous page, we use Multipass to spin up lightweight VMs quickly and easily. Multipass is a lightweight VM manager developed by Canonical. It creates VMs very quickly with a low resource footprint, and runs on Windows, Linux and macOS. A really handy tool that I recommend for spinning up VMs in no time!

Once Multipass is installed, we create the machines with the following commands:

```bash
multipass launch 22.04 -n ansible-main -c 2 -m 1G

multipass launch 22.04 -n web-server-1 -c 1 -m 1G

multipass launch 22.04 -n web-server-2 -c 1 -m 1G

multipass launch 22.04 -n lb-server -c 1 -m 1G
```

**Note: the "lb-server" VM will be used later for load balancing ;)**

I ran into an issue on my Ubuntu VM: Multipass assigns IP addresses via DHCP, so the IPs change regularly (no static IPs), and hostnames stop resolving correctly.

The solution is to automatically update the `/etc/hosts` file from the output of `multipass list`, with a script like this:

```bash
sudo tee /usr/local/bin/sync-multipass-hosts.sh <<'EOF'
#!/bin/bash
set -e

MARKER_START="# BEGIN MULTIPASS"
MARKER_END="# END MULTIPASS"
HOSTS_FILE="/etc/hosts"

BLOCK=$(multipass list --format csv | tail -n +2 | while IFS=',' read -r name state ip image release; do
  ip_clean=$(echo "$ip" | cut -d';' -f1)

  if [ -n "$ip_clean" ] && [ "$state" = "Running" ]; then
    echo "$ip_clean  $name"
  fi
done)

sudo sed -i "/$MARKER_START/,/$MARKER_END/d" "$HOSTS_FILE"

{
  echo "$MARKER_START"
  echo "$BLOCK"
  echo "$MARKER_END"
} | sudo tee -a "$HOSTS_FILE" > /dev/null

echo "✅ /etc/hosts synchronisé :"
echo "$BLOCK"
EOF

sudo chmod +x /usr/local/bin/sync-multipass-hosts.sh
```

To keep the hosts file up to date, we add a crontab entry that runs the script every five minutes:

```bash
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/local/bin/sync-multipass-hosts.sh > /dev/null 2>&1") | crontab -
```

So the hosts we have here:

```
$ multipass list

Name              State     IPv4
ansible-main      Running   10.3.241.40
web-server-1      Running   10.3.241.212
web-server-2      Running   10.3.241.176
```

end up in the following `/etc/hosts` file:

```
# BEGIN MULTIPASS
10.3.241.179 ansible-main
10.3.241.212 web-server-1
10.3.241.176 web-server-2
# END MULTIPASS
```

## Preparing the VMs for Ansible

The VMs must allow SSH connections, password authentication and root login. Obviously something to avoid in production, but this is a local lab for demonstration purposes 🙂 :

```bash
multipass exec web-server-1 -- sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf

multipass exec web-server-1 -- sh -c "echo 'root:ansible' | sudo chpasswd"

multipass exec web-server-1 -- sudo systemctl restart ssh
```

Same procedure for "**web-server-2**" and for "**lb-server**", which will be used later for load balancing.

However, while my Ubuntu host VM knows the Multipass IPs and hostnames, my `ansible-main` VM doesn't. A second script generates a file with the IP addresses and copies it into that VM:

```bash
#!/bin/bash
set -e

TARGET_VM="ansible-main"
TMP_FILE="$HOME/hostfile-multipass"

multipass list --format csv | tail -n +2 | while IFS=',' read -r name state ip image release; do
  ip_clean=$(echo "$ip" | cut -d';' -f1)

  if [ -n "$ip_clean" ] && [ "$state" = "Running" ]; then
    echo "$ip_clean  $name"
  fi
done > "$TMP_FILE"

multipass transfer "$TMP_FILE" "$TARGET_VM":/home/ubuntu/hostfile

multipass exec "$TARGET_VM" -- bash -c '
sudo sed -i "/# BEGIN MULTIPASS/,/# END MULTIPASS/d" /etc/hosts
{
  echo "# BEGIN MULTIPASS"
  cat /home/ubuntu/hostfile
  echo "# END MULTIPASS"
} | sudo tee -a /etc/hosts > /dev/null
'

rm "$TMP_FILE"
```

**Note: this script does not make the IPs static. It only keeps hostname resolution working.**

Then we create an inventory file, "**inventory.ini**". Of course, no hardcoded passwords in production 🙂 :

```ini
[web]
web-server-1
web-server-2

[lb-host]
lb-server

[web:vars]
ansible_ssh_user=root
ansible_ssh_pass=ansible
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
ansible_python_interpreter=/usr/bin/python3.10
```

Then, on the "**ansible-main**" VM, we install **python3-pip** and **ansible**, and add the install directory to the PATH:

```bash
export PATH=$PATH:~/.local/bin
```

## First error

When we try to ping the hosts:

```bash
ansible -m ping -i inventory all
```

An error says that the `sshpass` package is required in this case:

```
FAILED!

to use the 'ssh' connection type with passwords
you must install the sshpass program
```

We fix that, with an update for good measure:

```bash
sudo apt update

sudo apt install -y sshpass
```

## Checking that it works

We ping all the hosts listed in the inventory:

```bash
ansible -m ping -i hosts.ini all
```

And the output 😉 :

```
web-server-1 | SUCCESS
web-server-2 | SUCCESS
```

Output:

```
{
    "changed": false,
    "ping": "pong"
}
```
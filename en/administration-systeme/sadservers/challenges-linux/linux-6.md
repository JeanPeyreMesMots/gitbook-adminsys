# 6 - Paris, Manado & Moyogalpa

### <mark style="color:$warning;">Paris</mark>

A bit of hacking, like the good old days :D

#### Reconnaissance

Running processes:

```bash
ps -faux | grep "python"
# root  693  ... /usr/bin/python3 /home/admin/webserver.py
```

Testing the endpoint:

```bash
curl -v http://localhost:5000
# HTTP/1.1 200 OK
# Server: Werkzeug/3.1.4 Python/3.13.5
# ...
Unauthorized
```

A 200 response but "Unauthorized" content, so not a real 401: the app handles auth itself in the response body.

A small bash script to test a list of common login/password pairs (admin:admin, root:root, guest:guest, etc.):

```bash
for cred in "${creds[@]}"; do
  username="${cred%%:*}"
  password="${cred#*:}"
  response=$(curl -s -u "$username:$password" "$URL")
  if [[ ! "$response" =~ Unauthorized ]]; then
    echo "SUCCESS! avec $username:$password"
    exit 0
  fi
done
```

No luck. Looking for known exploits on the stack in use: nothing conclusive either.

#### The flaw: missing User-Agent header

An idea tested somewhat at random: removing the `User-Agent` header from the request entirely.

```bash
curl -v -u admin:admin -H "User-Agent:" http://localhost:5000
# HTTP/1.1 200 OK
# Content-Length: 35
Welcome! Password is FDZPmh5AX3oiJt
```

Bingo: without a User-Agent, the app returns a password in plain text in the response.

We spray the password with different logins (admin, root, sad, sadservers, guest...): each time the same "**Welcome!**" response comes back, regardless of the login.

So it wasn't a real login credential, but very likely the challenge's solution directly:

```bash
echo "FDZPmh5AX3oiJt" > ~/mysolution
```

Confirmed :)

### <mark style="color:$warning;">Manado</mark>

_(Quick note, to be detailed later)_

Exercise around the `sort` command and `xz` compression:

```bash
sort names > names_COPY
ll
# -rw-r--r-- 1 root  root  35147 Mar  2  2024 names
# -rw-r--r-- 1 admin admin 35148 Mar 23 16:43 names_COPY
```

We compress it:

```bash
xz -k names_COPY
# crée names_COPY.xz (9328 octets), garde l'original grâce à -k
rm names_COPY.xz
```

Attempt with the maximum compression level:

```bash
xz -9 names_COPY
# xz: names_COPY: Cannot allocate memory
```

Failed, not enough disk space. Let's try level 5:

```bash
xz -5 names_COPY
ll
# names_COPY.xz  9336 octets
```

It works; we copy the result into the solution folder:

```bash
cp names_COPY.xz solution/
```

### <mark style="color:$warning;">Moyogalpa</mark>

**Context:** a Go app secured by John and Mike, which they broke. The challenge gives us a spec:

* communication over HTTPS only;
* access restricted to only the necessary files (certificates + static files);
* rate limiting at 10 requests/second;
* running as a non-root user.

At first, I struggled because the app wasn't launched by hand (via a `go run ...`), but managed as a systemd service. It took me a while to decide I should look at the logs via `journalctl -u webapp` rather than hunting for a manually launched process.

```bash
sudo journalctl -u webapp
# open /home/webapp/pki/server.crt: permission denied
# open /home/webapp/pki/server.pem: permission denied
# can not access certificate/key file. sleeping for 10s and will retry
```

The permissions on the certificates are already wrong, so we start by fixing them:

```bash
ll /home/webapp/pki/
# ls: cannot open directory '/home/webapp/pki/': Permission denied
ll /home/webapp/
# drwx------ 2 root root 4096 Apr 10 2024 pki
```

```bash
sudo chmod -R 755 pki/
sudo chown -R admin: pki/
```

But it won't work:

```bash
open /home/webapp/pki/server.pem: permission denied
```

With an openssl loop, we can test the validity of each certificate file:

```bash
for cert in *.crt *.pem; do
    openssl x509 -in "$cert" -noout -dates 2>/dev/null || echo "Pas un certificat"
done
# CA.crt : OK
# server.crt : OK
# server.pem : Pas un certificat
```

`server.pem` isn't recognized as a certificate. Looking at its content:

```bash
head server.pem
# -----BEGIN RSA PRIVATE KEY-----
```

Makes sense: it's an RSA private key, not a certificate, so `openssl x509` can't read it. We still check that the key is valid and matches the certificate:

```bash
openssl rsa -in server.pem -check -noout
# RSA key ok

openssl rsa -noout -modulus -in server.pem | openssl md5
openssl x509 -noout -modulus -in server.crt | openssl md5
# mêmes hash des deux côtés → la paire clé/certificat est cohérente
```

The files themselves are fine. I adjusted the permissions on the static files just in case:

```bash
sudo chmod -R 755 static-files/
```

But with no effect on the main problem.

After putting the files back under the right owner (`webapp:webapp` rather than `admin`), and looking up the solution online, I found that the certificate needed to go in the system folder "_**/usr/local/share/ca-certificates/**_" to avoid having to use `--cacert` on every request:

```bash
sudo cp /home/webapp/pki/CA.crt /usr/local/share/ca-certificates/webappCA.crt
sudo chmod 644 /usr/local/share/ca-certificates/webappCA.crt
sudo update-ca-certificates
```

We test:

```bash
curl https://webapp:7000
# curl: (6) Could not resolve host: webapp
```

We'll get there :D The `webapp` hostname doesn't resolve because it's not in /etc/hosts; we add it:

```bash
echo "127.0.0.1 webapp" | sudo tee --append /etc/hosts
```

A new error, different this time:

```bash
curl https://webapp:7000
# Forbidden
```

Off to the logs:

```bash
open /home/webapp/static-files/users.html: permission denied
```

And there, the real culprit shows itself. Searching the "**permission denied**" message online, the app turns out to be confined by an AppArmor profile (`/etc/apparmor.d/usr.local.bin.webapp`), which allowed access to the certificates but not the static files. The profile looks like this:

```
/home/webapp/pki/ r,
/home/webapp/pki/server.pem r,
/home/webapp/pki/server.crt r,
# manquant : accès à static-files
```

We add the necessary lines:

```
/home/webapp/static-files/ r,
/home/webapp/static-files/* r,
```

And reload:

```bash
apparmor_parser -r /etc/apparmor.d/usr.local.bin.webapp
```

Final test:

```bash
curl https://webapp:7000/users.html
# <p>From Users Page</p>
```

Solved. In order, we fixed the classic permissions, checked the certificates, local DNS, and finally AppArmor.
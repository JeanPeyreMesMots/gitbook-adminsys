# 4 - Oaxaca, Melbourne & Lisbon

### <mark style="color:$warning;">Oaxaca</mark>

**Goal:** close a file opened by a process, without killing that process.

Straight to a search: _"close a file without killing its process"_, which leads to [this superuser thread](https://superuser.com/questions/963612/closing-open-file-without-killing-the-process).

We look at the open file and the process holding it:

```bash
ll /home/admin/somefile
# -rw-r--r-- 1 admin admin 0 Mar 12 16:16 /home/admin/somefile

lsof /home/admin/somefile
# COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
# bash    1037 admin   77w   REG  259,1        0 272875 /home/admin/somefile
```

The file is open by `bash` (PID 1037) on descriptor `77w` (write).

We can see it with `lsof -p`:

```bash
lsof -p 1037
# bash    1037 admin   77w   REG  259,1        0 272875 /home/admin/somefile
```

If we close it:

```bash
exec 77w>&-
# -bash: exec: 77w: not found
```

Syntax error: the `w` isn't part of the descriptor number, it's just an indicator in the `lsof` output. This is better:

```bash
exec 77>&-
lsof -p 1037
# (plus rien listé sur ce fichier)
```

The descriptor is closed, with the bash process still alive. Simpler than expected in the end.

### <mark style="color:$warning;">Melbourne</mark>

**Context:** a Python WSGI app (`/home/admin/wsgi.py`) is supposed to output "**Hello, world!**", behind Gunicorn, itself behind nginx. The expected chain: `curl → nginx → Gunicorn → wsgi.py`. Goal: have `curl localhost` return "Hello, world!".

Nginx is off, we turn it back on:

```bash
sudo systemctl status nginx
# Active: inactive (dead)
sudo systemctl start nginx
sudo systemctl status nginx
# Active: active (running)
```

Config tested, syntax OK:

```bash
sudo nginx -t
# syntax ok, test successful
```

But still not working. It used to work but not anymore :P :

```bash
curl http://localhost
# 502 Bad Gateway
```

Let's look at the wsgi file in question:

```python
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html'), ('Content-Length', '0'), ])
    return [b'Hello, world!']
```

If we try to launch it in the background:

```bash
gunicorn wsgi:application --daemon
```

We get a 502. What do the holy nginx logs say?

```bash
cat /var/log/nginx/error.log
# connect() to unix:/run/gunicorn.socket failed (2: No such file or directory)
```

So nginx is trying to reach a socket that doesn't exist. BUT, looking at Gunicorn's status, it was stopped:

```bash
sudo systemctl status gunicorn
# Active: inactive (dead)
sudo systemctl start gunicorn
sudo systemctl status gunicorn
# Active: active (running)
```

Still a 502 though. I lean toward the phantom socket theory, but looking closer:

```bash
ls -la /run/gunicorn.socket
# ls: cannot access '/run/gunicorn.socket': No such file or directory
ll /run/gunicorn.sock
# srw-rw-rw- 1 root root 0 Mar 12 18:17 /run/gunicorn.sock
```

The real socket is actually named `gunicorn.sock` (without the final "**et**"), not `gunicorn.socket`. This typo is visible in the nginx config:

```nginx
server {
    listen 80;
    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.socket;  # --> à corriger en "gunicorn.sock"
    }
}
```

We fix it, then restart the services involved. And there:

```bash
curl -I http://localhost
# HTTP/1.1 200 OK
# Content-Length: 0
```

The headers go through, but Content-Length is 0, so nothing in the response.

In **wsgi.py** itself, the file hardcodes `Content-Length: 0` while it actually returns `b'Hello, world!'`, hence the mismatch between the announced header and the real body. I still asked an AI to review the code, which gives in the end:

```python
def application(environ, start_response):
    status = '200 OK'
    output = b'Hello, world!'
    headers = [('Content-Type', 'text/html'), ('Content-Length', str(len(output)))]
    start_response(status, headers)
    return [output]
```

One of the changes it made sets `Content-Length` to be computed dynamically from the returned content. We hit it locally:

```bash
curl http://localhost
# Hello, world!
```

### <mark style="color:$warning;">Lisbon</mark>

**Context:** an etcd server with, apparently, an SSL certificate problem.

```bash
ps faux | grep etcd
# /usr/bin/etcd --cert-file /etc/ssl/certs/localhost.crt --key-file /etc/ssl/certs/localhost.key --advertise-client-urls=https://localhost:2379 --listen-client-urls=https://localhost:2379
```

I started from the idea that the SSL certificate needed renewing based on the system date. I went through several tutorials, all nginx / Let's Encrypt / certbot oriented, but nothing worked.

Setting the system date back to an earlier one (January 1, 2023), the certificate error did disappear... but another one appeared instead:

```bash
sudo date -s 01/03/2023
etcdctl get foo
# Error: client: response is invalid json. The endpoint is probably not valid etcd cluster endpoint.
```

So the certificate wasn't the root cause, just a symptom tied to the date, not the root cause.

We test the etcd endpoints directly over HTTPS:

```bash
curl https://localhost:2379/v2/keys/foo
# 404 Not Found (nginx)
curl https://localhost:2379/v2/
# 404 Not Found (nginx)
curl https://localhost:2379/
# Testing SSL
```

Still stuck on "Testing SSL...". Checking the nginx config: nothing wrong on the surface (listening on 443, syntactically valid config):

```bash
sudo nginx -t
# syntax ok, test successful
```

On to the iptables rules, the NAT table in particular:

```bash
sudo iptables -t nat -L
```

```
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
REDIRECT   tcp  --  anywhere             anywhere             tcp dpt:2379 redir ports 443
```

Found it: **all** TCP traffic destined for port 2379 (etcd) is forwarded to port 443 by iptables. That rule was causing the nginx 404s instead of etcd responses.&#x20;

We remove the redirect rules from the OUTPUT chain of the NAT table:

```bash
sudo iptables -t nat -F OUTPUT
```

Then check for anything new:

```bash
curl https://localhost:2379/v2/keys/foo
# {"action":"get","node":{"key":"/foo","value":"bar","modifiedIndex":4,"createdIndex":4}}
```

Solved.
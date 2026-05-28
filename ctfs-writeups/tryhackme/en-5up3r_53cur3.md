---
description: 'WriteUp of the Box 5up3r_53cur3, created by @Noobosaurus_R3x :'
---

# 🇬🇧 \[EN] 5up3r\_53cur3

Link :

{% embed url="https://tryhackme.com/room/5up3rs3cur3" %}

### 1 - Recognition

First let's start to discovering infos about the host using Nmap. Here we choose to search for all the opened ports and its services, with the -p- and -sV arguments, and an agressive scan with -A, all with the verbosity spitting with -vv, all with a SYN Scan -sS (note : some non-useful infos has been removed from the output) :

```shell
# nmap -vv -p- -sV -A -sS 10.10.113.169
Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-15 15:19 CET
Initiating Parallel DNS resolution of 1 host. at 15:19
Completed Parallel DNS resolution of 1 host. at 15:19, 0.01s elapsed
Initiating SYN Stealth Scan at 15:19
Scanning 10.10.113.169 [65535 ports]
Discovered open port 21/tcp on 10.10.113.169
Discovered open port 22/tcp on 10.10.113.169
Discovered open port 80/tcp on 10.10.113.169
Discovered open port 8080/tcp on 10.10.113.169
SYN Stealth Scan Timing: About 40.28% done; ETC: 15:20 (0:00:46 remaining)
Completed SYN Stealth Scan at 15:20, 59.67s elapsed (65535 total ports)
Initiating Service scan at 15:20
Scanning 4 services on 10.10.113.169
Completed Service scan at 15:20, 6.65s elapsed (4 services on 1 host)
Initiating OS detection (try #1) against 10.10.113.169
Retrying OS detection (try #5) against 10.10.113.169
Initiating Traceroute at 15:20
Completed Traceroute at 15:20, 0.07s elapsed
Initiating Parallel DNS resolution of 2 hosts. at 15:20
Completed Parallel DNS resolution of 2 hosts. at 15:20, 0.01s elapsed
NSE: Script scanning 10.10.113.169.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 15:20
NSE: [ftp-bounce 10.10.113.169:21] PORT response: 500 Illegal PORT command.
Completed NSE at 15:20, 12.74s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 15:20
Completed NSE at 15:20, 0.56s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 15:20
Completed NSE at 15:20, 0.00s elapsed
Nmap scan report for 10.10.113.169
Host is up, received echo-reply ttl 63 (0.065s latency).
Scanned at 2022-12-15 15:19:10 CET for 93s
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE REASON         VERSION
21/tcp   open  ftp     syn-ack ttl 63 vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.8.23.112
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 65534    65534          25 Nov 27 11:41 file.txt
22/tcp   open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ba0de89e97c5f0f141334db7b2f88c77 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCnyqpWcorTau5fARjuGtFnSz+p/ci9Vak5u2EKo9Kgc5FWpyG9x+72+MUPjOumD46nF4nh+wUtbfXzqDRXwMt1RAGlBMB+QWaLWQHFp/iIjFG6YBEo0CK5MkR1gzHWpdTKOhBaamzIrSCGkIyyZ6O7aZTXi9C7P8DWPKhQSVtyqYdObgU+99aa6ZnqiT6xU2Il6SBzD1k0C6Ev63iFTlIAwdkqP2vkBdpOkEEPcTBccaZK5WpeSQnzr/0izAF9BjzbvQNdBf3be0yWINBpCTDwzVRV+jEK87Pvwb2yiqLsSc0o7D27Vwo9g4IXc+JV2xM8RmeE5VImcmpfqSdrpaII3PXVEzYVFr8eUMEXnevhL3zKFs3Gfq7i47xwS7jbHJ1hUEKPyduArvlcCedHUiaOoa9hEQm+YdhLx3/8c9uxAFCdWxupqe955eemcWNN2O5ZEs88qn+cJrYA04mdH46kMu0gnjINJ4Kp+OfvIqERiEU4MO7fwDU7/I0I/d2WHvs=
|   256 a8b923323b9b2beb6d53f168cc7318e3 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBI+ivRfgkc0rhx+u68ey8K8wXn36EEusbdoqsJtHokVQ0BLOyplXHtNsfrJl1iaCnSm90nslwYeeTDTnC2xdAY8=
|   256 ac92265c9007e266444acd10d57230b7 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILKPZem8XUWDBaJ6fnUV6uEPPfGfqyw8gLzIojm8JnTG
80/tcp   open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
|_http-title: Apache2 Ubuntu Default Page: It works
| http-robots.txt: 227 disallowed entries (40 shown)
| /search /sdch /groups /index.html? /? /?hl=*& 
| /?hl=*&*&gws_rd=ssl /imgres /u/ /preferences /setprefs /default /m? /m/ /wml? 
| /wml/? /wml/search? /xhtml? /xhtml/? /xhtml/search? /xml? 
| /imode? /imode/? /imode/search? /jsky? /jsky/? /jsky/search? 
| /pda? /pda/? /pda/search? /sprint_xhtml /sprint_wml /pqa 
| /palm /gwt/ /bm9vb29vcGU= /NXVwM3JfYzRjaDM= /local? 
|_/local_url /shihui?
|_http-server-header: Apache/2.4.41 (Ubuntu)
8080/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
| http-robots.txt: 227 disallowed entries (40 shown)
| /search /sdch /groups /index.html? /? /?hl=*& 
| /?hl=*&*&gws_rd=ssl /imgres /u/ /preferences /setprefs /default /m? /m/ /wml? 
| /wml/? /wml/search? /xhtml? /xhtml/? /xhtml/search? /xml? 
| /imode? /imode/? /imode/search? /jsky? /jsky/? /jsky/search? 
| /pda? /pda/? /pda/search? /sprint_xhtml /sprint_wml /pqa 
| /palm /gwt/ /bm9vb29vcGU= /NXVwM3JfYzRjaDM= /local? 
|_/local_url /shihui?
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
|_http-server-header: Apache/2.4.41 (Ubuntu)

```

Interesting stuffs here, we have services running on common ports

* FTP (21) on **vsFTPD 3.0.3** :
* SSH (22) on **OpenSSH 8.2p1 Ubuntu 4ubuntu0.5** along with the public keys of the server
* HTTP Server (80, 8080) on **Apache 2.4.41**

We can see that nmap found a file called "file.txt" on the server. This has been possible by trying to connect to it using Anonymous. Assuming that the FTP server accept Anonymous connection, let's try to get the file on it ! :&#x20;

```
# ftp Anonymous@10.10.113.169
Connected to 10.10.113.169.
220 (vsFTPd 3.0.3)
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||8712|)
150 Here comes the directory listing.
-rw-r--r--    1 65534    65534          25 Nov 27 11:41 file.txt
226 Directory send OK.
ftp> get file.txt
local: file.txt remote: file.txt
229 Entering Extended Passive Mode (|||26615|)
150 Opening BINARY mode data connection for file.txt (25 bytes).
100% |***********************************|    25        8.08 KiB/s    00:00 ETA
226 Transfer complete.
25 bytes received in 00:00 (0.62 KiB/s)
ftp> exit
221 Goodbye.

┌──(root💀el-huervo)-[~/others_stuff/thm/5up3r_53cur3]
└─# cat file.txt 
Naaaah ! Not that fast !

```

....Would have been to easy. Let's try to go on the web page despite it :D :&#x20;

<figure><img src="../../.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

```
search
search/about
search/static
search/howsearchworks
sdch
groups
index.html?
?
?hl=
?hl=*&
?hl=*&gws_rd=ssl$
?hl=*&*&gws_rd=ssl
?gws_rd=ssl$
?pt1=true$
imgres
u/
preferences
setprefs
default
m?
m/
m/finance
wml?
wml/?
wml/search?
xhtml?
xhtml/?
xhtml/search?
xml?
imode?
imode/?
imode/search?
jsky?
jsky/?
jsky/search?
pda?
pda/?
pda/search?
sprint_xhtml
sprint_wml
pqa
palm
gwt/
bm9vb29vcGU=
NXVwM3JfYzRjaDM=
local?
local_url
shihui?
shihui/
products?
product_
products_
products;
print
books/
bkshp?*q=*
books?*q=*
books?*output=*
books?*pg=*
books?*jtp=*
books?*jscmd=*
books?*buy=*
books?*zoom=*
books?*q=related
books?*q=editions
books?*q=subject
books/about
booksrightsholders
books?*zoom=1*
books?*zoom=5*
books/content?*zoom=1*
books/content?*zoom=5*
ebooks/
ebooks?*q=*
ebooks?*output=*
ebooks?*pg=*
ebooks?*jscmd=*
ebooks?*buy=*
ebooks?*zoom=*
ebooks?*q=related
ebooks?*q=editions
ebooks?*q=subject
ebooks?*zoom=1*
ebooks?*zoom=5*
patents?
patents/download/
patents/pdf/
patents/related/
scholar
citations?
citations?user=
citations?*cstart=
citations?view_op=new_profile
citations?view_op=top_venues
scholar_share
s?
maps?*output=classic*
maps?*file=
maps/d/
maps?
mapstt?
mapslt?
maps/stk/
maps/br?
mapabcpoi?
maphp?
mapprint?
maps/api/js/
maps/api/js
maps/api/place/js/
maps/api/staticmap
maps/api/streetview
maps/_/sw/manifest.json
mld?
staticmap?
maps/preview
maps/place
maps/timeline/
help/maps/streetview/partners/welcome/
help/maps/indoormaps/partners/
lochp?
center
ie?
blogsearch/
blogsearch_feeds
advanced_blog_search
uds/
chart?
transit?
calendar$
calendar/about/
calendar/
cl2/feeds/
cl2/ical/
coop/directory
coop/manage
trends?
trends/music?
trends/hottrends?
trends/viz?
trends/embed.js?
trends/fetchComponent?
trends/beta
trends/topics
musica
musicad
musicas
musicl
musics
musicsearch
musicsp
musiclp
urchin_test/
movies?
wapsearch?
safebrowsing/diagnostic
safebrowsing/report_badware/
safebrowsing/report_error/
safebrowsing/report_phish/
reviews/search?
orkut/albums
cbk
recharge/dashboard/car
recharge/dashboard/static/
profiles/me
profiles
s2/profiles/me
s2/profiles
s2/oz
s2/photos
s2/search/social
s2/static
s2
transconsole/portal/
gcc/
aclk
cse?
cse/home
cse/panel
cse/manage
tbproxy/
imesync/
shenghuo/search?
support/forum/search?
reviews/polls/
hosted/images/
ppob/?
ppob?
accounts/ClientLogin
accounts/ClientAuth
accounts/o8
accounts/o8/id
topicsearch?q=
xfx7/
squared/api
squared/search
squared/table
qnasearch?
app/updates
sidewiki/entry/
quality_form?
labs/popgadget/search
buzz/post
compressiontest/
analytics/feeds/
analytics/partners/comments/
analytics/portal/
analytics/uploads/
alerts/manage
alerts/remove
alerts/
alerts/$
ads/search?
ads/plan/action_plan?
ads/plan/api/
ads/hotels/partners
phone/compare/?
travel/clk
hotelfinder/rpc
hotels/rpc
commercesearch/services/
evaluation/
chrome/browser/mobile/tour
compare/*/apply*
forms/perks/
shopping/suppliers/search
ct/
edu/cs4hs/
trustedstores/s/
trustedstores/tm2
trustedstores/verify
adwords/proposal
shopping?*
shopping/product/
shopping/seller
shopping/ratings/account/metrics
shopping/ratings/merchant/immersivedetails
shopping/reviewer
about/careers/applications/
about/careers/applications-a/
landing/signout.html
webmasters/sitemaps/ping?
ping?
gallery/
landing/now/ontap/
searchhistory/
maps/reserve
maps/reserve/partners
maps/reserve/api/
maps/reserve/search
maps/reserve/bookings
maps/reserve/settings
maps/reserve/manage
maps/reserve/payment
maps/reserve/receipt
maps/reserve/sellersignup
maps/reserve/payments
maps/reserve/feedback
maps/reserve/terms
maps/reserve/m/
maps/reserve/b/
maps/reserve/partner-dashboard
about/views/
intl/*/about/views/
local/cars
local/cars/
local/dealership/
local/dining/
local/place/products/
local/place/reviews/
local/place/rap/
local/tab/
localservices/*
finance
js/
nonprofits/account/
fbx
uviewer
maps/api/js/
maps/api/js
maps/api/place/js/
maps/api/staticmap
maps/api/streetview
imgres
imgres
```

Here we can give it a new shot :

```
# gobuster dir -u http://10.10.251.48 -w robots3.txt
===============================================================
Gobuster v3.3
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.251.48
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                robots3.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.3
[+] Timeout:                 10s
===============================================================
2022/12/15 18:04:17 Starting gobuster in directory enumeration mode
===============================================================
/?                    (Status: 200) [Size: 10918]
/?hl=                 (Status: 200) [Size: 10918]
/?gws_rd=ssl$         (Status: 200) [Size: 10918]
/?hl=*&*&gws_rd=ssl   (Status: 200) [Size: 10918]
/?hl=*&gws_rd=ssl$    (Status: 200) [Size: 10918]
/?pt1=true$           (Status: 200) [Size: 10918]
/index.html?          (Status: 200) [Size: 10918]
/?hl=*&               (Status: 200) [Size: 10918]
Progress: 236 / 287 (82.23%)===============================================================
2022/12/15 18:04:24 Finished
===============================================================
```

Still got nothing. But, if we look closely into the robots.txt file, we can see an interesting string... :

```
Disallow: /bm9vb29vcGU=
Disallow: /NXVwM3JfYzRjaDM=
```

When we see it, it obviously looks like a base64 string. If we aren't sure, we can still pass it to [CyberChef](https://gchq.github.io/CyberChef/) , the swiss army knife for encoded strings, URIs, zip... Absolute needed for CTFs. We can use the "magic" option to optain instant recognition of the string got if we still aren't sure of what type of string it is. Let's try the first one :

<figure><img src="../../.gitbook/assets/image (68).png" alt=""><figcaption></figcaption></figure>

That's some good trolling. And what about the second ? :&#x20;

<figure><img src="../../.gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>

Here we got it ! Another subdirectory called **5up3r\_c4ch3**, let's paste it and search for it !<br>

<figure><img src="../../.gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

Uh, nice... White page. Let's be serious for a minute, if we look into the source code :

<figure><img src="../../.gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

We're on a good path ;) Let's feed this to cyberchef one more time :

<figure><img src="../../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

### 2 - Getting an access :&#x20;

```
jeannoobie@sup3r-s3cur3:~$ history 
    1  su - root
    2  su - root
    3  ls -la
    4  echo 'set +o history' > .bashrc
    5  ls -la
    6  bash
jeannoobie@sup3r-s3cur3:~$
```

Holy noobo :D

To remeditate this, just do the opposite of the "echo" and redirect it to your .bashrc ;) Don't forget to do "source .bashrc" to reload the modifications :

```
jeannoobie@sup3r-s3cur3:~$ echo 'set -o history' > .bashrc
jeannoobie@sup3r-s3cur3:~$ source .bashrc 
jeannoobie@sup3r-s3cur3:~$ echo "Now we're good"
Now we're good
jeannoobie@sup3r-s3cur3:~$ history 
    1  su - root
    2  su - root
    3  ls -la
    4  echo 'set +o history' > .bashrc
    5  ls -la
    6  bash
    7  su - root
    8  su - root
    9  ls -la
   10  echo 'set +o history' > .bashrc
   11  ls -la
   12  bash
   13  echo "Now we're good"
   14  history 
jeannoobie@sup3r-s3cur3:~$ 
```

### 3 - Getting the first flag :

Now let's check for some files. I've put an alias to avoid typing "ls -l" all the time (and no, I'm not lazy) :

```
jeannoobie@sup3r-s3cur3:~$ alias 'll=ls -l'
jeannoobie@sup3r-s3cur3:~$ ll
total 8
-rw-rw-r-- 1 jeannoobie jeannoobie 65 Nov 27 14:08 RouteToRoot.txt
-rw-rw-r-- 1 jeannoobie jeannoobie 20 Nov 27 13:26 user.txt
jeannoobie@sup3r-s3cur3:~$ cat user.txt 
n***{*************}
jeannoobie@sup3r-s3cur3:~$ 
```

And we got the flag, hidden here but it start with the **"n"** character ;)

<figure><img src="../../.gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

### 3 - Becoming r00t :&#x20;

Now that we have our first user flag, we may want to acquire the super-root powers, right ? Let's start it by checking the other interesting file, **"RouteToRoot.txt"** :

```
jeannoobie@sup3r-s3cur3:~$ ll
total 8
-rw-rw-r-- 1 jeannoobie jeannoobie 65 Nov 27 14:08 RouteToRoot.txt
-rw-rw-r-- 1 jeannoobie jeannoobie 20 Nov 27 13:26 user.txt
jeannoobie@sup3r-s3cur3:~$ cat RouteToRoot.txt 
Cherche les GET requests et tu trouveras le flag, jeune Padawan.
jeannoobie@sup3r-s3cur3:~$
```

"Search for the GET requests and you will find the flag, young Padawan"

<figure><img src="../../.gitbook/assets/image (53).png" alt=""><figcaption></figcaption></figure>

"GET Requests", sounds like some networking stuffs for us right ?

Maybe we can take a look at the requests on the web server, using our favorite network-sword **Wireshark** : <br>

<figure><img src="../../.gitbook/assets/image (56).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>

Nothing on the page where we found the SSH creditentials. Maybe in the robots.txt page ?

<figure><img src="../../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

There is still the Apache default web page :&#x20;

<figure><img src="../../.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

```
jeannoobie@sup3r-s3cur3:/var/log/apache2$ cat error.log
[Sat Dec 17 16:34:55.297039 2022] [mpm_event:notice] [pid 718:tid 139795583835200] AH00489: Apache/2.4.41 (Ubuntu) configured -- resuming normal operations
[Sat Dec 17 16:34:55.297140 2022] [core:notice] [pid 718:tid 139795583835200] AH00094: Command line: '/usr/sbin/apache2'
[Sat Dec 17 16:35:24.696349 2022] [mpm_event:notice] [pid 718:tid 139795583835200] AH00493: SIGUSR1 received.  Doing graceful restart
[Sat Dec 17 16:35:25.823221 2022] [mpm_event:notice] [pid 718:tid 139795583835200] AH00489: Apache/2.4.41 (Ubuntu) configured -- resuming normal operations
[Sat Dec 17 16:35:25.823239 2022] [core:notice] [pid 718:tid 139795583835200] AH00094: Command line: '/usr/sbin/apache2'
jeannoobie@sup3r-s3cur3:/var/log/apache2$ 

```

Now let's check the **error.log**.1 :

```
jeannoobie@sup3r-s3cur3:/var/log/apache2$ ll
total 150348
-rw-r----- 1 root adm     10799 Dec 17 17:37 access.log
-rw-r----- 1 root adm 153906428 Nov 27 16:57 access.log.1
-rw-r----- 1 root adm       688 Dec 17 16:35 error.log
-rw-r----- 1 root adm     11042 Nov 28 00:36 error.log.1
-rw-r----- 1 root adm     14364 Dec 17 17:58 other_vhosts_access.log
-rw-r--r-- 1 root adm      1540 Nov 28 00:36 other_vhosts_access.log.1
jeannoobie@sup3r-s3cur3:/var/log/apache2$ cat error.log.1
[Sun Nov 27 14:01:44.236975 2022] [core:error] [pid 2184:tid 140150269007616] [client 192.168.30.157:32776] AH00126: Invalid URI in request GET /../../../../../../../../../../../../etc/shadow HTTP/1.1
[...]
[Sun Nov 27 14:01:45.939389 2022] [core:error] [pid 2183:tid 140150260614912] [client 192.168.30.157:34034] AH00126: Invalid URI in request GET /../../../../../../../../../boot.ini HTTP/1.1
[Sun Nov 27 14:01:45.942934 2022] [core:error] [pid 2184:tid 140150578464512] [client 192.168.30.157:34188] AH00126: Invalid URI in request GET /../../../../winnt/repair/sam._ HTTP/1.1
[Sun Nov 27 14:01:45.963024 2022] [core:error] [pid 2184:tid 140150419977984] [client 192.168.30.157:34190] AH00126: Invalid URI in request GET ////./../.../boot.ini HTTP/1.1
[Sun Nov 27 14:01:46.028399 2022] [core:error] [pid 2183:tid 140150419977984] [client 192.168.30.157:34198] AH00126: Invalid URI in request GET /DomainFiles/*//../../../../../../../../../../etc/passwd HTTP/1.1
[Sun Nov 27 14:01:46.181120 2022] [core:error] [pid 2183:tid 140150461941504] [client 192.168.30.157:34260] AH00126: Invalid URI in request GET /../../../../../../../../../../etc/passwd HTTP/1.1
[Sun Nov 27 14:01:46.201075 2022] [core:error] [pid 2184:tid 140150478726912] [client 192.168.30.157:34426] AH00126: Invalid URI in request GET /%2E%2E/%2E%2E/%2E%2E/%2E%2E/%2E%2E/windows/win.ini HTTP/1.1
[Sun Nov 27 14:01:46.203644 2022] [core:error] [pid 2184:tid 140150378014464] [client 192.168.30.157:34428] AH00126: Invalid URI in request GET /%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd HTTP/1.1
[Sun Nov 27 14:01:47.856118 2022] [core:error] [pid 2183:tid 140150478726912] [client 192.168.30.157:35614] AH00126: Invalid URI in request GET /file/../../../../../../../../etc/ HTTP/1.1
[client 192.168.30.157:44452] AH01630: client denied by server configuration: /var/www/html/.htpasswd
[Sun Nov 27 14:01:51.771853 2022] [authz_core:error] [pid 2183:tid 140150436763392] [client 192.168.30.157:44452] AH01630: client denied by server configuration: /var/www/html/.htaccess
[Sun Nov 27 14:01:51.818572 2022] [core:error] [pid 2183:tid 140150386407168] [client 192.168.30.157:44452] AH00126: Invalid URI in request GET ////../../data/config/microsrv.cfg HTTP/1.1
[Sun Nov 27 14:01:51.821260 2022] [core:error] [pid 2184:tid 140150394799872] [client 192.168.30.157:44790] AH00126: Invalid URI in request GET ////////../../../../../../etc/passwd HTTP/1.1
[Sun Nov 27 14:01:52.361710 2022] [negotiation:error] [pid 2184:tid 140150586857216] (2)No such file or directory: [client 192.168.30.157:45054] AH00683: cannot access type map file: /var/www/html/index.html.var
[Sun Nov 27 14:02:04.271525 2022] [core:error] [pid 2183:tid 140150411585280] [client 192.168.30.157:43340] AH00126: Invalid URI in request GET /sdk/%2E%2E/%2E%2E/%2E%2E/%2E%2E/%2E%2E/%2E%2E/etc/vmware/hostd/vmInventory.xml HTTP/1.1
[Sun Nov 27 14:02:04.906543 2022] [core:error] [pid 2183:tid 140150461941504] [client 192.168.30.157:43864] AH00126: Invalid URI in request GET /../../windows/dvr2.ini HTTP/1.1
[Sun Nov 27 14:02:04.911716 2022] [core:error] [pid 2184:tid 140150445156096] [client 192.168.30.157:44106] AH00126: Invalid URI in request GET /htdocs/../../../../../../../../../../../etc/passwd HTTP/1.1
[Sun Nov 27 14:02:04.953817 2022] [core:error] [pid 2184:tid 140150428370688] [client 192.168.30.157:44112] AH00126: Invalid URI in request GET /help/../../../../../../../../../../../../../../../../etc/shadow HTTP/1.1
[Sun Nov 27 14:02:33.491800 2022] [core:error] [pid 2183:tid 140150260614912] [client 192.168.30.157:45000] AH00135: Invalid method in request QIOT / HTTP/1.1
[Sun Nov 27 14:26:42.138572 2022] [mpm_event:notice] [pid 2182:tid 140150597483584] AH00491: caught SIGTERM, shutting down
[...]
```

Looks like there are some juicy informations here ! Someone tried to access to important ressources, like trying the **"/../../../../../../etc/shadow"** famous way to access hashed passwords stored on Unix OSs, or even the bootfile **boot.ini**, a text file that contains the boot options for computers with BIOS firmware running NT-based operating system prior to Windows Vista ! That's funny somehow :D

But, trying these path on the web page doesn't work, we're still stuck :/

<figure><img src="../../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (66).png" alt=""><figcaption></figcaption></figure>

No base64 or encoded string in the file. Maybe if we take a look at the **other\_vhosts\_access.log** file ?

```
jeannoobie@sup3r-s3cur3:/var/log/apache2$ ll
total 150328
-rw-r----- 1 root adm       515 Dec 17 18:50 access.log
-rw-r----- 1 root adm 153906428 Nov 27 16:57 access.log.1
-rw-r----- 1 root adm       688 Dec 17 18:51 error.log
-rw-r----- 1 root adm     11042 Nov 28 00:36 error.log.1
-rw-r----- 1 root adm      1026 Dec 17 18:56 other_vhosts_access.log
-rw-r--r-- 1 root adm      1540 Nov 28 00:36 other_vhosts_access.log.1
jeannoobie@sup3r-s3cur3:/var/log/apache2$ cat other_vhosts_access.log
127.0.0.1:80 127.0.0.1 - - [17/Dec/2022:18:51:15 +0000] "GET /cm9vdDpNb3RkZVBhc3NlVHJvcExvbmdQb3VyRXRyZUJydXRlRm9yY2VQYXJVbkRlYmlsZQ== HTTP/1.1" 404 434 "-" "curl/7.68.0"
127.0.0.1:80 127.0.0.1 - - [17/Dec/2022:18:52:09 +0000] "GET /cm9vdDpNb3RkZVBhc3NlVHJvcExvbmdQb3VyRXRyZUJydXRlRm9yY2VQYXJVbkRlYmlsZQ== HTTP/1.1" 404 434 "-" "curl/7.68.0"
127.0.0.1:80 127.0.0.1 - - [17/Dec/2022:18:53:09 +0000] "GET /cm9vdDpNb3RkZVBhc3NlVHJvcExvbmdQb3VyRXRyZUJydXRlRm9yY2VQYXJVbkRlYmlsZQ== HTTP/1.1" 404 434 "-" "curl/7.68.0"
127.0.0.1:80 127.0.0.1 - - [17/Dec/2022:18:54:03 +0000] "GET /cm9vdDpNb3RkZVBhc3NlVHJvcExvbmdQb3VyRXRyZUJydXRlRm9yY2VQYXJVbkRlYmlsZQ== HTTP/1.1" 404 434 "-" "curl/7.68.0"
127.0.0.1:80 127.0.0.1 - - [17/Dec/2022:18:55:03 +0000] "GET /cm9vdDpNb3RkZVBhc3NlVHJvcExvbmdQb3VyRXRyZUJydXRlRm9yY2VQYXJVbkRlYmlsZQ== HTTP/1.1" 404 434 "-" "curl/7.68.0"
127.0.0.1:80 127.0.0.1 - - [17/Dec/2022:18:56:01 +0000] "GET /cm9vdDpNb3RkZVBhc3NlVHJvcExvbmdQb3VyRXRyZUJydXRlRm9yY2VQYXJVbkRlYmlsZQ== HTTP/1.1" 404 434 "-" "curl/7.68.0"
jeannoobie@sup3r-s3cur3:/var/log/apache2$
```

Without waiting no more, let's feed it to CyberChef !<br>

<figure><img src="../../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

Yup, that looks nasty guys, let's get it over with ! :

```
jeannoobie@sup3r-s3cur3:/var/log/apache2$ su - root
Password: #Type the password obtained here ;)
root@sup3r-s3cur3:~#
root@sup3r-s3cur3:~# ll
ll: command not found
root@sup3r-s3cur3:~# ls -l
total 12
-r-x------ 1 root root  131 Nov 27 17:12 rabbitHole.sh
-rw-r--r-- 1 root root   31 Nov 27 17:18 root.txt
drwx------ 4 root root 4096 Nov 27 11:42 snap
root@sup3r-s3cur3:~# cat root.txt 
n***{*********************}
root@sup3r-s3cur3:~#
```

<figure><img src="../../.gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>

### Conclusion :

That was an easy and funny box to complete, thanks to @Noobosaurus\_r3x for making and publishing it on THM ;) You can joing his Discord server, it's a real gold-mine (only in French of course :D) :&#x20;

[https://discord.gg/BXakAbF5<br>](https://discord.gg/BXakAbF5)\
Thank you for reading, and have a nice day !

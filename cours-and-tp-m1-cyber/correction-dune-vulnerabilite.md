---
description: >-
  Correction d'une vulnérabilité sur un plugin Wordpress, et renforcement de la
  configuration du site
---

# 🔐 Correction d'une vulnérabilité

### Installation de la VM :&#x20;

Nous partons sur une VM de 32 Go d'espace totale. Voici les points de montage LVM à monter :

```
/; = 5 Go 
/var; = 10 Go
/var/log; = 2 Go
/var/tmp; = 5 Go
/tmp; = 5 Go
/home; = 2 Go
/srv/www; = 2 Go
```

Pour /var/log et /srv/www, bien attention à ajouter les options de montage suivantes :

* nosuid : ignore complètement les bits setuid et setgid
* nodev : ignore les fichiers de périphériques
* noexec : interdit l'exécution de tout programme sur ce point de montage

Après le partitionnement :

```bash
root@ubuntu-server:/etc/apache2/sites-available# df -h
Filesystem                   Size  Used Avail Use% Mounted on
tmpfs                        867M  1.6M  866M   1% /run
/dev/mapper/myVolume-ROOT    7.8G  4.8G  2.7G  65% /
tmpfs                        4.3G     0  4.3G   0% /dev/shm
tmpfs                        5.0M  4.0K  5.0M   1% /run/lock
/dev/mapper/myVolume-TMP     4.9G  740K  4.6G   1% /tmp
/dev/mapper/myVolume-HOME    2.0G  226M  1.6G  13% /home
/dev/mapper/myVolume-VAR     9.8G  1.5G  7.8G  17% /var
/dev/mapper/myVolume-lv--0  2.0G   27M  1.8G   2% /srv/www
/dev/mapper/myVolume-VAR--LOG
                             2.0G   37M  1.8G   3% /var/log
/dev/mapper/myVolume-VAR--TMP
                             4.9G  112K  4.6G   1% /var/tmp
tmpfs                        867M  140K  867M   1% /run/user/1000

root@ubuntu-server:/etc/apache2/sites-available# lvdisplay
  --- Logical volume ---
  LV Path                /dev/myVolume/ROOT
  LV Name                ROOT
  VG Name                myVolume
  LV UUID                7XfPcV-U97H-Mfmy-aI5O-4e2g-AjFM-tLX2gC
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2023-03-14 21:47:02 +0000
  LV Status              available
  # open                 1
  LV Size                8.00 GiB
  Current LE             2048
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0

  --- Logical volume ---
  LV Path                /dev/myVolume/VAR
  LV Name                VAR
  VG Name                myVolume
  LV UUID                LS2BOd-1qal-g0iW-ouvv-TBVz-cfcV-cgAVU3
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2023-03-14 21:47:02 +0000
  LV Status              available
  # open                 1
  LV Size                10.00 GiB
  Current LE             2560
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:1

  --- Logical volume ---
  LV Path                /dev/myVolume/VAR-LOG
  LV Name                VAR-LOG
  VG Name                myVolume
  LV UUID                5BjYH3-u3yJ-2VlY-xO7

[...]

```

Les partitions sont alors complètes et prêtes à l'emploi.

### Installation du serveur apache :

A l'aide du tutoriel :

```bash
sudo - 

apt install software-properties-common 

add-apt-repository ppa:ondrej/php 

apt update 

apt install apache2 

apt install php5.6 php5.6-fpm a2enmod proxy_fcgi setenvif a2enconf php5.6-fpm 

apt install php5.6-mysql php5.6-mbstring php5.6-xml php5.6-gd php5.6- curl 

apt install mariadb-server mariadb-client apt-get install unzip
```

On configure ensuite le wordpress :

```bash
cd /tmp/ 

wget https://wordpress.org/wordpress-4.0.7.zip && unzip wordpress-4.0.7.zip 

sudo mv wordpress /srv/www/ && cd /srv/www/ && chown -R root:www-data wordpress/
```

Puis dans le dossier de configuration d'apache :

```php
<VirtualHost *:80>
DocumentRoot /srv/www/wordpress
 <Directory /srv/www/wordpress>
	Options FollowSymLinks 
	AllowOverride Limit Options FileInfo 
	DirectoryIndex index.php 
	Require all granted
 </Directory>
 <Directory /srv/www/wordpress/wp-content>
	Options FollowSymLinks
	Require all granted
 </Directory>
</VirtualHost>
```

Reste maintenant à activer le module Wordpress :

```bash
a2ensite wordpress 

a2enmod rewrite 

a2dissite 000-default service 

apache2 reload
```

Il faut maintenant configurer la base de données :

```bash
mysql -u root -p
```

```sql
CREATE DATABASE wordpress_db; 

CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'password'; 

GRANT ALL ON wordpress_db.* TO 'wp_user'@'localhost' IDENTIFIED BY 'password'; 

FLUSH PRIVILEGES; Exit
```

Permettre l'installation du plugin :

```bash
vim php.ini :

sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 10M/g' /etc/php/5.6/fpm/php.ini 

echo "define('FS_METHOD', 'direct');" >> /srv/www/wordpress/wp-config.php 

systemctl restart php5.6-fpm.service && systemctl restart apache2.service
```

Restreindre l'execution de code php dans le répertoire "/uploads" :

```bash
root@ubuntu-server:/srv/www/wordpress/wp-content# sudo chmod -R 600 uploads/
root@ubuntu-server:/srv/www/wordpress/wp-content# ll
total 28
drwxrwx--- 6 root www-data 4096 mars  15 00:23 ./
drwxr-x--- 5 root www-data 4096 mars  16 08:24 ../
-rwxrwx--- 1 root www-data   28 janv.  8  2012 index.php*
drwxrwx--- 4 root www-data 4096 mars  15 00:23 plugins/
drwxrwx--- 5 root www-data 4096 août   4  2015 themes/
drwxrwx--- 2 root www-data 4096 mars  15 00:23 upgrade/
drw------- 3 root www-data 4096 mars  15 00:23 uploads/
root@ubuntu-server:/srv/www/wordpress/wp-content#
```

Interdire l'exécution de code dans le panel admin de wordpress :&#x20;

<figure><img src="../.gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

Ainsi que dans Apache :&#x20;

<figure><img src="../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

Et les secrets de l'application. Les valeurs sont aléatoires mais il s'agit de test : <br>

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

Enfin, affiner les permissions :

Pour le ".htaccess" : <br>

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

/wp-content : <br>

<figure><img src="../.gitbook/assets/image (62).png" alt=""><figcaption></figcaption></figure>

wp-content/plugins/\* : <br>

<figure><img src="../.gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

\[...]

wp-content/themes/\* : <br>

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

### Patchage de la faille :&#x20;

Un contrôle tout simple d'extension nous permet de patcher la faille : <br>

<figure><img src="../.gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

Pas de reverse shell en vue, ni de fichiers en cache : <br>

<figure><img src="../.gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

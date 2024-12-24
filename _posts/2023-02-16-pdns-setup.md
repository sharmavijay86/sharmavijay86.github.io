---
layout: post
title:  "powerdns setup with recursor and pdnsmanager webui"
summary: "powerdns setup with recursor and pdnsmanager webui"
author: vijay
date: '2023-02-16 16:35:23 +0530'
category: devops
thumbnail: /assets/img/posts/devops.jpg
keywords: 
permalink: /blog/setup-powerdns-with-recursor-and-web-ui/
usemathjax: true
---

## Setup powerdns with recursor and pdnsmanager web ui   
- In ubuntu there would be systemdresolver already running on port 53, hence we will first disable that.   
```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved
systemctl mask systemd-resolved
```   
- Now open resolv file and keep entry whatever you requires.   
**Note:** This is just for information if you want to make entries static install packages ifupdown and resolvconf   

  
```bash
apt update
apt-get install pdns-server pdns-recursor pdns-backend-mysql mysql-server -y
```
- Create database and user for pdns.
```sql
create database pdns;
create user pdns@localhost identified by pdns;
create user pdns@localhost identified by 'pdns';
grant all on pdns.* to pdns@localhost;

```
- import the sql schema
```bash
mysql pdns < /usr/share/pdns-backend-mysql/schema/schema.mysql.sql 
```

- Now edit pdns config file to make required changes   
```bash
vim /etc/powerdns/pdns.conf
```   
set bellow entries

```bash
allow-axfr-ips=127.0.0.1 <ip of your secondary nameserver>
config-dir=/etc/powerdns
daemon=yes
disable-axfr=no
guardian=yes
local-address=0.0.0.0
local-port=54
master=yes
slave=yes
module-dir=/usr/lib/x86_64-linux-gnu/pdns
setgid=pdns
setuid=pdns
socket-dir=/var/run
version-string=powerdns
include-dir=/etc/powerdns/pdns.d
```   
save and exit   
Lets validate if configuration is correct.
```bash
pdns_server --daemon=no --guardian=no --loglevel=9
```

- Now we will setup powerdns recursor   

```bash
vim /etc/powerdns/recursor.conf

forward-zones=mylab.local=127.0.0.1:54
forward-zones-recurse=.=1.1.1.1,.=8.8.8.8
local-address=0.0.0.0
local-port=53
```   
you can use your own forwarders in above config instead 1.1.1.1 and 8.8.8.8
edit mysql for pdns now   
```bash
vim /etc/powerdns/pdns.d/pdns.local.gmysql.conf 

launch=gmysql

gmysql-host=localhost
gmysql-port=3306
gmysql-dbname=pdns
gmysql-user=pdns
gmysql-password=pdns
gmysql-dnssec=no
```   
save and exit   
- Lets validate if configuration is correct.
```bash
pdns_server --daemon=no --guardian=no --loglevel=9
```
- Restart services    
```bash
systemctl restart pdns
systemctl restart pdns-recursor
```    
Now we will create and setup mysql database for dns records    

- create database and user   
```sql
create database powerdns;
grant all on powerdns.* to 'pdns'@'localhost' identified by 'secret';
```    
#### Setting up pdnsmanager    

Install required php and apache packages

```bash
apt install php php-apcu php-mysql apache2 -y
```   
Enable apache modules   
```bash
a2enmod rewrite 
a2enmod ssl
```   
Setup virtual host in apache   

```bash
vim /etc/apache2/sites-enabled/default.conf

<VirtualHost _default_:443>
    ServerAdmin webmaster@localhost

    ServerName pdns.example.com

    DocumentRoot /var/www/html/frontend

    RewriteEngine On
    RewriteRule ^index\.html$ - [L]
    RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-f
    RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-d
    RewriteRule !^/api/\.* /index.html [L]

    Alias /api /var/www/html/backend/public
    <Directory /var/www/html/backend/public>
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule ^ index.php [QSA,L]
    </Directory>

</VirtualHost>
```   

Make changes in above as per your need ( keep ssl settingsi have not included them )   
Now get the webui package download , untar and set

```bash
wget https://dl.pdnsmanager.org/pdnsmanager-2.0.1.tar.gz
tar -xvf pdnsmanager-2.0.1.tar.gz
cd pdnsmanager-2.0.1
mv backend /var/www/html/
mv frontend /var/www/html/
chown -R www-data:www-data /var/www/html 
```   

access url now with /setup
use same database entry as given above in mysql

Done enjoy!   

#### replication on master slave for powerdns   

powerdns uses mysql as backend in above example hence do all settings on slave server just like above except mysql database. Instead folow guide to [perform mysql database replication](/blog/easy-way-of-mysql-multimaster-setup/)

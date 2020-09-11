## Setup powerdns with recursor and pdnsmanager web ui   
In ubuntu there would be systemdresolver already running on port 53, hence we will first disable that.   
```
root@pdns:~# systemctl stop systemd-resolved
root@pdns:~# systemctl disable systemd-resolved
root@pdns:~# systemctl mask systemd-resolved
```   
Now open resolv file and keep entry whatever you requires.   
Note : This is just for information if you want to make entries static install packages ifupdown and resolvconf   

```
vim /etc/resolv.conf

echo "deb http://repo.powerdns.com/ubuntu bionic-auth-41 main" >> /etc/apt/sources.list.d/powerdns.list

```   
enable repository   
```
vim /etc/apt/preferences.d/powerdns
Package: pdns-*
Pin: origin repo.powerdns.com
Pin-Priority: 600

curl -s https://repo.powerdns.com/FD380FBB-pub.asc | sudo apt-key add -
```   
Now install packages   
```
apt update
apt-get install pdns-server pdns-recursor pdns-backend-mysql mysql-server -y
```

Once packages get installed edit pdns config file to make required changes   
```
vim /etc/powerdns/pdns.conf
```   
set bellow entries

```
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
Now we will setup powerdns recursor   

```
vim /etc/powerdns/recursor.conf

forward-zones=mylab.local=127.0.0.1:54
forward-zones-recurse=.=1.1.1.1,.=8.8.8.8
local-address=0.0.0.0
local-port=53
```   
you can use your own forwarders in above config instead 1.1.1.1 and 8.8.8.8
edit mysql for pdns now   
```
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
restart services    
```
systemctl restart pdns
systemctl restart pdns-recursor
```    
Now we will create and setup mysql database for dns records    
Lets tune some mysql entries for more read request    
```
vim /etc/mysql/mysql.conf.d/mysqld.cnf


--- InnoDB section
innodb_log_file_size = 64M
default-storage-engine=INNODB
innodb_buffer_pool_size=1G
innodb_buffer_pool_instances = 2
innodb_autoinc_lock_mode = 2
innodb_doublewrite = 1
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit = 2
innodb_lock_wait_timeout = 60
innodb_locks_unsafe_for_binlog = 1
innodb_stats_on_metadata = 0
transaction-isolation=READ-COMMITTED

service mysql restart
```    
create database and user   
```
mysql> create database powerdns;
mysql> grant all on powerdns.* to 'pdns'@'localhost' identified by 'secret';
```    
#### Setting up pdnsmanager    

Install required php and apache packages

```
apt install php php-apcu php-mysql apache2 -y
```   
Enable apache modules   
```
a2enmod rewrite 
a2enmod ssl
```   
Setup virtual host in apache   

```
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

```
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

powerdns uses mysql as backend in above example hence do all settings on slave server just like above except mysql database. Instead folow guide to [perform mysql database replication](mysqlrep.md)

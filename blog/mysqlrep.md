## Mysql 5.7 database replication    
#### Steps to perform on Master    
Edit conf file    
```
vim /etc/mysql/mysql.conf.d/mysqld.cnf
```   
make bellow changes   
```
bind-address            = 192.168.0.161
server-id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log
```    
login to mysql and run query    
```
create user 'slave'@'192.168.0.68' identified by 'redhat123';
grant replication slave on *.* to 'slave'@'192.168.0.68';
flush privileges;
FLUSH TABLES WITH READ LOCK;
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      771 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```   
take complete database backup to copy on slave server    
```
mysqldump --all-databases --master-data > data.sql
```    
transfer backup dump to slave server    
```
scp data.sql ubuntu@192.168.0.68:~/
```   
Now unlock tables here in master mysql   
```
UNLOCK TABLES;
```

#### Steps to run on slave    
Edit file and make changes as bellow    
```
vim /etc/mysql/mysql.conf.d/mysqld.cnf

bind-address            = 192.168.0.68
server-id               = 2
log_bin                 = /var/log/mysql/mysql-bin.log
```    
restore database dump from master   
```
mysql  < /home/ubuntu/data.sql
```   
Now login to mysql  and run query    
```
stop slave;
change master to master_host='192.168.0.161',master_user='slave',master_password='redhat123',master_log_file='mysql-bin.000001',master_log_pos=771;
start slave;
SHOW SLAVE STATUS \G;
```

Done!   If required restart mysql service.    
```
systemctl restart mysql
```

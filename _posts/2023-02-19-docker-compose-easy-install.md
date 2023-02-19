---
layout: post
title:  "how to setup docker compose with example"
summary: "how to setup docker compose with example"
author: vijayk
date: '2023-02-19 08:35:23 +0530'
category: k8s
thumbnail: /assets/img/posts/kubernetes3.jpg
keywords: 
permalink: /blog/docker-compose-easy-setup/
usemathjax: true
---


# Docker compose install  
- Intallation of docker-compose can be done via binary download. Follow these commands-
```python
wget https://github.com/docker/compose/releases/download/v2.13.0/docker-compose-linux-x86_64

sudo mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
- You can validate the installation by running bellow command
```python
docker-compose version
```


- Sample docker-compose yaml file-
```python
services:
  db:
    # We use a mariadb image which supports both amd64 & arm64 architecture
    image: mariadb:10.6.4-focal
    # If you really want to use MySQL, uncomment the following line
    #image: mysql:8.0.27
    command: '--default-authentication-plugin=mysql_native_password'
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=somewordpress
      - MYSQL_DATABASE=wordpress
      - MYSQL_USER=wordpress
      - MYSQL_PASSWORD=wordpress
    expose:
      - 3306
      - 33060
  wordpress:
    image: wordpress:latest
    ports:
      - 80:80
    restart: always
    environment:
      - WORDPRESS_DB_HOST=db
      - WORDPRESS_DB_USER=wordpress
      - WORDPRESS_DB_PASSWORD=wordpress
      - WORDPRESS_DB_NAME=wordpress
volumes:
  db_data:
```

- To run the stack -
```python
docker-compose up -d 
```
- to start , stop  & restart the stack-

```python
docker-compose stop

docker-compose start

docker-compose restart
```

- To remove the stack 
```python
docker-compose down
```

- To prune the volumes
```python
docker volume prune
```
 - Example Mysql with Adminer
```python
version: '3.7'
services:
  mysql_db_container:
    image: mysql:latest
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
    ports:
      - 3306:3306
    volumes:
      - mysql_db_data_container:/var/lib/mysql
  adminer_container:
    image: adminer:latest
    environment:
      ADMINER_DEFAULT_SERVER: mysql_db_container
    ports:
      - 8080:8080

volumes:
  mysql_db_data_container:
```

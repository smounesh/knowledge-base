## Seafile-Selhosted fileserver(alternative to dropbox,google-drive)

1. Seafile is a fileserver written in C and php and latest 9.0 version is written in go self-hosted alternative to google drive.

2. It performs better than nextcloud and handles large file uploads well.

 3. Seafile supports multiorganization, multitenant and cluster deployment

### docker-compose deployment

1. docker-compose file to run seafile, memcache and mariadb

```yaml
# Filename: /srv/seaile/docker-compose.yml
# Purpose: To start seafile along with memcache and mariadb
# Url: https://drive.smounesh.in

version: "2.4"
services:
  seafile:
    image: seafileltd/seafile-mc:latest
    container_name: drive.smounesh.in
    restart: always

    mem_limit: 2G

    environment:
      - DB_HOST=mariadb
      - DB_ROOT_PASSWD=${MYSQL_ROOT_PASSWORD}
      - TIME_ZONE=Asia/Kolkata
      - SEAFILE_SERVER_LETSENCRYPT=false
      - SEAFILE_SERVER_HOSTNAME=${SEAFILE_SERVER_HOSTNAME}
      - SEAFILE_ADMIN_EMAIL=${SEAFILE_ADMIN_EMAIL}
      - SEAFILE_ADMIN_PASSWORD=${SEAFILE_ADMIN_PASSWORD}

    volumes:
      - ./conf:/shared/seafile
      - ./logs:/shared/logs
      - ./nginx:/shared/nginx

    labels:
      - com.centurylinklabs.watchtower.enable=true
      - traefik.enable=true
      - traefik.http.routers.seafile.rule=Host(`drive.smounesh.in`)
      - traefik.http.routers.seafile.tls=true
      - traefik.http.routers.seafile.tls.certresolver=lets-encrypt
      #- traefik.http.routers.seafile.service=seafile-svc
      #- traefik.http.services.seafile-svc.loadbalancer.server.port=80

    depends_on:
       - mariadb

  memcached:
    image: memcached:1.6
    container_name: memcache-drive.smounesh.in
    restart: always

    mem_limit: 2G
    entrypoint: memcached -m 256

    labels:
      - com.centurylinklabs.watchtower.enable=true

    depends_on:
       - mariadb

  mariadb:
    image: mariadb:10.5
    container_name: mariadb-drive.smounesh.in
    restart: always

    mem_limit: 2G

    environment:
      -  MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}

    volumes:
      - ./database:/var/lib/mysql

    labels:
      - com.centurylinklabs.watchtower.enable=true

    healthcheck:
      test: ["CMD", "mysqladmin" ,"ping", "-h", "localhost"]
      interval: 5s
      timeout: 180s
      retries: 3
```


 2. Envfile to store credentials
```sh
SEAFILE_SERVER_HOSTNAME=drive.smounesh.in
SEAFILE_ADMIN_EMAIL=alita@smounesh.in
SEAFILE_ADMIN_PASSWORD=alita@2019

MYSQL_ROOT_PASSWORD=root_pass
```
 
### Reference: 
 - [Seafile deployment with docker(manual.seafile.com)](https://manual.seafile.com/docker/deploy_seafile_with_docker)

### Todo:
  1 . Solve access denied for user 'root'@'localhost' (using password: no) error on mariadb
 2. Setup google sso for user login authentication and authorization





## Learn sql with dvd_rental database with adminer, mysql and postgres

 1. The below docker-compose file wil spin up postgres and mysql with dvd rental database and adminer services
 2. Mysql with bind mounts will not work in all environments like windows so create it using named volumes to solve the file permission overhead for beginners.

```yaml
# Filename: docker-compose.yml
# Purpose: A postgres with dvd-rental database and a mysql with sakila database with adminer sql web-client to learn sql queries

# Connect postgresql on adminer using:
#   server: postgresql, username: dvdrental, password: dvdrental, database: dvdrental

# Connect mysql on adminer using:
#   server: mysql, username: sakila, password: p_ssW0rd, database: sakila

# run these services using the cmd in powershell or cmd:
#   docker-compose up -d

# stop these  services using the cmd in powershell or cmd:
#   docker-compose down

version: "2.4"
services:
 adminer:
   image: docker.bitsathy.ac.in/shankarmounesh.cs20/adminer:latest
   container_name: adminer
   restart: always
   ports:
    - 80:8080

 postgresql:
   image: docker.bitsathy.ac.in/shankarmounesh.cs20/dvdrental:postgres-15
   container_name: postgres
   restart: always

   ports:
    - 5432:5432

   volumes:
     - ./postgres-data:/var/lib/postgresql/data

  
#  mysql:
#    image: docker.bitsathy.ac.in/shankarmounesh.cs20/sakiladb:mysql-8
#    container_name: mysql
#    restart: always

#    ports:
#     - 3306:3306

#    volumes:
#      - mysql-data:/var/lib/mysql

# volumes:
#   mysql-data:
```
### References:

 1.  [PostgreSQL Sample Database with (postgresqltutorial.com)](https://www.postgresqltutorial.com/postgresql-getting-started/postgresql-sample-database/)
 2.  [Build the postgres image with dvd rental database from scratch(github.com)](https://github.com/kedziorski/test-db-pg-dvdrental)
 3.  [Mysql image with sakila example database(github.com)](https://github.com/sakiladb/mysql)





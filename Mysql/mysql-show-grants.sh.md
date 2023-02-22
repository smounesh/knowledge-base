# MySQL Show Grants in SQL Syntax

This script shows mysql all the users and their privileges as SQL grant statements. The output can be backed up and run on a new MySQL server to recreate all the users and their privileges.

```bash
#!/bin/bash

# Filename: mysql-show-grants.sh
# Shows all MySQL users and their grant privileges in SQL format
# The output can be sourced by the mysql client to recreate all user permissions on a different server
#
# On Ubuntu to remove mysql system users from the output use:
# mysql-show-grants.sh | grep -vE '(mysql.sys|mysql.sesssion|debian-sys-maint)'

# Set MySQL root user and password if required
#export MYSQL_USER="-u root -pPASSWORD"
export MYSQL_USER=""
export MYSQL="/usr/bin/mysql"

${MYSQL} ${MYSQL_USER} --skip-column-names -A \
    -e"SELECT CONCAT('SHOW GRANTS FOR ''',user,'''@''',host,''';') FROM mysql.user WHERE user<>''" | \
        ${MYSQL} ${MYSQL_USER} --skip-column-names -A | \
        sed 's/$/;/g'
```
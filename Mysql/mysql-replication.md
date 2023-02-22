# Ubuntu 20.04 MySQL 8.0 (not MariaDB!)

Ubuntu 20.04 MySQL 8.0 master/slave replication setup instructions.

1. master: `172.16.0.211
1. slave:  `172.16.0.212

## Master

```txt
/etc/mysql/mysql.conf.d/mysqld.cnf
    server-id               = 1
```

```sql
mysql> CREATE USER 'replication_user'@'172.16.0.212' IDENTIFIED WITH mysql_native_password BY '546b395b2cee57a5dcd';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'172.16.0.212';
```

```sh
-- latest version syntax
mysqldump --all-databases --routines --events --apply-replica-statements --delete-source-logs --single-transaction > /tmp/mysqldump.sql

-- old version syntax
mysqldump --all-databases --routines --events --apply-slave-statements --delete-master-logs --single-transaction > /tmp/mysqldump.sql
```

## Slave

```txt
/etc/mysql/mysql.conf.d/mysqld.cnf
    server-id               = 2
```

```sql
mysql> CHANGE MASTER TO MASTER_HOST='172.16.0.211', MASTER_USER='replication_user', MASTER_PASSWORD='546b395b2cee57a5dcd';
```

```sh
mysql < /tmp/mysqldump.sql
```

```sql
mysql> reset slave; -- to make slave forget last known replication position
mysql> start slave;
mysql> show slave status\G      -- Ensure Seconds_Behind_Master is 0
```


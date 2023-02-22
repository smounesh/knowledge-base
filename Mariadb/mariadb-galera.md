# MariaDB Galera Cluster on Ubuntu 22.04

## Required Deb Packages

`apt install mariadb-server`

 to purge mariadb along with its data `apt-get remove --purge *mysql\*`
```txt
root@db1:/etc/mysql/mariadb.conf.d$ dpkg -l mariadb\* | grep ^ii
ii  mariadb-client-10.6        1:10.6.11-0ubuntu0.22.04.1 arm64        MariaDB database client binaries
ii  mariadb-client-core-10.6   1:10.6.11-0ubuntu0.22.04.1 arm64        MariaDB database core client binaries
ii  mariadb-common             1:10.6.11-0ubuntu0.22.04.1 all          MariaDB common configuration files
ii  mariadb-server             1:10.6.11-0ubuntu0.22.04.1 all          MariaDB database server (metapackage depending on the latest version)
ii  mariadb-server-10.6        1:10.6.11-0ubuntu0.22.04.1 arm64        MariaDB database server binaries
ii  mariadb-server-core-10.6   1:10.6.11-0ubuntu0.22.04.1 arm64        MariaDB database core server files
```

## Prerequisites

| hostname | IP            |
| -------- | ------------- |
| db1      | 172.31.10.175 |
| db2      | 172.31.10.159 | 

Add static `/etc/host` entries are present for `db1` and `db2` on both the servers. Do not depend on DNS as it adds unnecessary dependencies.

## MariaDB Galera Config Fragment for db1

```txt
# Filename: /etc/mysql/mariadb.conf.d/zz-galera.cnf
# Local galera cluster config
#

[galera]
wsrep_on                 = ON
wsrep_provider           = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name       = "Galera-Cluster"

# Ensure /etc/host entires are present for db1, db2 
wsrep_cluster_address    = "gcomm://db1,db2"
wsrep_sst_method         = rsync

binlog_format            = row
default_storage_engine   = InnoDB
innodb_autoinc_lock_mode = 2

# Allow server to accept connections on all interfaces.
bind-address = 0.0.0.0

# Galera Node Configuration
wsrep_node_address="172.31.10.175"
wsrep_node_name="db1"
```

## MariaDB Galera Config Fragment for db2

```
# Filename: /etc/mysql/mariadb.conf.d/zz-galera.cnf
# Local galera cluster config
#

[galera]
wsrep_on                 = ON
wsrep_provider           = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name       = "Galera-Cluster"

# Ensure /etc/host entires are present for db1, db2 
wsrep_cluster_address    = "gcomm://db1,db2"
wsrep_sst_method         = rsync

binlog_format            = row
default_storage_engine   = InnoDB
innodb_autoinc_lock_mode = 2

# Allow server to accept connections on all interfaces.
bind-address = 0.0.0.0

# Galera Node Configuration
wsrep_node_address="172.31.10.159"
wsrep_node_name="db2"
```

## Starting the Cluster

1. Stop all mariadb instances on db1 and db2 `systemctl stop mariadb.service`.
2. On db1, run  `sudo galera_new_cluster` and check `mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'"`.
3. On db2 start MariaDB normally `systemctl start mariadb.service`.
4. After ensuring db1 and db2 are functioning normally, we can now start mariadb normally on db1. `killall mariadbd; systemctl start mariadb`.
5. To check if galera is running normally, run the below command to confirm the cluster size.

```txt
mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size'"

+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 2     |
+--------------------+-------+
```

## References
1. Galera State Transfers: https://mariadb.com/kb/en/introduction-to-state-snapshot-transfers-ssts/
2. How to: https://www.digitalocean.com/community/tutorials/how-to-configure-a-galera-cluster-with-mariadb-on-ubuntu-18-04-servers
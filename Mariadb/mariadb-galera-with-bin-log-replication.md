# Combining MariaDB Galera with Bin Log Replication

MariaDB Galera offers sync replication and therefore the performance of a galera cluster is that of the slowest node. Also Galera introduces transaction latency as to avoid data loss, the write set replication mechanism waits for other cluster nodes to also commit the transaction before acknowledging. Galera therefore requires low latency and high bandwidth connection between the cluster nodes.

For disaster recovery scenarios, having an off-site MariaDB replica using traditional bin log shipping is the only solution.

## Architecture

This setup works in tandem with Galera cluster and a single DB host is configured to function as MariaDB master node for bin log shipping.
The off-site MariaDB server acts as a bin log replication target and offers async replication.

```
Site 1: MariaDB Cluster                                           Site 2: MariaDB Bin Log Replication                  
 +----------------------------------------------------+           +----------------------------------------------------+
 | +-----------------------------------------------+  |           | +-----------------------------------------------+  |
 | | db1:                                          | ==============>| db3:                                          |  |
 | |  1. galera ws_replication zz-galera.cnf       |  |           | |  1. bin log replication zz-binlog-slave.cnf   |  |
 | |  2. bin log replication zz-binlog-master.cnf  |  |           | |                                               |  |
 | +-----------------------------------------------+  |           | +-----------------------------------------------+  |
 |                                                    |           |                                                    |
 | +-----------------------------------------------+  |           |                                                    |
 | | db2:                                          |  |           |                                                    |
 | |  1. galera ws_replication zz-galera.cnf       |  |           |                                                    |
 | |                                               |  |           |                                                    |
 | +-----------------------------------------------+  |           |                                                    |
 +----------------------------------------------------+           +----------------------------------------------------+

ASCII art created using: https://textik.com
```

## Steps
1. MariaDB config files as specified below.
2. After that, bin log replication setup is similar to [[mysql-replication]].

## DB1 MariaDB Config

```txt
# Filename: /etc/mysql/mariadb.conf.d/zz-binlog-master.cnf
# MariaDB log shipping replication setup
# This setup coexists with Galera cluster
#  - db1 and db2 are part of galera cluster within a single DC
#  - db1 is master for log shipping replication to offsite MariaDB slave

[mysqld]
server-id         = 1
log_bin           = /var/log/mysql/mysql-bin.log
expire_logs_days  = 4
max_binlog_size   = 100M
log_slave_updates = 1
```

## DB3 MariaDB Config

```
# Filename: /etc/mysql/mariadb.conf.d/zz-binlog-slave.cnf

[mysqld]
server-id = 3
```

## Caveats and Limitations

 - Galera SST over rsync copies the entire DB files from a donor node to the target node. This does not generate any binary logs.
 - When such SST event occurs in the bin log master node, the slave will not have those DB updates and replication will fail.
 - This is a rare event, but must be considered in the design and monitoring tools. See [[mysql-replication-check.sh]].

## References

1. Galera State Transfers: https://mariadb.com/kb/en/introduction-to-state-snapshot-transfers-ssts/
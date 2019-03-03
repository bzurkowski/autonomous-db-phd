# ProxySQL setup

## Proxy VM setup

Create security group:

```bash
$ openstack security group create proxysql && \
  openstack security group rule create --proto icmp proxysql && \
  openstack security group rule create --proto tcp --dst-port 22 proxysql && \
  openstack security group rule create --proto tcp --dst-port 3306 proxysql && \
  openstack security group rule create --proto tcp --dst-port 6032 proxysql && \
  openstack security group rule create --proto tcp --dst-port 6033 proxysql
```

Spawn VM:

```bash
$ network_id=$(openstack network list |grep internal |awk '{print $2}')
$ openstack server create \
    --image ubuntu_mariadb \
    --flavor test.large \
    --security-group proxysql \
    --nic net-id=$network_id \
    --user-data /etc/kolla/trove-taskmanager/cloudinit/proxysql.cloudinit \
    proxysql
```

## Installation

Add YUM repository:

```bash
$ cat <<EOF | tee /etc/yum.repos.d/proxysql.repo
[proxysql_repo]
name= ProxySQL YUM repository
baseurl=http://repo.proxysql.com/ProxySQL/proxysql-2.0.x/centos/\$releasever
gpgcheck=1
gpgkey=http://repo.proxysql.com/ProxySQL/repo_pub_key
EOF
```

Install packages:

```bash
$ yum install proxysql
```

Start proxy service:

```bash
$ systemctl start proxysql
$ systemctl enable proxysql
```

## Configuration

Configuration based on ProxySQL guide: https://proxysql.com/blog/proxysql-native-galera-support

Log into proxy VM and connect to admin interface:

```bash
$ mysql -h127.0.0.1 -P6032 -uadmin -padmin
```

Create Galera hostgroup:

* 1 - OFFLINE
* 2 - WRITES
* 3 - READS
* 4 - WRITE BACKUPS

```sql
INSERT INTO mysql_galera_hostgroups
            (offline_hostgroup,
             writer_hostgroup,
             reader_hostgroup,
             backup_writer_hostgroup,
             active,
             max_writers,
             writer_is_also_reader,
             max_transactions_behind)
          VALUES
            (1,2,3,4,1,1,2,100);
```

Add SQL backends:

```sql
INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight)
    VALUES (2,'10.10.10.172',3306,100);

INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight)
    VALUES (3,'10.10.10.169',3306,10);

INSERT INTO mysql_servers(hostgroup_id,hostname,port,weight)
    VALUES (3,'10.10.10.183',3306,100);
```

Define query rules:

```sql
INSERT INTO mysql_query_rules (active, match_digest, destination_hostgroup, apply)
    VALUES (1, '^SELECT.*',3, 0);

INSERT INTO mysql_query_rules (active, match_digest, destination_hostgroup, apply)
    VALUES (1, '^SELECT.* FOR UPDATE',2, 1);
```

Create Proxy Monitor user via Trove:

```bash
$ trove user-create 33e546d4-b2b8-476f-94cf-ee49838e80db proxymon proxymon
```

Set username and password for Proxy Monitor (user must exist on each backend):

```sql
SET mysql-monitor_username='proxymon';
SET mysql-monitor_password='proxymon';
```

For reference: https://github.com/sysown/proxysql/wiki/Monitor-Module

(Optionally, for slow setups) Increase backend connection timeouts:

```sql
SET mysql-monitor_connect_timeout=30000;
SET mysql-monitor_ping_timeout=30000;
SET mysql-monitor_read_only_timeout=30000;
SET mysql-monitor_replication_lag_timeout=30000;
SET mysql-monitor_connect_timeout=30000;

SET mysql-connect_timeout_server=30000;
SET mysql-connect_timeout_server_max=30000;
```

Create test user:

```sql
INSERT INTO mysql_users (username, password, active, default_hostgroup) VALUES ('sbtest', 'sbtest', 1, 2);
```

Load configuration into runtime:

```sql
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
LOAD SCHEDULER TO RUNTIME;
SAVE SCHEDULER TO DISK;
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

Verify configuration:

```sql
SELECT hostgroup,srv_host,status,ConnUsed,MaxConnUsed,Queries,Latency_us FROM stats.stats_mysql_connection_pool ORDER BY srv_host;
+-----------+--------------+--------+----------+-------------+---------+------------+
| hostgroup | srv_host     | status | ConnUsed | MaxConnUsed | Queries | Latency_us |
+-----------+--------------+--------+----------+-------------+---------+------------+
| 3         | 10.10.10.103 | ONLINE | 0        | 0           | 0       | 3883       |
| 4         | 10.10.10.103 | ONLINE | 0        | 0           | 0       | 3883       |
| 3         | 10.10.10.145 | ONLINE | 0        | 0           | 0       | 4603       |
| 4         | 10.10.10.145 | ONLINE | 0        | 0           | 0       | 4603       |
| 2         | 10.10.10.151 | ONLINE | 0        | 0           | 0       | 5136       |
+-----------+--------------+--------+----------+-------------+---------+------------+
```

Verify Galera status for each node:

```sql
SELECT hostname, port, time_start_us, primary_partition, read_only, wsrep_local_state FROM mysql_server_galera_log ORDER BY time_start_us DESC LIMIT 3;
+--------------+------+------------------+-------------------+-----------+-------------------+
| hostname     | port | time_start_us    | primary_partition | read_only | wsrep_local_state |
+--------------+------+------------------+-------------------+-----------+-------------------+
| 10.10.10.151 | 3306 | 1551633033901918 | YES               | NO        | 4                 |
| 10.10.10.145 | 3306 | 1551633033893493 | YES               | NO        | 4                 |
| 10.10.10.103 | 3306 | 1551633033891254 | YES               | NO        | 4                 |
+--------------+------+------------------+-------------------+-----------+-------------------+
```

Verify monitor checks:

```sql
SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 30;
+--------------+------+------------------+-------------------------+---------------+
| hostname     | port | time_start_us    | connect_success_time_us | connect_error |
+--------------+------+------------------+-------------------------+---------------+
| 10.10.10.151 | 3306 | 1551640007369753 | 10042322                | NULL          |
| 10.10.10.103 | 3306 | 1551640006778577 | 10037439                | NULL          |
| 10.10.10.103 | 3306 | 1551639956537071 | 10023157                | NULL          |
+--------------+------+------------------+-------------------------+---------------+
```

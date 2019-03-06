# Auto-scaling scenario

Spawn database cluster:

```bash
network_id=$(openstack network list |grep internal |awk '{print $2}')
    trove cluster-create sppa-test-1 mariadb 10.1.32 \
        --instance "flavor=test.large,volume=4,nic='net-id=$network_id'" \
        --instance "flavor=test.large,volume=4,nic='net-id=$network_id'" \
        --instance "flavor=test.large,volume=4,nic='net-id=$network_id'"
```

Disable DNS resolving on each node (`/etc/mysql/my.cnf`):

```bash
[mysqld]
skip-host-cache
skip-name-resolve
```

Start populating data:

```bash
export instance_ip=10.10.10.152

docker run \
  --rm=true \
  --name=sb-prepare \
  severalnines/sysbench \
  sysbench \
      --db-driver=mysql \
      --table-size=100000 \
      --tables=100 \
      --threads=1 \
      --mysql-host=10.10.10.152 \
      --mysql-port=6033 \
      --mysql-user=sbtest \
      --mysql-password=sbtest \
      --mysql-ignore-errors=all \
      --report-interval=5 \
      --debug \
      /usr/share/sysbench/oltp_read_write.lua \
      prepare
```



```bash
docker run \
  --rm=true \
  --name=sb-prepare \
  severalnines/sysbench \
  sysbench \
      --db-driver=mysql \
      --threads=1 \
      --time=3600 \
      --mysql-host=10.10.10.152 \
      --mysql-port=6033 \
      --mysql-user=sbtest \
      --mysql-password=sbtest \
      --mysql-ignore-errors=all \
      --report-interval=5 \
      --debug \
      /usr/share/sysbench/bulk_insert.lua \
      prepare
```

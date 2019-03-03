# Cheatsheet

Create Trove instance:

```bash
$ network_id=$(openstack network list |grep internal |awk '{print $2}')
$ trove create mariadb-test test.small \
    --size 1 \
    --volume_type default \
    --datastore mariadb \
    --datastore_version 10.1.32 \
    --nic net-id=$network_id \
```

Create Trove cluster:

```bash
$ network_id=$(openstack network list |grep internal |awk '{print $2}')
$ trove cluster-create galera-test mariadb 10.1.32 \
    --instance "flavor=test.small,volume=1,nic='net-id=$network_id'" \
    --instance "flavor=test.small,volume=1,nic='net-id=$network_id'" \
    --instance "flavor=test.small,volume=1,nic='net-id=$network_id'"
```

List last Monasca measurements related to disk usage on vdb device for each Trove instance:

```bash
$ instance_id=<INSTANCE_ID>
$ monasca \
      --os-username monitoring \
      --os-password monitoring \
      --os-project-name monasca_control_plane \
      measurement-list \
      --dimensions resource_id=$instance_id,device=vdb \
      --group_by resource_id \
      disk.space_used_perc \
      -10
```

List last Monasca measurements related to disk usage on vdb device for given Trove instance:

```bash
$ instance_id=<INSTANCE_ID>
$ monasca \
      --os-username monitoring \
      --os-password monitoring \
      --os-project-name monasca_control_plane \
      measurement-list \
      --dimensions resource_id=$instance_id,device=vdb \
      disk.space_used_perc \
      -10
```

Define disk usage alert for Trove instances:

```bash
$ monasca \
    --os-username monitoring \
    --os-password monitoring \
    --os-project-name monasca_control_plane \
    alarm-definition-create \
    --match-by resource_id \
    trove-instance-low-disk-space \
    "disk.space_used_perc{device=vdb} > 20"
```

List Monasca alarms:

```bash
$ monasca \
    --os-username monitoring \
    --os-password monitoring \
    --os-project-name monasca_control_plane \
    alarm-list \
    |grep trove-instance-low-disk-space \
    |awk '{print $6,"\t",$15}'
```

Delete Monasca alarm definition:

```bash
$ alarm_id=$(monasca \
                --os-username monitoring \
                --os-password monitoring \
                --os-project-name monasca_control_plane \
                alarm-definition-list \
                |grep trove-instance-low-disk-space \
                |awk '{print $4}')

$ monasca \
    --os-username monitoring \
    --os-password monitoring \
    --os-project-name monasca_control_plane \
    alarm-definition-delete \
    $alarm_id
```

Create Sysbench database and user:

```bash
$ instance_id=<INSTANCE_ID>
$ trove database-create $instance_id sbtest
$ trove user-create $instance_id sbtest sbtest --databases sbtest
```

Create Sysbench data set:

```bash
$ instance_ip=<INSTANCE_IP>
$ docker run \
      --rm=true \
      --name=sb-prepare \
      severalnines/sysbench \
      sysbench \
          --db-driver=mysql \
          --table-size=100000 \
          --tables=10 \
          --threads=1 \
          --mysql-host=$instance_ip \
          --mysql-port=3306 \
          --mysql-user=sbtest \
          --mysql-password=sbtest \
          /usr/share/sysbench/oltp_read_write.lua \
          prepare
```

Flavors

```bash
$ openstack flavor create -id 6 --ram 2048 --disk 5 --vcpus 2 --public test.small
$ openstack flavor create -id 6 --ram 4096 --disk 5 --vcpus 4 --public test.medium
$ openstack flavor create -id 6 --ram 8192 --disk 10 --vcpus 8 --public test.large


+----+-------------+-------+------+-----------+-------+-----------+
| ID | Name        |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+-------------+-------+------+-----------+-------+-----------+
| 6  | test.small  |  2048 |    5 |         0 |     1 | True      |
| 7  | test.medium |  4096 |    5 |         0 |     4 | True      |
| 8  | test.large  |  8192 |   10 |         0 |     8 | True      |
+----+-------------+-------+------+-----------+-------+-----------+
```

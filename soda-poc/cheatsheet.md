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

List last Monasca measurements related to disk usage on given Trove instance:

```bash
$ instance_id=<INSTANCE_ID>
$ monasca \
      --os-username monitoring \
      --os-password monitoring \
      --os-project-name monasca_control_plane \
      measurement-list \
      --dimensions resource_id=$instance_id \
      --group_by resource_id \
      disk.space_used_perc{device=vdb} \
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
      "disk.space_used_perc{device=vdb} > 80"
```

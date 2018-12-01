# Cheatsheet

Create server:

```bash
$ network_id=$(openstack network list |grep internal |awk '{print $2}')
$ openstack sample-server create \
      --image cirros \
      --flavor test.small \
      --key-name host-key \
      --network $network_id \
      demo1
```

# Multinode OpenStack deployment with Kolla Ansible

## Prepare all nodes

Install dependencies and base packages:

```bash
$ yum install -y python-devel \
                 libffi-devel \
                 gcc \
                 openssl-devel \
                 libselinux-python

$ yum install -y net-tools telnet wget htop bridge-utils
```

Install Python and upgrade pip:

```bash
$ yum install -y epel-release
$ yum install -y python-pip
$ pip install -U pip
```

Disable firewall:

```
$ systemctl stop firewalld
$ systemctl disable firewalld
```

Disable selinux:

```
$ sed -i -r "s,^SELINUX=.+$,SELINUX=permissive," /etc/selinux/config
$ setenforce permissive
```

Install Docker:

```bash
$ yum install -y yum-utils device-mapper-persistent-data lvm2
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ yum install -y docker-ce
```

Create Docker drop-in unit file:

```bash
$ mkdir -p /etc/systemd/system/docker.service.d

$ cat > /etc/systemd/system/docker.service.d/kolla.conf <<EOF
[Service]
MountFlags=shared
EOF
```

Reload systemd daemon and enable Docker:

```bash
$ systemctl daemon-reload
$ systemctl start docker
$ systemctl enable docker
```

Verify installation:

```bash
$ docker run hello-world
```

Generate SSH keys:

```bash
$ ssh-keygen -t rsa -b 4096 -C "operator@soda-os-poc"
```

## Prepare deployment node

Populate `/etc/hosts` file, e.g.:

```bash
192.168.100.233 osc1
192.168.100.222 comp1
192.168.100.231 comp2
```

Distribute public key to OSC and compute nodes:

```bash
$ ssh-copy-id osc1
$ ssh-copy-id comp1
$ ssh-copy-id comp2
```

Install Ansible:

```bash
$ yum install -y ansible
$ pip install -U ansible
```

Configure Ansible in `/etc/ansible/ansible.cfg`:

```bash
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

Clone Kolla Ansible repository:

```bash
$ git clone https://github.com/soda-research/kolla-ansible.git
$ git checkout soda-poc
```

Setup virtualenv:

```bash
$ pip install virtualenv
$ mkdir ~/.venvs
$ virtualenv ~/.venvs/kolla
$ source ~/.venvs/kolla/bin/activate
```

Install Kolla Ansible:

```bash
$ pip install ./kolla-ansible
```

Copy sample Kolla configuration and inventory:

```bash
$ cp -r /root/.venvs/kolla/share/kolla-ansible/etc_examples/kolla /etc/
$ cp /root/.venvs/kolla/share/kolla-ansible/ansible/inventory/* .
```

Clone repository with custom deployment configuration:

```bash
$ git clone https://github.com/soda-research/soda-poc.git
```

Copy custom configuration to Kolla configuration directory:

```bash
$ cp -r ./soda-poc/kolla/etc/kolla/config /etc/kolla/
$ cp ./soda-poc/kolla/etc/kolla/globals.yml /etc/kolla/
```

Review and adjust parameters in `/etc/kolla/globals.yml`, in particular:

* `kolla_internal_vip_address`
* `kolla_external_vip_address`
* `network_interface`
* `kolla_external_vip_interface`
* `docker_registry`

Review and adjust configuration content in copied files, in particular:

* `/etc/kolla/config/neutron/ml2_plugin.ini`

Modify deployment inventory in `/root/multinode`:

```ini
[control]
osc1

[network]
osc1

[compute]
comp1
comp2

[monitoring]
osc1

[storage]
comp1
comp2
```

Generate passwords:

```bash
$ kolla-genpwd
```

Simplify passwords in `/etc/kolla/password.yml`, in particular:

* `keystone_admim`
* `database_password`
* `rabbitmq_password`

## Prepare OSC

Populate `/etc/hosts` file, e.g.:

```bash
192.168.100.233 osc1
192.168.100.222 comp1
192.168.100.231 comp2
```

Distribute public key to compute nodes:

```bash
$ ssh-copy-id comp1
$ ssh-copy-id comp2
```

[Start local Docker registry](https://gist.github.com/sfoolish/090c93a1ff417cbed3b148b07f501ab2):

```bash
$ docker run -d \
    --name registry \
    --restart=always \
    -p 4000:5000 \
    -v registry:/var/lib/registry \
    registry:2
```

Download archive with Monasca images into `/root/` directory.

Push Monasca images:

```bash
$ gunzip monasca-docker-images.tar.gz
$ docker load -i monasca-docker-images.tar
```

Re-tag pushed images:

```bash
docker tag 695614bd3c20 kolla/centos-source-monasca-thresh:master
docker tag 450e8913a681 kolla/centos-source-monasca-api:master
docker tag 60a15ff5c04d kolla/centos-source-monasca-agent:master
docker tag 3c0f411686fc kolla/centos-source-monasca-log-api:master
docker tag 4b74129e2c0f kolla/centos-source-monasca-notification:master
docker tag 810cf62c3aad kolla/centos-source-monasca-base:master
docker tag 80105d1988f8 kolla/centos-source-monasca-grafana:master
```

Restart registry as cache-through proxy:

```bash
$ docker stop registry
$ docker run -d \
    --name registry \
    --restart=always \
    -p 4000:5000 \
    -v registry:/var/lib/registry \
    -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io \
    registry:2
```

Clone repository with custom deployment configuration:

```bash
$ git clone https://github.com/soda-research/soda-poc.git
```

## Prepare storage nodes

Setup Cinder volume group:

```bash
free_device=$(losetup -f)
fallocate -l 250G /var/lib/cinder_data.img
losetup $free_device /var/lib/cinder_data.img
pvcreate $free_device
vgcreate cinder-volumes $free_device
```

Enable loopback device on boot:

```bash
chmod +x /etc/rc.d/rc.local

echo "# Setup loopback device for Cinder volume group" >> /etc/rc.d/rc.local
echo 'free_device=$(losetup -f)' >> /etc/rc.d/rc.local
echo 'losetup $free_device /var/lib/cinder_data.img' >> /etc/rc.d/rc.local
```

## Run deployment

Ping nodes:

```bash
ansible -i multinode all -m ping
```

Bootstrap nodes:

```bash
$ kolla-ansible -i multinode bootstrap-servers
```

Run prechecks:

```bash
$ kolla-ansible -i multinode prechecks
```

Run deployment:

```bash
$ kolla-ansible -i multinode deploy
```

## Post deployment

Run post deployment:

```bash
$ kolla-ansible post-deploy
```

Update credentials with endpoint type appropriate for older versions of OpenStack clients:

```bash
$ echo "export OS_ENDPOINT_TYPE=internal" >> /etc/kolla/admin-openrc.sh
```

Upload credentials to OSC:

```bash
$ rsync /etc/kolla/admin-openrc.sh root@osc1:/etc/kolla/admin-openrc.sh
```

## Post deployment: OSC

Source credentials:

```bash
$ . /etc/kolla/admin-openrc.sh
```

Source credentials on each session:

```bash
$ echo ". /etc/kolla/admin-openrc.sh" >> ~/.bashrc
```

Install OpenStack clients:

```bash
$ pip install python-openstackclient \
              python-troveclient \
              python-glanceclient \
              python-neutronclient \
              python-vitrageclient \
              python-monascaclient
```

Download resource init script:

```bash
$ wget https://raw.githubusercontent.com/openstack/kolla-ansible/master/tools/init-runonce
$ chmod +x init-runonce
```

Run resource init script:

```bash
$ . ./init-runonce
```

Create additional networks:

```bash
$ openstack network create \
      --provider-network-type flat \
      --provider-physical-network \
      mgmt internal

$ openstack subnet create \
      --subnet-range 192.168.200.0/24 \
      --dhcp \
      --allocation-pool start=192.168.200.100,end=192.168.200.200 \
      --dns-nameserver 8.8.8.8 \
      --network \
      internal internal-subnet
```

Create additional flavors:

```bash
$ openstack flavor create \
      --id 6 \
      --disk 5 \
      --ram 1024 \
      --vcpus 1 \
      --public \
      test.small
```

Create volume types:

```bash
$ openstack volume type create default
```

## Post deployment: Trove

Update Trove CLI code with additional patches:

```bash
$ cd /opt/stack/python-troveclient
$ git remote add soda https://github.com/soda-research/python-troveclient.git
$ git fetch soda
$ git checkout soda-poc
```

Put Trove guest images into `/root/images/` directory.

Create MariaDB image in Glance:

```bash
$ openstack image create --disk-format qcow2 \
                         --min-disk 5 \
                         --min-ram 1024 \
                         --public \
                         --file /root/images/centos_mariadb_10.1.32.qcow2 \
                         centos_mariadb
```

Load guest image into Trove via Trove API container:

```bash
$ docker exec -it trove_api /bin/bash

DATASTORE_TYPE=mariadb
VERSION=10.1.32
IMAGEID=<GLANCE_IMAGE_ID>
PATH_TROVE=/trove

trove-manage datastore_update "$DATASTORE_TYPE" ""
trove-manage datastore_version_update "$DATASTORE_TYPE" "$VERSION" "$DATASTORE_TYPE" $IMAGEID "$PACKAGES" 1
trove-manage datastore_update "$DATASTORE_TYPE" "$VERSION"
trove-manage db_load_datastore_config_parameters "$DATASTORE_TYPE" "$VERSION" "$PATH_TROVE"/trove/templates/$DATASTORE_TYPE/validation-rules.json
```

Build Trove source distribution:

```bash
$ python /opt/stack/trove/setup.py sdist
```

Create cloudinit directory:

```bash
$ mkdir -p /etc/kolla/trove-taskmanager/cloudinit
```

Add datastore cloudinit files:

```bash
$ cp -r /root/soda-poc/kolla/etc/kolla/config/trove/cloudinit/ /etc/kolla/trove-taskmanager/cloudinit
```

Setup OSC publis key in OSC authorized keys (for code sync between guest and OSC):

```bash
$ cat /root/.ssh/id_rsa$ .pub >> /root/.ssh/authorized_keys
```

## Post deployment: Monasca

Create monitoring user:

```bash
$ openstack user create --project monasca_control_plane \
                        --password monitoring \
                        monitoring

$ openstack role add admin --project monasca_control_plane \
                           --user monitoring

# $ openstack role add monasca_read_only_user --project monasca_control_plane \
#                                             --user monitoring
```

## Post deployment: Vitrage

Install Monasca client in Vitrage Graph container:

```bash
$ docker exec -u0 vitrage_graph pip install python-monascaclient
```

Copy datasource value manifests:

```bash
$ cp -r /root/soda-poc/kolla/etc/kolla/config/vitrage/datasources_values/ /etc/kolla/vitrage-graph/datasources_values
$ cp -r /root/soda-poc/kolla/etc/kolla/config/vitrage/datasources_values/ /etc/kolla/vitrage-collector/datasources_values
```

Restart Vitrage containers:

```bash
$ for i in $(docker ps |grep vitrage |awk '{print $1}'); do echo $i; docker restart $i; done
```

Add Vitrage user to admin project:

```bash
$ openstack role add --user vitrage \
                     --project \
                     admin admin
```

## Test deployment

Create sample server:

```bash
$ network_id=$(openstack network list |grep internal |awk '{print $2}')
$ openstack server create \
      --image cirros \
      --flavor test.small \
      --key-name mykey \
      --network $network_id \
      soda-poc
```

Create sample database instance:

```bash
$ network_id=$(openstack network list |grep internal |awk '{print $2}')
$ trove create soda-poc test.small \
      --size 1 \
      --volume_type default \
      --datastore mariadb \
      --datastore_version 10.1.32 \
     --nic net-id=$network_id \
```

Create sample database cluster:

```bash
$ network_id=$(openstack network list |grep internal |awk '{print $2}')
$ trove cluster-create soda-poc mariadb 10.1.32 \
      --instance "flavor=test.small,volume=1,nic='net-id=$network_id'" \
      --instance "flavor=test.small,volume=1,nic='net-id=$network_id'" \
      --instance "flavor=test.small,volume=1,nic='net-id=$network_id'"
```

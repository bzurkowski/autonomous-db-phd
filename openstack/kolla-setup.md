# Multinode OpenStack deployment with Kolla Ansible

## Prepare all nodes

Install Python and upgrade pip:

```bash
$ yum install -y epel-release
$ yum install -y python-pip
$ pip install -U pip
```

Install dependencies:

```bash
$ yum install -y python-devel \
                 libffi-devel \
                 gcc \
                 openssl-devel \
                 libselinux-python
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

Install and configure git:

```bash
$ yum install -y git

$ git config --global user.email "operator@soda-os-poc"
$ git config --global user.name "operator"
```

Clone Kolla Ansible repository:

```bash
$ git clone https://git.openstack.org/openstack/kolla-ansible.git
```

Fetch all references:

```bash
git fetch origin "+refs/changes/*:refs/remotes/origin/changes/*"
```

Checkout:

```bash
git checkout -b soda-poc 25ef71c20bce25b9b79d04a9b8232f27f4ac8a78
```

Cherry-pick Kolla Ansible patches:

```bash
$ git remote add bz https://github.com/bzurkowski/kolla-ansible.git
$ git fetch bz
$ git cherry-pick 05a9f9abb9a47ea3e45ce4c566e0ea5b871982fe..dd7de79136c826c5de3135eb91fd4b185cc14ead
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

Modify deployment configuration in `/etc/kolla/globals.yml`:

```yaml
---
config_strategy: "COPY_ALWAYS"

kolla_base_distro: "centos"
kolla_install_type: "source"
openstack_release: "master"

kolla_source_version: "master"
trove_dev_mode: "yes"
vitrage_dev_mode: "yes"
mistral_dev_mode: "yes"

kolla_internal_vip_address: "192.168.200.50"
kolla_external_vip_address: "192.168.100.50"

docker_registry: "192.168.200.115:4000"

network_interface: "eth2"

#kolla_external_vip_interface: "{{ network_interface }}"
#api_interface: "{{ network_interface }}"
#storage_interface: "{{ network_interface }}"

neutron_external_interface: "eth1"

neutron_plugin_agent: "linuxbridge"

enable_cinder: "yes"
enable_cinder_backup: "yes"
enable_cinder_backend_lvm: "yes"
enable_haproxy: "yes"
enable_heat: "no"
enable_horizon: "yes"
enable_horizon_mistral: "{{ enable_mistral | bool }}"
enable_horizon_trove: "{{ enable_trove | bool }}"
enable_mistral: "yes"
enable_monasca: "yes"
enable_trove: "yes"
enable_vitrage: "yes"
```

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

Apply Nova config patches:

```bash
mkdir -p /etc/kolla/config/nova/
cat > /etc/kolla/config/nova/nova-compute.conf <<EOF
[libvirt]
virt_type = qemu
cpu_mode = none
EOF
```

Apply Trove config patches:

```bash
mkdir -p /etc/kolla/config/trove/
cat > /etc/kolla/config/trove/trove-taskmanager.conf <<EOF
[mariadb]
icmp = True
tcp_ports = 22,3306,4444,4567,4568
EOF
```

Apply Monasca config patches:

```bash
$ echo "log.message.format.version=0.9.0.0" >> /etc/kolla/config/kafka.server.properties
```

Generate passwords:

```bash
$ kolla-genpwd
```

Update password for `keystone_admim`.

## Prepare OSC

Populate `/etc/hosts` file, e.g.:

```bash
192.168.100.233 osc1
192.168.100.222 comp1
192.168.100.231 comp2
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

Push Monasca images:

```bash
$ gunzip monasca-docker-images.tar.gz
$ docker load -i monasca-docker-images.tar

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

Source credentials:

```bash
$ . /etc/kolla/admin-openrc.sh
```

Update credentials with endpoint type appropriate for older versions of OpenStack clients:

```bash
$ echo "export OS_ENDPOINT_TYPE=internal" >> /etc/kolla/admin-openrc.sh
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
              python-vitrageclient
```

Create networks:

```bash
$ openstack network create --provider-network-type flat --provider-physical-network mgmt internal

$ openstack subnet create --subnet-range 192.168.122.0/24 --dhcp --allocation-pool start=192.168.122.100,end=192.168.122.200 --dns-nameserver 8.8.8.8 --network internal internal-subnet
```

Create flavors:

```bash
$ openstack flavor create --id 6 \
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

Create cloudinit directory:

```bash
$ mkdir -p /etc/kolla/trove-taskmanager/cloudinit
```

Populate cloudinit file for MariaDB in `/etc/kolla/trove-taskmanager/cloudinit/mariadb.cloudinit`:

```bash
#!/usr/bin/env bash

mkdir -p /home/trove/.ssh/
cat <<EOT >> /home/trove/.ssh/authorized_keys
<SSH_PUBLIC_KEY>
EOT

mkdir -p /root/.ssh/
cat <<EOT >> /root/.ssh/id_rsa
<SSH_PRIVATE_KEY>
EOT
chmod 600 /root/.ssh/id_rsa

ln -s /etc/trove/conf.d/trove-guestagent.conf /etc/trove/trove-guestagent.conf

rsync \
  -e "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null" -a \
  --exclude .git \
  --exclude-from /opt/trove/.gitignore \
  root@<OSC_INTERNAL_IP>:/opt/stack/trove/ /opt/trove-rsync

chown -R trove /opt/trove-rsync

systemctl stop trove-guest

if [[ -d /opt/trove-rsync ]]; then
    mv /opt/trove /opt/trove-orig
    mv /opt/trove-rsync /opt/trove
fi

systemctl start trove-guest

guest_id=$(awk -F "=" '/guest_id/ {print $2}' /etc/trove/conf.d/guest_info.conf)

monasca-setup \
  --username monitoring \
  --password monitoring \
  --project_name monasca_control_plane \
  --project_domain_name default \
  --user_domain_name default \
  --keystone_url http://<OSC_INTERNAL_VIP>:5000/v3 \
  --monasca_url http://<OSC_INTERNAL_VIP>:8070/v2.0 \
  --system_only \
  --service trove \
  --dimensions resource_id:${guest_id},resource_type:trove.instance \
  --verbose
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

Add Vitrage user to admin project:

```bash
$ openstack role add --user vitrage \
                     --project \
                     admin admin
```

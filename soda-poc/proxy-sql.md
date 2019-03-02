# ProxySQL setup

## VM setup

Create security group:

```bash
$ openstack security group create proxysql && \
  openstack security group rule create --proto icmp proxysql && \
  openstack security group rule create --proto tcp --dst-port 22 proxysql && \
  openstack security group rule create --proto tcp --dst-port 3306 proxysql && \
  openstack security group rule create --proto tcp --dst-port 6032 proxysql &&
```

Spawn VM:

```bash
$ network_id=$(openstack network list |grep internal |awk '{print $2}')
$ openstack server create \
    --image centos_mariadb \
    --flavor test.small \
    --security-group proxysql \
    --nic net-id=$network_id \
    --user-data /etc/kolla/trove-taskmanager/cloudinit/mariadb.cloudinit \
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

Connect to admin interface:

```bash
$ proxy_ip=$(openstack server )
$ mysql -P6032 -uadmin -padmin -h $proxy_ip
```

# Kolla image building

Clone Kolla repository:

```bash
$ git clone https://git.openstack.org/openstack/kolla.git
```

Setup virtualenv for Kolla build script:

```bash
$ pip install virtualenv
$ mkdir ~/.venvs
$ virtualenv ~/.venvs/kolla
$ source ~/.venvs/kolla/bin/activate
```

Install Kolla:

```bash
$ pip install ./kolla
```

Generate configuration:

```bash
$ pip install tox
$ cd ./kolla
$ tox -e genconfig
```

Build images, e.g. for Monasca:

```bash
$ kolla-build -t source -b centos monasca
```




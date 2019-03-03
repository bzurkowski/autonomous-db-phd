# Trove guest image builder

## Installation

Switch to `stack` user and activate DIB virtual env:

```bash
$ su stack && cd /opt/stack
$ source ~/.venvs/dib/bin/activate
```

Clone Trove Image Builder:

```
$ git clone https://github.com/soda-research/trove-image-builder.git
```

Fetch required DIB elements:

```bash
$ git clone https://github.com/openstack/diskimage-builder.git
$ git clone https://github.com/openstack/tripleo-image-elements.git
$ git clone https://github.com/soda-research/trove.git -b soda-poc
```

Create log directory:

```bash
$ sudo mkdir -p /var/log/trove-image-builder
$ sudo chown -R stack:stack /var/log/trove-image-builder
```

Create output image directory:

```bash
$ mkdir -p /opt/stack/images
```

Synchronize SSH keys from controller:

```bash
$ rsync -r root@<CONTROLLER_IP>:/root/.ssh/ /opt/stack/.ssh
$ cd /opt/stack/.ssh
$ rm authorized_keys
$ cat id_rsa.pub > authorized_key
```

## Usage

Run image building script:

```bash
$ ./trove-image-builder/build_image --controller-ip=<CONTROLLER_IP>
```

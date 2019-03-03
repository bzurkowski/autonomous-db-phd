# Disk image builder

## Installation

Install dependencies:

```bash
$ yum install -y epel-release
$ yum install -y qemu kpartx squashfs-tools
```

Install Python, pip and virtualenv:

```bash
$ yum install -y python-pip
$ pip install -U pip
$ pip install virtualenv
```

Create `stack` user (pass: stack):

```bash
$ useradd -G wheel -d /opt/stack -s /bin/bash -p "$(openssl passwd -1 stack)" stack
$ su stack
```

Prepare virtual env for DIB package:

```bash
$ mkdir ~/.venvs
$ virtualenv ~/.venvs/dib
$ source ~/.venvs/dib/bin/activate
```

Install DIB package:

```bash
$ pip install diskimage-builder
```

## Usage

```
Usage: disk-image-create [OPTION]... [ELEMENT]...

Options:
    -a i386|amd64|armhf|arm64 -- set the architecture of the image(default amd64)
    -o imagename -- set the imagename of the output image file(default image)
    -t qcow2,tar,tgz,squashfs,vhd,docker,aci,raw -- set the image types of the output image files (default qcow2)
       File types should be comma separated. VHD outputting requires the vhd-util
       executable be in your PATH. ACI outputting requires the ACI_MANIFEST
       environment variable be a path to a manifest file.
    -x -- turn on tracing (use -x -x for very detailed tracing).
    -u -- uncompressed; do not compress the image - larger but faster
    -c -- clear environment before starting work
    --logfile -- save run output to given logfile (implies DIB_QUIET=1)
    --checksum -- generate MD5 and SHA256 checksum files for the created image
    --image-size size -- image size in GB for the created image
    --image-cache directory -- location for cached images(default ~/.cache/image-create)
    --max-online-resize size -- max number of filesystem blocks to support when resizing.
       Useful if you want a really large root partition when the image is deployed.
       Using a very large value may run into a known bug in resize2fs.
       Setting the value to 274877906944 will get you a 1PB root file system.
       Making this value unnecessarily large will consume extra disk space
       on the root partition with extra file system inodes.
    --min-tmpfs size -- minimum size in GB needed in tmpfs to build the image
    --mkfs-options -- option flags to be passed directly to mkfs.
       Options should be passed as a single string value.
    --no-tmpfs -- do not use tmpfs to speed image build
    --offline -- do not update cached resources
    --qemu-img-options -- option flags to be passed directly to qemu-img.
       Options need to be comma separated, and follow the key=value pattern.
    --root-label label -- label for the root filesystem.  Defaults to 'cloudimg-rootfs'.
    --ramdisk-element -- specify the main element to be used for building ramdisks.
       Defaults to 'ramdisk'.  Should be set to 'dracut-ramdisk' for platforms such
       as RHEL and CentOS that do not package busybox.
    --install-type -- specify the default installation type. Defaults to 'source'. Set to 'package' to use package based installations by default.
    --docker-target -- specify the repo and tag to use if the output type is docker. Defaults to the value of output imagename
    -n skip the default inclusion of the 'base' element
    -p package[,p2...] [-p p3] -- extra packages to install in the image.  Runs once, after 'install.d' phase.  Can be specified multiple times
    -h|--help -- display this help and exit
    --version -- display version and exit

Environment Variables:
  (this is not a complete list)

  * ELEMENTS_PATH: specify external locations for the elements.  As for $PATH
  * DIB_NO_TIMESTAMP: no timestamp prefix on output.  Useful if capturing output
  * DIB_QUIET: 1=do not output log output to stdout; 0=always ouptut to stdout.  See --logfile

NOTE: At least one distribution root element must be specified.

NOTE: If using the VHD output format you need to have a patched version of vhd-util installed for the image
      to be bootable. The patch is available here: https://github.com/emonty/vhd-util/blob/master/debian/patches/citrix
      and a PPA with the patched tool is available here: https://launchpad.net/~openstack-ci-core/+archive/ubuntu/vhd-util

Examples:
    disk-image-create -a amd64 -o ubuntu-amd64 vm ubuntu
    export ELEMENTS_PATH=~/source/tripleo-image-elements/elements
    disk-image-create -a amd64 -o fedora-amd64-heat-cfntools vm fedora heat-cfntools
```

# Configuration file bits for Proxmox LXC Container
*This is tested in Proxmox. I'm not sure if running LXC/LXD containers directly on a host are different.*
## Common bits:
Architecture:
```
arch: amd64
```
## Physical(?) cores:
```
cores: 3
```
## Nesting:
https://ubuntu.com/blog/nested-containers-in-lxd
```
features: nesting=1
```
## Hostname:
```
hostname: container
```
## Memory/Swap (MiB?):
No swap, live dangerously 😎
```
memory: 6144
swap: 0
```
## Folder mountpoints:
Eg. Host mountpoint inside of container\
Mountpoints iterate starting from 0
```
# Host has /HDD1 directory from ZFS pool, mounted in container as /media/HDD1
mp0: /HDD1,mp=/media/HDD1
mp1: /NVR,mp=/media/NVR
```
## Networking:
```
net0: name=eth0,bridge=vmbr0,hwaddr=BC:24:11:C7:91:86,ip=dhcp,ip6=dhcp,type=veth
# VMBR1 is setup as a local bridge between all of the containers, not routed for lan
net1: name=eth1,bridge=vmbr1,hwaddr=BC:24:11:0A:DB:AD,ip=192.168.50.3/24,type=veth
```
## Start container on system start:
```
onboot: 1
```
## Container OS type:
```
ostype: debian
```
## RootFS:
Host stored on "local-zfs", named "subvol-102-disk-0" (LXC ID is 102, first disk in the container), 4GB in size
```
rootfs: local-zfs:subvol-102-disk-0,size=4G
```
## Tags:
```
tags: proxmox-helper-scripts
```
## Device Passthrough
*THIS HAS ONLY BEEN TESTED WITH INTEL iGPU IN PRIVILEGED CONTAINERS*
```
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
lxc.cgroup2.devices.allow: c 188:* rwm
lxc.cgroup2.devices.allow: c 189:* rwm
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.cgroup2.devices.allow: c 29:0 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
```
---
layout: post
title:  "Moving Proxmox-LXC container"
date:   2018-7-24
tags:
  - proxmox
  - LXC
  - Docker
  - NFS
  - container
  - apparmor
  - migrate
---


Proxmox supports migration of container in shutdown mode as well as taking snapshot of container. One caveat that needs to be consider for container snapshot feature is that the under laying disk file system for the proxmox host need to support snapshot. I had a requirement to take complete backup of a container and move it to another proxmox host, incidently this Proxmox host was not part of the cluster and was configured on regular lvm file system which does not support snapshot of container. This was tackled using the below steps:

* Container test-move needs to be moved to another proxmox box

```
root@proxmox1:~$pct list

VMID       Status     Lock         Name                
111        running                 app1   
114        stopped                 db-store
119        running                 consul  
120        running                 test-move  

root@proxmox1:~$pct enter 120
root@test-move:~# pwd
/root
root@test-move:~# mkdir test
root@test-move:~# cd test
root@test-move:~/test# echo "test 1 test 1 test 2 test 2" > test-move
root@test-move:~/test# ls
test-move
root@test-move:~/test# cat test-move 
test 1 test 1 test 2 test 2
```

* Shutdown & take backup of the container 

root@proxmox1:~$pct stop 120

* Copy the container image to a shared partition

I have a NFS shared partition mounted on all proxmox host. Copying the container image from /var/lib/vz/images/<container id>/container-id.raw to a nfs shared partition

``` 
root@proxmox1:/mnt/bkp/images$cp /var/lib/vz/images/120/vm-120-disk-1.raw .
root@proxmox1:/mnt/bkp/images$ls -lth
total 635M
-rw-r----- 1 root root 5.0G Feb 26 20:48 vm-120-disk-1.raw
``` 

* Copy the configuration file for the container to nfs shared 

```
root@proxmox1:~$cat /etc/pve/nodes/red/lxc/120.conf 
arch: amd64
cores: 1
hostname: test-move
memory: 512
net0: name=eth1,bridge=vmbr3,gw=192.168.56.1,hwaddr=A6:01:FE:E0:67:36,ip=192.168.56.161/16,type=veth
ostype: ubuntu
rootfs: local:110/vm-120-disk-1.raw,size=5G
swap: 512
```

Before restoring the containers raw image from nfs share on the destination proxmox box, create a dummy place holder container on the destination proxmox host with similar cpu, ram, disk size.
This should create the necessary files and entries in /etc/pve/ and a file entry at /etc/pve/.rrd 

* Stop this place holder container and delete/move its raw file

```
root@proxmox2:pct stop 160
root@proxmox2:/var/lib/vz/images$mv 160/vm-160-disk-1.raw /tmp/
```

* Copy the raw image from nfs share to /var/lib/vz/images/<container id>

```
root@proxmox2:/var/lib/vz/images/160$cp  /mnt/bkp/images/vm-120-disk-1.raw vm-160-disk-1.raw
root@proxmox2:/var/lib/vz/images/160$ls
vm-160-disk-1.raw
```

* Copy the configuration file content from nfs share to /etc/pve/nodes/studio/lxc/<container id>.conf

```
root@proxmox2:/etc/pve$cat nodes/studio/lxc/160.conf
arch: amd64
cores: 1
hostname: test-move
memory: 512
net0: name=eth1,bridge=vmbr3,gw=192.168.56.1,hwaddr=A6:01:FE:E0:67:36,ip=192.168.56.161/16,type=veth
ostype: ubuntu
rootfs: local:120/vm-120-disk-1.raw,size=5G
swap: 512
```

* Replace the string referencing the old container id with new container id in the configuration file

```
rootfs: local:120/vm-120-disk-1.raw,size=5G
TO
rootfs: local:160/vm-160-disk-1.raw,size=5G
```

* Start the container and verify its content

```
root@proxmox2:/etc/pve$pct list
VMID       Status     Lock         Name                
130        stopped                 test-snap           
160        running                 test-move 


root@proxmox2:/etc/pve$pct enter 160
root@test-move:/# ls
bin  boot  dev  etc  fastboot  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@test-move:/# cd
root@test-move:~# ls
test
root@test-move:~# cd test
root@test-move:~/test# ls
test-move
root@test-move:~/test# cat test-move 
test 1 test 1 test 2 test 2
```

The above procedure can also be used for creating backup of raw image files of the container.
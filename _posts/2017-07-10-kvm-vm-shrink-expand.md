---
layout: post
title:  "KVM VM shrink-expand"
date:   2017-06-25
tags:
  - KVM
  - VM
  - guestfish
  - virsh
  - libvirt
  - qemu-img
  - qcow2
  - shrink vm
  - expand vm
---

***Problem*: Disk space on the Host machine is filling up.**

* The total used disk space is @ 94%, disk space on the server need to be freed up
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            252G   39G  214G  16% /dev
tmpfs            51G  1.4M   51G   1% /run
/dev/dm-0       413G  367G   25G  94% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
none            5.0M     0  5.0M   0% /run/lock
none            252G   12K  252G   1% /run/shm
none            100M     0  100M   0% /run/user
/dev/sda1       237M   77M  148M  35% /boot
```

***Analysis*: The host machine has Multiple VM's which are occupying large amount of disk space than the disk space actually used.**  

* List all the running VM's
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# virsh list --all
 Id    Name                           State
----------------------------------------------------
 2     oracle-01.prod.setup       running
 3     centos-01.prod.setup       running
 8     sql-01.prod.setup          running
 10    ntp-01.prod.setup          running
 11    config-01.prod.setup       running
 12    dns-01.prod.setup          running
 23    maas-01.prod.setup         running
```
* Using the disk usage command on /var/lib/libvirt/images, I was able to narrow down down on a VM disk image which was occupying substantial amount of disk space.

```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# ls -lh
total 145G
-rw-r--r-- 1 root root  145G Sep 20 15:46 system.qcow2
```
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# qemu-img info system.qcow2
image: system.qcow2
file format: qcow2
virtual size: 146G (157286400000 bytes)
disk size: 11G
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
```
* Actual disk utilization inside the VM is 2.4GB out of 145GB.

```
amold@maas-01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           799M  8.6M  790M   2% /run
/dev/vda1       145G  2.4G  136G   2% /
tmpfs           3.9G   12K  3.9G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
```

***Solution*: Shrink the disk image to appropriate size on the Host machine and reclaim the free disk space.**

* Mark the empty disk space in the guest VM - maas-01 by filling the free disk space with zero's using multiple zero files. 
```
dd if=/dev/zero of=zero-file bs=1M count=50000
dd if=/dev/zero of=zero-file2 bs=1M count=60000
dd if=/dev/zero of=zero-file2 bs=1M count=10000
```
* Delete the zero files after creation 

``amold@maas-01:~$ rm -frv zero-file2 zero-file3 zero-file
``

* Logout of the VM & back to the KVM nodes 

* Shutdown the maas-01 VM
```
(prod)root@host: virsh stop  maas-01.prod.setup
```

* Rename the original disk image for the VM
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# mv system.qcow2 system.qcow2.original
```
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# ls -lh
total 150G
-rw-r--r-- 1 root root 140G Sep 19 22:40 system.qcow2.original
```
* Use qemu-img command to ceate a new disk image
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# qemu-img convert -O qcow2 system.qcow2.original system.qcow2
```
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# ls -lh
total 150G
-rw-r--r-- 1 root root  11G Sep 20 15:46 system.qcow2
-rw-r--r-- 1 root root 140G Sep 19 22:40 system.qcow2.original
```
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# qemu-img info system.qcow2
image: system.qcow2
file format: qcow2
virtual size: 146G (157286400000 bytes)
disk size: 11G
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
```
The new disk image is the resized/shrunk qcow2 file for the maas-01 VM.
The new disk image has discarded all the zero disk data and considered the only the non zero bits. 

* Start the maas-01 VM with the newly created disk image. 
```
(prod)root@host: virsh start maas-01.prod.setup
```
* Login to the maas-01 VM, the disk size reflected is as earlier.
```
amold@maas-01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           799M  8.6M  790M   2% /run
/dev/vda1       145G  2.3G  136G   2% /
tmpfs           3.9G   12K  3.9G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
```
This need to be resized as per actual disk usage.

* Shutdown the VM from host

```
(prod)root@host: virsh destroy maas-01.prod.setup
```
* Use virt-df to view the VM's disk filesystem details

```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# virt-df system.qcow2
Filesystem                           1K-blocks       Used  Available  Use%
system.qcow2:/dev/sda1               1451481468    2405216   46935740    5%
```
* Use guestfish to modify the guest VM's filesystem.

This is an amazing tool to tweak the guest VM filesystem, to view - modify files, run commands, remove packages inside the disk image.
It is advisible to shutdown the VM before trying any write operation to the disk image to avoid data corruption.  More on guestfish @ http://libguestfs.org/guestfish.1.html

I am using the guestfish shell to perform disk resize for maas-01, same can be done using guestfish command. 
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# guestfish -a system.qcow2

Welcome to guestfish, the guest filesystem shell for
editing virtual machine filesystems and disk images.

Type: 'help' for help on commands
      'man' to read the manual
      'quit' to quit the shell

><fs>run
><fs>e2fsck-f /dev/sda1
><fs>resize2fs-size /dev/sda1 20G
><fs>quit
```

* Start the maas-01 VM 
```
(prod)root@host: virsh start maas-01.prod.setup
```
* View the resized disk changes inside the VM

```
amold@maas-01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           799M  8.6M  790M   2% /run
/dev/vda1        20G  2.3G   17G  13% /
tmpfs           3.9G   12K  3.9G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
```

While working on resizing the disk image file, I though to reverse the process for expanding the disk image to increase the disk space to  an adequate size in the VM.

* Shutdown the VM
```
(prod)root@host: virsh destroy maas-01.prod.setup
```
* Use guestfish shell to execute resize2fs on the VM partition 

```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# guestfish -a system.qcow2

Welcome to guestfish, the guest filesystem shell for
editing virtual machine filesystems and disk images.

Type: 'help' for help on commands
      'man' to read the manual
      'quit' to quit the shell

><fs>run
><fs>e2fsck-f /dev/sda1
><fs>resize2fs-size /dev/sda1 50G
><fs>quit
```
* Start VM & view the expanded partition
```
(prod)root@host: virsh start maas-01.prod.setup
```
```
(prod)root@host:/var/lib/libvirt/images/maas-01.prod.setup# virt-df system.qcow2
Filesystem                           1K-blocks       Used  Available  Use%
system.qcow2:/dev/sda1                51481468    2405216   46935740    5%
```

* SSH to the VM and verify the partition 

```
amold@maas-01:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           799M  8.6M  790M   2% /run
/dev/vda1        50G  2.3G   45G   5% /
tmpfs           3.9G   12K  3.9G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
```




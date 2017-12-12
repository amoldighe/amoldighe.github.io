---
layout: post
title:  "Reviving a corrupt qemu-kvm VM"
date:   2017-09-15
tags:
  - KVM
  - VM
  - blkid
  - virsh
  - libvirt
  - qemu-img
  - qemu-nbd
  - corrupt VM
  - tune2fs
  - growpart
  - ubuntu cloud image
---

***Problem*: Production VM has boot errors, due to short grub timeout of 2 sec, cannot boot the VM in single user mode.**

***Solution*: Copy the necessary data from the existing VM disk image to a new VM disk image.**

* Download the cloud disk image (ubuntu 14.04)   

wget -c https://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img 

* Rename and move the disk image to /var/lib/libvirt/

Verify the disk image size 

```
root@hp-envy:~# qemu-img info /var/lib/libvirt/images/ubuntu-server-1404-NEW.img
image: /var/lib/libvirt/images/newdisk-ubuntu1404.img
file format: qcow2
virtual size: 2.2G (2361393152 bytes)
disk size: 250M
cluster_size: 65536
Format specific information:
    compat: 0.10
    refcount bits: 16
```

* Resize disk image

The size of the new disk image is inadequate for production use, as need a larger root disk partion 

```
root@hp-envy:~# qemu-img resize /var/lib/libvirt/images/ubuntu-server-1404-NEW.img 50G
Image resized.

root@hp-envy:~# qemu-img info /var/lib/libvirt/images/ubuntu-server-1404-NEW.img
image: /var/lib/libvirt/images/ubuntu-server-1404-NEW.img
file format: qcow2
virtual size: 50G (53687091200 bytes)
disk size: 250M
cluster_size: 65536
Format specific information:
    compat: 0.10
    refcount bits: 16
```

* Load the nbd module, this will allow attaching the cloud image using qemu-nbd  

```
modprobe nbd
```

* Verify if nbd module is loaded 

``` 
root@hp-envy:~# lsmod | grep nbd
nbd                    32768  0
```

* Once nbd module is loaded, you should find number of object in /dev starting with nbd:

```
root@hp-envy:~# ls /dev/nbd*
/dev/nbd0  /dev/nbd10  /dev/nbd12  /dev/nbd14  /dev/nbd2  /dev/nbd4  /dev/nbd6  /dev/nbd8
/dev/nbd1  /dev/nbd11  /dev/nbd13  /dev/nbd15  /dev/nbd3  /dev/nbd5  /dev/nbd7  /dev/nbd9
```

* Attach the cloud image to one of the nbd device

Please note : I have copied the downloaded image to /var/lib/libvirt/images/ 

```
root@hp-envy:~# qemu-nbd --connect=/dev/nbd0 /var/lib/libvirt/images/ubuntu-server-1404-NEW.img 
```

* Use fdisk to check for the partitions on the mounted cloud image

```
root@hp-envy:/mnt# fdisk -l /dev/nbd0
Disk /dev/nbd0: 50 GiB, 53687091200 bytes, 104857600 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x000ae720

Device      Boot Start     End Sectors Size Id Type
/dev/nbd0p1 *     2048 4194303 4192256   2G 83 Linux
```

* I will be using growpart to extend the partition 

```
root@hp-envy:~# growpart /dev/nbd0 1
CHANGED: partition=1 start=2048 old: size=4192256 end=4194304 new: size=104855519,end=104857567

root@hp-envy:~# fdisk -l /dev/nbd0
Disk /dev/nbd0: 50 GiB, 53687091200 bytes, 104857600 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x000ae720

Device      Boot Start       End   Sectors Size Id Type
/dev/nbd0p1 *     2048 104857566 104855519  50G 83 Linux
```

* Mount the new disk image 

```
root@hp-envy:~# mount /dev/nbd0p1 /mnt/newVM/

root@hp-envy:~# cd /mnt/newVM/

root@hp-envy:/mnt/newVM# ls
bin  boot  dev  etc  home  initrd.img  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  vmlinuz

root@hp-envy:/mnt/newVM# ls -lh
total 92K
drwxr-xr-x  2 root root 4.0K Dec  8 23:31 bin
drwxr-xr-x  3 root root 4.0K Dec  8 23:31 boot
drwxr-xr-x  3 root root 4.0K Dec  8 23:27 dev
drwxr-xr-x 89 root root 4.0K Dec  8 23:31 etc
drwxr-xr-x  2 root root 4.0K Apr 11  2014 home
lrwxrwxrwx  1 root root   34 Dec  8 23:30 initrd.img -> boot/initrd.img-3.13.0-137-generic
drwxr-xr-x 21 root root 4.0K Dec  8 23:29 lib
drwxr-xr-x  2 root root 4.0K Dec  8 23:28 lib64
drwx------  2 root root  16K Dec  8 23:31 lost+found
drwxr-xr-x  2 root root 4.0K Dec  8 23:27 media
drwxr-xr-x  2 root root 4.0K Apr 11  2014 mnt
drwxr-xr-x  2 root root 4.0K Dec  8 23:27 opt
drwxr-xr-x  2 root root 4.0K Apr 11  2014 proc
drwx------  2 root root 4.0K Dec  8 23:31 root
drwxr-xr-x  2 root root 4.0K Dec  8 23:31 run
drwxr-xr-x  2 root root 4.0K Dec  8 23:31 sbin
drwxr-xr-x  2 root root 4.0K Dec  8 23:27 srv
drwxr-xr-x  2 root root 4.0K Mar 13  2014 sys
drwxrwxrwt  2 root root 4.0K Dec  8 23:31 tmp
drwxr-xr-x 10 root root 4.0K Dec  8 23:27 usr
drwxr-xr-x 12 root root 4.0K Dec  8 23:31 var
lrwxrwxrwx  1 root root   31 Dec  8 23:30 vmlinuz -> boot/vmlinuz-3.13.0-137-generic

```

Please note the new disk image has kernel version 3.13.0-137-generic

* Mount the old/existing disk image 

```
root@hp-envy:~# qemu-nbd --connect=/dev/nbd1 /var/lib/libvirt/images/ubuntu-server-1404.img 

root@hp-envy:~# fdisk -l /dev/nbd1
Disk /dev/nbd1: 2.2 GiB, 2361393152 bytes, 4612096 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x000c5984

Device      Boot Start     End Sectors  Size Id Type
/dev/nbd1p1 *     2048 4612095 4610048  2.2G 83 Linux

root@hp-envy:~# mount /dev/nbd1p1 /mnt/oldVM/

root@hp-envy:~# mount /dev/nbd1p1 /mnt/oldVM/
root@hp-envy:~# ls -lh /mnt/oldVM/
total 92K
drwxr-xr-x  2 root root 4.0K Nov 14 02:13 bin
drwxr-xr-x  3 root root 4.0K Nov 14 02:13 boot
drwxr-xr-x  3 root root 4.0K Nov 14 02:10 dev
drwxr-xr-x 89 root root 4.0K Dec 12 00:48 etc
drwxr-xr-x  3 root root 4.0K Nov 16 19:52 home
lrwxrwxrwx  1 root root   34 Nov 14 02:12 initrd.img -> boot/initrd.img-3.13.0-135-generic
drwxr-xr-x 21 root root 4.0K Nov 14 02:11 lib
drwxr-xr-x  2 root root 4.0K Nov 14 02:11 lib64
drwx------  2 root root  16K Nov 14 02:13 lost+found
drwxr-xr-x  2 root root 4.0K Nov 14 02:10 media
drwxr-xr-x  2 root root 4.0K Apr 11  2014 mnt
drwxr-xr-x  2 root root 4.0K Nov 14 02:10 opt
drwxr-xr-x  2 root root 4.0K Apr 11  2014 proc
drwx------  4 root root 4.0K Nov 17 21:13 root
drwxr-xr-x  2 root root 4.0K Nov 14 02:12 run
drwxr-xr-x  2 root root 4.0K Nov 14 02:13 sbin
drwxr-xr-x  2 root root 4.0K Nov 14 02:10 srv
drwxr-xr-x  2 root root 4.0K Mar 13  2014 sys
drwxrwxrwt  2 root root 4.0K Dec 12 00:48 tmp
drwxr-xr-x 10 root root 4.0K Nov 14 02:10 usr
drwxr-xr-x 12 root root 4.0K Nov 14 02:13 var
lrwxrwxrwx  1 root root   31 Nov 14 02:12 vmlinuz -> boot/vmlinuz-3.13.0-135-generic

```

Please note the old disk image has older kernel version 3.13.0-135-generic

* Copy root data "/" partition of the old VM to the new VM

```
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/dev ./
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/etc ./
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/home ./
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/l* ./
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/m* ./
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/o* ./
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/p* ./
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/r* ./
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/s* ./
root@hp-envy:/mnt/newVM# cp -rp /mnt/oldVM/u* ./
```

* Check the block device UUID

We have copied the /etc/fstab from old VM to new VM in the earlier step, which contains the UUID of the block device attached to the root filesystem on the image disk. The below command displays the UUID of the block device attached.

```
root@hp-envy:~# blkid /dev/nbd0p1
/dev/nbd0p1: LABEL="cloudimg-rootfs" UUID="d50acd3c-2275-465e-9499-f7d3de94618a" TYPE="ext4" PARTUUID="000ae720-01"

root@hp-envy:~# blkid /dev/nbd1p1
/dev/nbd1p1: LABEL="cloudimg-rootfs" UUID="97cea586-4b5c-4710-8d23-5cae6c12ae40" TYPE="ext4" PARTUUID="000c5984-01"
```

As the /etc/fstab entry corresponds to the UUID of the OLD VM 's block device, this UUID needs to be mapped to new VM.

Please note, if tune2fs commands prompts for a fsck on the file system, go ahead a do the same as below.
If not then, tune2fs should set the new UUID for block device. 

```
root@hp-envy:~# tune2fs /dev/nbd0p1 -U 97cea586-4b5c-4710-8d23-5cae6c12ae40
tune2fs 1.43.5 (04-Aug-2017)

Please run e2fsck -f on the filesystem.

root@hp-envy:~# e2fsck -f /dev/nbd0p1
e2fsck 1.43.5 (04-Aug-2017)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
cloudimg-rootfs: 84074/90112 files (0.0% non-contiguous), 252861/360448 blocks

root@hp-envy:~# tune2fs /dev/nbd0p1 -U 97cea586-4b5c-4710-8d23-5cae6c12ae40
tune2fs 1.43.5 (04-Aug-2017)
Setting UUID on a checksummed filesystem could take some time.
Proceed anyway (or wait 5 seconds) ? (y,N) <proceeding>
```

Both disk image partitions should have the same UUID for it's block device. 

```
root@hp-envy:~# blkid /dev/nbd0p1
/dev/nbd0p1: LABEL="cloudimg-rootfs" UUID="97cea586-4b5c-4710-8d23-5cae6c12ae40" TYPE="ext4" PARTUUID="000ae720-01"

root@hp-envy:~# blkid /dev/nbd1p1
/dev/nbd1p1: LABEL="cloudimg-rootfs" UUID="97cea586-4b5c-4710-8d23-5cae6c12ae40" TYPE="ext4" PARTUUID="000c5984-01"
```

* Disconnect the nbd device

```
root@hp-envy:~# qemu-nbd --disconnect /dev/nbd0
/dev/nbd0 disconnected

root@hp-envy:~# qemu-nbd --disconnect /dev/nbd1
/dev/nbd1 disconnected
```

* I am going to boot the existing / OLD VM (which is actually not corrupted & still bootable) for skae of displaying the diferences

```
root@hp-envy:/var/lib/libvirt/images# virsh start ubuntu14
Domain ubuntu14 started

root@hp-envy:/var/lib/libvirt/images# virsh list --all
 Id    Name                           State
----------------------------------------------------
 3     ubuntu14                       running


root@hp-envy:/var/lib/libvirt/images# virsh console ubuntu14
Connected to domain ubuntu14
Escape character is ^]

$ uname -a
Linux cfg01 3.13.0-135-generic #184-Ubuntu SMP Wed Oct 18 11:55:51 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            493M   12K  493M   1% /dev
tmpfs           100M  376K  100M   1% /run
/dev/vda1       2.2G  799M  1.3G  39% /
```

* Shutdown the OLD VM 

```
root@hp-envy:/var/lib/libvirt/images# virsh destroy ubuntu14
Domain ubuntu14 destroyed

root@hp-envy:/var/lib/libvirt/images# virsh list --all
 Id    Name                           State
----------------------------------------------------
 -     ubuntu14                       shut off
```

* As mentioned earlier, we have the OLD disk image & the NEW disk image with different kernel version

```
root@hp-envy:/var/lib/libvirt/images# ls -lh
total 1.3G
-rw-r--r-- 1 root         root 366K Nov 17 19:40 cloud.img
-rw-r--r-- 1 libvirt-qemu kvm  364K Nov 17 21:07 config-node.iso
-rw-r--r-- 1 root         root 393M Dec 12 02:00 ubuntu-server-1404.img
-rw-r--r-- 1 root         root 915M Dec 12 02:10 ubuntu-server-1404-NEW.img
```

* Switch the images, by renaming to the original disk image file name used in virsh-install command while spawning the existing VM.

```
root@hp-envy:/var/lib/libvirt/images# mv ubuntu-server-1404.img ubuntu-server-1404-OLD.img

root@hp-envy:/var/lib/libvirt/images# mv ubuntu-server-1404-NEW.img ubuntu-server-1404.img

root@hp-envy:/var/lib/libvirt/images# virsh start ubuntu14
Domain ubuntu14 started

root@hp-envy:/var/lib/libvirt/images# virsh list --all
 Id    Name                           State
----------------------------------------------------
 4     ubuntu14                       running


root@hp-envy:/var/lib/libvirt/images# virsh console ubuntu14
Connected to domain ubuntu14
Escape character is ^]

$ uname -a
Linux cfg01 3.13.0-137-generic #186-Ubuntu SMP Mon Dec 4 19:09:19 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            493M   12K  493M   1% /dev
tmpfs           100M  376K  100M   1% /run
/dev/vda1        50G  940M   47G   2% /

```

Note the size of the VM's root partition is 50 GB


* ***Reference Links*** 

[https://linux.die.net/man/1/qemu-img](https://linux.die.net/man/1/qemu-img)
[https://www.jamescoyle.net/tag/qemu-nbd](https://www.jamescoyle.net/tag/qemu-nbd)
[https://www.systutorials.com/docs/linux/man/1-growpart/](https://www.systutorials.com/docs/linux/man/1-growpart/)
[http://landoflinux.com/linux_tune2fs_command.html](http://landoflinux.com/linux_tune2fs_command.html)




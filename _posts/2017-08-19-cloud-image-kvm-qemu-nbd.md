---
layout: post
title:  "Cloud Image on KVM using qemu-nbd"
date:   2017-08-19
tags:
  - KVM
  - trusty-server-cloudimg-amd64-disk1.img
  - ubuntu
  - virsh
  - cloudimg
  - qemu-nbd
  - modprob
  - qemu-utils
  - virt-install
---

The goal of this exercise is to download a cloud image and spawn a KVM - Virtual Machine. Cloud images have cloud-init preinstalled which can be used to inject a user key, hostname and other metadata into the virtual machine being spawned. We will not be using cloud-init injection, the approch we will employ, is to bake a new user and a password in the cloud image to gain access to the spawned KVM-VM. This approch is a hack, which can be also be used to gain access to a VM, where a user is locked out OR for setting a backdoor in your cloud images available on Openstack cloud. Please note, I am using Ubuntu cloud (Trusty - 14.04) image for this exercise. 
 
* Download the cloud image file

``` 
wget -c https://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img
```

* Rename the image file

```
mv trusty-server-cloudimg-amd64-disk1.img ubuntu-server-1404.img
```

* Install qemu utils

```
apt-get install qemu-utils
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
root@hp-envy:~# qemu-nbd --connect=/dev/nbd0 /var/lib/libvirt/images/ubuntu-server-1404.img 
```

* Create a mount point directory 

```
root@hp-envy:~# mkdir /mnt/srcVM
```

* Use fdisk to check for the partitions on the mounted cloud image

```
root@hp-envy:~# fdisk -l /dev/nbd0
Disk /dev/nbd0: 2.2 GiB, 2361393152 bytes, 4612096 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x000c5984

Device      Boot Start     End Sectors  Size Id Type
/dev/nbd0p1 *     2048 4612095 4610048  2.2G 83 Linux
```

* nbd0p1 is displayed as the first partition, mount the same 

```
root@hp-envy:~# mount /dev/nbd0p1 /mnt/srcVM
```

* Check the root filesystem on the image

```
root@hp-envy:~# cd /mnt/srcVM/
root@amol-HP-ENVY-15-Notebook-PC:/mnt/srcVM# ls
bin  boot  dev  etc  home  initrd.img  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  vmlinuz
```

* Use chroot jail to execute commands inside of the mounted root filesystem

```
root@hp-envy:~# chroot /mnt/srcVM/

root@hp-envy:/# ls
bin   dev  home        lib    lost+found  mnt  proc  run   srv  tmp  var
boot  etc  initrd.img  lib64  media       opt  root  sbin  sys  usr  vmlinuz
```

* Add a new user to image filesystem 

```
root@hp-envy:/# useradd amol
```

* Set password for the new user

```
root@hp-envy:/# passwd amol
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
```

* Modify the sshd config to allow password authentication

```
root@hp-envy:/# sed -i "s/PasswordAuthentication no/PasswordAuthentication yes/g" /etc/ssh/sshd_config
```

* Exit chroot & Unmount the image file

``` 
root@hp-envy:~# umount /mnt/srcVM/
root@hp-envy:~# qemu-nbd --disconnect /dev/nbd0 
/dev/nbd0 disconnected
```

* Start the cloud image 

```
root@hp-envy:~# virt-install --name ubuntu14 --vcpus 1 --memory 1024 --disk path=/var/lib/libvirt/images/ubuntu-server-1404.img,bus=virtio,cache=writeback 
--graphics vnc,listen=0.0.0.0 
--network bridge:virbr0,model=virtio --noautoconsole --os-type=linux --import 
WARNING  No operating system detected, VM performance may suffer. Specify an OS with --os-variant for optimal results.

Starting install...
Creating domain...                                                                                                             |    0 B  00:00:00     
Domain creation completed.

```

* List the VM

```
root@hp-envy:~# virsh list --all
 Id    Name                           State
----------------------------------------------------
 12    ubuntu14                       running
```

* Use virt-manager to view the console of the newly booted VM

You might see few warning message from cloud-init trying to reach the bridge newtwork gateway for metadata, we will discuss this in another post.

```
2017-11-16 14:22:35,359 - url_helper.py[WARNING]: Calling 'http://192.168.122.1//latest/meta-data/instance-id' failed [112/120s]: request error [HTTPConnectionPool(host='192.168.122.1', port=80): Max retries exceeded with url: //latest/meta-data/instance-id (Caused by <class 'socket.error'>: [Errno 111] Connection refused)]
2017-11-16 14:22:42,367 - url_helper.py[WARNING]: Calling 'http://192.168.122.1//latest/meta-data/instance-id' failed [119/120s]: request error [HTTPConnectionPool(host='192.168.122.1', port=80): Max retries exceeded with url: //latest/meta-data/instance-id (Caused by <class 'socket.error'>: [Errno 115] Operation now in progress)]
2017-11-16 14:22:49,374 - DataSourceCloudStack.py[CRITICAL]: Giving up on waiting for the metadata from ['http://192.168.122.1//latest/meta-data/instance-id'] after 126 seconds
```

* Login using console 

```
root@hp-envy:~# virsh console ubuntu14
Connected to domain ubuntu14
Escape character is ^]

Ubuntu 14.04.5 LTS ubuntu ttyS0

ubuntu login: amol
Password: 
Last login: Thu Nov 16 17:52:53 UTC 2017 on tty1
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 3.13.0-135-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Thu Nov 16 17:52:53 UTC 2017

```

Congratulations !! you have successfully booted a cloud image on KVM.

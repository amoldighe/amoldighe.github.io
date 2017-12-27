---
layout: post
title:  "Cloud Image & CloudInit"
date:   2017-10-10
tags:
  - KVM
  - trusty-server-cloudimg-amd64-disk1.img
  - ubuntu
  - virsh
  - cloudimg
  - uuidgen
  - mkisofs
  - qemu-utils
  - virt-install
---

In one of our earlier exercise while booting a cloud image on KVM, we noticed the following warning messages.

```
2017-11-16 14:22:35,359 - url_helper.py[WARNING]: Calling 'http://192.168.122.1//latest/meta-data/instance-id' failed [112/120s]: request error [HTTPConnectionPool(host='192.168.122.1', port=80): Max retries exceeded with url: //latest/meta-data/instance-id (Caused by <class 'socket.error'>: [Errno 111] Connection refused)]
2017-11-16 14:22:42,367 - url_helper.py[WARNING]: Calling 'http://192.168.122.1//latest/meta-data/instance-id' failed [119/120s]: request error [HTTPConnectionPool(host='192.168.122.1', port=80): Max retries exceeded with url: //latest/meta-data/instance-id (Caused by <class 'socket.error'>: [Errno 115] Operation now in progress)]
2017-11-16 14:22:49,374 - DataSourceCloudStack.py[CRITICAL]: Giving up on waiting for the metadata from ['http://192.168.122.1//latest/meta-data/instance-id'] after 126 seconds
```

The message indicate that the above python script are trying to connect to network gateway IP 192.168.122.1 to fetch meta-data for the VM. As this is a cloud image we are trying to boot on KVM, it has cloud-init prebaked in the image. Cloud-init is a collection of python scripts configured to fetch metadata like - cloud specific host information like instance ID numbers, hostname, IP address, etc. User data can also be passed to the VM using cloud-init which can execute various commands, scripts to configure the VM. More details regarding cloud-init can be refered at the links ate the end of the post.

Cloud-init is configured to fetch this metadata or userdata from a metadata server OR a config-drive. To run a cloud image locally on KVM, we will be setting up a config drive to fetch this meta-data. Cloud-init is programmed to search for the config-drive to fetch meta-data before looking for meta-data @ 169.254.169.254 OR the gateway IP in the above case. We will be creating a config-drive iso containing the metadata and attach the same to as a cdrom drive while booting the VM as follows :

* Create a directory to house the meta-data & user-data files.

```
root@hp-envy:~# ls -lh cloud-configdir/
total 4.0K
-rw-r--r-- 1 root root 533 Dec 27 21:13 meta-data
-rw-r--r-- 1 root root   0 Dec 27 21:16 user-data
```

PLEASE NOTE - both these files are required, even if you do not want to pass as user-data to the VM.

* Generate UUID using command uuidgen

```
root@hp-envy:/mnt/srcVM# uuidgen 
378e5f70-de64-4944-a93f-190e29e49956
``` 

* Add the following meta data for KVM -VM

```
root@hp-envy:~# cat cloud-configdir/meta-data 
instance-id: 378e5f70-de64-4944-a93f-190e29e49956
hostname: node-1
local-hostname: node-1
public-keys:
  - |      
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD 
```

* Generate the file for iso drive

```
root@hp-envy:~# mkisofs -o /var/lib/libvirt/images/config-drv.iso -V cidata -r -J --quiet cloud-configdir
```

* OR use this script - [create-config-drive](https://github.com/larsks/virt-utils/blob/master/create-config-drive) to generate the iso file 

```
root@amol-HP-ENVY-15-Notebook-PC:~# bash ./create-config-drive.sh -k /home/amol/.ssh/id_rsa.pub -h cfg01 /var/lib/libvirt/images/config-drv.iso
adding pubkey from /home/amol/.ssh/id_rsa.pub
generating configuration image at /var/lib/libvirt/images/config-drv.iso
```

* Spawn the VM using the cloud image & config drive iso

``` 
root@hp-envy:/var/lib/libvirt/images# virt-install --name ubuntu14-new --vcpus 1 --memory 1024 --disk path=/var/lib/libvirt/images/ubuntu-server-1404-NEW.img,bus=virtio,cache=writeback --disk path=/var/lib/libvirt/images/config-drv.iso,device=cdrom --graphics vnc,listen=0.0.0.0 --network bridge:virbr0,model=virtio --noautoconsole --os-type=linux --boot hd
WARNING  No operating system detected, VM performance may suffer. Specify an OS with --os-variant for optimal results.

Starting install...
Creating domain...                                                                                                             |    0 B  00:00:00     
Domain creation completed.
```

* List the new VM

```
root@hp-envy:~# virsh list --all
 Id    Name                           State
----------------------------------------------------
 1     ubuntu14                       running
 5     ubuntu14-new                   running
```

* List the active network to further list the DHCP IP assigned to the VM

```
root@hp-envy:~# virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
```

```
root@hp-envy:~# virsh net-dhcp-leases default
 Expiry Time          MAC address        Protocol  IP address                Hostname        Client ID or DUID
-------------------------------------------------------------------------------------------------------------------
 2017-12-27 23:25:04  52:54:00:e4:53:22  ipv4      192.168.122.238/24        cfg01           -
 2017-12-27 23:30:53  52:54:00:f4:2f:df  ipv4      192.168.122.210/24        node-1          -
```

* SSH to the new VM

```
root@hp-envy:~# ssh -i /home/amol/.ssh/id_rsa ubuntu@192.168.122.210
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 3.13.0-137-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Wed Dec 27 15:51:01 UTC 2017

  System load:  0.05              Processes:           73
  Usage of /:   1.9% of 49.18GB   Users logged in:     0
  Memory usage: 5%                IP address for eth0: 192.168.122.210
  Swap usage:   0%

  Graph this data and manage this system at:
    https://landscape.canonical.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.

New release '16.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Wed Dec 27 15:51:24 2017 from hp-envy
ubuntu@node-1:~$ 
```

Congratulations again!! you have successfully booted a cloud image on KVM.


* Reference Links

[https://www.digitalocean.com/community/tutorials/how-to-use-cloud-config-for-your-initial-server-setup](https://www.digitalocean.com/community/tutorials/how-to-use-cloud-config-for-your-initial-server-setup)
[https://www.cloudsigma.com/an-introduction-to-server-provisioning-with-cloudinit](https://www.cloudsigma.com/an-introduction-to-server-provisioning-with-cloudinit)
[http://ibm-blue-box-help.github.io/help-documentation/nova/Metadata_service_FAQ/](http://ibm-blue-box-help.github.io/help-documentation/nova/Metadata_service_FAQ/)







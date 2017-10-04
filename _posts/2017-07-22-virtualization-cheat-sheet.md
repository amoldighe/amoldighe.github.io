---
layout: post
title:  "Virtualization cheat sheet"
date:   2017-07-22
tags:
  - KVM
  - VM
  - guestfish
  - virsh
  - libvirt
  - qemu-img
  - qcow2
  - virt-install
  - interface
 ---

* Create a disk image

```
qemu-img create -f qcow2 /home/amol/Documents/ISO/oraclelinu66-2.qcow2 6G
```

Formatting '/home/amol/Documents/ISO/oraclelinu66-2.qcow2', fmt=qcow2 size=6442450944 encryption=off cluster_size=65536 lazy_refcounts=off refcount_bits=16

 
* Create a VM 

```
virt-install --name oracle66-2 --vcpus=1 --ram=2048 --disk path=/home/amol/Documents/ISO/oraclelinu66-2.qcow2,bus=virtio,cache=writeback --graphics vnc,listen=0.0.0.0 --network bridge:virbr0,model=virtio --noautoconsole --os-type=linux --os-variant=rhel6 --location=/home/amol/Documents/ISO/OracleLinux66_V52218-01.iso

Starting install...
Retrieving file .treeinfo...                                                                                                   | 1.5 kB  00:00:00     
Retrieving file vmlinuz...                                                                                                     | 4.0 MB  00:00:00     
Retrieving file initrd.img...                                                                                                  |  34 MB  00:00:00     
Creating domain...                                                                                                             |    0 B  00:00:01     
Domain installation still in progress. You can reconnect to 
the console to complete the installation process.
```

* Modify file content inside a Virtual Machine using guestfish

```
guestfish --rw -a /tmp/OracleLinux-66.img -i edit /etc/ssh/sshd_config
```

* Remove a package from qcow2 image, without booting into the Virtual Machine

```
guestfish --rw -a /tmp/OracleLinux-66.img -i command "yum remove cloud-init -y"

Loaded plugins: security
Setting up Remove Process
Resolving Dependencies
--> Running transaction check
---> Package cloud-init.noarch 0:0.7.4-2.el6 will be erased
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package             Arch            Version               Repository      Size
================================================================================
Removing:
 cloud-init          noarch          0.7.4-2.el6           @epel          1.7 M

Transaction Summary
================================================================================
Remove        1 Package(s)

Installed size: 1.7 M
Downloading Packages:
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Erasing    : cloud-init-0.7.4-2.el6.noarch                                1/1
  Verifying  : cloud-init-0.7.4-2.el6.noarch                                1/1

Removed:
  cloud-init.noarch 0:0.7.4-2.el6

Complete!
```

* Modify user password for a VM using guestfish  

```
guestfish --rw -a /tmp/OracleLinux-66.img -i command "bash -c 'echo admin:amol123 | chpasswd'"
```

* Attach a new interface to an existing VM 

```
virsh attach-interface --domain oracle66-2 --type bridge --source br-new --model virtio
```

This will attach an additional bridge interface to the VM and same will be visible as a new interface inside the VM.

 





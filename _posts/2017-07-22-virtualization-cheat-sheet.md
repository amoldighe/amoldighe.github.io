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

* Create a disk image for virtual machine

```
qemu-img create -f qcow2 /home/amol/Documents/ISO/oraclelinu66-2.qcow2 6G

Formatting '/home/amol/Documents/ISO/oraclelinu66-2.qcow2', fmt=qcow2 size=6442450944 encryption=off cluster_size=65536 lazy_refcounts=off refcount_bits=16
```
 
* Create a domain and spawn a virtual machine 

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

* Attach a new interface to an existing virtual machine 

```
virsh attach-interface --domain oracle66-2 --type bridge --source br-new --model virtio
```

This will attach an additional bridge interface to the VM and same will be visible as a new interface inside the VM.

* Logs location for openstack VM 
 
Openstack would crreate a VM log file at the below path with id of the VM

```
/var/lib/nova/instances/1212b55f-5a62-406a-89ca-706c03657251/console.log
```

qemu will create a log file at the below path with id of the instance 

```
/var/log/libvirt/qemu/instance-000039c8.log
```

The instance & openstack VM id can be grep from the qemu process  

```
root@cpu-node:~# ps -ef | grep instance-000039c8

root     50890 32187  0 19:50 pts/3    00:00:00 grep --color=auto instance-000039c8

libvirt+ 64262     1  2 Jun01 ?        13:03:17 qemu-system-x86_64 -enable-kvm -name instance-000039c8 -S -machine pc-i440fx-trusty,accel=kvm,usb=off -cpu host -m 65536 -realtime mlock=off -smp 16,sockets=16,cores=1,threads=1 -uuid 1212b55f-5a62-406a-89ca-706c03657251 -smbios type=1,manufacturer=OpenStack Foundation,product=OpenStack Nova,version=12.0.4,serial=34383137-3630-4753-4835-3531584c3753,uuid=1212b55f-5a62-406a-89ca-706c03657251,family=Virtual Machine -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/instance-000039c8.monitor,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=discard -no-hpet -no-shutdown -boot strict=on -device piix3-usb-uhci,id=usb,bus=pci.0,addr=0x1.0x2 -drive file=rbd:volumes/volume-afa54dd1-dd47-40a5-acd6-5ff4d653277e:id=volumes:key=AQDgMEFY27UcERAAX4p8GvgdTytz0i/aoMK7JA==:auth_supported=cephx\;none:mon_host=192.168.12.130\:6789\;192.168.12.131\:6789\;192.168.12.132\:6789,if=none,id=drive-virtio-disk0,format=raw,serial=afa54dd1-dd47-40a5-acd6-5ff4d653277e,cache=writeback -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1 -drive file=rbd:volumes/volume-55d25150-3c43-49e3-ac44-d3b64ee53320:id=volumes:key=AQDgMEFY27UcERAAX4p8GvgdTytz0i/aoMK7JA==:auth_supported=cephx\;none:mon_host=192.168.12.130\:6789\;192.168.12.131\:6789\;192.168.12.132\:6789,if=none,id=drive-virtio-disk1,format=raw,serial=55d25150-3c43-49e3-ac44-d3b64ee53320,cache=writeback -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x5,drive=drive-virtio-disk1,id=virtio-disk1 -netdev tap,ifname=tapa42f54ba-91,script=,id=hostnet0,vhost=on,vhostfd=27 -device virtio-net-pci,netdev=hostnet0,id=net0,mac=02:a4:2f:54:ba:91,bus=pci.0,addr=0x3 -chardev file,id=charserial0,path=/var/lib/nova/instances/1212b55f-5a62-406a-89ca-706c03657251/console.log -device isa-serial,chardev=charserial0,id=serial0 -chardev pty,id=charserial1 -device isa-serial,chardev=charserial1,id=serial1 -device usb-tablet,id=input0 -vnc 0.0.0.0:3 -k en-us -device cirrus-vga,id=video0,bus=pci.0,addr=0x2 -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x6 -msg timestamp=on
```






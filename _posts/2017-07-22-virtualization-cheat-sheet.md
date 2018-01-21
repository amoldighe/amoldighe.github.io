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

* Tools for playing with qemu-kvm disk image, disk partitions, filesystem

There are virt & qemu tools available to tweak disk images :

[virt-resize](https://linux.die.net/man/1/virt-resize)

[qemu-img](https://linux.die.net/man/1/qemu-img)

While disk partition can be altered with :

[growpart](https://www.systutorials.com/docs/linux/man/1-growpart/)

Next is the filesystem which can be resized using :

[resize2fs](https://linux.die.net/man/8/resize2fs)

Tuning filesystem parameters:

[tune2fs](http://landoflinux.com/linux_tune2fs_command.html)

Mounting VM disk image:

[qemu-nbd](https://www.jamescoyle.net/tag/qemu-nbd)

 
* Command for KVM - Guest / Virtual Machine stat check

List VM and fetch its hardware status information

```
root@hp-envy:/mnt/srcVM# virsh list 
 Id    Name                           State
----------------------------------------------------
 1     ubuntu14                       running

root@hp-envy:/mnt/srcVM# virsh dominfo ubuntu14
Id:             1
Name:           ubuntu14
UUID:           f55e0174-6e75-4ed8-a09c-705cacfe8653
OS Type:        hvm
State:          running
CPU(s):         1
CPU time:       8.9s
Max memory:     1048576 KiB
Used memory:    1048576 KiB
Persistent:     yes
Autostart:      disable
Managed save:   no
Security model: apparmor
Security DOI:   0
Security label: libvirt-f55e0174-6e75-4ed8-a09c-705cacfe8653 (enforcing)
```

Get VM CPU details:

```
root@hp-envy:~# virsh domstats --vcpu ubuntu14
Domain: 'ubuntu14'
  vcpu.current=1
  vcpu.maximum=1
  vcpu.0.state=1
  vcpu.0.time=7560000000
  vcpu.0.wait=0
  vcpu.0.halted=yes

"vcpu.current" - current number of online virtual CPUs
"vcpu.maximum" - maximum number of online virtual CPUs
"vcpu.<num>.state" - state of the virtual CPU <num>
"vcpu.<num>.time" - virtual cpu time spent by virtual CPU <num> (in microseconds)
"vcpu.<num>.wait" - virtual cpu time spent by virtual CPU <num> waiting on I/O (in microseconds)
"vcpu.<num>.halted" - virtual CPU <num> is halted: yes or no 
``` 

```
root@hp-envy:~# virsh domstats --cpu-total ubuntu14
Domain: 'ubuntu14'
  cpu.time=9323162046
  cpu.user=1400000000
  cpu.system=2220000000

"cpu.time" - total cpu time spent for this domain in nanoseconds
"cpu.user" - user cpu time spent in nanoseconds
"cpu.system" - system cpu time spent in nanoseconds

```

VM memory stats

```
root@hp-envy:~# virsh dommemstat  ubuntu14
actual 1048576
swap_in 0
last_update 1514374267
rss 288460

swap_in           - The amount of data read from swap space (in kB)
actual            - Current balloon value (in KB)
rss               - Resident Set Size of the running domain's process (in kB)
last-update       - Timestamp of the last update of statistics (in seconds)
```

Print a table showing the brief information of all block devices associated with domain.

```
root@hp-envy:~# virsh domblklist  ubuntu14
Target     Source
------------------------------------------------
vda        /var/lib/libvirt/images/ubuntu-server-1404.img
hda        /var/lib/libvirt/images/config-node.iso
```

Get block device size info for a domain. 

```
root@hp-envy:~# virsh domblkinfo  ubuntu14 vda --human
Capacity:       2.199 GiB
Allocation:     393.324 MiB
Physical:       393.375 MiB

root@hp-envy:~# virsh domblkinfo  ubuntu14 hda --human
Capacity:       364.000 KiB
Allocation:     364.000 KiB
Physical:       364.000 KiB
```

VM block device stats.

```
root@hp-envy:~# virsh domblkstat  ubuntu14 --human
Device: 
 number of read operations:      4735
 number of bytes read:           104040094
 number of write operations:     245
 number of bytes written:        4592640
 number of flush operations:     28
 total duration of reads (ns):   38175594855
 total duration of writes (ns):  1831031913
 total duration of flushes (ns): 5564971530
```

List interface used for the Virtual Machine

```
root@hp-envy:~# virsh domiflist ubuntu14
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      bridge     virbr0     virtio      52:54:00:e4:53:22
```

List the Virtual Machine data transfer details.
 
```
root@hp-envy:~# virsh domifstat ubuntu14 vnet0
vnet0 rx_bytes 142202
vnet0 rx_packets 2626
vnet0 rx_errs 0
vnet0 rx_drop 0
vnet0 tx_bytes 6300
vnet0 tx_packets 59
vnet0 tx_errs 0
vnet0 tx_drop 0
```

List the active network 

```
root@hp-envy:~# virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
```

Get a list of dhcp leases for all network interfaces connected to the given virtual network

```
root@hp-envy:~# virsh net-dhcp-leases default 
 Expiry Time          MAC address        Protocol  IP address                Hostname        Client ID or DUID
-------------------------------------------------------------------------------------------------------------------
 2017-12-27 20:15:35  52:54:00:e4:53:22  ipv4      192.168.122.238/24        cfg01           -
```



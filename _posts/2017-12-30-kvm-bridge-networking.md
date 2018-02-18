---
layout: post
title:  "KVM Networking - Bridge"
date:   2017-12-30
tags:
  - KVM
  - ubuntu
  - virsh
  - Bridge
  - virbr
  - vnet
  - virtual bridge
  - virt-install
  - virsh net-dumpxml
  - netplan
  - ubuntu 17.10
  - yaml
  - netplan apply
  - brctl show
---

In a bridged network, the guest VM is on the same network as your host machine i.e. if your host machine IP is 192.168.122.25 then your VM will have a IP address like 192.168.122.30. The virtual machine can be accessed by all computers in your host network as it is part of same network IP range. 

To explore bridge interface, I have setup a Ubuntu Desktop VM (hostname = ubuntu-desk) on my Laptop which will act as a HOST machine. This host machine ubuntu-desk has a VM running on it in bridge mode.  


* Add the bridge interface on host machine 

```
root@ubuntu-desk:~# brctl addbr vmbr0

root@ubuntu-desk:~# ip a | grep vmbr0 -B6
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:d9:22:19 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.192/24 brd 192.168.122.255 scope global dynamic ens3
       valid_lft 2254sec preferred_lft 2254sec
    inet6 fe80::1a0c:acc4:1487:b3a5/64 scope link 
       valid_lft forever preferred_lft forever
3: vmbr0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether de:84:3e:7a:06:4f brd ff:ff:ff:ff:ff:ff                                                                                                  
```

* Attach it to the physical interface

```
root@ubuntu-desk:~# brctl addif vmbr0 ens3

root@ubuntu-desk:~# ip a | grep vmbr0 -A 4
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master vmbr0 state UP group default qlen 1000
    link/ether 52:54:00:d9:22:19 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.192/24 brd 192.168.122.255 scope global dynamic ens3
       valid_lft 2237sec preferred_lft 2237sec
    inet6 fe80::1a0c:acc4:1487:b3a5/64 scope link 
       valid_lft forever preferred_lft forever
3: vmbr0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 52:54:00:d9:22:19 brd ff:ff:ff:ff:ff:ff
```

You would notice vmbr0 is attached in your physical interface


* The bridge interface will also reflect in brctl show

```
root@ubuntu-desk:~# brctl show
bridge name	bridge id		STP enabled	interfaces
vmbr0		8000.525400d92219	no		ens3
```

* Run dhclient to get IP from dhcp server for vmbr0

```
root@ubuntu-desk:~# dhclient vmbr0
```

* Start the VM by connecting it to the vmbr0 bridge on host machine 

I have connected the bridge network directly to an existing VM on the host machine using virtual machine manager.
Following is the dump of it's xml.

```
root@ubuntu-desk:~# virsh list 
 Id    Name                           State
----------------------------------------------------
 1     br-node1                       running

```

```
root@ubuntu-desk:~# virsh dumpxml 1 | grep bridge -A5
    <interface type='bridge'>
      <mac address='52:54:00:4a:f2:00'/>
      <source bridge='vmbr0'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
```

* brctl will show the virtual interface to the bridge is UP

```
root@ubuntu-desk:~# brctl show
bridge name	bridge id		STP enabled	interfaces
vmbr0		8000.525400d92219	no		ens3
                                                        vnet0           
```

* Following interfaces are created on the host machine.  

```
root@ubuntu-desk:~# ip a | grep vmbr0 -A11

2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master vmbr0 state UP group default qlen 1000
    link/ether 52:54:00:d9:22:19 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.192/24 brd 192.168.122.255 scope global dynamic ens3
       valid_lft 3543sec preferred_lft 3543sec
    inet6 fe80::1a0c:acc4:1487:b3a5/64 scope link 
       valid_lft forever preferred_lft forever
3: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:d9:22:19 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.192/24 brd 192.168.122.255 scope global vmbr0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fed9:2219/64 scope link 
       valid_lft forever preferred_lft forever
4: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master vmbr0 state UNKNOWN group default qlen 1000
    link/ether fe:54:00:4a:f2:00 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:fe4a:f200/64 scope link 
       valid_lft forever preferred_lft forever
```

* The VM on the host machine has a IP on the same network 192.168.122.xxx

<img src="{{ site.baseurl }}/img/vm-bridge.png">


What we saw above is a temporary solution to build & attach a bridge network.
For a permanent solution the bridge interface details need to be added to /etc/network/interfaces file on the host machine.

In my case my host machine is running Ubuntu 17.10 Desktop version, where I discovered that network configuration is managed by Netplan instead of the /etc/network/interfaces file. Digging further on Dr Google, led me to few good webpages which I have listed below, this gave me an understanding as where to add this new yaml configuration in netplan for adding an permanent bridge interface.

* Add a new network configuration YAML file 

``` 
root@ubuntu-desk:~# cat  /etc/netplan/01-network-vmbr0.yaml
network:
    version: 2
    renderer: networkd
    ethernets:
        ens3:
            dhcp4: true
    bridges:
        vmbr0:
            interfaces: [ens3]
            dhcp4: true
            parameters:
                stp: false
                forward-delay: 0
```

Ubuntu desktop uses Network Manager as renderer for network configuration yaml. The network manager GUI can be accessed through nm-connection-editor command to configure/add a new network interface. I did not want to use Network Manager GUI and use a configuration file to make my bridge configuration persistent, hence I thought of using networkd as the renderer for to manage my network configuration to setup a permanent bridge interface.

* Apply the configuration in yaml file.

``` 
root@ubuntu-desk:~#netplan apply
```
 
* This should bring up the bridge interface post every restart

```
root@ubuntu-desk:~# ip a | grep vmbr0 -A 10

2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master vmbr0 state UP group default qlen 1000
    link/ether 52:54:00:d9:22:19 brd ff:ff:ff:ff:ff:ff
3: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 0e:32:57:ff:6f:f5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.133/24 brd 192.168.122.255 scope global dynamic vmbr0
       valid_lft 3267sec preferred_lft 3267sec
    inet6 fe80::c32:57ff:feff:6ff5/64 scope link 
       valid_lft forever preferred_lft forever
4: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master vmbr0 state UNKNOWN group default qlen 1000
    link/ether fe:54:00:4a:f2:00 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:fe4a:f200/64 scope link 
       valid_lft forever preferred_lft forever
```


* ***Reference Links***

[https://wiki.ubuntu.com/Netplan](https://wiki.ubuntu.com/Netplan)

[https://websiteforstudents.com/configure-static-ip-addresses-on-ubuntu-18-04-beta/](https://websiteforstudents.com/configure-static-ip-addresses-on-ubuntu-18-04-beta/)

[https://www.dedoimedo.com/computers/kvm-bridged.html](https://www.dedoimedo.com/computers/kvm-bridged.html)

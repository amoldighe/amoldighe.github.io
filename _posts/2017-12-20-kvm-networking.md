---
layout: post
title:  "KVM Networking"
date:   2017-12-20
tags:
  - KVM
  - trusty-server-cloudimg-amd64-disk1.img
  - ubuntu
  - virsh
  - NAT
  - Host-Only
  - Bridge
  - virbr
  - vnet
  - virtual bridge
  - virt-install
  - virsh net-autostart
  - virsh net-dumpxml
  - virsh net-dhcp-leases
  - virsh net-define
  - virsh net-list
  - virsh net-info
  - virsh net-start
  - brctl show
---


KVM - Networking Adding NAT interface & Host Only Interface

As the title suggest, we are going to explore networking for Virtual (guest) machines and host machine, specifically NAT interface and Host Only Interface. I will be covering Bridge networking in a later post. To start with, I am going to list a few concepts for better understanding of kvm networking. Lets go through the type of networks a Virtual Machine can be attached to:

* Host-Only: The VM will be assigned a IP which is only accessible by the host machine where the VM is running on. Only the host machine can talk to the VM, no other host can access it.

* NAT: The VM will be assigned a IP on different subnet that the host machine, but the VM has ability to talk to ouside network just like the host machine, but hosts from outside cannot access to your VM directly.

* Bridged: The VM will be in the same network as your host, if your host IP is 192.168.100.25 then your VM will be like 192.168.100.30. It can be accessed by all computers in your host network.

The below interfaces will be noticed on the host machine:  

virbr is a virtual bridge interfaces provided by libvirt library. It's role is to act like a bridge, it switch packets (at layer 2) between the interfaces (real or other) that are attached. There is a gateway IP attached to this virtual bridge which is visible on the host machine. The bridge is connected to a virtual router which provides DHCP which basically gives the virtual machine an IP address on that subnet which the bridge connects to.

vnet interfaces is a type of virtual interface called tap interfaces. It is attached to a qemu-kvm emulator process for reading-writing data between the host and the VM. A vnet will be added to a bridge interface for plugging the VM into virtual network. 


* Currently have a default network

```
root@amol-hp-elite:~# virsh net-list --all
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
```

* Get details of default network

```
root@amol-hp-elite:~# virsh net-info default
Name:           default
UUID:           8f7e6477-61df-4468-83fc-966d41d22302
Active:         yes
Persistent:     yes
Autostart:      yes
Bridge:         virbr0
```

* Let see the virtual bridge interface  

```
root@amol-hp-elite:~# brctl show 
bridge name     bridge id               STP enabled     interfaces
virbr0          8000.52540035ac06       yes             virbr0-nic
                                                        vnet0
                                                        vnet1
```

virbr0 virtual bridge is attached to interface vnet0 & vnet1 

* Let see what type of network is configured on default network

```
root@amol-hp-elite:~# virsh net-dumpxml default
<network connections='1'>
  <name>default</name>
  <uuid>8f7e6477-61df-4468-83fc-966d41d22302</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:35:ac:06'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```

The output indicates that default network is configured a forward for NAT. Hence the VM on this network should be able to connect to outside networks.

* Check the VM's and IP's assigned on this network

```
root@amol-hp-elite:~# virsh net-dhcp-leases default
 Expiry Time          MAC address        Protocol  IP address                Hostname        Client ID or DUID
-------------------------------------------------------------------------------------------------------------------
 2018-01-21 12:56:45  52:54:00:58:e4:7f  ipv4      192.168.122.113/24        ubuntu1404      -
 2018-01-21 12:56:45  52:54:00:ae:45:66  ipv4      192.168.122.181/24        node-1          -
```

The above explains the default NAT network, lets track steps to create another NAT network.
  

* Create a network configuration xml

```
root@amol-hp-elite:~# cat nat.xml
<network>
  <name>natntw</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr1' stp='on' delay='0'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.10' end='192.168.100.254'/>
    </dhcp>
  </ip>
</network>
```
We have define the network name, network mode is set to NAT, a new bridge name as virbr1, network gateway, network subnet mask and IP range.

* Define the new network

```
root@amol-hp-elite:~# virsh net-define nat.xml                             
Network natntw defined from nat.xml  
```

Network bridge details are setup as soon as the new network is defined.

```
root@amol-hp-elite:~# virsh net-info natntw                                
Name:           natntw               
UUID:           0c3b9a99-c14c-4317-84c4-e2fe245becbc                       
Active:         no                   
Persistent:     yes                  
Autostart:      no                   
Bridge:         virbr1               
```

* Start the network 

```
root@amol-hp-elite:~# virsh net-start natntw                                                                                                          
Network natntw started               
```

This should start the virtual bridge virbr1 on the host machine

```
8: virbr1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000                                              
    link/ether 52:54:00:b8:ff:b7 brd ff:ff:ff:ff:ff:ff                     
    inet 192.168.100.1/24 brd 192.168.100.255 scope global virbr1          
       valid_lft forever preferred_lft forever                             
```

* Set network to autostart

```
root@amol-hp-elite:~# virsh net-autostart natntw
Network natntw marked as autostarted
```

* I am going to repeat the above steps to create a host-only (isolated) network for VMs using the below xml file

```
root@amol-hp-elite:~# cat hostonly.xml
<network>
  <name>hostntw</name>
  <bridge name='virbr2' stp='on' delay='0'/>
  <ip address='192.168.110.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.110.10' end='192.168.110.254'/>
    </dhcp>
  </ip>
</network>
```

* Virtual bridge virbr2 is started after starting the host only network - hostntw 

```
root@amol-hp-elite:~# ip a | grep virbr2 -A 7
10: virbr2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:48:7b:d9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.110.1/24 brd 192.168.110.255 scope global virbr2
       valid_lft forever preferred_lft forever
11: virbr2-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr2 state DOWN group default qlen 1000
    link/ether 52:54:00:48:7b:d9 brd ff:ff:ff:ff:ff:ff
```

* Lets list all the virtual networks

```
root@amol-hp-elite:~# virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
 hostntw              active     yes           yes
 natntw               active     yes           yes
```

* I have created a VM and attached to the hostntw network, let see it's behaviour

```
root@amol-hp-elite:~# virt-install --name node-2 --vcpus=1 --memory 512 --disk path=/var/lib/libvirt/images/trusty-server-cloudimg-amd64-disk1-node2.i
mg,bus=virtio,cache=writeback --disk path=/var/lib/libvirt/images/config-drive-node2.iso,device=cdrom --graphics vnc,listen=0.0.0.0 --network bridge:v
irbr2,model=virtio --noautoconsole --os-type=linux --boot=hd                                                                                          
WARNING  No operating system detected, VM performance may suffer. Specify an OS with --os-variant for optimal results.                                

Starting install...                  
Creating domain...                                                                                                             |    0 B  00:00:00     
Domain creation completed.           
```

As we are aware that virbr2 is the bridge attached to hostntw, we should get an interface attached on hostntw connected to the VM's process.

```
root@amol-hp-elite:/var/lib/libvirt/images# brctl show virbr2
bridge name     bridge id               STP enabled     interfaces
virbr2          8000.525400487bd9       yes             virbr2-nic
                                                        vnet2
```

The vnet2 interface is attached to the VM - node-2

```
root@amol-hp-elite:~# virsh dumpxml node-2 | grep vnet -B 3 -A 4
    <interface type='bridge'>
      <mac address='52:54:00:d7:8d:61'/>
      <source bridge='virbr2'/>
      <target dev='vnet2'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
```

IP assigned to VM on hostntw 

```
root@amol-hp-elite:/var/lib/libvirt/images# virsh net-dhcp-leases hostntw
 Expiry Time          MAC address        Protocol  IP address                Hostname        Client ID or DUID
-------------------------------------------------------------------------------------------------------------------
 2018-01-21 14:26:15  52:54:00:d7:8d:61  ipv4      192.168.110.116/24        node-2          -
```

I am able to ssh to the VM but cannot ping outside the VM's hostonly network.

```
root@amol-hp-elite:~# ssh -i /home/amol/.ssh/id_rsa ubuntu@192.168.110.116 
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 3.13.0-139-generic x86_64)        

 * Documentation:  https://help.ubuntu.com/                                

  System information as of Sun Jan 21 07:57:53 UTC 2018                    

  System load:  0.05              Processes:           71                  
  Usage of /:   36.5% of 2.13GB   Users logged in:     0                   
  Memory usage: 10%               IP address for eth0: 192.168.110.116     
  Swap usage:   0%                   

  Graph this data and manage this system at:                               
    https://landscape.canonical.com/ 

  Get cloud support with Ubuntu Advantage Cloud Guest:                     
    http://www.ubuntu.com/business/services/cloud                          

0 packages can be updated.           
0 updates are security updates.      


Last login: Sun Jan 21 07:57:56 2018 from 192.168.110.1                    
ubuntu@node-2:~$ ping google.com     
ping: unknown host google.com        
```

* Next we will try to attach a NAT interface/network to the VM -node-2 and try to reach outside through the NAT interface.

I will be using the newly created natntw network having the IP subnet 192.168.100.XXX to accomplish this.

``` 
root@amol-hp-elite:~# virsh net-info natntw
Name:           natntw
UUID:           0c3b9a99-c14c-4317-84c4-e2fe245becbc
Active:         yes
Persistent:     yes
Autostart:      yes
Bridge:         virbr1

root@amol-hp-elite:~# virsh net-dhcp-leases natntw
 Expiry Time          MAC address        Protocol  IP address                Hostname        Client ID or DUID
-------------------------------------------------------------------------------------------------------------------


root@amol-hp-elite:~# brctl show virbr1
bridge name     bridge id               STP enabled     interfaces
virbr1          8000.525400b8ffb7       yes             virbr1-nic

```

* Attached natntw to VM node-2

```
root@amol-hp-elite:~# virsh attach-interface --domain node-2 --type bridge --source virbr1 --model virtio --config                                    
Interface attached successfully      
```

Please note - I had to restart the VM, as restarting networking did not work, maybe because I had ssh'ed in the VM using eth0 

Virtual bridge got an vnet3 interface attached for the VM node-2 for network natntw

```
root@amol-hp-elite:~# brctl show virbr1                                                                                                               
bridge name     bridge id               STP enabled     interfaces         
virbr1          8000.525400b8ffb7       yes             virbr1-nic         
                                                        vnet3         
```

IP from natntw network was assigned to the VM

```
root@amol-hp-elite:~# virsh net-dhcp-leases natntw                         
 Expiry Time          MAC address        Protocol  IP address                Hostname        Client ID or DUID                                        
-------------------------------------------------------------------------------------------------------------------                                   
 2018-01-21 14:56:36  52:54:00:c6:3c:03  ipv4      192.168.100.197/24        node-2          -                                                        
```

* Test outside connectivity from inside the VM

```
root@amol-hp-elite:~# ssh -i /home/amol/.ssh/id_rsa ubuntu@192.168.110.116
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 3.13.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Sun Jan 21 08:26:31 UTC 2018

  System load: 0.0               Memory usage: 9%   Processes:       52
  Usage of /:  36.5% of 2.13GB   Swap usage:   0%   Users logged in: 0

  Graph this data and manage this system at:
    https://landscape.canonical.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.


Last login: Sun Jan 21 08:09:43 2018 from 192.168.110.1

ubuntu@node-2:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:d7:8d:61 brd ff:ff:ff:ff:ff:ff
    inet 192.168.110.116/24 brd 192.168.110.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fed7:8d61/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:c6:3c:03 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.197/24 brd 192.168.100.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fec6:3c03/64 scope link 
       valid_lft forever preferred_lft forever

ubuntu@node-2:~$ ping google.com
PING google.com (172.217.27.206) 56(84) bytes of data.
64 bytes from bom07s15-in-f14.1e100.net (172.217.27.206): icmp_seq=1 ttl=52 time=30.4 ms
64 bytes from bom07s15-in-f14.1e100.net (172.217.27.206): icmp_seq=2 ttl=52 time=69.5 ms
64 bytes from bom07s15-in-f14.1e100.net (172.217.27.206): icmp_seq=3 ttl=52 time=52.8 ms
^C
--- google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 30.477/50.960/69.533/16.003 ms

```

Ping to google.com is now working as it connecting to outside network using the newly attached NAT interface on VM's eth1.



* ***Reference Links***

[https://kashyapc.fedorapeople.org/virt/create-a-new-libvirt-bridge.txt](https://kashyapc.fedorapeople.org/virt/create-a-new-libvirt-bridge.txt)
[https://jamielinux.com/docs/libvirt-networking-handbook/nat-based-network.html](https://jamielinux.com/docs/libvirt-networking-handbook/nat-based-network.html)
[https://libvirt.org/formatnetwork.html#examplesNAT](https://libvirt.org/formatnetwork.html#examplesNAT)





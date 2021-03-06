---
layout: post
title:  "Cacti Setup & Graphing"
date:   2017-05-23
tags:
  - cacti
  - shell
  - graphs
  - monitoring
---

I am using Cacti for creating graphs for cpu, memory, load average and network traffic for various private cloud nodes. We have multiple cloud environment and have 200+ node for each cloud. Due to the large number of nodes, its time consuming to discover each node in Cacti dashboard and configure graph through dashboard. Below is my solution to setup Cacti nodes, discover them and create graphs for same. 

* **Cacti Server Installation** 

> sudo apt-get install cacti cacti-spine rrdtool

Cacti dashboard should be available on the server IP.

* **Cacti client installation on multiple nodes** 

In our infrastructure we have different type of nodes like - compute, controller, storage. We also have a config node which has salt installed. Each of this type further having multiple nodes e.g. compute1, compute2.... controller1, controller2...
The below ***for*** loop along with ***salt*** will allow to execute commands on all nodes of different type of node e.g.
  
> for i in compute-nodes controller-nodes storage-nodes ; do salt "$i*" cmd.run "lsb_release -a"; done

**OR** we can take one type of node and execute command using salt from config node.

> salt 'cpu*' cmd.run 'apt-get update'
> 
> salt 'cpu*' cmd.run 'apt-get install snmp snmpd smistrip --dry-run'
> 
> salt 'cpu*' cmd.run 'apt-get install snmp snmpd smistrip -y'
> 
> salt 'cpu*' cmd.run 'service snmpd stop'
> 
> salt 'cpu*' cmd.run 'service snmpd status'
> 
> salt 'cpu*' cmd.run 'apt-get install libsnmp-dev -y'
> 
> salt 'cpu*' cmd.run 'net-snmp-config --create-snmpv3-user -ro -A
> clouddevops jiocloudservices'
> 
> salt-cp 'cpu*' /home/amold/snmpd.conf /etc/snmp/
> 
> salt 'cpu*' cmd.run 'service snmpd start'

* **Incase you have firewall on nodes, add to the following rules to iptables to allow UDP communication for SNMP**

> iptables -I INPUT -p udp -m udp --dport 161 -j ACCEPT
> 
> iptables -I INPUT -p udp -m udp --dport 162 -j ACCEPT 
> 
> iptables-save > /etc/iptables/rules.v4
> 

 * **Cacti client node addition to Cacti Dashboard :**

Login to cacti server, to add the clients Hostname and Host IP to cacti dashboard.

Get the list of client hostname & host IP address in a file in following format :

>  clientnode1 <space> 192.168.0.1
> 
>  clientnode2 <space> 192.168.0.2
> 
>  clientnode3 <space> 192.168.0.3
> 

Run below command to fetch the hostname - IP from file and add to cacti dashboard by piping to shell

> less /tmp/control-nodes | awk '{print "sudo php -q
> /usr/share/cacti/cli/add_device.php --description=" \$1 " --ip="$2 "
> --avail=snmp --template=9 --community=jiocloudservices"}' | sh

This should add the host to cacti dashboard and assign a unique id, which will be used for addition of graphs for cpu, memory, load and interface.

* **Addition of cpu, memory, load average graphs** (**script method**)

> bash ./[cacti-cpu-mem-graph.sh](https://github.com/amoldighe/cacti-setup-add-graph/blob/master/cacti-cpu-mem-graph.sh)    host-id-list-file

**OR**

* **Addition of cpu, memory, load average graphs** (**manual method**)

List the id's for all the hosts added to cacti in the earlier step, use the below command

> sudo php -q /usr/share/cacti/cli/add_graphs.php --list-hosts

List graph template to be used 

> sudo php -q /usr/share/cacti/cli/add_graphs.php --list-graph-templates

Use the below comands for genarating graphs for cpu, load average, memory using the above graph templates

> sudo php -q /usr/share/cacti/cli/add_graphs.php  --host-id=host-id
> --graph-template-id=40 --graph-type=cg 
> 
> sudo php -q /usr/share/cacti/cli/add_graphs.php  --host-id=host-id
> --graph-template-id=41 --graph-type=cg 
> 
> sudo php -q /usr/share/cacti/cli/add_graphs.php  --host-id=host-id
> --graph-template-id=42 --graph-type=cg

* **Addition of interface graphs** (***script method***)

> bash ./[cacti-interface-graph.sh](https://github.com/amoldighe/cacti-setup-add-graph/blob/master/cacti-interface-graph.sh)   host-id-list-file

**OR**

* **Addition of interface graphs** (***manual method***)

List the snmp queries to be used while adding graphs for interface 

> sudo php -q /usr/share/cacti/cli/add_graphs.php --list-snmp-queries

List the snmp field type for interface graphing

> sudo php -q /usr/share/cacti/cli/add_graphs.php --host-id=47 --list-snmp-fields

List all the interfaces with IP

> sudo php -q /usr/share/cacti/cli/add_graphs.php --host-id=47 --snmp-field=ifIP --list-snmp-values 

Add interface graph creation to Cacti dashboard

> sudo php -q /usr/share/cacti/cli/add_graphs.php --host-id=47
> --snmp-query-id=1 --snmp-query-type-id=13 --snmp-field=ifIP --snmp-value=10.144.166.171 --graph-template-id=2 --graph-type=ds
> 
> sudo php -q /usr/share/cacti/cli/add_graphs.php --host-id=47
> --snmp-query-id=1 --snmp-query-type-id=13 --snmp-field=ifIP --snmp-value=192.168.0.120 --graph-template-id=2 --graph-type=ds
> 
> sudo php -q /usr/share/cacti/cli/add_graphs.php --host-id=47
> --snmp-query-id=1 --snmp-query-type-id=13 --snmp-field=ifIP --snmp-value=192.168.2.171 --graph-template-id=2 --graph-type=ds
> 
> sudo php -q /usr/share/cacti/cli/add_graphs.php --host-id=47
> --snmp-query-id=1 --snmp-query-type-id=13 --snmp-field=ifIP --snmp-value=192.168.4.171 --graph-template-id=2 --graph-type=ds

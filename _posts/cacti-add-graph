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

* **Incase you have firewall on nodes, add to the following rules to iptables**

> iptables -I INPUT -p udp -m udp --dport 161 -j ACCEPT
> iptables -I INPUT -p udp -m udp --dport 162 -j ACCEPT 
> iptables-save > /etc/iptables/rules.v4

 * **Cacti client node addition to Cacti Dashboard :**

Login to cacti server, to add all the clients to cacti dashboard.

Get the list of client machine name & IP address in a file in following format - 
    	clientnode1 <space> 192.168.0.1
    	clientnode2 <space> 192.168.0.2
    	clientnode3 <space> 192.168.0.3

Run below command to fetch the hostname - IP from file and add to cacti dashboard by piping to shell

> less /tmp/control-nodes | awk '{print "sudo php -q
> /usr/share/cacti/cli/add_device.php --description=" \$1 " --ip="$2 "
> --avail=snmp --template=9 --community=jiocloudservices"}' | sh

* **Addition of cpu, memory, load average graphs** (**script method**)

> bash ./cacti-cpu-mem-graph.sh    host-id-list-file

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

> bash ./cacti-interface-graph.sh   host-id-list-file

**OR**

* **Addition of interface graphs** (***manual method***)

List the snmp queries to be used while adding graphs for interface 

> sudo php -q /usr/share/cacti/cli/add_graphs.php --list-snmp-queries
>
> Known SNMP Queries:(id, name) 
> 1       SNMP - Interface Statistics 
> 2       ucd/net -  Get Monitored Partitions
> 3       Karlnet - Wireless Bridge Statistics 
> 4       Netware - Get Available Volumes 
> 6       Unix - Get Mounted Partitions 
> 7       Netware - Get Processor Information 
> 8       SNMP - Get Mounted Partitions 
> 9       SNMP - Get Processor Information


List the snmp field type for interface graphing

> (jpe2)amold@consul-03:~$ sudo php -q
> /usr/share/cacti/cli/add_graphs.php --host-id=47 --list-snmp-fields
>
> Known SNMP Fields for host-id 47: (name) 
> hrStorageAllocationUnits
> hrStorageDescr 
> hrStorageIndex 
> ifAlias 
> ifDescr 
> ifHighSpeed 
> ifHwAddr
> ifIndex 
> ifIP 
> ifName 
> ifOperStatus 
> ifSpeed 
> ifType

List all the interfaces with IP

> (jpe2)amold@consul-03:~$ sudo php -q
> /usr/share/cacti/cli/add_graphs.php --host-id=47 --snmp-field=ifIP
> --list-snmp-values 
> 
>Known values for ifIP for host 47: (name)
> 10.144.166.171
> 127.0.0.1
> 192.168.0.120
> 192.168.2.171
> 192.168.4.171

List all the interfaces with IP

> (jpe2)amold@consul-03:~$ sudo php -q
> /usr/share/cacti/cli/add_graphs.php --host-id=47 --snmp-field=ifIP
> --list-snmp-values 
> 
> Known values for ifIP for host 47: (name)
> 10.144.166.171
> 127.0.0.1
> 192.168.0.120
> 192.168.2.171
> 192.168.4.171

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

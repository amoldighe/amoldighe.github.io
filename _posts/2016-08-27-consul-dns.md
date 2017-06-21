---
layout: post
title: Consul DNS
---

Our Private Cloud setup is using Consul for service discovery and monitoring. While working with Consul, I thought of trying to setup Consul DNS, which was never adopted on our cloud environment due to lack of a solid use case for same. Nevertheless it was a good learning, and I thought of documenting Consul DNS, as the information was scattered across various websites.

In this post, I will try to address: 
* Consul DNS setup 
* Consul DNS forward using dnsmasq

As a default behaviour, Consul will listen on 127.0.0.1:8600 for DNS queries i.e. on the localhost where consul agent is running. The consul DNS service works in the top level domain of “consul.”. The logical concept of data centers is used to subdivide the domain. The datacenter must be configured for each Consul service. Each datacenter contains two namespaces - “nodes” and “services”. Every node is automatically registered with its name in the nodes.<datacenter>.consul zone and services live in the services.<datacenter>.consul.zone. 

I have a consul cluster setup of 3 servers & 2 clients

```
root@ubuntuserver3:~# consul members
Node           Address            Status  Type    Build  Protocol  DC
ubuntuclient1  192.168.56.5:8301  alive   client  0.7.0  2         consultest
ubuntuclient2  192.168.56.6:8301  alive   client  0.7.0  2         consultest
ubuntuserver1  192.168.56.2:8301  alive   server  0.7.0  2         consultest
ubuntuserver2  192.168.56.3:8301  alive   server  0.7.0  2         consultest
ubuntuserver3  192.168.56.4:8301  alive   server  0.7.0  2         consultest
```

As I had mentioned DNS works on consul nodes by default on port 8600, hence I can run a dig on anyone of my consul nodes for services or nodes.

The below dig response will give IP of all nodes running consul service on datacenter - consultest.

```
root@ubuntuserver3:~# dig @127.0.0.1 -p 8600 consul.service.consultest.consul. ANY

; <<>> DiG 9.9.5-3ubuntu0.8-Ubuntu <<>> @127.0.0.1 -p 8600 consul.service.consultest.consul. ANY
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15776
;; flags: qr aa rd; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;consul.service.consultest.consul. IN	ANY

;; ANSWER SECTION:
consul.service.consultest.consul. 0 IN	A	192.168.56.3
consul.service.consultest.consul. 0 IN	A	192.168.56.2
consul.service.consultest.consul. 0 IN	A	192.168.56.4

;; Query time: 4 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu Sep 29 21:34:35 IST 2016
;; MSG SIZE  rcvd: 98
```

The below dig response will give IP of a node for ubuntuserver1 under datacenter - consultest.

```
root@ubuntuclient2:~# dig @127.0.0.1 -p 8600 ubuntuserver1.node.consultest.consul. ANY

; <<>> DiG 9.9.5-3-Ubuntu <<>> @127.0.0.1 -p 8600 ubuntuserver1.node.consultest.consul. ANY
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 10921
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;ubuntuserver1.node.consultest.consul. IN ANY

;; ANSWER SECTION:
ubuntuserver1.node.consultest.consul. 0	IN A	192.168.56.2

;; Query time: 16 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu Sep 29 21:49:43 IST 2016
;; MSG SIZE  rcvd: 70
```

Ok ... the above is straight forward.

Lets assume I have an existing dns service like dnsmasq running on my environment and I also have consul dns setup. Now how do I make dns tools like dig & nslookup work with consul without distrupting my existing dns setup.

Dnsmasq setup on ubuntuserver2

```
root@ubuntuserver2:~# ps -axf | grep dnsmasq
 1762 pts/0    S+     0:00          \_ grep --color=auto dnsmasq
  948 ?        S      0:00 /usr/sbin/dnsmasq -x /var/run/dnsmasq/dnsmasq.pid -u dnsmasq -r /var/run/dnsmasq/resolv.conf -7 /etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new
```
  
using the following settings in /etc/dnsmasq.conf 

```
# Configuration file for dnsmasq.
#
log-queries
log-facility=/var/log/dnsmasq.log
addn-hosts=/etc/dnsmasq_static_hosts.conf
server=8.8.8.8
```

with a static host file defining the hostname - ip address mapping in /etc/dnsmasq_static_hosts.conf

```
192.168.56.2    ubuntuserver1    ubuntuserver1.local.tld
192.168.56.3    ubuntuserver2    ubuntuserver2.local.tld
192.168.56.4    ubuntuserver3    ubuntuserver3.local.tld
192.168.56.5    ubuntuclient1    ubuntuclient1.local.tld
192.168.56.6    ubuntuclient2    ubuntuclient2.local.tld
```

Hence nslookup for :

```
root@ubuntuclient2:~# nslookup ubuntuserver3.local.tld
Server:		192.168.56.3
Address:	192.168.56.3#53

Name:	ubuntuserver3.local.tld
Address: 192.168.56.4

root@ubuntuclient2:~# nslookup ubuntuserver2
Server:		192.168.56.3
Address:	192.168.56.3#53

Name:	ubuntuserver2
Address: 192.168.56.3
Name:	ubuntuserver2
Address: 127.0.1.1
```

But this will not resolve "consul." domain 

```
root@ubuntuclient2:~# nslookup ubuntuserver1.node.consultest.consul
;; connection timed out; no servers could be reached
```

To make this work with consul domain, we need to forward any query comming to our dns server for consul domain to consul port 8600

Edit /etc/dnsmasq.d/10-consul and add an entry for consul on port 8600

```
server=/consul/127.0.0.1#8600
```

Restart dnsmasq service 

```
root@ubuntuserver2:~# service dnsmasq restart
 * Restarting DNS forwarder and DHCP server dnsmasq                          [ OK ]
```

Please note /etc/resolv.conf in dnsmasq server need to have an entry for :

```
root@ubuntuserver2:~# cat /etc/resolv.conf 
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 127.0.0.1
```

While /etc/resolv.conf on other servers need to have an entry pointing to dnsmasq server

```
root@ubuntuclient2:~# cat /etc/resolv.conf 
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 192.168.56.3
```

Now you should be able to resolve consul domain along with domain names configured under dnsmasq

```
root@ubuntuclient2:~# nslookup ubuntuserver1.node.consultest.consul
Server:		192.168.56.3
Address:	192.168.56.3#53

Name:	ubuntuserver1.node.consultest.consul
Address: 192.168.56.2

root@ubuntuclient2:~# nslookup consul.service.consultest.consul
Server:		192.168.56.3
Address:	192.168.56.3#53

Name:	consul.service.consultest.consul
Address: 192.168.56.3
Name:	consul.service.consultest.consul
Address: 192.168.56.4
Name:	consul.service.consultest.consul
Address: 192.168.56.2

root@ubuntuclient2:~# nslookup ubuntuserver1
Server:		192.168.56.3
Address:	192.168.56.3#53

Name:	ubuntuserver1
Address: 192.168.56.2
```




                             



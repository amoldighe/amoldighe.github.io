---
layout: post
title:  "Consul for Monitoring"
date:   2017-06-25
tags:
  - consul
  - monitoring
  - ubuntu 14.04
  - key value store
  - serf 
  - raft
  - consul cluster
---

I usually do not write setup guides as there are abundant of websites doing a good job of explaining setup and configuration for a opensource project. For Consul, somehow I felt the need to record the setup that we are rocking for production monitoring using Consul since past one year. 
Consul is a versatile tool which can be used for Service Discovery, DNS, Key Value store.  We thought of leveraging Consul's service discovery mechanism and use it as our monitoring solution. Consul turns out to be a lightweight solution for setting up monitoring, as its memory & cpu footprint is relativity small compared to other monitoring solutions. It is available in a single binary which can be configured for a Server or Client setup.  Please note your Consul monitoring solution will be as good as how well you build your scripts which will be executed by consul and how you configure Consul.

The server cluster consist of a bootstrap server, server1 and server2.
Below configuration is for setting up the Consul server cluster.

* Copy the consul binary to /usr/local/bin/consul on all hosts

* Create the below directory structure for Consul setup
```
/etc/consul.d/bootstrap
/etc/consul.d/server/
/etc/consul.d/script/
/etc/consul.d/ssl
/etc/consul.d/ui
```
* Generate a encrypt key for Consul using command
```
> consul keygen
RYBg5+WxobPhqUpHW5RthA==
```
* Generate self signed certificate
Reference - http://www.thegeekstuff.com/2009/07/linux-apache-mod-ssl-generate-key-csr-crt-file

* Configure the Bootstrap Server
Create a configuration file /etc/consul.d/bootstrap/config.json with the below lines
```
  {
    "bootstrap" : true,
    "server" : true,
    "datacenter" : "DS1",
    "data_dir" : "/var/consul",
    "ui_dir" : "/etc/consul.d/ui",
    "encrypt" : "RYBg5+WxobPhqUpHW5RthA==",
    "enable_script_checks" : true,
    "ca_file": "/etc/consul.d/ssl/ca.cert",
    "cert_file": "/etc/consul.d/ssl/consul.cert",
    "key_file": "/etc/consul.d/ssl/consul.key",
    "log_level" : "INFO",
    "enable_syslog" : true,
    "bind_addr": "192.168.12.161",
    "start_join" : ["192.168.12.161", "192.168.12.162", "192.168.12.163"],
    "node_name": "consul-01 ",
  }
```
* Configure Server1
Create a configuration file /etc/consul.d/server/config.json with the below lines
```
{
    "bootstrap" : false,
    "server" : true,
    "datacenter" : "DS1",
    "data_dir" : "/var/consul",
    "ui_dir" : "/etc/consul.d/ui",
    "encrypt" : "RYBg5+WxobPhqUpHW5RthA==",
    "enable_script_checks" : true,
    "ca_file": "/etc/consul.d/ssl/ca.cert",
    "cert_file": "/etc/consul.d/ssl/consul.cert",
    "key_file": "/etc/consul.d/ssl/consul.key",
    "log_level" : "INFO",
    "enable_syslog" : true,
    "bind_addr": "192.168.12.162",
    "start_join" : ["192.168.12.161", "192.168.12.162", "192.168.12.163"],
    "node_name": "consul-02",
} 
```

I am using this server to expose http service which defaults on port 8500 to https 

* Configure Server2
Create a configuration file /etc/consul.d/server/config.json with the below lines
```
{
    "bootstrap" : false,
    "server" : true,
    "datacenter" : "DS1",
    "data_dir" : "/var/consul",
    "ui_dir" : "/etc/consul.d/ui",
    "encrypt" : "RYBg5+WxobPhqUpHW5RthA==",
    "enable_script_checks" : true,
    "ca_file": "/etc/consul.d/ssl/ca.cert",
    "cert_file": "/etc/consul.d/ssl/consul.cert",
    "key_file": "/etc/consul.d/ssl/consul.key",
    "log_level" : "INFO",
    "enable_syslog" : true,
    "bind_addr": "192.168.12.163",
    "start_join" : ["192.168.12.161", "192.168.12.162", "192.168.12.163"],
    "node_name": "consul-03",
}
```
* Configure the client

Create the following directories on client 
```
/etc/consul.d/script/
/etc/consul.d/client/
```
Create a configuration file /etc/consul.d/client/config.json with the below lines
```
{
    "bootstrap" : false,
    "server" : false,
    "datacenter" : "JSE-2",
    "data_dir" : "/var/consul",
    "encrypt" : "RYBg5+WxobPhqUpHW5RthA==",
    "log_level" : "INFO",
    "enable_syslog" : true,
    "bind_addr": "192.168.12.96",
    "start_join" : ["192.168.12.161", "192.168.12.162", "192.168.12.163"],
    "node_name": "mtr-01",
}
```

* Create data directory for consul 
The data directory is required on all servers & clients @ /var/consul/ 
Consul store the data required by serf & raft protocol in their respective directories. Consul uses a consensus protocol to provide consistency, this consensus protocol is based on Raft.Only consul server participate in raft to elect a leader. Consul uses a gossip protocol to manage membership and broadcast messages to the cluster, this gossip protocol is implemented by Serf.

* Autostart consul bootstrap on Ubuntu 14.04 
On 14.04 I am using upstart configuration to start consul on start up using the below configuration on server side.
Setup a server configuration file   -   /etc/init/consul.conf
```
# Consul Agent (Upstart unit)
description "Consul Agent"
start on (local-filesystems and net-device-up IFACE!=lo)
stop on runlevel [06]
exec  consul agent -config-dir /etc/consul.d/bootstrap -advertise=192.168.12.162 >> /var/log/consul 2>&1
respawn
respawn limit 10 10
kill timeout 10
```

* Autostart consul server on Ubuntu 14.04 
Setup a server configuration file   -   /etc/init/consul.conf
```
# Consul Agent (Upstart unit)
description "Consul Agent"
start on (local-filesystems and net-device-up IFACE!=lo)
stop on runlevel [06]
exec  consul agent -config-dir /etc/consul.d/server -advertise=192.168.12.163 >> /var/log/consul 2>&1
respawn
respawn limit 10 10
kill timeout 10
```

* Autostart consul client on Ubuntu 14.04 
Setup a client configuration file   -   /etc/init/consul.conf
```
# Consul Agent (Upstart unit)
description "Consul Agent"
start on (local-filesystems and net-device-up IFACE!=lo)
stop on runlevel [06]
exec  consul agent -config-dir /etc/consul.d/client -advertise=192.168.12.96 >> /var/log/consul 2>&1
respawn
respawn limit 10 10
kill timeout 10
```

* Autostart consul client on Ubuntu 16.04 
Setup a client configuration file  - /etc/systemd/system/consul-client.service
```
[Unit]
Description=Consul Server
After=network.target

[Service]
User=root
Group=root
Environment="GOMAXPROCS=2"
ExecStart=/usr/local/bin/consul agent -config-dir /etc/consul.d/client
ExecReload=/bin/kill -9 $MAINPID
KillSignal=SIGINT
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

* Manage Consul on Ubuntu 14.04 using commands 
```
start consul
stop consul
status consul
```
* Manage Consul on Ubuntu 16.04 using commands 
```
systemctl status consul-client.service
systemctl start consul-client.service
systemctl stop consul-client.service
```
Starting consul should expose the service over following ports :
```
(jse2)root@consul-02:/etc/systemd/system# netstat -ntlp | grep consul
tcp        0      0 192.168.12.162:8300     0.0.0.0:*               LISTEN      21996/consul
tcp        0      0 192.168.12.162:8301     0.0.0.0:*               LISTEN      21996/consul
tcp        0      0 192.168.12.162:8302     0.0.0.0:*               LISTEN      21996/consul
tcp        0      0 127.0.0.1:8500          0.0.0.0:*               LISTEN      21996/consul
tcp        0      0 127.0.0.1:8600          0.0.0.0:*               LISTEN      21996/consul
```
dns - The DNS server,  Default 8600. 
http - The HTTP API,  Default 8500. 
serf_lan - The Serf LAN port. Default 8301. 
serf_wan - The Serf WAN port. Default 8302. 
server - Server RPC address. Default 8300. 

In case you have IPTABLES enabled on your consul nodes, see to it that you add the above ports 

Verify the server cluster 
```
root@consul-01:/etc/consul.d/bootstrap# consul members | grep server
consul-01  192.168.12.161:8301  alive   server  0.9.0  2         dc1
consul-02  192.168.12.162:8301  alive   server  0.9.0  2         dc1
consul-03  192.168.12.163:8301  alive   server  0.9.0  2         dc1
```

* Consul for Monitoring

Define the check in config.json for each of the host (bootstrap, server, client ) which need to be monitored.

```
    "checks": [
        {
            "id" : "check_cpu_utilization",
            "notes" : "Greater than 50% warn, greater than 75% fail.",
            "name" : "CPU Utilization",
            "script" : "/etc/consul.d/script/cpu_utilization.sh",
            "interval" : "60s"
        },
        {
          "id" : "check_mem_utilization",
          "notes" : "Greater than 50% warn, greater than 75% fail.",
          "name" : "MEM Utilization",
          "script" : "/etc/consul.d/script/mem_utilization.sh",
          "interval" : "60s"
        },

```
Add the actual script file /etc/consul.d/script/cpu_utilization.sh
to print exit codes.
OK = exit 0
Warning = exit 1
Critical = exit 2

```
#!/bin/bash

CPU_UTILIZATION=`top -b -n1 | grep "Cpu(s)" | awk '{print $2 + $4}'`
CPU_UTILIZATION=${CPU_UTILIZATION%.*}

echo "CPU: "$CPU_UTILIZATION"%"

if (( $CPU_UTILIZATION > 75 ));
then
    exit 2
fi

if (( $CPU_UTILIZATION > 50 ));
then
    exit 1
fi

exit 0
```

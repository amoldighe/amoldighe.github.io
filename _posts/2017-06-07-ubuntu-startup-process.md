---
layout: post
title:  "Exploring Ubuntu Process Startups"
date:   2017-06-07
tags:
  - ubuntu
  - startup
  - process
  - systemV
  - upstart
  - systemd
---

In this article I am going to explore Ubuntu startup process which has been through major upgrade from the traditional System V to Upstart to Systemd.

* **Traditional Startup - SystemV**

System V init tools (SysVinit) uses startup scripts in /etc/init.d/ . This is the traditional service management package for Linux, containing the init program which is the first process that is run when the kernel has finished initializing. It also contains script to start and stop services. Specifically, files in /etc/init.d are shell scripts that respond to start, stop, restart, reload commands to manage a particular service. These scripts can be invoked directly or via some other trigger which is a symbolic link in /etc/rc?.d/.

/etc/init.d scripts are the old way of doing things based on System V standard. These scripts are fired only in a particular sequence, so no real dependencies can be established.

To enable a script to startup at system boot, make the script executable and copy to /etc/init.d/
Next used update-rc.d command to set the runlevel to start the script as well as mentioned the runlevel to stop the script. 

update-rc.d myscript.sh defaults

> #update-rc.d myscript.sh defaults
> Adding system startup for /etc/init.d/myscript.sh ...
> /etc/rc0.d/K20myscript.sh -> ../init.d/myscript.sh
> /etc/rc1.d/K20myscript.sh -> ../init.d/myscript.sh
> /etc/rc6.d/K20myscript.sh -> ../init.d/myscript.sh
> /etc/rc2.d/S20myscript.sh -> ../init.d/myscript.sh
> /etc/rc3.d/S20myscript.sh -> ../init.d/myscript.sh
> /etc/rc4.d/S20myscript.sh -> ../init.d/myscript.sh
> /etc/rc5.d/S20myscript.sh -> ../init.d/myscript.sh

On the next boot, myscript.sh would be started on runlevel 2,3,4,5 and shutdown on runleve 0,1,6 
Also note that it has a default priority of 20, i.e. it will be started before and killed after any other script with higher priority.

> # update-rc.d myscript.sh start 20 2 3 4 . start 30 5 . stop 80 0 1 6 .
> Adding system startup for /etc/init.d/myscript.sh ...
> /etc/rc0.d/K80myscript.sh -> ../init.d/myscript.sh
> /etc/rc1.d/K80myscript.sh -> ../init.d/myscript.sh
> /etc/rc6.d/K80myscript.sh -> ../init.d/myscript.sh
> /etc/rc2.d/S20amyscript.sh -> ../init.d/myscript.sh
> /etc/rc3.d/S20myscript.sh -> ../init.d/myscript.sh
> /etc/rc4.d/S20myscript.sh -> ../init.d/myscript.sh
> /etc/rc5.d/S30myscript.sh -> ../init.d/myscript.sh

With the above parameters set for myscript.sh, on the next boot, myscript.sh would be started on runlevel 2,3,4 with priority 20 and on runleve 5 with priority 30 and shutdown on runleve 0,1,6 with priority 80 

* **Upstart**

Upstart has been developed with the intent to substitute all the /etc/init.d/ scripts, with upstart startup configuration files are stored at /etc/init/ 
Upstart's model for starting processes (jobs) is "greedy event-based", i. e. all available jobs whose startup events happen are started as early as possible. A new job merely needs to install its configuration file into /etc/init/ to become active. 
/etc/init contains configuration files used by Upstart. Files in /etc/init are configuration files telling Upstart how and when to start, stop, reload the configuration, or query the status of a service. 

Here's an example of a startup configuration file for Consul, stored @ /etc/init/consul.conf

> # Consul Agent (Upstart unit)
> description "Consul Agent"
> start on (local-filesystems and net-device-up IFACE!=lo)
> stop on runlevel [06]
> 
> exec  consul agent -config-dir /etc/consul.d/server -advertise=192.168.7.162 >> /var/log/consul 2>&1
> respawn
> respawn limit 10 10
> kill timeout 10
> 

The configuration file defines the job description, runleve to start and stop the script, exec to execute a command.
Following commands are used for :
    * starting consul agent - start consul
    * stopping consul agent - stop consul
    * checking status of consul agent - status consul


* **Systemd**

Systemd's model for starting processes (units) is "lazy dependency-based", i. e. a unit will only start if and when some other starting unit depends on it.  A new unit needs to add itself as a dependency of a unit of the boot sequence (commonly multi-user.target) in order to become active. The basic object that systemd manages and acts upon is a "unit". Units can be of many types, but the most common type is a "service" (indicated by a unit file ending in .service). The service unit files are stored @ /etc/systemd/system/

Here's an example of consul client added to systemd startup on Ubuntu 16.04, located @ /etc/systemd/system/consul.service

> [Unit]
> Description=Consul Server
> After=network.target
> 
> [Service]
> User=root
> Group=root
> Environment="GOMAXPROCS=2"
> ExecStart=/usr/local/bin/consul agent -config-dir /etc/consul.d/client
> ExecReload=/bin/kill -9 $MAINPID
> KillSignal=SIGINT
> Restart=on-failure
> 
> [Install]
> WantedBy=multi-user.target

To manage services on a systemd enabled server, our main tool is the systemctl command.

> systemctl start consul.service
> 
> systemctl status consul.service
> 
> systemctl stop consul.service

By default, most systemd unit files are not started automatically at boot, they need to be hooked to be started automatically using enable option.
Similary disbale will unhook them from startup.

> systemctl enable consul.service
> 
> systemctl disable consul.service

View the service unit file using:

> systemctl cat consul.service

Edit the service unit file using:

> systemctl edit consul.service

The unit file resides in /etc/systemd/system/consul.service and can be view or updated directly using a text editor.
After edit to load the new configuration, reload the service unit file

> systemctl daemon-reload



* ***Helpful Bookmarks*** 

https://www.debuntu.org/how-to-managing-services-with-update-rc-d/ 
https://www.digitalocean.com/community/tutorials/the-upstart-event-system-what-it-is-and-how-to-use-it
https://www.digitalocean.com/community/tutorials/systemd-essentials-working-with-services-units-and-the-journal
https://wiki.ubuntu.com/SystemdForUpstartUsers



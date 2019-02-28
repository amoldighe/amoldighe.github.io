Since past few months I have been working on Proxmox container solution which is based of LXC containers. Proxmox is similar to Docker in terms of its containerzation philisophy, but different in terms of how the technology is implemented. Here's a comparision of Docker, LXC and Virtual Machine, this should give some prespective on what I mean in my earlier statement.

|Docker | LXC | VM|
|--|--|--|
|Very quick OS boot/intialization | Very quick OS boot/intialization | OS booting takes time|
|Requires custom baked container images | Requires custom baked container images | Standard OS iso can be used for booting|
|Shared kernel of the base machine | Shared kernel of the base machine | Full functional kernel for each VM|
|Runs a stripped down OS | Contains a fully functional OS with its own filesystem | Contains a fully functional OS with its own filesystem|
|Docker is designed to run one application per container | Multiple application can be installed within a LXC container | Runs multiple applications|
|Lightweight as compared to LXC & VM | Lightweight compared to VM | Increased resource consumption than containers|
|Filesystem is based of read only layers via AUFS | Provides complete independent filesystem | Provides complete independent filesystem|
|Emhemeral data storage within container, persistent data storage is supported using external storage mounts | Persistent data can be stored within LXC | Persistent data can be stored within VM|
|Cross platform solution | LXC is linux only solution | Cross platform solution|


Implementing a proxmox standalone server or a cluster is fairly simple and covered in their documentation. Proxmox uses corosync for clustering and has a very good UI for managing its server / cluster. What I liked about Proxmox is the supports creation of containers as well as creation of VM's. One thing to remember while setting up a Proxmox cluster is to use the base server partition as lvm-thin or ZFS or any other storage file system which supports creation of snapshot. In case standard LVM storage is used on the base machine, it will restrict creation of container & VM snapshot as no metadata is stored in case of standard LVM. Lets cover some more of Proxmox based on the two issues and its solution that I came across while using this container technology.


## ISSUE 1 : LXC container fails to start 


* LXC container fails to start with ERROR in /var/log/lxc/<container_id>.log

```

      lxc-start 20181016102940.512 ERROR    lxc_sync - sync.c:__sync_wait:57 - An error occurred in another process (expected sequence number 5)
      lxc-start 20181016102940.512 ERROR    lxc_start - start.c:__lxc_start:1365 - Failed to spawn container "155".
      lxc-start 20181016102941.255 ERROR    lxc_conf - conf.c:run_buffer:405 - Script exited with status 32.
      lxc-start 20181016102941.255 ERROR    lxc_start - start.c:lxc_fini:546 - Failed to run lxc.hook.post-stop for container "155".
      lxc-start 20181016102946.260 ERROR    lxc_start_ui - tools/lxc_start.c:main:366 - The container failed to start.
      lxc-start 20181016102946.260 ERROR    lxc_start_ui - tools/lxc_start.c:main:368 - To get more details, run the container in foreground mode.
      lxc-start 20181016102946.260 ERROR    lxc_start_ui - tools/lxc_start.c:main:370 - Additional information can be obtained by setting the --logfile and --logpriority options.
      lxc-start 20181016103925.588 ERROR    lxc_apparmor - lsm/apparmor.c:apparmor_process_label_set:234 - No such file or directory - failed to change apparmor profile to lxc-container-default-cgns
      lxc-start 20181016103925.588 ERROR    lxc_sync - sync.c:__sync_wait:57 - An error occurred in another process (expected sequence number 5)
      lxc-start 20181016103925.588 ERROR    lxc_start - start.c:__lxc_start:1365 - Failed to spawn container "155".
      lxc-start 20181016103926.340 ERROR    lxc_conf - conf.c:run_buffer:405 - Script exited with status 32.
      lxc-start 20181016103926.340 ERROR    lxc_start - start.c:lxc_fini:546 - Failed to run lxc.hook.post-stop for container "155".
      lxc-start 20181016103931.344 ERROR    lxc_start_ui - tools/lxc_start.c:main:366 - The container failed to start.
      lxc-start 20181016103931.344 ERROR    lxc_start_ui - tools/lxc_start.c:main:368 - To get more details, run the container in foreground mode.
      lxc-start 20181016103931.344 ERROR    lxc_start_ui - tools/lxc_start.c:main:370 - Additional information can be obtained by setting the --logfile and --logpriority options.
```

* The error indicate an apparmor issue, check the apparmor modules loaded with command 

```
root@proxmox:~$aa-status
apparmor module is loaded.
0 profiles are loaded.
0 profiles are in enforce mode.
0 profiles are in complain mode.
0 processes have profiles defined.
0 processes are in enforce mode.
0 processes are in complain mode.
0 processes are unconfined but have a profile defined.

```
Try loading the apparmor profiles 

```
root@proxmox:~$apparmor_parser -R /etc/apparmor.d/
apparmor_parser: Unable to remove "/usr/bin/lxc-start".  Profile doesn't exist
```

## SOLUTION 1:

* First load the lxc-container profile

```
root@proxmox:~$apparmor_parser -r /etc/apparmor.d/lxc-containers

root@proxmox:~$aa-status
apparmor module is loaded.
4 profiles are loaded.
4 profiles are in enforce mode.
   lxc-container-default
   lxc-container-default-cgns
   lxc-container-default-with-mounting
   lxc-container-default-with-nesting
0 profiles are in complain mode.
0 processes have profiles defined.
0 processes are in enforce mode.
0 processes are in complain mode.
0 processes are unconfined but have a profile defined.
```

* Now try starting the container, it shoukd work

* Load rest of the profiles in /etc/apparmor.d/

```
root@proxmox:/etc/apparmor.d$apparmor_parser -r /etc/apparmor.d/
root@proxmox:/etc/apparmor.d$aa-status
apparmor module is loaded.
6 profiles are loaded.
6 profiles are in enforce mode.
   /usr/bin/lxc-start
   /usr/sbin/named
   lxc-container-default
   lxc-container-default-cgns
   lxc-container-default-with-mounting
   lxc-container-default-with-nesting
0 profiles are in complain mode.
25 processes have profiles defined.
16 processes are in enforce mode.
   lxc-container-default-cgns (31021) 
   lxc-container-default-cgns (31305) 
   lxc-container-default-cgns (31325) 
   lxc-container-default-cgns (31364) 
   lxc-container-default-cgns (31469) 
   lxc-container-default-cgns (31473) 
   lxc-container-default-cgns (31491) 
   lxc-container-default-cgns (31684) 
   lxc-container-default-cgns (31689) 
   lxc-container-default-cgns (31697) 
   lxc-container-default-cgns (31962) 
   lxc-container-default-cgns (31967) 
   lxc-container-default-cgns (31968) 
   lxc-container-default-cgns (32020) 
   lxc-container-default-cgns (32024) 
   lxc-container-default-cgns (32025) 
0 processes are in complain mode.
9 processes are unconfined but have a profile defined.
   /usr/bin/lxc-start (7185) 
   /usr/bin/lxc-start (11478) 
   /usr/bin/lxc-start (13471) 
   /usr/bin/lxc-start (18829) 
   /usr/bin/lxc-start (24157) 
   /usr/bin/lxc-start (26359) 
   /usr/bin/lxc-start (26710) 
   /usr/bin/lxc-start (30969) 
   /usr/sbin/named (1402) 
```


## ISSUE 2: Container fails to mount NFS path

To share data across containers and Proxmox host machine, we had setup a NFS box. The mounts on Proxmox host machine were working flawlessly but container failed to mount NFS path.

```
root@container1:~$mount 192.168.50.4:/home/bkp /mnt/bkp/
mount: block device 192.168.100.4:/home/bkp is write-protected, mounting read-only
mount: cannot mount block device 192.168.50.4:/home/bkp read-only
```
## SOLUTION 2: 

* Tail syslog on container host 

```
Jan 29 19:10:10 proxmox kernel: [36785137.211208] audit: type=1400 audit(1548769210.717:50467): apparmor="DENIED" operation="mount" info="failed type match" error=-13 profile="lxc-container-default-cgns" name="/mnt/bkp/" pid=3535 comm="mount" fstype="nfs" srcname="192.168.50.4:/home/bkp"
```

The error indicates that the apparmor profile is denying the NFS mount 

* On the Proxmox host, add the below line to /etc/apparmor.d/lxc/lxc-default-cgns 

```
root@proxmox:~$cat /etc/apparmor.d/lxc/lxc-default-cgns 
# Do not load this file.  Rather, load /etc/apparmor.d/lxc-containers, which
# will source all profiles under /etc/apparmor.d/lxc

profile lxc-container-default-cgns flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/lxc/container-base>

  # the container may never be allowed to mount devpts.  If it does, it
  # will remount the host's devpts.  We could allow it to do it with
  # the newinstance option (but, right now, we don't).
  deny mount fstype=devpts,
  mount fstype=cgroup -> /sys/fs/cgroup/**,
  mount options=(rw, nosuid, noexec, remount, relatime, ro, bind),
}

```

* Add mount to fstab
```
root@container1:~$cat /etc/fstab
192.168.50.4:/home/bkp /mnt/bkp nfs rw,async,hard,intr 0 0
```

* Incase of error
```
root@container1:~$mount -a
mount: wrong fs type, bad option, bad superblock on 192.168.50.4:/home/bkp,
       missing codepage or helper program, or other error
       (for several filesystems (e.g. nfs, cifs) you might
       need a /sbin/mount.<type> helper program)
       In some cases useful info is found in syslog - try
       dmesg | tail  or so
```

* Install nfs-common package and then try mount command




https://forum.proxmox.com/threads/lxc-aa_profile-is-deprecated-and-was-renamed-to-lxc-apparmor-profile.38505/

https://unix.stackexchange.com/questions/254956/what-is-the-difference-between-docker-lxd-and-lxc

https://www.linkedin.com/pulse/docker-vs-lxc-virtual-machines-phucsi-nguyen





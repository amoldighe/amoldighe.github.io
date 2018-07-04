---
layout: post
title:  "My system is slow"
date:   2018-2-11
tags:
  - System slow
  - Throughput
  - IOPS
  - Latency
  - USE methodology
  - Linux performance tools
---

Investigating a slow performing system

* Check system resources - provide output of tools displaying output of cpu, ram, disk io, network io performance
* Ask customer question to quantify the slowness he is facing

Below are the methodologies to resolve system issue.

* ***Problem Statement Method***
 
Ask question to customer related to the performance issue - what, when, where, which, changes ?

* What makes you think there is a performance problem ?

* Has the system ever performed well ?

* What has changed recently - hardware, software, load ?

* Can the performance degradation be expressed in terms of latency or run time ?

* Does the performance affect other people or other application or its just you ?

* What is the environment - hardware, software, instance type, version, configuration ?

* ***Workload characterization method***

Who is causing the load ? 
Get information regarding the PID, UID, IP address.

Why is the load called ? 
Is it related to some code, a directory path, stack trace.
 
What is the load ? 
IOPs, through put, read / write

How is the load changed overtime ?

* ***USE methodologies*** 

Check these for every resource:

***Utilization*** - Average time the system resource was busy serviceing a request.
The metric for utilization can be defined as a percentage over time interval e.g. "one of the cpu core is running at 95% utilization"

***Saturation*** - A degree to which a system resource has extra work which it cannot service or is being queued. 
The queue length of task to be serviced by the cpu. 

***Errors*** - Count of error for each system resource. 
Each of the system resource writes error messages to a log file which help in investigation of an issue.

* ***Check System Metrices***
 Check output of performance tools like metrics or graphs depicting the metrices for 

```
    CPUs: cpu load 
    Memory: memory capacity load
    Network interfaces : recieve data, transmit data 
    Storage devices: IOPS, Capacity, Throughput, Latency 
```
    
To explain the above terms for storage device metrices, I am going to take reference from an article by [Rickard Nobel] (http://rickardnobel.se/storage-performance-iops-latency-throughput/) here is some information in extract from the same article which I feel is important for our explaination.

Throughput is usually expressed in Megabytes / Second (MB/s). The maximum throughput for a disk could be for example 170 MB/s.

IOPS means IO operations per second, which means the amount of read or write operations that could be done in one seconds time. A certain amount of IO operations will also give a certain throughput of Megabytes each second, so these two are related. A third factor is however involved: the size of each IO request. Depending on the operating system and the application/service that needs disk access it will issue a request to read or write a certain amount of data at the same time. This is called the IO size and could be for example 4 KB, 8 KB, 32 KB and so on. The minimum amount of data to read/write is the size of one sector, which is 512 byte only.

``
Average IO size x IOPS = Throughput in MB/s
``

Each IO request will take some time to complete, this is called the average latency. This latency is measured in milliseconds (ms) and should be as low as possible. There are several factors that would affect this time. Many of them are physical limits due to the mechanical constructs of the traditional hard disk.

***Linux Performance Tools***

```
Observebility Tools - uptime, top, atop, htop, ps, vmstat, mpstat, iostat, free, strace, tcpdump, netstat, pidstat, lsof, swapon, sar, ss

Benchmarking Tools - fio, dd, sysbench, iperf

Tuning Tools - sysctl, ethtool, ip, route, nice, ulimit, chcpu, tune2fs, ionice, hdparm

Static Tools - df, ip, route, lsblk, lsscsi, swapon, lscpu, lshw, sysctl, lspci, ldd, sysctl

```

***Profiling***
 
Check output of system components for a certain time period to locate the problem area.



* ***Reference Links***

[Brendan Gregg USE method](http://www.brendangregg.com/usemethod.html)

[Brendan Gregg Video](https://youtu.be/FJW8nGV4jxY?list=PLwZOquYxKJS5_4UhfSCvOa-89LBuI0IIQ)

[Rickard Nobel](http://rickardnobel.se/storage-performance-iops-latency-throughput/)


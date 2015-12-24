title: Trouble Shooting Your Linux Server In One Minute
date: 2015-12-23 23:09:52
tags:
	- linux
	- bash
	- trouble shooting
	- performance
	- shell
	- server
---

If you noticed a weirly performing Linux server, what things are you going to check in the first minute after jumping on the server? Assuming it's not soooo bad which is denying all incoming requests. :)

If you are alerted by some preestablished metric thresholds, like "low disk space", "low free mem" etc. The best practice is probably just fill in the prescription: throw out some logs or kill some long running processes. Here I want to talk about more debugging checks, and gives you a quick understanding of what is happening on the box at the moment.

<!-- more -->

Among so many metrics, what are most important ones to look at? Or which ones can deliver most information you need about a Linux server. Brendan Gregg proposed a method called **[USE (Utilization, Saturation, Errors)](http://www.brendangregg.com/usemethod.html)** in his book << System Performance: Enterprise and the Cloud >>. The key concept of this method is: For every resource, check utilization, saturation, and errors(0).

Terminology definitions:

* Resource: all physical server functional components (CPUs, disks, busses, ...)
* Utilization: the average time that the resource was busy servicing work
* Saturation: the degree to which the resource has extra work which it can't service, often queued
* Errors: the count of error events


uptime/w
---
```bash uptime
~$ uptime
 06:37:07 up 217 days, 18:11,  1 user,  load average: 1.61, 0.68, 0.13
```
```bash w
~$ w
 06:37:07 up 217 days, 18:11,  1 user,  load average: 1.61, 0.68, 0.13
USER     TTY      FROM              LOGIN@   IDLE   JCPU   PCPU WHAT
zwang    pts/2    jump.xxxxxxxxxxx 18:12    0.00s  0.07s  0.00s w
```

A very light weight command that quickly shows you how many long the system has been running, and load average.

**Load Average** is very important 3 numbers that show system load over the past 1, 5 and 15 minutes accordingly. It calculates by from 0.00 to 1.00 or higher, where 0.00 means no load and 1.00 means full load capacity, per core of you CPU.

For the above example, my system load is 1.61 duraing the past 1 minute, which is not at capacity because it's a dual-core system(capacity is 2 in this case).

vmstat
---
```bash Load Stats - Mem and CPU
~$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0 289336 259920 290900 883276    0    0     2    35    0    1  0  0 99  0  0
 0  0 289336 259920 290904 883272    0    0     0   352   75   77  0  0 100  0  0
 0  0 289336 259672 290904 883276    0    0     0     0   82  100  0  0 100  0  0
...
 ```
That `1` means pring out a snapshot every second.


* `si` and `so` means swap mem in/out, if they are not 0, it means no free memory is available right now.
* `r` means # of threads waiting for run time, so r > # of CPU means load is beyond capacity.
* `id` means CPU idle time; `us` and `sy` means user CPU time and system CPU time respectively; `st` means *steal time*, it happens when VM's hypervisor is taking current system's CPU and use it elsewhere (other VM etc).

CPU idle and user time + system time both tells you if CPU is busy, but us (user time) and sy (system time) can give you more info, for example, if sy is much higher than us, it possiblely means system is stucking on I/O.

mpstat
---
```bash Load Stats Per Core
~$ mpstat -P ALL 1
Linux 2.6.18-238.19.1.el5xen (my-linux) 	12/24/2015

07:13:48 AM  CPU   %user   %nice    %sys %iowait    %irq   %soft  %steal   %idle    intr/s
07:13:49 AM  all    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00     68.00
07:13:49 AM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00     46.00
07:13:49 AM    1    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00     21.00
```

Similar metrics with `vmstat`, but display stat per core so you can check if each core is load balanced.

iostat
---
```bash I/O stats
~$ iostat -dmx 5
Linux 2.6.18-238.19.1.el5xen (30.zwang.user.nym2) 	12/24/2015

Device:         rrqm/s   wrqm/s   r/s   w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await  svctm  %util
sda1              0.02     9.30  0.25  7.95     0.00     0.07    17.56     0.05    5.54   0.64   0.53
sda2              0.02     0.07  0.01  0.03     0.00     0.00    27.79     0.00   99.32   1.21   0.00
```
This is a very useful command because you are likely to see I/O bound very often.
`-d` means only display device utilization (exclude CPU), `-m` means in megabyte format, `-x` means extra stats (always good :).

* `r/s`, `w/s`, `rMB/s` and `wMB/s` means # of R/W requests, and size of R/W sent to CPU per second.
* `avgqu-sz` shows # of requests wait in device's queue. If this is greater than # of cores, it means your system is overloaded.
* `%util` is the % of CPU time when requests were issued. When it becomes 100%, the device is saturated.


dmesg
---
```bash System's Latest Error Message
~$ dmesg | tail
Swap cache: add 1157149, delete 1156672, find 70807/80358, race 0+13
Free swap  = 0kB
Total swap = 4194296kB
Free swap:            0kB
...
phpunit[7388]: segfault at 00002b40285c2018 rip 00000000005b5bf8 rsp 00007fff84a77be0 error 4
```

This would display last 10 system message, helpful on finding possible system failures.

ifconfig
---
Everyone knows to check IP address with this command, but few knows it also shows your # of error and dropped packets.
```bash ifconfig | grep error
~$ ifconfig | grep error
          RX packets:262418135 errors:0 dropped:0 overruns:0 frame:0
          TX packets:53768807 errors:0 dropped:0 overruns:0 carrier:0
          RX packets:6416646 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6416646 errors:0 dropped:0 overruns:0 carrier:0
```

free
---
```bash Buffers and Cached
~$ free -m
             total       used       free     shared    buffers     cached
Mem:          2048       1852        195          0        292        870
-/+ buffers/cache:        689       1358
Swap:         4095        282       3813
```

While you can retrive memory usage via a lot ways, `free` can quickly show you stats of `buffers` and `cached`.

* A buffer is a temporary location assigned to an application that's used by **system**, it buffers data pior to I/O;
* Unlike buffer, cache is used for filesystems and store frequently used data from your disk for fast access, similar idea like LRU in many software caching application.

If both values are reaching 0, it means system is currently under high I/O usage, which can be verified by `iostat -dmx`.

More about free see [this post at linuxnix](http://www.linuxnix.com/find-ram-size-in-linuxunix/).

netstat
---
```bash Show All Listened TCP/UDP Port
~$ netstat -tulpn
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State       PID/Program name
tcp        0      0 :::3306                     :::*                        LISTEN      -
tcp        0      0 :::16011                    :::*                        LISTEN      -
tcp        0      0 :::11211                    :::*                        LISTEN      -
...
udp        0      0 0.0.0.0:11211               0.0.0.0:*                               -
...
```

There is a lot arguments, but each has its meaning. `-tu` means show TCP and UDP only, because those are what you are looking for most of time; `-l` means display listening sockets only; `-p` will display PID/Program of the application that owns the port, also keep in mind that you mind need sudo access to view this column. Above result was ran by me (user), which shows no Program name, but can be viewed by super user:
```bash Usage of Port 3306
~$ sudo netstat -tulpn | grep :3306
tcp        0      0 :::3306                     :::*                        LISTEN      22571/mysqld
```
Now it shows 3306 is listened by mysql.

See more about netstat at [this post at cyberciti](http://www.cyberciti.biz/faq/what-process-has-open-linux-port/).

top
---
Alright, finally `top`. IMO this is the most powerful tool that gives you a lot information about your linux server, and it probably covered/overlapped most of the commands output we listed above. Unlike `top`, which is like printing snapshot of a collection your system stats, commands like `vmstat`, `iostat` can roll out stats constantly so you can view some metric across a duration, for debugging/logging purposes or what not. Furthermore, with the combination of awk/sed etc, you will be able to do more cool things.

You can do a lot of things with `top`, but usually I use it when I am trying to get a high level idea of resource utilization of each application.

Conclusion, I often find it's overwhelming to learn so many different Linux shell commands, among which overlaps with each other's functionality sometimes too. I find it's useful to learn them based on what you want to do, and learn them with specific purpose, check out [**this post**](http://www.brendangregg.com/USEmethod/use-linux.html) from Brendan Gregg to find out debugging different system component with best shell commands.
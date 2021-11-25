# chrony对时

## 安装chrony

```shell
yum install chrony
```

查看时间源

```shell
[root@controller3 ceph]# docker exec -it chrony bash
(chrony)[root@controller3 /]# chronyc sourcestats -v
210 Number of sources = 5
                             .- Number of sample points in measurement set.
                            /    .- Number of residual runs with same sign.
                           |    /    .- Length of measurement set (time).
                           |   |    /      .- Est. clock freq error (ppm).
                           |   |   |      /           .- Est. error in freq.
                           |   |   |     |           /         .- Est. offset.
                           |   |   |     |          |          |   On the -.
                           |   |   |     |          |          |   samples. \
                           |   |   |     |          |          |             |
Name/IP Address            NP  NR  Span  Frequency  Freq Skew  Offset  Std Dev
==============================================================================
172.20.41.136              16   9   969     +0.017      0.025   +521us  7026ns
tick.ntp.infomaniak.ch     21  12  344m     -0.418      0.469    +50us  3563us
stratum2-1.ntp.led01.ru.>  23  13  413m     -0.117      0.054    +47ms   483us
ntp6.flashdance.cx         34  17   12h     -0.103      0.287    -42ms  6104us
139.199.214.202            19   9  344m     -0.167      0.088   -713us   599us
(chrony)[root@controller3 /]# 
```

配置文件

```shell

  1 # Use public servers from the pool.ntp.org project.
  2 # Please consider joining the pool (http://www.pool.ntp.org/join.html).
  3 #server 0.centos.pool.ntp.org iburst
  4 #server 1.centos.pool.ntp.org iburst
  5 #server 2.centos.pool.ntp.org iburst
  6 #server 3.centos.pool.ntp.org iburst
  7 server 172.20.41.136 iburst   #指定时间源
  8 # Record the rate at which the system clock gains/losses time.
  9 driftfile /var/lib/chrony/drift
 10 
 11 # Allow the system clock to be stepped in the first three updates
 12 # if its offset is larger than 1 second.
 13 makestep 1.0 3
 14 
 15 # Enable kernel synchronization of the real-time clock (RTC).
 16 rtcsync
 17 
 18 # Enable hardware timestamping on all interfaces that support it.
 19 #hwtimestamp *
 20 
 21 # Increase the minimum number of selectable sources required to adjust
 22 # the system clock.
 23 #minsources 2
 24 
 25 # Allow NTP client access from local network.
 26 #allow 192.168.0.0/16
 27 
 28 # Serve time even if not synchronized to a time source.
 29 #local stratum 10
 30 
 31 # Specify file containing keys for NTP authentication.
 32 #keyfile /etc/chrony.keys
 33 
 34 # Get TAI-UTC offset and leap seconds from the system tz database.
 35 #leapsectz right/UTC
 36 
 37 # Specify directory for log files.
 38 logdir /var/log/chrony
 39 
 40 # Select which information is logged.
 41 #log measurements statistics tracking

```

重启chrony服务

```shell
systemctl restat chronyd
```


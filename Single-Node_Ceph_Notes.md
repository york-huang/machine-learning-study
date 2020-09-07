# Single-Node Ceph Notes

Single-node Ceph PoC (Proof-of-Concept) cluster on single Hyper-V (Windows 10 Pro, OS Build 19041) VM instance with CentOS 8.2.2004, deployed and managed with Cephadm (cephadm 15.2.4).

## Prerequisite

* Ceph cluster is supposed to be run on nodes with fixed IPs, to make this possible for Hyper-V VMs, we need Hyper-V NAT virtual network adapter, check this [tutorial](https://www.cnblogs.com/wswind/p/11007613.html).

* CentOS 8 with XFS file system chosen on installation, as suggest in Ceph document:

```
We recommend using the xfs file system when running mkfs. (btrfs and ext4 are not recommended and no longer tested.)
```

* Cephadm will handle the firewalld things automatically, however, we can still check against the firewall status with following commands,

```bash
netstat -tunlp  # check on which ports are being listened
firewall-cmd --list-all  # check which services and ports are added to firewalld

# BTW: the firewalld services config files can be found at /usr/lib/firewalld/services/, e.g., in file ceph.xml we can see port 6800-7300 are added, which are really nice.
```

* NTP or Chronyd must be setup for time synchronization because:

```
Ceph daemons pass critical messages to each other, which must be processed before daemons reach a timeout threshold. If the clocks in Ceph monitors are not synchronized, it can lead to a number of anomalies.
```

* Cephadm installs logrotate config (`/etc/logrotate.d/ceph-<fsid>`) by default even the system has no logrotate package installed. If you want this to function properly, just install the logrotate manually.

* Cephadm requires the host name to be a single string, i.e., no `.localdomain` part. Can use `hostnamectl` tool to adjust the host name on CentOS.

* Systemd, which is required for Ceph cluster, concepts explained:  [link1](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files), [link2](https://www.computernetworkingnotes.com/linux-tutorials/systemd-unit-configuration-files-explained.html); and there are some command [examples](https://www.computernetworkingnotes.com/linux-tutorials/how-to-use-the-systemctl-command-to-manage-systemd-services.html).

* Using Huawei's CentOS mirror, [link](https://mirrors.huaweicloud.com/).

* Cephadm installation guides, multiple sources:

  * (Compact and clean) Cephadm + CentOS 8, [link](https://www.server-world.info/query?os=CentOS_8&p=ceph15&f=9).

  * Another detailed guide for Cephadm + CentOS 8, [link](https://blog.csdn.net/networken/article/details/106870859).

  * Cephadm + docker on CentOS 4-nodes cluster, very detailed, [link](https://segmentfault.com/a/1190000023292938).

  * The official guide, [link](https://docs.ceph.com/docs/master/cephadm/install/).
  
  * Complete guide on 3-nodes cluster on CentOS 7, outdated but very informative, [link](http://xuxiaopang.com/2016/10/10/ceph-full-install-el7-jewel/).

## Ceph configuration concepts and notes

* Configurations are interpreted in the following order,

```
Each Ceph daemon, process, and library will pull its configuration from several sources, listed below. Sources later in the list will override those earlier in the list when both are present.

* the compiled-in default value

* the monitor cluster’s centralized configuration database

* a configuration file stored on the local host

* environment variables

* command line arguments

* runtime overrides set by an administrator

One of the first things a Ceph process does on startup is parse the configuration options provided via the command line, environment, and local configuration file. The process will then contact the monitor cluster to retrieve configuration stored centrally for the entire cluster. Once a complete view of the configuration is available, the daemon or process startup will proceed.
```

* Monitor configure database

```
The monitor cluster manages a database of configuration options that can be consumed by the entire cluster, enabling streamlined central configuration management for the entire system. The vast majority of configuration options can and should be stored here for ease of administration and transparency.

A handful of settings may still need to be stored in local configuration files because they affect the ability to connect to the monitors, authenticate, and fetch configuration information. In most cases this is limited to the mon_host option, although this can also be avoided through the use of DNS SRV records.
```

```
Once one monitor is reached, all other current monitors are discovered, so the mon host configuration option only needs to be sufficiently up to date such that a client can reach one monitor that is currently online. The MGR, OSD, and MDS daemons will bind to any available address and do not require any special configuration.
```

* Basic concepts of Ceph configuration, [link](https://ceph.readthedocs.io/en/latest/rados/configuration/ceph-conf/), must be read with great carefulness.

* Accessing cluster key components through the daemon's admin socket, [link](https://ceph.readthedocs.io/en/latest/rados/operations/monitoring/#using-the-admin-socket).

* Choosing the number of placement groups, [link](https://ceph.readthedocs.io/en/latest/rados/operations/placement-groups/#choosing-the-number-of-placement-groups), or simply use the PGCalc [calculator](https://ceph.com/pgcalc/).

```
In a nutshell, more OSDs mean faster recovery and a lower risk of cascading failures leading to the permanent loss of a Placement Group. Having 512 or 4096 Placement Groups is roughly equivalent in a cluster with less than 50 OSDs as far as data durability is concerned.

As long as there are one or two orders of magnitude more Placement Groups than OSDs, the distribution should be even. For instance, 256 placement groups for 3 OSDs, 512 or 1024 placement groups for 10 OSDs etc.

When using multiple data pools for storing objects, you need to ensure that you balance the number of placement groups per pool with the number of placement groups per OSD so that you arrive at a reasonable total number of placement groups that provides reasonably low variance per OSD without taxing system resources or making the peering process too slow.
```

* PG states explained, [link](https://ceph.readthedocs.io/en/latest/rados/operations/pg-states/).

```
Do not let your cluster reach its full ratio when removing an OSD. Removing OSDs could cause the cluster to reach or exceed its full ratio.
```

## Cephadm containerized deployment

* Add/remove components (daemons, etc.) through `ceph orch (cephadm-backed)` interface, [link](https://ceph.readthedocs.io/en/latest/mgr/orchestrator/). We should use this to manage the cluster if the deployment is done with cephadm (the containerized deployment). However, this document doesn't exactly match the latest `ceph orch (cephadm-backed)` command parameters, we need to use it with great carefulness. Moreover, it might be better use the `ceph orch apply` commands with a given spec file as input, check the [placement specification](https://ceph.readthedocs.io/en/latest/mgr/orchestrator/#placement-specification) and [OSD service specification](https://ceph.readthedocs.io/en/latest/cephadm/drivegroups/#drivegroups) respectively. Command `ceph orch ls --export` can print out the current deployment in a service specification format which we can make use of as kind of template.

* The proper way to restart cluster (deployed with Cephadm on CentOS 8) is `systemctl restart ceph.target`.

* By default, Ceph cluster (deployed by Cephadm) will not write log to files, instead, we need to retrieve them from journald, with command like `journalctl -u <ceph-mon/osd/mgr/*-systemd-service-unit>`. Also we can adjust the log verbosity level like [this](https://ceph.readthedocs.io/en/latest/rados/troubleshooting/troubleshooting-mon/#preparing-your-logs).

* Notes on single-node deployment, [link](https://ceph.readthedocs.io/en/latest/rados/troubleshooting/troubleshooting-pg/#one-node-cluster), especially, if you are trying to create a cluster on a single node, you must change the default of the `osd crush chooseleaf type` setting from 1 (meaning host or node) to 0 (meaning osd) in your Ceph configuration file before you create your monitors and OSDs. This tells Ceph that an OSD can peer with another OSD on the same host.

* Basic operations, [link](https://ceph.readthedocs.io/en/latest/rados/operations/operating/).

* Block device quick start, [link](https://www.server-world.info/query?os=CentOS_8&p=ceph15&f=3).

## Single-node cluster

* Quick guide: single node with multiple OSDs replicated, [link](https://www.jianshu.com/p/d17cbcb16a62).

* `osd_crush_chooseleaf_type` is the key to single-node cluster, check this [link](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/1.2.3/html/installation_guide_for_centos_x86_64/create_a_cluster) for more information or the following:

```
Set a CRUSH leaf type to the largest serviceable failure domain for your replicas under the [global] section of your Ceph configuration file. The default value is 1, or host, which means that CRUSH will map replicas to OSDs on separate separate hosts. For example, if you want to make three object replicas, and you have three racks of chassis/hosts, you can set osd_crush_chooseleaf_type to 3, and CRUSH will place each copy of an object on OSDs in different racks. For example:

osd_crush_chooseleaf_type = 3
The default CRUSH hierarchy types are:

type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root
```

* Result:

```
[ceph: root@node01 /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME        STATUS  REWEIGHT  PRI-AFF
-1         0.02959  root default
-3         0.02959      host node01
 0    hdd  0.00999          osd.0        up   1.00000  1.00000
 1    hdd  0.00980          osd.1        up   1.00000  1.00000
 2    hdd  0.00980          osd.2        up   1.00000  1.00000

[ceph: root@node01 /]# ceph -w
  cluster:
    id:     3c658122-dd50-11ea-84f6-00155dd6e108
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum node01 (age 10h)
    mgr: node01.qlddlw(active, since 10h)
    osd: 3 osds: 3 up (since 7m), 3 in (since 44m)

  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 0 B
    usage:   3.2 GiB used, 27 GiB / 30 GiB avail
    pgs:     1 active+clean
```

## Troubleshootings

* **DO NOT** disable IPv6 in kernel command line parameters otherwise Cephadm would fail to install prometheus and related components for "port already in use" reason (since the Cephadm detects the port usability in both IPv4 and IPv6 contexts).

* Ceph orchestrator [bug](https://tracker.ceph.com/issues/45726) with version 15.2.4.

* Troubleshooting with cephx error and recovering a broken cluster, [link](https://segmentfault.com/a/1190000012348586).

* Useful Ceph blog post collection, [link](http://www.xuxiaopang.com/2020/10/09/list/).

* If `ceph -s` doesn't finish, it means something went wrong with the monitors. You can contact each monitor individually asking them for their status, regardless of a quorum being formed. This can be achieved using `ceph tell mon.ID mon_status`, ID being the monitor’s identifier. You should perform this for each monitor in the cluster. In section [Understanding mon_status](https://ceph.readthedocs.io/en/latest/rados/troubleshooting/troubleshooting-mon/#understanding-mon-status) we will explain how to interpret the output of this command. For the rest of you who don’t tread on the bleeding edge, you will need to ssh into the server and use the monitor’s admin socket. Please jump to [Using the monitor’s admin socket](https://ceph.readthedocs.io/en/latest/rados/troubleshooting/troubleshooting-mon/#using-the-monitor-s-admin-socket).

* When playing with the rbd-related commands, you may encounter the `failed to load rbd kernel module` error, it can be solved by simply `dnf install librbd1 && modprobe rbd`, see [link](https://blog.csdn.net/HYZX_9987/article/details/103636488).

## Benchmarking

According to this [guide](https://tracker.ceph.com/projects/ceph/wiki/Benchmark_Ceph_Cluster_Performance):

* Benchmark disks.

```
[root@node01 tmp]# dd if=/dev/zero of=here bs=1G count=1 oflag=direct
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 5.61522 s, 191 MB/s
```
* Benchmark network.

Single-node networking:

```
[root@node01 ~]# iperf3 -c 192.168.56.10
Connecting to host 192.168.56.10, port 5201
[  5] local 192.168.56.10 port 51708 connected to 192.168.56.10 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  3.55 GBytes  30.5 Gbits/sec    0   3.12 MBytes
[  5]   1.00-2.00   sec  3.40 GBytes  29.2 Gbits/sec    0   3.12 MBytes
[  5]   2.00-3.00   sec  3.44 GBytes  29.5 Gbits/sec    0   3.12 MBytes
[  5]   3.00-4.00   sec  3.47 GBytes  29.8 Gbits/sec    0   3.12 MBytes
[  5]   4.00-5.00   sec  3.48 GBytes  29.9 Gbits/sec    3   3.12 MBytes
[  5]   5.00-6.00   sec  2.55 GBytes  21.9 Gbits/sec    7   2.25 MBytes
[  5]   6.00-7.00   sec  3.62 GBytes  31.1 Gbits/sec    1   3.06 MBytes
[  5]   7.00-8.00   sec  3.26 GBytes  28.0 Gbits/sec    3   3.18 MBytes
[  5]   8.00-9.00   sec  3.09 GBytes  26.6 Gbits/sec   18   3.18 MBytes
[  5]   9.00-10.00  sec  3.49 GBytes  29.9 Gbits/sec    0   3.18 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  33.4 GBytes  28.7 Gbits/sec   32             sender
[  5]   0.00-10.04  sec  33.4 GBytes  28.5 Gbits/sec                  receiver

iperf Done.
```

* Benchmark storage cluster (low-level).

Write:

```
[ceph: root@node01 /]# rados bench -p scbench 10 write --no-cleanup
hints = 1
Maintaining 16 concurrent writes of 4194304 bytes to objects of size 4194304 for up to 10 seconds or 0 objects
Object prefix: benchmark_data_node01_1569
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0       0         0         0         0         0           -           0
    1      16        16         0         0         0           -           0
    2      16        16         0         0         0           -           0
    3      16        16         0         0         0           -           0
    4      16        20         4    3.9976         4     3.66637     3.38555
    5      16        27        11   8.79207        28      4.9936     3.78128
    6      16        35        19   12.6501        32     2.77277     4.27503
    7      16        36        20   11.4145         4     2.34223     4.17839
    8      16        37        21   10.4879         4     3.86408     4.16342
    9      16        41        25   11.0991        16     4.10731     4.05346
   10      16        42        26   10.3892         4     4.35738     4.06514
   11       7        42        35   12.7146        36     5.01434     4.36914
Total time run:         11.6698
Total writes made:      42
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     14.3962
Stddev Bandwidth:       13.9088
Max bandwidth (MB/sec): 36
Min bandwidth (MB/sec): 0
Average IOPS:           3
Stddev IOPS:            3.4772
Max IOPS:               9
Min IOPS:               0
Average Latency(s):     4.17318
Stddev Latency(s):      1.23208
Max latency(s):         6.20708
Min latency(s):         1.61562
```

Sequential read:

```
[ceph: root@node01 /]# rados bench -p scbench 10 seq
hints = 1
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0      16        16         0         0         0           -           0
Total time run:       0.155492
Total reads made:     42
Read size:            4194304
Object size:          4194304
Bandwidth (MB/sec):   1080.44
Average IOPS:         270
Stddev IOPS:          0
Max IOPS:             41
Min IOPS:             41
Average Latency(s):   0.0493663
Max latency(s):       0.115975
Min latency(s):       0.0125881
```

Random read:

```
[ceph: root@node01 /]# rados bench -p scbench 10 rand
hints = 1
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0      16        16         0         0         0           -           0
    1      16       304       288   1149.53      1152    0.108819   0.0526388
    2      15       595       580   1158.46      1168   0.0130736   0.0530025
    3      15       854       839   1117.29      1036    0.115514   0.0557318
    4      16      1103      1087   1085.36       992   0.0468514    0.057555
    5      16      1382      1366   1090.63      1116    0.112379   0.0572858
    6      16      1648      1632   1086.17      1064   0.0239004   0.0576594
    7      16      1931      1915   1092.68      1132   0.0143503   0.0573722
    8      16      2206      2190   1093.45      1100   0.0250431   0.0573589
    9      15      2480      2465   1094.12      1100    0.019276   0.0573806
   10      12      2753      2741   1095.02      1104   0.0268442   0.0574487
Total time run:       10.0655
Total reads made:     2753
Read size:            4194304
Object size:          4194304
Bandwidth (MB/sec):   1094.03
Average IOPS:         273
Stddev IOPS:          13.3204
Max IOPS:             292
Min IOPS:             248
Average Latency(s):   0.0574556
Max latency(s):       0.229901
Min latency(s):       0.00493496
```

* Benchmark cluster storage (high-level).

Write (rbd 1GB image):

```
[ceph: root@node01 /]# rbd bench --io-type write -p rbdbench image01
bench  type write io_size 4096 io_threads 16 bytes 1073741824 pattern sequential
  SEC       OPS   OPS/SEC   BYTES/SEC
    1      6640   5884.98    23 MiB/s
    2      8560      3798    15 MiB/s
    3      9856    3058.2    12 MiB/s
    4     12672   2986.78    12 MiB/s
    6     14336   2389.58   9.3 MiB/s
    7     18032   1821.53   7.1 MiB/s
    8     21680   2062.22   8.1 MiB/s
    9     23376   2315.83   9.0 MiB/s
   10     25520   2179.81   8.5 MiB/s
   11     29360   2862.77    11 MiB/s
   12     32432   2777.74    11 MiB/s
   14     33264   1861.46   7.3 MiB/s
   15     35280   2006.38   7.8 MiB/s
   18     35648   1209.15   4.7 MiB/s
   19     39248   1276.68   5.0 MiB/s
   20     40272   1003.31   3.9 MiB/s
   ...truncated...
```

Write (mount the rbd image and benchmark with `dd`):

```
[root@node01 rbd-image01]# dd if=/dev/zero of=here bs=512M count=1 oflag=direct
1+0 records in
1+0 records out
536870912 bytes (537 MB, 512 MiB) copied, 3.308 s, 162 MB/s
```

* PVE (Proxmox Virtual Environment) has decent support and [documents](https://www.proxmox.com/en/downloads?task=callelement&format=raw&item_id=398&element=f85c494b-2b32-4109-b8c1-083cca2b7db6&method=download&args[0]=d5d1d6e8def72b615f469bac1c721c2d) on Ceph hardware requirement and [benchmarking](https://www.proxmox.com/en/downloads?task=callelement&format=raw&item_id=393&element=f85c494b-2b32-4109-b8c1-083cca2b7db6&method=download&args[0]=96de10a8ce7f24b3ebfe368b20b061a0).

## Other notes

```
We recommend copying the Ceph Storage Cluster’s keyring file to nodes where you will run administrative commands, because it contains the client.admin key.
```

```
(For scenarios which use separated device for bluestore journal) When using a mixed spinning and solid drive setup it is important to make a large-enough block.db logical volume for Bluestore. Generally, block.db should have as large as possible logical volumes.

The general recommendation is to have block.db size in between 1% to 4% of block size. For RGW workloads, it is recommended that the block.db size isn’t smaller than 4% of block, because RGW heavily uses it to store its metadata. For example, if the block size is 1TB, then block.db shouldn’t be less than 40GB. For RBD workloads, 1% to 2% of block size is usually enough.

If not using a mix of fast and slow devices, it isn’t required to create separate logical volumes for block.db (or block.wal). Bluestore will automatically manage these within the space of block.
```

```
If an OSD is down and the degraded condition persists, Ceph may mark the down OSD as out of the cluster and remap the data from the down OSD to another OSD. The time between being marked down and being marked out is controlled by mon osd down out interval, which is set to 600 seconds by default.
```

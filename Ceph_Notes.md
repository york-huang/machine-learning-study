# Ceph Study Notes

### High availability notes

* Ceph best practices dictate that you should run operating systems, OSD data and OSD journals on separate drives. However, when using a single device type (for example, spinning drives), the journals should be colocated: the logical volume (or partition) should be in the same device as the data logical volume. When using a mix of fast (SSDs, NVMe) devices with slower ones (like spinning drives) it makes sense to place the journal on the faster device, while data occupies the slower device fully.

* A cluster will run fine with a single monitor; however, a single monitor is a single-point-of-failure. To ensure high availability in a production Ceph Storage Cluster, you should run Ceph with multiple monitors so that the failure of a single monitor WILL NOT bring down your entire cluster. When a Ceph Storage Cluster runs multiple Ceph Monitors for high availability, Ceph Monitors use Paxos to establish consensus about the master cluster map. A consensus requires a majority of monitors running to establish a quorum for consensus about the cluster map (e.g., 1; 2 out of 3; 3 out of 5; 4 out of 6; etc.). So 3 monitors is the minimum requirement of high availability ([link](https://serverfault.com/questions/989863/high-availability-for-ceph-storage)).

* Ceph provides a default path where Ceph Monitors store data. For optimal performance in a production Ceph Storage Cluster, we recommend running Ceph Monitors on separate hosts and drives from Ceph OSD Daemons. As leveldb is using `mmap()` for writing the data, Ceph Monitors flush their data from memory to disk very often, which can interfere with Ceph OSD Daemon workloads if the data store is co-located with the OSD Daemons.

* The pool type which may either be **replicated** to recover from lost OSDs by keeping multiple copies of the objects or **erasure** to get a kind of generalized RAID5 capability (like RAID5, the simplest erasure coded pool requires at least three hosts; the default erasure code profile will split the data into 2 equal-sized chunks, and have 2 parity chunks of the same size, it will take as much space in the cluster as a 2-replica pool but can sustain the data loss of 2 chunks out of 4, it is described as a profile with k=2 and m=2, meaning the information is spread over four OSD (k+m == 4) and two of them can be lost). The replicated pools require more raw storage but implement all Ceph operations. The erasure pools require less raw storage but only implement a subset of the available operations. 

* When you deploy OSDs they are automatically placed within the CRUSH map under a host node named with the hostname for the host they are running on. This, combined with the default CRUSH failure domain, ensures that replicas or erasure code shards are separated across hosts and a single host failure will not affect availability. For larger clusters, however, administrators should carefully consider their choice of failure domain. Separating replicas across racks, for example, is common for mid- to large-sized clusters.

### CRUSH algorithm

* The [paper](https://ceph.com/wp-content/uploads/2016/08/weil-crush-sc06.pdf).

* CRUSH is implemented as a pseudo-random, deterministic function that maps an input value, typically an object or object group identifier, to a list of devices on which to store object replicas.

* CRUSH can calculate any new data’s storage target without consulting a central allocator (metadata / directory, etc.), it relies on a compact cluster description and deterministic mapping function, which makes CRUSH quite different from other early counterparts.

* Given a single integer input value $\mathcal x$, CRUSH will output an ordered list $\vec R$ of $\mathcal n$ distinct storage targets. CRUSH utilizes a strong multi-input integer hash function whose inputs include $\mathcal x$, making the mapping completely deterministic and independently calculable using only the cluster map, placement rules, and $\mathcal x$. The distribution is pseudo-random in that there is no apparent correlation between the resulting output from similar inputs or in the items stored on any storage device. We say that CRUSH generates a declustered distribution of replicas in that the set of devices sharing replicas for one item also appears to be independent of all other items. 

#### Hierarchical cluster map

* The cluster map is composed of devices and buckets, both of which have numerical identifiers and weight values associated with them. Weights, assigned by administrator, corresponds to devices' capability, and it leads to the amount of data they are responsible for storing. Bucket weights are defined as the sum of the weights of the items they contain.

* Buckets can contain any number of devices or other buckets, allowing them to form interior nodes in a storage hierarchy in which devices are always at the leaves.

### Ceph architecture and workflow

* Ceph Clients retrieve a Cluster Map from a Ceph Monitor, and write objects to pools. The pool’s size or number of replicas, the CRUSH rule and the number of placement groups determine how Ceph will place the data. Each pool has a number of placement groups. CRUSH maps PGs to OSDs dynamically. When a Ceph Client stores objects, CRUSH will map each object to a placement group.

* The CRUSH algorithm maps each object to a placement group and then maps each placement group to one or more Ceph OSD Daemons. This layer of indirection allows Ceph to rebalance dynamically when new Ceph OSD Daemons and the underlying OSD devices come online. With a copy of the cluster map and the CRUSH algorithm, the client can compute exactly which OSD to use when reading or writing a particular object.

* Ceph clients use the following steps to compute PG IDs.

  1. The client inputs the pool name and the object ID. (e.g., pool = “liverpool” and object-id = “john”)

  2. Ceph takes the object ID and hashes it.

  3. Ceph calculates the hash modulo the number of PGs. (e.g., 58) to get a PG ID.

  4. Ceph gets the pool ID given the pool name (e.g., “liverpool” = 4)

  5. Ceph prepends the pool ID to the PG ID (e.g., 4.58).

* In a typical write scenario, a client uses the CRUSH algorithm to compute where to store an object, maps the object to a pool and placement group, then looks at the CRUSH map to identify the primary OSD for the placement group. The client writes the object to the identified placement group in the primary OSD. Then, the primary OSD with its own copy of the CRUSH map identifies the secondary and tertiary OSDs for replication purposes, and replicates the object to the appropriate placement groups in the secondary and tertiary OSDs (as many OSDs as additional replicas), and responds to the client once it has confirmed the object was stored successfully.

* The Ceph Storage Cluster was designed to store at least two copies of an object (i.e., size = 2), which is the minimum requirement for data safety. For high availability, a Ceph Storage Cluster should store more than two copies of an object (e.g., size = 3 and min size = 2) so that it can continue to run in a degraded state while maintaining data safety.

* When a series of OSDs are responsible for a placement group, that series of OSDs, we refer to them as an *Acting Set*. An *Acting Set* may refer to the Ceph OSD Daemons that are currently responsible for the placement group, or the Ceph OSD Daemons that were responsible for a particular placement group as of some epoch. The Ceph OSD daemons that are part of an *Acting Set* may not always be up. When an OSD in the *Acting Set* is up, it is part of the *Up Set*. The *Up Set* is an important distinction, because Ceph can remap PGs to other Ceph OSD Daemons when an OSD fails.  In an *Acting Set* for a PG containing osd.25, osd.32 and osd.61, the first OSD, osd.25, is the *Primary*. If that OSD fails, the *Secondary*, osd.32, becomes the *Primary*, and osd.25 will be removed from the *Up Set*.

* An erasure coded pool stores each object as K+M chunks. It is divided into K data chunks and M coding chunks. The pool is configured to have a **size** (number of replicas) of K+M so that each chunk is stored in an OSD in the acting set. The rank of the chunk is stored as an attribute of the object. For instance an erasure coded pool is created to use five OSDs (K+M = 5) and sustain the loss of two of them (M = 2).

* Ceph provides three types of clients: Ceph Block Device, Ceph File System, and Ceph Object Storage. A Ceph Client converts its data from the representation format it provides to its users (a block device image, RESTful objects, CephFS filesystem directories) into objects for storage in the Ceph Storage Cluster. The objects Ceph stores in the Ceph Storage Cluster are not striped. Ceph Object Storage, Ceph Block Device, and the Ceph File System stripe their data over multiple Ceph Storage Cluster objects. Ceph Clients that write directly to the Ceph Storage Cluster via librados must perform the striping (and parallel I/O) for themselves to obtain these benefits. To be noted that, striping is independent of object replicas. Since CRUSH replicates objects across OSDs, stripes get replicated automatically.

* Three important variables determine how Ceph stripes data:

  * Object Size: Objects in the Ceph Storage Cluster have a maximum configurable size (e.g., 2MB, 4MB, etc.). The object size should be large enough to accommodate many stripe units, and should be a multiple of the stripe unit.

  * Stripe Width: Stripes have a configurable unit size (e.g., 64kb). The Ceph Client divides the data it will write to objects into equally sized stripe units, except for the last stripe unit. A stripe width, should be a fraction of the Object Size so that an object may contain many stripe units.

  * Stripe Count: The Ceph Client writes a sequence of stripe units over a series of objects determined by the stripe count. The series of objects is called an object set. After the Ceph Client writes to the last object in the object set, it returns to the first object in the object set.

### Deployment hints

#### Useful resources

* Deployment on AWS EC2 instances with Ceph-deploy (for version Nautilus), [link](https://blog.risingstack.com/ceph-storage-deployment-vm/).

* Deployment on Centos 8 clusters with or without Cephadm, [link1]( https://www.server-world.info/en/note?os=CentOS_8&p=ceph15&f=9), [link2](https://www.server-world.info/query?os=CentOS_8&p=ceph15&f=1).

* SUSE Enterprise Storage 6 deployment guide, [link](https://documentation.suse.com/ses/6/single-html/ses-deployment/).

* Rook, deploying Ceph to an existing Kubernetes clusters, [link](https://rook.io/docs/rook/v1.3/ceph-quickstart.html).

* There are two types of disk space needed to run on OSD: the space for the disk journal (for FileStore) or WAL/DB device (for BlueStore), and the primary space for the stored data. The minimum (and default) value for the journal/WAL/DB is 6 GB. The minimum space for data is 5 GB, as partitions smaller than 5 GB are automatically assigned the weight of 0. So although the minimum disk space for an OSD is 11 GB, we do not recommend a disk smaller than 20 GB, even for testing purposes.

* Another consideration is whether to share disks between an OSD, a monitor node, and the operating system on the server. The answer is simple: if possible, dedicate a separate disk to OSD, and a separate server to a monitor node. Although Ceph supports directory-based OSDs, an OSD should always have a dedicated disk other than the operating system one. If it is really necessary to run OSD and monitor node on the same server, run the monitor on a separate disk by mounting the disk to the /var/lib/ceph/mon directory for slightly better performance.

### Operation hints

* How Ceph Calculates Data Usage? The `usage` value (returned from command `ceph status`) reflects the **actual** amount of raw storage used. The `xxx GB / xxx GB` value means the amount available (the lesser number) of the overall storage capacity of the cluster. The **notional** number reflects the size of the stored data **before** it is replicated, cloned or snapshotted. Therefore, the amount of data actually stored typically exceeds the notional amount stored, because Ceph creates replicas of the data and may also use storage capacity for cloning and snapshotting. At the meantime, the `RAW STORAGE` section from the output of command `ceph df` reflects the actual usage of storage which includes overheads of replicas, clones and snapshots.

* Placement group status explained.

  * **Active**. Once Ceph completes the peering process, a placement group may become active. The active state means that the data in the placement group is generally available in the primary placement group and the replicas for read and write operations.

  * **Clean**. When a placement group is in the clean state, the primary OSD and the replica OSDs have successfully peered and there are no stray replicas for the placement group. Ceph replicated all objects in the placement group the correct number of times.

* As long as there are one or two orders of magnitude more Placement Groups than OSDs, the distribution should be even. For instance, 256 placement groups for 3 OSDs, 512 or 1024 placement groups for 10 OSDs etc. If you have more than 50 OSDs, we recommend approximately 50-100 placement groups per OSD to balance out resource usage, data durability and distribution. If you have less than 50 OSDs, choosing among the preselection mentioned previously is best. For a single pool of objects, you can use the following formula to get a baseline (where pool size is either the number of replicas for replicated pools or the K+M sum for erasure coded pools; the result should always be rounded up to the nearest power of two):

```
             (OSDs * 100)
Total PGs =  ------------
              pool size
```

* Online placement group calculator, [link](https://ceph.com/pgcalc/).

*  Do not let your cluster reach its full ratio before adding an OSD. OSD failures that occur after the cluster reaches its near full ratio may cause the cluster to exceed its full ratio.

### Misc

* 2-way / 3-way mirroring, [link](https://www.starwindsoftware.com/resource-library/2-way-vs-3-way-synchronous-mirroring-comparison/).
# Single-node Rook/Ceph on a 2-nodes k3s Kubernetes cluster

All works are done on a Windows 10 laptop (Surface Book 1st gen): two Hyper-V VMs are used to run a k3s Kubernetes cluster (1 master node + 1 agent node), then the Rook/Ceph is deployed on top of it, with a single-node setup, i.e., one mon, one mgr and one osd (on a raw block device) on one single node (VM).

Such deployment is served for the purpose of proof-of-concept and testing.

### Prepare the k3s Kubernetes cluster

Perform the following steps on two Hyper-V VMs.

* Grab the Alpine Linux v3.12 virt image ISO file, install as instructed [here](https://wiki.alpinelinux.org/wiki/Alpine_Install:_from_a_disc_to_a_virtualbox_machine_single_only), some notes should be taken:

  * For the agent node, attach a secondary virtual disk (such as /dev/sdb, leave it raw, no filesystem, no Alpine installation on it) which is used as the ceph-volume later. **NOTE**: I tried a 5GB disk for it but failed in `ceph-volume lvm` step, then I used a 10GB one instead and succeeded in bringing up the OSD on it finally.

  * In Hyper-V firmware settings, disable secure boot to prevent potential issues during system boot.
  
  * In Hyper-V network settings, connect the NIC to the default switch instead of the WSL one since the latter would cause DHCP issues during Alpine installation. By doing this, you can't access Alpine from WSL (WSL and Alpine not in a same network), instead, use PowerShell to do SSH, see below. (More Hyper-V network issues explained in below sections.)

  * In Hyper-V memory settings, either disable dynamic memory or specify a reasonable upper limit, otherwise it would consume excessive amount of memory of the host machine.

  * In Hyper-V management/integration services, enable all items which are supposed to be worked with Alpine's `hvtools`, see below.

  * Windows 10 (with latest update) has SSH client built into the PowerShell, it's convenient to use it to access Alpine.

* (OPTIONAL maybe) After Alpine installed to disk, add `hvtools` (Hyper-V tools) as instructed [here](https://wiki.alpinelinux.org/wiki/Hyper-V_guest_services).

* Add `containerd` which is required by `k3s`.

* Bring up the `cgroups` by `rc-update add cgroups default` then `service cgroups start`, as suggested [here](https://teada.net/k3s-on-alpine-linux/). (Without this step, container service such as docker might not function normally.)

* (OPTIONAL, it's not required by `k3s`) Add `docker` to Alpine, as instructed [here](http://www.imooc.com/article/287437). There are some issues with `cgroups` on Alpine (can see "docker daemon not started" problems in docker daemon log) so the previous `cgroups` step must be done,

  * In order to eliminate the error of "No swap limit support" seen from `docker info` output,  edit `/etc/default/grub`, append `cgroup_enable=memory swapaccount=1` to kernel command line parameters `GRUB_CMDLINE_LINUX_DEFAULT`.

  * Make the grub changes effective, `grub-mkconfig -o /boot/grub/grub.cfg`.

  * Reboot, then check the `cgroups` status by `mount|grep cgroup`, kernel parameters by `cat /proc/cmdline`, also the `docker` status by `docker info`.

* ~~Install `k3d`, `kubectl` and create a small cluster as instructed on other notes.~~ This plan is abandoned since raw block persistent volume (`k3d` cluster runs containers as nodes that Rook Ceph needs raw block PV for Ceph OSD) is troublesome on current setup.

* Install `lvm2` and `rc-update add lvm default`, it's required by Rook/Ceph deployment as stated [here](https://rook.io/docs/rook/v1.3/ceph-prerequisites.html).

* Install `k3s` as instructed [here](https://rancher.com/docs/k3s/latest/en/installation/), make sure one node is installed as `k3s server` while another is `k3s agent`. Better use the script method since it will add `k3s` to the system service, also the `kubectl` symlink, `k3s-killall.sh` and `k3s-uninstall.sh` to the system which are really useful, however, it might be difficult to use it directly (cause the github content server connection issue), so it's reasonable to try the steps below:

  * Download the `k3s` binary from GitHub releases page, copy it to `/usr/local/bin`.

  * Clone the `k3s` repo.

  * Run the `install.sh` script from the repo with environment variable `INSTALL_K3S_SKIP_DOWNLOAD=true`, as instructed [here](https://rancher.com/docs/k3s/latest/en/installation/install-options/). For agent installation, don't forget to set the `K3S_URL` and `K3S_TOKEN` variables accordingly, such as,

```bash
K3S_URL=https://172.20.200.52:6443 K3S_TOKEN=K10c4fea3d5484aa43193a26a8d7645599f479774e5c8857a06c8a682384c5f6202::server:702b424a2ebe8fa897bf65f18f19d13b INSTALL_K3S_SKIP_DOWNLOAD=true ./install.sh
```

* Verify the `k3s` setup by doing checks with `kubectl`, like `kubectl cluster-info`, `kubectl get node` and `kubectl get pod --all-namespaces`. 

### Deploy Rook/Ceph

Following these [steps](https://rook.io/docs/rook/v1.3/ceph-quickstart.html) with some modifications,

```bash
git clone --single-branch --branch release-1.3 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
kubectl create -f common.yaml
kubectl create -f operator.yaml
```

Use the following as `cluster-custom.yaml`,

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [global]
    osd_pool_default_size = 1
---
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: ceph/ceph:v14.2.10  # v15 should be ok, not tried yet.
  dataDirHostPath: /var/lib/rook
  mon:
    count: 1
    allowMultiplePerNode: true
  dashboard:
    enabled: true
  # cluster level storage configuration and selection
  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
    - name: "worker01-1b456cff"  # node's 'name' field should match its 'kubernetes.io/hostname' label, check it with 'kubectl describe node xxxx -n rook-ceph'.
      devices:  # specific devices to use for storage can be specified for each node
      - name: "sdb"  # Whole storage device
      config:  # configuration can be specified at the node level which overrides the cluster level config
        storeType: bluestore
```

Then create it,

```bash
kubectl create -f cluster-custom.yaml
```

Use `kubectl get pod -n rook-ceph` to confirm the status of the deployment. Furthermore, one can deploy `ceph-toolbox` to do more with the ceph things, as instructed [here](https://rook.io/docs/rook/v1.3/ceph-toolbox.html).

### Hyper-V network issues

* Hyper-V virtual ethernet adapters (both `default switch` and `WSL switch`) network (IP range) change on every reboot of the host Windows, causing the IP assigned to VMs invalid after host reboot. It's a problem since the changing network prevents lots of services from functioning correctly, including the Kubernetes and Ceph cluster we setup in previous sections, see this thread: https://github.com/microsoft/WSL/issues/4210#issuecomment-629135832.

* Simply put, all that we need is Hyper-V VMs have working static IPs which are able to survive in host (Windows 10, in my case) reboot. A possible workaround (**NOT** yet verified) is Hyper-V NAT, which is documented [here](https://www.cnblogs.com/wswind/p/11007613.html).

* Configure static IPs on Alpine, see [here](https://wiki.alpinelinux.org/wiki/Configure_Networking#IPv4_Static_Address_Configuration). 

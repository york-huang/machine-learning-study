# Learning OpenStack Deployment

Major deployment methods: [manual](https://docs.openstack.org/install-guide/), [DevStack](https://docs.openstack.org/devstack/latest/), [TripleO](http://tripleo.org/index.html) / [TripleO-Quickstart](https://docs.openstack.org/tripleo-quickstart/latest/), study notes as below.

### Manual install

OpenStack Ussuri on CentOS 8 step-by-step installation guide, [link](https://www.server-world.info/query?os=CentOS_8&p=openstack_ussuri&f=1). (BTW: this website has many other useful tutorials for other software and Linux distro.)

### DevStack

* All-in-one-VM method, using Windows 10 Hyper-V.

* Tried Centos 8.2.2004 as VM OS, failed; tried Ubuntu 18.04 LTS, succeeded after a few error-and-trials.

* Mainly followed the official [guide](https://docs.openstack.org/devstack/latest/index.html).

* Used this `local.conf`

```
[[local|localrc]]

# Password for KeyStone, Database, RabbitMQ and Service
ADMIN_PASSWORD=StrongAdminSecret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# Host IP - get your Server/VM IP address from ip addr command
HOST_IP=<your-VM-IP>  # shall be optional; noted that it might be changed after VM restarted by the hypervisor.

SKIP_PATH_SANITY=1  # it might cause unnecessary interruptions.

disable_service tempest  # tempest seems not to work with DevStack.
```

* When `g-api did not start` error encountered, check `/var/log/syslog` see if you can find error messages about boto3 module, if yes, install it with pip manually.

* If mysqld not started after `stack.sh` error  interruptions, tried `unstack.sh -a && clean.sh && stack.sh`.

* Messages on installation completed as below.

```
This is your host IP address: 172.17.154.134
This is your host IPv6 address: ::1
Horizon is now available at http://172.17.154.134/dashboard
Keystone is serving at http://172.17.154.134/identity/
The default users are: admin and demo
The password: StrongAdminSecret

Services are running under systemd unit files.
For more information see:
https://docs.openstack.org/devstack/latest/systemd.html

DevStack Version: victoria
Change: 3b37b95684d351af19bccae4d0fee4135aa00857 Merge "Allow plugins to override initial network creation" 2020-07-09 15:11:25 +0000
OS Version: Ubuntu 18.04 bionic

2020-07-12 14:48:04.977 | stack.sh completed in 1137 seconds.
```

### TripleO

TripleO stands for "OverStack On OverStack", which means TripleO manages to deploy a production OpenStack cloud (called overcloud) by firstly provisioning a small OpenStack cloud (called undercloud) then leveraging the facilities of it to carry out the actual deployment work. 

### TripleO-Quickstart

An Ansible-based automation for development, testing and proof-of-concept deployment, the concept is as below,

- A baremetal host required (called VIRTHOST).
- A virtual environment setup on the VIRTHOST with libvirt (KVM or QEMU as hypervisor).
- Necessary VM instances then provisioned on the virtual environment of the VIRTHOST, some instances deployed as undercloud, others deployed as overcloud.
- VM instances are usually deployed with pre-built qcow2 images.

Very powerful, swift, yet flexible, as long as you understand the TripleO concepts, networking, and Ansible mechanism such as playbook, roles, etc.

Instead of setting up the virtual environment on a baremetal, there is an alternative method to do similar deployment on a public cloud, it's called OVB (OpenStack Virtual Baremetal), described [here](https://openstack-virtual-baremetal.readthedocs.io/en/latest/index.html).

### Other useful resources

- [Setting up OVN (Open Virtual Network)](https://docs.openstack.org/neutron/ussuri/install/ovn/tripleo_install.html) with TripleO-Quickstart.
- TripleO-Quickstart [cheatsheet](https://superuser.openstack.org/articles/new-tripleo-quick-start-cheatsheet/).
- YAML quick [tutorial](https://rollout.io/blog/yaml-tutorial-everything-you-need-get-started/).
- Ansible 2 [tutorial](https://serversforhackers.com/c/an-ansible2-tutorial).
- [TripleO-Validations](https://docs.openstack.org/tripleo-quickstart/latest/validations.html), methods and tools to validate the deployment.
- [JSON/YAML converter](https://www.json2yaml.com/).
- IP CIDR [calculator](http://www.subnet-calculator.com/cidr.php), decimal/binary [convertor](https://www.rapidtables.com/convert/number/decimal-to-binary.html).
- What is cloud-native? [link](https://www.ibm.com/cloud/learn/cloud-native).
- Introduction to Kubenetes, [link](https://www.redhat.com/en/topics/containers/what-is-kubernetes).
- Docker on WSL2, [link](https://dev.to/bartr/install-docker-on-windows-subsystem-for-linux-v2-ubuntu-5dl7).

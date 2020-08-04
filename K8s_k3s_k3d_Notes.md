# K8s / k3s / k3d Notes

## K8s (Kubernetes) concepts

Notes are mainly taken from this [article](https://www.digitalocean.com/community/tutorials/an-introduction-to-kubernetes).

### Architecture and components

K8s is a system for running and coordinating (so called orchestration) containerized applications across a cluster of machines (bare metal or VM).

A K8s cluster consists of one or more **master servers** and multiple **worker nodes**. Master servers act as a gateway and brain for the cluster by exposing an API for users or clients, health checking other servers, deciding how best to split up and assign work (known as “scheduling”), and orchestrating communication between other components. Worker nodes are responsible for accepting and running workloads using local and external resources. Typically, workloads (applications and services) assigend to worker nodes are run in containers, so that the nodes receive work instructions from the master servers and create or destroy containers accordingly, adjusting networking rules to route and forward traffic appropriately.

Users interact with the cluster by communicating with the main API server (provided by master servers) either directly or with clients and libraries. To start up an application or service, a declarative plan is submitted in JSON or YAML defining what to create and how it should be managed. The master server then takes the plan and figures out how to run it on the infrastructure by examining the requirements and the current state of the system. 

#### Master server components

* etcd. A lightweight distributed key-value store, however, in a k3s/k3d cluster, some other data stores are used instead, such as sqlite or dqlite.

* kube-apiserver. Acts as the bridge between various components to maintain cluster health and disseminate information and commands by providing a RESTful interface. Specifically, a client called **kubectl** is available as a default method of interacting with the Kubernetes cluster from a local computer by communicating with the kube-apiserver.

* kube-controller-manager. Manages different controllers that regulate the state of the cluster, manage workload life cycles, and perform routine tasks. 

* kube-scheduler. Reads in a workload’s operating requirements, analyzes the current infrastructure environment, and places the work on an acceptable node or nodes.

* cloud-controller-manager. Act as the glue that allows Kubernetes to interact providers with different capabilities, features, and APIs while maintaining relatively generic constructs internally.

#### Worker node components

* A container runtime. Such as docker, rkt, runc and containerd, responsible for starting and managing containers, applications encapsulated in a relatively isolated but lightweight operating environment.

* kubelet. Main contact point for each workder node, responsible for  relaying information to and from the control plane services (provided by master servers), as well as interacting with the etcd store to read configuration details or write new values, controlling the container runtime to launch or destroy containers as needed.

* kube-proxy. Forwards requests to the correct containers, does primitive load balancing, and be generally responsible for making sure the networking environment is predictable and accessible, but isolated where appropriate.

### Objects and workloads

* Pods.  The most basic unit K8s deals with, consist of containers that operate closely together, share a life cycle, and should always be scheduled on the same node. They are managed entirely as a unit and share their environment, volumes, and IP space. 

* Replication controllers. Defines a pod template and control parameters to scale identical replicas of a pod horizontally by increasing or decreasing the number of running copies.
a component that acts as a basic internal load balancer and ambassador for pods. A service groups together logical collections of pods that perform the same function to present them as a single entity.
* Replication sets. A replacement of replication controllers, with greater flexibilities.

* Deployments. Built with replications sets, designed to ease the life cycle management of replicated pods.

* Stateful sets. Specialized pod controllers that offer ordering and uniqueness guarantees.

* Daemon sets.  Specialized form of pod controller that run a copy of a pod on each node in the cluster (or a subset, if specified). 

* Jobs.  Task-based workflow where the running containers are expected to exit successfully after some time once they have completed their work. There are also cron jobs, which provides an interface to run jobs with a scheduling component.

* Services. A component that acts as a basic internal load balancer and ambassador for pods by  grouping together logical collections of pods that perform the same function to present them as a single entity.

* Volumes. Abstraction layer that allows data to be shared by all containers within a pod and remain available until the pod is terminated. Container failures within the pod will not affect access to the shared files. Once the pod is terminated, the shared volume is destroyed. In contrast, **persistent volumes** is not tied to the pod life cycle, once a pod is done with a persistent volume, the volume’s reclamation policy determines whether the volume is kept around until manually deleted or removed along with the data immediately. Persistent data can be used to guard against node-based failures and to allocate greater amounts of storage than is available locally.

### Example: deploy Nginx to Kubernetes cluster

Notes taken from [this](https://devopscube.com/kubernetes-deployment-tutorial/) comprehensive Kubernetes Nginx deployment tutorial. 

* Overview: Namespace -> Deployment -> Pod <- Service. In Kubernetes, pods are the basic units that get deployed in the cluster. Kubernetes deployment is an abstraction layer for the pods. The main purpose of the deployment object is to maintain the resources declared in the deployment configuration (typically the YAML files) in its desired state.

* Deployment key concepts:

  * A Deployment can schedule multiple pods. A pod as a unit cannot scale by itself.

  * A Deployment represents a single purpose with a group of PODs.

  * A single POD can have multiple containers and these containers inside a single POD shares the same IP (so-called podIP) and can talk to each other using localhost address.

  * To access a Deployment with one or many PODs, you need a Kubernetes **Service endpoint mapped to the deployment** using labels and selectors.

* Namespace and Deployment sample YAML files.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: deployment-demo
  labels:
    apps: web-based
  annotations:
    type: demo
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-quota
  namespace: deployment-demo
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
  namespace: deployment-demo
  annotations:
    monitoring: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: "2Gi"
            cpu: "1000m"
          requests: 
            memory: "1Gi"
            cpu: "500m"
```

* Service sample YAML file.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: deployment-demo
spec:
  ports:
  - nodePort: 30500
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: NodePort
```

With `app: nginx` in `labels` section as a selector, the service will be able to match the pods in previous Nginx deployment as the deployment and the pods have the same label. So automatically all the requests coming to the Nginx service will be sent to the Nginx deployment. Now, we are able to access the Nginx service on any one of the Kubernetes node IP on port 30500.

## k3d deployment notes

[k3s](https://k3s.io/) is kind of lightweight Kubernetes which can setup a complete functional Kubernetes cluster with a few lines of command, while [k3d](https://k3d.io/) is a docker/container version of k3s, which is able to run a multi-nodes Kubernetes cluster on a single machine (also can run the cluster on multiple machines, like a typical Kubernetes cluster setup). I've tried k3d (v3.0.0) on WSL2 for some time, notes are made as below.

### Install notes

* Follow the [instructions](https://k3d.io/#installation) on official website to get k3d command installed.
* docker is required.
* kubectl is also required, install instructions can be found [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
* Create and run a k3d cluster with multiple nodes (containers actually) as [instructed](https://github.com/rancher/k3d/#usage) on the official website.

#### Install helm

Helm is the package manager for Kubernetes cluster, a brilliant introduction can be found [here](https://www.digitalocean.com/community/tutorials/an-introduction-to-helm-the-package-manager-for-kubernetes). Install instruction can be found [here](https://www.linode.com/docs/kubernetes/how-to-install-apps-on-kubernetes-with-helm-3/), since Helm 2 is about to be deprecated, I used Helm 3 (v3.2.4) here.

##### Example: install and run Kubernetes dashboard

* Follow the K8s dashboard helm install instructions [here](https://hub.helm.sh/charts/k8s-dashboard/kubernetes-dashboard), complete the installation, the commands I used are as below (NOTE: it would install dashboard into namespace 'default'; the version I used is Kubernetes dashboard 2.0.3 / helm chart version 2.3.0).

```bash
helm repo add k8s-dashboard https://kubernetes.github.io/dashboard
helm install -g k8s-dashboard/kubernetes-dashboard  # -g let helm generate a random name-postfix.
```
The message below will be showed upon installation completed (follow it, we can see the dashboard sign in page in the browser, the authentication method is detailed in next section).

```bash
Get the Kubernetes Dashboard URL by running:
  export POD_NAME=$(kubectl get pods -n default -l "app.kubernetes.io/name=kubernetes-dashboard,app.kubernetes.io/instance=kubernetes-dashboard-1595513335" -o jsonpath="{.items[0].metadata.name}")
  echo https://127.0.0.1:8443/
  kubectl -n default port-forward $POD_NAME 8443:8443
```

* Follow the instructions [here](https://freshbrewed.science/getting-started-with-k3s/index.html), prepare a YAML file as below (you can change the name `dashboard-admin` to other names you choose).

```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: default
```

* Create the service account and the cluster role binding (bind it to ClusterRole/cluster-admin so that you can access everything through the dashboard) as below,

```bash
kubectl apply -f dashboard-admin.yaml -n default  # since the dashboard is installed under namespace 'default'
```

Grab the bearer token for the newly created dashboard-admin account,

```bash
kubectl -n default describe secret $(kubectl -n default get secret | grep dashboard-admin | awk '{print $1}')
```

Copy the token from the above output, use it in the dashboard sign in page, that is it.

### Useful commands

After the cluster is created and running, the following commands are useful for mangement.

* The default KUBECONFIG is `~/.kube/config`.

* `k3d cluster list`, `k3d node list` and `k3d kubeconfig get <cluster-name>`.

* NOTE: `cluster name` in k3d is **NOT** equal to `namespace name` in kubectl, don't get confused.

* `kubectl cluster-info` and `kube cluster-info dump`, detailed information of the cluster.

* Find out all pods names and status: `kubectl get pods --namespace <cluster namespace, e.g., 'kube-system'>`.

* Similar commands include `kubectl get svc ...`, `kubectl get jobs ...`, `kubectl get deployments ...`, `kubectl get nodes ...`, etc.

* Find out more details such as which node a given pod is running on, `kubectl describe pod <pod name> --namespace <cluster namespace>`.

* Find out details for a given node (pods running, resources allocated, network, conditions, events, etc.), `kubectl describe node <node name> --namespace <cluster namespace>`. 

* Find out logs for a given pod, `kubectl logs <pod name> --namespace <cluster namespace>` (NOTE: the `pod name` must be the full name showed in the output of `kubectl get pods ...`).

* Directly access the given pod (its first container, by default), `kubectl exec -it -n <cluster namespace> <pod> -- <command such as /bin/sh>`.

* By using the option `-o wide` in `kubectl get ...` commands, we can see more useful information in the output table.

* Find out what containers are actually running as nodes of the cluster, `docker ps -a`.

* Find out more about a given container (such as how the network and volumes are attached to it), `docker container inspect <container id or name>`, furthermore, one can really look inside the running container by `docker exec -it <container id or name> /bin/sh`.

* In k3d cluster, pod `local-path-provisioner` is deployed by default, we can find out some useful information by running commands as below:

  * Introduction and usage can be found on [here](https://github.com/rancher/local-path-provisioner/blob/master/README.md).

  * `kubectl get storageclass -n <cluster namespace>`

* The [configmap](https://kubernetes.io/docs/concepts/configuration/configmap/) is also worthy of a look,

  * `kubectl get configmap -n <cluster namespace>`

  * `kubectl describe configmap/<config name> -n <cluster namespace>`

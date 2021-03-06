# Kubernetes Notes

### Basic Terminologies :

**Pods**
: Pod is group of containers. Pod can group together can container images into a single deployable unit. Most of the time
one container is run in one pod unless there is dependency.

**Node:**  Kubernetes runs workload by placing container into Pods to run on Nodes.
A node may be a  virtual or physical machine depending on the cluster.

**Cluster:** A kubernetes cluster is set of nodes running an applications.
Kubernetes cluster consist of one master node and set of worker nodes. These nodes can be either
physical computers or virtual machines. The master node controls the state of the cluster.

**Services:** Kubernetes services provide load balancing, naming and discovery to isolate one microservice from another

**Namespace:** Namespace provide isolation and access control, so that each microservice can control the degree to which other services interact with it
. Namespace is a way to a Kubernetes user to organize different clusters within just one physical cluster.

### K8S Architecture

Kubernetes cluster have two types of nodes.
1. **Master Node**
2. **Worker Node**

![alt text](https://github.com/Shaad7/notes/blob/master/images/K8S_Archi.png?raw=true 
"Kubernetes Architecture")

**API Server:** Users interact with k8s cluster via API Server in master node. Client of this api server can
be Kubernetes Dashboard(UI) or Kubectl(CLI) or any script.

**Scheduler:** Kubernetes Scheduler assigns pods to different nodes. It checks the requirements of pods and find 
a node which meets the requirement to run the pod. It also schedules nodes so that the pods are distributed 
among all the nodes.

**Controller Manager:** Control Manager is responsible for overall health of the cluster. It ensures correct
number of pod running as needed.

**ETCD:** It is called brain of the cluster. It is a key-value storage system. It stores the current state of 
the cluster. It stores nodes, controller and all the state related things. Any component of cluster can query etcd.

**Kubelet:** Kubelet is responsible for running the pods in a worked node. When scheduler schedules node, kubelet
runs the node according to pod spec. It tries to create a new pod if pod fails. If it can not 
restart failed pod, master detects a node failure and schedule the pod in another node.

**Kube-proxy:** Kube-proxy is responsible for maintaining network across the cluster. It maintains internet among
nodes, pods and containers.

**Container-Runtime (Docker):** It runs the container inside the pod.

### Kubernetes Objects

**Namespace** provides a degree of isolation withing the cluster. Namespace-based scoping 
is applicable only for namespaced objects(i.e. Deployments, Services, etc.) and not for cluster
-wide objects(e.g. StorageClass, Nodes, PersistentVolumes, etc.).Each Kubernetes resources
can only be in one namespace.

**Label:** Labels are key-value pairs that are attached to objects such as pods. Labels are intended to be used
to identifying attributes of objects that are meaningful and relevant to users.

**Label selectors :** Using label selector, the client/user can identify a set of objects.

**Field Selectors :** Used to select k8s resources based on the value of one or more resource fields.

To get all the running pods
```shell
kubectl get pods --field-selector status.phase=Running
```
**Finalizers :** Finalizers are namespaced keys that tell k8s to wait specific conditions are met before it fully 
deletes resources marked for deletion. It can be used as garbage collector

**Annotations:** Annotations can be used in k8s objects like key-value pair. We can store different
information about the object in annotations. It works like sort of comments . Build, release or image
information like timestamps, release ID, registry address can be recorded in annotations. 


## Pods

Pods are the smallest deployable unit in kubernetes. A pod usually runs one container but can also run multiple container
that needs to work together. A pod natively provides shared networking and storage for its containers.

Each Pod is assigned a unique IP address. Containers inside a Pod can communicate between each other using 
`localhost`. Containers are not in same Pod can communicate using Pods IP addresses.

*Pod is not a process but environment for running containers*

Usually we don't need to create Pod manually. Deployment, StatefulSet and DaemonSet manages pods.
PodTemplates are specification for creating Pods and are included in workload resources such as Deployment,
StatefulSet, DaemonSet.

When the pod template is updated, the controller creates new pod based on updated template and replace the
existing one.

**Init Container**
: Init container runs in pod before actual container. Init container usually contains utilities or set up scripts
not present in an app image.

**Pod Topology Spread Constraint** 
: Using `topologyKey` we can put pods on different zones and nodes.

**Pod Termination** 
: API Server updates the pod status and the pod in the api server is considered dead beyond grace period(30s). Kubelet
notices the pod updates and start graceful shutdown. Control Plane removes the shutting-down Pod from Endpoints. The 
pod can no longer provide service. Other objects no longer consider the pod valid. When the graceful shutdown period
is over, kubelet triggers forcible shutdown.

**Pod Disruption Budget**
: Pod disruption budget tries to ensure a minimum number of pod running always. It prevents voluntary disruption
such a directly deleting a pos or updating a deployment pod template causing restart. However, it can not 
prevent involuntary disruption such as hardware failure or kernel panic. Though it counts both disruption.

## Node

Node in a kubernetes cluster is a physical or virtual machine which holds the pods. There are two ways to add nodes in API
server 
1. The kubelet on a node self-registers to the control plane
2. Human user manually add a Node object

***Heartbeats*** sent by k8s nodes to determine health of each node. There are two form of heartbeats.
1. updates to the `.status` of Node
2. updates Lease objects within the kube-node-lease namespace. Each node has an associated Lease Object

**Graceful Node Shutdown**
: kubelet ensures tha pod follows the normal pod termination process. During graceful shutdown kubelet terminates pod in 
two phases , termination of regular pods and critical pods. There is specific time duration to shut down critical 
pods.

**Controller**
: A controller tracks at least one k8s resource type, and it is responsible for making the current state of the object
come closer to desired state. Controller might carry the action out itself or will send messages to the API server 

**Cloud Controller Manager**
: The could-controller-manager is a k8s `control plane` component that embeds cloud specific control logic.
It lets us link our cluster into cloud provides api.

**Garbage Collection**
: Kubernetes checks for and deletes objects that no longer have owner references, like the pods left behind when
ReplicaSet is deletes. We can control how and when garbage collection deletes resources that have owner references
using Kubernetes finalizers.

1. ***Foreground cascading deletion***: k8s API Server first removes the dependents, then the owner.
2. ***Background cascading deletion***: k8s API Server first removes the owner, then removes the orphan objects.

_The kubelet only garbage collects the containers it manages._

## Deployment

A deployment provides declarative updates for Pods and ReplicaSets. We can create, update, delete and manage Pods or
ReplicaSets using Deployment.

[_Sample YAML to create Deployment_](https://github.com/Shaad7/notes/blob/master/sample-yaml/deployment.yaml)


To apply deployment
```shell
kubectl apply -f FilePath
```

To get all the deployment : 
```shell
kubectl get deployment
```

To update the image of the pods the deployment created 
```shell
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```

To check the revisions of nginx deployment
```shell
kubectl rollout history deployment/nginx-deployment
```

To see details of revision 2
```shell
kubectl rollout history deployment/nginx-deployment --revision=2
```

To rollback to a specific version we can use `--to-revision=2`
```shell
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

To scale a deployment 
```shell
kubectl scale deployment/nginx-deployment --replicas=10
```

## ReplicaSet

ReplicaSet is used to maintain a specific number of identical Pods running at any given time. ReplicaSet creates or
acquires nodes according to the manifest. ReplicaSet maintains owner relation with its Pods.

[_Sample YAML to create ReplicaSet_](https://github.com/Shaad7/notes/blob/master/sample-yaml/replicaset.yaml)


_A ReplicaSet only ensures a certain number of Pods are running at any given time. However, Deployment manages
ReplicaSet and provides other useful feature such as rollback. So most of the time we use Deployment_

If newly created Pods(from manifest) do not have a Controller as their owner reference and match the selector of a
ReplicaSet, the Pods will be immediately acquired by ReplicaSet.

***Pod Deletion Cost*** 
: We can assign an integer value cost with each pod. When scaling down the pod with lower cost will be removed first.

## StatefulSets

StatefulSet is used to manage a set of Pods which runs stateful applications like database.
StatefulSet maintains sticky identity for each Pod and the Pods are not interchangeable. 
If an app requires Stable,unique network identifiers and/or persistent storage and/or ordered 
deployment/scaling and/or automated rolling updates, StatefulSet is used.

[_Sample YAML to create StatefulSet_](https://github.com/Shaad7/notes/blob/master/sample-yaml/statefulset.yaml)


StatefulSet uses a headless service so that using it all the pods can be uniquely identified. Unlike Deployment,
we can't just forward service to any pod, we need to identify the all the Pods uniquely.

Each Pod can be identified as `$podname.$(governing headless service domain)` where Headless service domain 
can be constructed as `$(service name).$(namespace).svc.cluster.local`

_When the StatefulSet Controller creates a Pod, it adds a label,
StatefulSet.kubernetes.io/pod-name, that is set to the name of the Pod._

- When StatefulSet fails due to node failure and the control plane creates replacement Pod, the StatefulSet retains 
existing PersistentVolumeClaim.
- When Pod is deleted due to deletion or scaling of StatefulSet , we can configure if we want to 
delete or retain the data.

## DaemonSet

A DaemonSet ensures that all Nodes run a copy of Pod or all the zone run a copy of a Pod. DaemonSet automatically
adds Pod to newly created Nodes.

There are many programs that need to run on a system without intervention from the user. These programs are often
referred to as background processes or ***Daemons***

_Once the DaemonSet is created, selectors and labels can not be changed_

Taints prohibit certain nodes from scheduling pods on them whereas toleration allow(but not require) certain pods to 
scheduled on nodes with matching taints.

We can perform rolling updates on DaemonSet.

[_Sample YAML to create DaemonSet_](https://github.com/Shaad7/notes/blob/master/sample-yaml/daemon-set.yaml)


## Jobs

A Job creates one or more Pods and will continue execution until a specified number of Pod
terminates successfully. A job performs a task and terminates. It does not run forever.

To list all the Pods belong to Job `sampleJob`
```shell
pods=$(kubectl get pods --selector=job-name=sampleJob --output=jsonpath='{.items[*].metadata.name}')
echo $pods
```

#### Types of Jobs
- **Non-parallel Jobs** : One pod is started unless the pod fails. Job is completed when one pod 
terminates successfully.
- **Parallel Jobs with fixed completion count** : Job completes when there are `spec.completion`
successful Pods.
- **Parallel Jobs with a work queue** : `.spec.parallelism` number of pods start. If one Pod
terminates successfully, the Job is successful.

[_Sample YAML to create Job_](https://github.com/Shaad7/notes/blob/master/sample-yaml/jobs.yaml)



### CronJob

A Cronjob creates Jobs on a repeating schedule(given in Cron format). It is used to perform
scheduled actions such as backups, report generation, and so on.

If the `startingDeadlineSeconds` field is set, the controller counts how many missed jobs occurred
from the value of `startingDeadlineSeconds` rather than from the last scheduled time until now.
If `startingDeadlineSeconds` is 200, the controller counts how many missed jobs occurred in the last
200 seconds.

[_Sample YAML to create CronJob_](https://github.com/Shaad7/notes/blob/master/sample-yaml/cronjob.yaml)


## Services

In k8s, a Service is an abstraction which defines a logical set of Pods and a policy
by which to access them. Service is used to communicate between two k8s objects or maintain communication between outside
world and kubernetes objects. We can't use Pods IP address for communication because it changes
when go gets restarted. Service enables decoupling components.

A service named `my-service` which targets TCP port 9376 on any Pod with the app=MyApp label
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

If service have no selector, Endpoints object is not created by k8s.

***EndPoint*** When a service matches pods with selector, the pods IP address and Port is stored in Endpoint object.
When the pod is restarted, Endpoint object is updated

EndpointSlices works like Endpoint, but it is more scalable and more suitable when we have large cluster.

#### Kube-Proxy
kube-proxy is responsible for implementing a form of virtual IP for Services. The IP of k8s objects are not accessible
from outside world. Thus, they are called Virtual IP.

***User space proxy mode***
: In this mode, kube-proxy watches the k8s control plane for addition and removal of Service and Endpoint objects.
kube-proxy in userspace mode chooses a backend via a round-robin algorithm. If the chosen pod does not respond,
kube-proxy will retry with a different backend Pod.


***iptables proxy mode***
: In this mode, kube-proxy watches the k8s control plane for addition and removal of Service and Endpoint objects.
kube-proxy in iptables mode chooses a backend at random. kube-proxy only sees backend that are healthy using 
readiness probes. So the selected pod should be healthy.


***IPVS proxy mode***
: In `ipvs` mode, kube-proxy watches k8s Services and Endpoints, calls  `netlink` interface to create IPVS rules
accordingly and sync IPVS rules with k8s Services and Endpoint ts periodically. kube-proxy in IPVS mode redirect 
traffic with lower latency than kube-proxy in iptables mode with much better performance when synchronising proxy rules.
IPVS provides more options for balancing traffic to backend Pods such as Round-Robin, Least Connection, Destination 
Hashing, Source Hashing, Shortest Expected Delay, Never Queue.

_To run kube-proxy in IPVS mode, we must make IPVS available on the node before starting kube-proxy._


#### Multi Port Service
If we need to we can expose more than one port. While using multiple port for a Service, we must give all port  names
so that these are unambiguous.

***DNS***
: If DNS has been enabled in cluster then all Pods should automatically be able to resolve Services by their
DNS names. If a service is called `my-service` and it is in `my-ns` namespace, the control plane and the DNS
Service acting together create a DNS record for `my-service.my-ns`. Pods can find the service by resolving `my-service.my-ns`.
Pods in the same namespace can find the Service only by using `my-service`

***Headless Services***
: For Headless Services, a cluster IP is not allocated, kube-proxy does not handle these Services. There is no
load balancing or proxying done by the platform for them. 

- For headless Services with selector, `Endpoints` are created and recorded in the API. DNS configuration is modified
to return A records that point directly to the Pods backing the Service.
- If selector is not defined `Endpoint` object is not created.

### Service Type

#### Cluster IP
Service creates a virtual IP inside  the cluster to enable communication between different
services such as between a set of backend pods and a set of frontend pods. Cluster IP used 
by the Service proxies.

#### NodePort
NodePort Service listens to a Port on the node(s) and forward requests on that port to a port on the Pod. We can 
access NodePort service from outside the cluster. We can use any nodes IP and `nodeport` to get service. NodePort 
Service forward requests to Pod randomly, or we can set up our own load balancing solutions.


![alt text](https://github.com/Shaad7/notes/blob/master/images/NodePort.png?raw=true
"IPVS proxy mode")

[_Sample YAML to create NodePort Service_](https://github.com/Shaad7/notes/blob/master/sample-yaml/nodeport.yaml)


#### Load Balancer
LoadBalancer does not use any nodes IP, and it distributes load across different Pods. 
It is provided by cloud service provider. It is resolves single point failure from NodePort
Service. Load Balancer generates an external IP for public use.

[_Sample YAML to create Load Balancer Service_](https://github.com/Shaad7/notes/blob/master/sample-yaml/load-balancer.yaml)

Load Balancer uses node port internally which can be disabled. If disabled Load Balancer will 
forward the traffic to Pods directly.

We can specify load balancer implementation overriding default cloud provider implementation.

#### Service IP Address

Unlike Pod IP addresses, which actually route to a fixed destination, Service IPs are not 
actually answered by a single host. Instead, kube-proxy uses iptables to define virtual IP
addresses which are transparently redirected as needed.

## Topology Aware Hints

When calculating the endpoints for a Service, the EndpointSlice controller considers the topology(region and zone) of
each endpoint and populated the hints' field to allocate it to a zone. Kube proxy can consume those hints, and use them to 
influence how traffic to is routed. 

The EndpointSlice controller is responsible for setting hints on EndpointSlices when this feature is enabled. 
The controller allocated a proportional amount of endpoints to each zone.

## DNS for Services and Pods

Kubernetes DNS schedules a DNS Pod and Service on the cluster, and configures  kubelet to tell individual containers to
use the  DNS Service's IP to resolve DNS names.

A DNS query may return different results based on the namespace of the pod making it. DNS queries that 
don't specify a namespace are limited to the pod's namespace. To access services in other namespaces we need
to specify the namespace in DNS query.

### DNS Records

- Services : `my-svc.my-namespace.svc.cluster-domain.example` resolves to one IP for services and  a set of IP for headless
services. SRV Records has the form `_my-port-name._my-port-protcol.my-svc.my-namespace.svc.cluster-domain.example` and 
it resolves to a port number and the domain name.
- Pods : `pod-ip-adress.deployment-name.my-namespace.svc.cluster-domain.example` is an example resolution for Pod
created by Deployment. `deployment-name` field is omitted if we create bare Pod. A pod with `hostname` set to "foo"
and `subdomain` set to "bar", in namespace "my-namespace", will have the domain name `foo.bar.my-namespace.svc.cluster-domain.example`

_Pod with no hostname but with subdomain will only create the A or AAAA record for the headless service._

## Ingress

Ingress is used to manage communication between k8s cluster and outside world. Ingress can direct different api paths 
to different k8s services.

We can expose one service as default backend through ingress.

A fanout configuration routes traffic from a single IP address to more than one Service, based
on the HTTP URL being requested. [_Sample YAML to create fanout ingress_](https://github.com/Shaad7/notes/blob/master/sample-yaml/fanout-ingress.yaml)

Name-based virtual hosts support routing HTTP traffic to multiple host names at the same IP address(
ingress IP ) and then forwarded to multiple service. [_Sample YAML to create Name based virtual hosting_](https://github.com/Shaad7/notes/blob/master/sample-yaml/name-based-ingress.yaml)

Ingress can be secured(HTTPS) by specifying a Secret that contains a TLS private key and certificate. We create the 
then secret and specify the secret name when creating Ingress.

## EndpointSlice 

EndpointSlices provide a simple way to track network endpoints withing a Kubernetes cluster. They offer a more
scalable and extensible alternative to Endpoints.

EndpointSlice contains references to a set of network endpoints. The control plane automatically creates
EndpointSlices for any k8s Service that has a selector specifies. These Endpoints include references to all 
the Pods that match the Service selector.

[_Sample EndpointSlice resource for the example
Kubernetes Service_](https://github.com/Shaad7/notes/blob/master/sample-yaml/endpoint.yaml)

EndpointSlice can act as the source of truth for kube-proxy when it comes to how to route internal traffic.

## Volumes

A volume is directory, possible with some data in it which is accessible to the containers in a pod. Ephemeral volume
types have a lifetime of a pod but persistent volumes exist beyond the lifetime of a pod. For any kind of volumes in a 
given pod, data is preserved across container restarts.

**A ConfigMap provides a way to inject configuration data into pods.** This [_Sample Configuration_](https://github.com/Shaad7/notes/blob/master/sample-yaml/configmap-pod.yaml) shows how to 
mount `log-config` ConfigMap onto a Pod called `configmap-pod`. 

### emptyDir

An `emptyDir` volume is first created when a Pod is assigned to a node, and exists as long as that Pod is running
on that node. `emptyDir` volume is initially empty, all containers in the Pod can read and write the same files in the 
`emptyDir` volume, though that volume can be mounted at the same or different paths in each container. `emptyDir` 
can be used to checkpointing a long computation for recovery from crashes. [_Sample Configuration of emptyDir_](https://github.com/Shaad7/notes/blob/master/sample-yaml/empty-dir.yaml).

### hostPath

A `hostPath` volume mounts a file or directory from the host node`s filesystem into Pod. It presents many security risk.
It should be avoided when possible and should be used in Read-Only mode when needed.
[_Sample Configuration of hostPath_](https://github.com/Shaad7/notes/blob/master/sample-yaml/hostPath.yaml).

### local

A local volume represents a mounted local storage device such as disk, partition or directory. Local volume can be
used a statically created PersistentVolume. Dynamic provisioning is not supported. `nodeAffinity`
must be set when crating local volume so that k8s scheduler can schedule the volume in correct storage.

[_A Sample YAML file to create a Persistent Local Volume_](https://github.com/Shaad7/notes/blob/master/sample-yaml/local-volume.yaml)

### nfs
An `nfs` volume allows an existing Network File System share to be mounted into a Pod. NFS volume can persist data
when Pod is removed, and it can be pre-populated with data and data can be shared between pods.

### persistentVolumeClaim
A `persistentVolumeClaim` volume is used to mount a PersistentVolume into a Pod. PersistentVolumeClaims are a way
for users to "claim" durable storage without knowing the details of the particular cloud environment.

### secret
A `secret` volume is used to pass sensitive information, such as passwords, to Pods. Secrets can be stored in
k8s api and mount them as files for use by pods.

### Using subPath

Sometimes it is useful to share one volume for multiple uses/containers in a single Pod. Using subPath we 
can specify a subPath for different containers.

[_A Sample YAML file to create a Volume Sub Path_](https://github.com/Shaad7/notes/blob/master/sample-yaml/volume-subPath.yaml)

### csi
Container Storage Interface(CSI) defines a standard interface for k8s to expose 
arbitrary storage systems to container workloads. csi volume can be used
through a reference to a PersistentVolumeClaim.

### Mount Propagation

Mount propagation allows for sharing volumes mounted by a container to other container
in the same pod or even to other pods on the same node.

## Persistent Volumes

A **PersistentVolume(PV)** is a piece of storage in the cluster that has been provisioned by an admin or 
dynamically provisioned by Storage Classes.

A **PersistentVolumeClaim(PVC)** is a request for storage by a user. It is similar to a Pod. Pod consumes
node resources and PVC consumes PV resources.

### Life Cycle of a Volume and Claim

- **Provisioning :** Allocating the memory. Can be static or dynamic(based on claim)
- **Binding :** PVC with specific storage amount and access modes are requested. If a matching PV is found
, the PVC and PV are bound together.
- **Using :** Pods use claims as volumes.
- **Reclaiming :** When a user is done with their volume, they can delete the PVC objects from the API
that allows reclamation of the resource. Volumes can either be Retained, Recycled, or Deleted using reclaim policy.

_A PV can be reserved for a PVC_

[_A Sample YAML file to create a NFS Persistent Volume._](https://github.com/Shaad7/notes/blob/master/sample-yaml/persistent-volumes.yaml)

**Node Affinity** should be explicitly set for local volumes.

### PersistentVolumeClaims

PVC is used by Pods to claim volume. Pods use volume through PVC. Claims can specify label selector
to filter the set of volumes. Only the volumes whose labels match the selector can be bound to the claim.

[_A Sample YAML file to create a PersistentVolumeClaim._](https://github.com/Shaad7/notes/blob/master/sample-yaml/persistent-volume-claim.yaml)

_Pods access storage by using the claim as a volume. Claims must exist in the same namespace as the POd using the claim_

### Projected Volumes

A projected volume maps several existing volumes sources into the same directory. `secret`, `downwardAPI`, 
`configMap` and `serviceAccountToken` volumes can be projected.
[_A Sample YAML file to create a Projected Volumes._](https://github.com/Shaad7/notes/blob/master/sample-yaml/projected-volume.yaml)

## Ephemeral Volumes

Ephemeral Volumes don't care whether the data is stored persistently across restarts. Usually they are
used for caching services or provide application some read only files like configuration data or secret keys.

### Types of ephemeral volumes
- emptyDir : empty at Pod startup, with storage coming locally from the kubelet based directory or RAM
- configMap, downwardAPI, secret : inject different kinds of k8s data into a Pod
- CSI ephemeral volumes : Provided by special CSI drivers.
- generic ephemeral volumes : can be provided by all storage drivers that also support persistent volumes

`emptyDir`, `configMap`, `downwardAPI`, `secret` are provided as local ephemeral storage and managed by kubelet.
Generic ephemeral volumes can be provided by third-party CSI storage drivers, but also by any other
storage driver that supports dynamic provisioning.

### Generic ephemeral volumes
Generic ephemeral volumes provide a per-pod directory for scratch data that is usually empty after provisioning.
They may also provide some additional features : 
- Storage can be local or network-attached
- Volumes can have a fixed size that Pods are not able to exceed
- Volumes may have some initial data
- Snapshotting, Cloning, resizing and storage capacity tracking are supported

We use Generic ephemeral volumes when creating a Pod. When such a Pod gets created, the ephemeral volume
controller then creates an actual PVC object in the same namespace as the Pod and ensures that the PVC 
gets deleted when the Pod gets deleted.

## Storage Class

A StorageClass provides a way for admin to describe the "classes" of storage they  offer. Different classes
might map to quality-of-service levels, or backup policies. To use different types of storage, we can use
storage classes when creating PVC. A cluster admin can define and expose multiple flavors of storage each with
different set of parameters using Storage Class.

**Storage Class dynamically creates  `Persistent Volume` when Pod request for storage using `Persistent
 Volume Claim`**

Each StorageClass contains the fields `provisioner`,
parameters, and `reclaimPolicy` which are used when a PersistentVolume belonging to the class needs to be
dynamically provisioned. Using provisioner we can specify from where volume should be created. (Local 
or NFS server or cloud service provider)

If `volumeBindingMode` is set to `Immediate`, volume binding and dynamic provisioning occurs once
the PersistentVolumeClaim is created.

### Dynamic Volume Provisioning

Dynamic Volume Provisioning allows storage volumes to be created when volume is requested rather than creating
volumes beforehand. It ensures that user don't have to worry about the complexity and nuances of how 
storage is provisioned, but still can select form multiple storage options.

To use dynamic provisioning, a cluster admin needs to pre-create one or more StorageClass objects for users.
StorageClass objects define provisioner and which parameters should be passed to that provisioner when 
dynamic provisioning is invoked.

## Volume Snapshots

A `VolumeSnapshotContent` is a snapshot taken from a volume in the cluster that has been provisioned by an
admin. It is a resource in the cluster just like PV.

A `VolumeSnapshot` is a request for snapshot of a volume by a user. It is similar to a PVC.

A `VolumeSnapshotClass` provides a way to describe the "classes" of storage when provisioning a volume snapshot.
It is like StorageClass.

_VolumeSnapshot support is only available for CSI drivers_

In dynamic provisioning, the snapshot common controller creates `VolumeSnapshotContent` objects automatically.
For pre-provisioned snapshots, cluster admin is responsible for creating `VolumeSnapshotContent`.

Volume snapshot classes have a driver that determines what CSI volume plugin is used for provisioning
VolumeSnapshots.

### CSI Volume Cloning

When creating a PVC, we can refer another existing PVC as data source. It is called Volume Cloning.

It is only supported for CSI Drivers. A PVC can only be cloned when source and destination is in the same 
namespace and same storage class. Cloning can only be performed between two volumes that use the same
VolumeMode setting. The storage of new PVC should be same or larger than existing source PVc.

## ConfigMaps

A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps
as environment variables, CL arguments or as configuration files in a volume. ConfigMap is used to 
decouple environment specific configuration from container image so that the applications are easily portable.

[A sample configMap YAML file](https://github.com/Shaad7/notes/blob/master/sample-yaml/config-map.yaml). 
Here is a [sample Pod](https://github.com/Shaad7/notes/blob/master/sample-yaml/configmap-demo-pod.yaml) that consumes the ConfigMap.


## Secrets

A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key.
Because Secrets can be created independently of the Pods that use them, there is less risk of the Secret 
being exposed during workflow of creating, viewing, and editing Pods. Secret can be mounted as data volumes
or exposed as environment variables to be used by a container in a Pod.

_Secrets are similar to ConfigMaps but are intended to hold confidential data_

### Types of Secret

- **Opaque secrets :** Default Secret type to store any key-value pair.
- **Service Account Token Secrets :** It is used to store a token that identifies a service account.
- **Docker Config Secrets :** Stores credentials for accessing Docker registry for images.  `~/.coker/config.json`
files is provided as base64 encoded string in  the secret.
- **Basic authentication Secret :** Contains 'username' and `password` for basic authentication
- **SSH authentication secrets :** Used for passing `ssh-privateke` for SSH authentication.
- **TLS secrets :** It can be used for storing a certificate(`tls.crt`) and its associated key(`tls.key`) that  
are typically used for TLS. Can be created form command line using `tls` subcommand.
- **Bootstrap token Secrets :** It is designed for tokens used during the node bootstrap process.

[A sample Service Account Token Secret YAML File](https://github.com/Shaad7/notes/blob/master/sample-yaml/service-account-token-secret.yaml).
Here is a [sample pod](https://github.com/Shaad7/notes/blob/master/sample-yaml/secret-demo-pod.yaml) 
that consumes the Secret.

Instead of mounting the whole secret, we can take keys and create volume for it, and 
we can also mount that volume in specific path. Permission can be managed for whole
secret as well as specific keys when mounting it in Pod directory.

### Immutable Secrets

Secrets and ConfigMaps can be immutable. Immutable secrets protects accidental 
updates and reduce load on kube-apiserver by closing watches for secrets marked as 
immutable. 

## Scheduling

### Node Affinity

Node Affinity is a property of Pods that attracts them to a set of nodes.

We can make some rules which will be used to schedule Pods to Nodes. These rules can be soft/preference so
if the scheduler can't satisfy them, the pod will still be scheduled. There is two types of affinity
, "node affinity" and "inter-pod affinity/anti-affinity". Node affinity is used to schedule Pods to Nodes.
Node affinity is specified as files `nodeAffinity` of field `affinity` in  the PodSpec.

[_A Sample YAML file of a Pod with Node Affinity_](https://github.com/Shaad7/notes/blob/master/sample-yaml/node-affinity.yaml)


**Inter-pod affinity and anti-affinity** allow you to constrain which nodes your pod is eligible to be
scheduled based on labels on pods that are already running on the node rather than based on labels on the
nodes.

[_A Sample YAML file of a Deployment with Inter Pod Anti-Affinity_](https://github.com/Shaad7/notes/blob/master/sample-yaml/inter-pod-affinity.yaml)

### Taints and Tolerations

Taints allow a node to repel a set of pods. Tolerations are applied to Pods, and allow(but do not require)
the pods to schedule onto nodes with matching taints. 

To set taint on a node `node1`
```shell
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

- `NoSchedule` means if the Pod does not have the taint as toleration, it will not be schedules on that Node.
- `NoExecute` is used to evict Pods from the Node that does not tolerate the applied taint.
-  `PreferNoSchedule` will try not to schedule the Pod on that Node.

[_A Sample YAML file of a Pod with tolerations_](https://github.com/Shaad7/notes/blob/master/sample-yaml/taint-toleration.yaml)

## Role Based Access Control (RBAC)

Role-based access control (RBAC) is a method of regulating access to computer or network resources
on the roles of individual users within organization.

**Role** is used to set permissions within a particular namespace.

**Cluster Role** is used to set permissions across the cluster.

[Sample YAML to create a Role](https://github.com/Shaad7/notes/blob/master/sample-yaml/role.yaml)

Both Role and ClusterRole have must define resources and verbs. The role says that we can use those 
verbs on the specified resources.

### RoleBinding and ClusterRoleBinding

A RoleBinding grants the permissions defined in a role to a user or set of users. A ClusterRoleBinding 
 binds a ClusterRole with users.

[Sample YAML](https://github.com/Shaad7/notes/blob/master/sample-yaml/role.yaml)
to create a RoleBinding which binds previously create pod-reader Role with a User. 

We can also bind ClusterRole with a User using RoleBinding. The User will only get 
access a specified namespace. In this way, we can create a global role and 
grant permission to different user in different namespace.

![alt text](https://github.com/Shaad7/notes/blob/master/images/rbac.png?raw=true
"Rule Back Access Control")


### Aggregated ClusterRoles

Using Aggregated ClusterRoles we can combine multiple ClusterRoles into one 
ClusterRole. It works using labels and labelSelector in aggregationRule.

### Subjects

A RoleBinding or ClusterRoleBinding binds a role to a subjects. Subjects can be groups, users or
ServiceAccounts.


## Horizontal Pod Autoscaling

_HorizontalPodAutoscaler_ updates a workload resource (Deployment or StatefulSet) with the aim of automatically
scaling the workload to match demand. If the load increases and the memory and cpu utilization of 
existing resources increases above a certain threshold, _HorizontalPodAutoscaler_ scales the workload 
resource. And again when the load decreases, the resource will be scaled down to minimum eventually.

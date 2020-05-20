---
layout: post
title:  "k8s Cluster Usage"
date:   2019-12-25 22:59:00 -0400
categories: k8s linux
logo: k8s.jpg
---
Expanding on [this previous post]({% post_url 2019-12-14-raspi-k8s-cluster %}), this post details some basic fundamentals
of Kubernetes (k8s) and its purpose. It is intended to be some quick high-level summaries of many core functions and some
quick hits for interacting with and managing workloads on k8s.

### Warning

Please note this is *NOT* a comprehensive tutorial on k8s, its functionalities, and interacting to perform deployments.
These topics take multiple books, resources, experimentation and research to accomplish. This post is specifically some
fundamentals around interacting with k8s using `kubectl` and performing simple deployments to experiment with the technology.

### Cluster and Command Interaction

`kubectl` is the command used to interact with a k8s cluster from the command line. To view its version and get some general
information about your cluster, the following commands may be useful.

**Note**: In order for this command to function as expected, you need to have a k8s cluster configuration specified. Usually
this is via a file in your home directory `~/.kube/config` which `kubectl` looks for by default (and can be copied from the
k8s cluster you're specifically wishing to interact with).

```bash
# list the kubectl version and information
kubectl version

# view the configuration kubectl knows about
kubectl config view

# list information about the cluster
kubectl cluster-info

# show the cluster nodes and their status information
kubectl get nodes

# get detailed information about a specific node
kubectl describe nodes node1

# list API versions
kubectl api-versions

# show the status of various core components of a k8s cluster
kubectl get componentstatuses

# get all namespaces for the cluster
kubectl get namespaces
```

#### Labeling Cluster Nodes

It is sometimes useful to provide labels to nodes (for instance, SSD-backed storage, GPU-specific performance, etc.),
especially in the context of DaemonSets (explained later). Labeling is pretty straightforward:

```bash
# label a node `node_name` indicating it is
# gpu-specific/focused
kubectl label nodes node_name gpu=true

# find nodes with a label 'gpu' set to 'true'
kubectl get nodes --selector="gpu=true"
```

### Pods

Pods are the basic atomic unit of k8s, consisting of 1 to many containers. When you wish to deploy containers into a k8s
cluster, you do so via creating a Pod, which results in all containers within the Pod sharing the same IP, permissions, etc.

```bash
# list pods in the kube-system namespace
kubectl get pods --namespace=kube-system

# get detailed information about a specific pod
kubectl describe pods etcd-node1 --namespace=kube-system

# port forward a pod's port 80 to a local port 8080
kubectl port-forward etcd-node1 8080:80 --namespace=kube-system
```

#### Copying Files To/From Pods

You can also copy files to/from pod containers - this is generally not a best practice (as you want an immutable infrastructure)
but can be useful for debugging and troubleshooting issues.

```bash
# copy a file from a pod to the local file system
kubectl cp <POD_NAME>:/etc/hosts ./pod_hosts_file

# copy a file from the local file system to a pod
kubectl cp ./some_file <POD_NAME>:/tmp/some_file
```

#### Troubleshooting Pods

There are some additional commands that are useful in troubleshooting pod containers:

```bash
# log into an instance
kubectl exec -it nginx bash

# run an arbitrary command
kubectl exec -it nginx whoami

# get logs
kubectl logs nginx

# continuously stream latest logs
kubectl logs nginx -f

# get logs from previously-running instances
# (useful for troubleshooting crash loops)
kubectl logs nginx --previous
```

#### Labeling Pods

Although not restricted to Pods alone (labels can be used in various use cases and on various sources), Pods represent a decent
example of how to work with labels:

```bash
# create a deployment comprised of a minimum
# of 2 nginx instances, exposing port 80, and
# having labels:
#   app: nginx
#   env: dev
kubectl run nginx \
            --generator=run-pod/v1 \
            --replicas=3 \
            --image=nginx:1.17 \
            --port=80 \
            --labels='app=nginx,env=dev'

# show labels associated with the nginx pod
kubectl get pods nginx --show-labels

# delete the 'app' label on the nginx pod
kubectl label pods nginx 'app-'

# show all pods having an 'env' label set to 'int'
kubectl get pods --selector='env=int'

# show all pods having an 'env' label set to 'int' or 'dev'
kubectl get pods --selector='env in (int,dev)'

# show all pods not having an 'env' label set
kubectl get pods --selector='!env'

# update/overwrite the existing 'env' label on the nginx pod
kubectl label pods nginx 'env=int'

# show what the 'env' label is set to for all pods
kubectl get pods -L env
```

#### ReplicaSets to Manage Pods

ReplicaSets are configurations that manage the availability of pods by selecting information through labels. It is
used to ensure the specified pods to be running match the actual state of the k8s pods actually running, bringing
up or terminating instances as appropriate. The configuration for a ReplicaSet is inclusive of a Pod configuration
template, which is how the ReplicaSet indicates the creation of new Pods when necessary. However, as such, the
configuration is also a bit more complicated than can be performed on the command line and is typically managed
through `yaml` configurations. Some of the benefits of a ReplicaSet are:

* Addition of new Pods if the number of Pods is below the replica threshold.
* Removal of existing Pods if the number of Pods is above the replica threshold.
* Automatic scaling (horizontal) of pods based on metrics such as CPU, memory, etc. (requires the `heapster` Pod
be running in the `kube-system` namespace, which manages the associated metrics).

Below are a few commands to interact with a ReplicaSet object:

```bash
# show all replicasets
kubectl get replicasets

# get information about a specific replicaset
kubectl describe replicasets nginx

# scale a replicaset number of pods up/down
kubectl scale replicasets nginx --replicas=4

# create an autoscaler based on max CPU of 80% in a pod
kubectl autoscale replicasets --min=3 --max=10 --cpu-percent=80

# get the autoscalers
kubectl get horizontalpodautoscalers

# describe details about an autoscaler
kubectl describe horizontalpodautoscalers nginx
```

#### Portability and Configuration

Keeping Pods portable is possibly by loosely-coupling configuration and sensitive information from the Pod (ConfigMaps
and Secrets, respectively).

ConfigMaps are a way to store configuration information for Pods, which can be exposed to the Pod via filesystem-mounted
files (names of keys, content containing values), environment variables, or command-line arguments.

```bash
# create a configmap from a file
kubectl create configmaps test-config1 --from-file=test-config.txt

# create a configmap free-form
kubectl create configmaps test-config2 \
    --from-literal=param1=some-value \
    --from-literal=param2=some-value

# get a configmap
kubectl get configmaps test-config1

# describe information about a configmap
kubectl describe configmaps test-config1

# output the raw configmap data
kubectl get configmaps test-config1 -o yaml

# update/overwrite a configmap with a new value
kubectl replace -f configmap-file.yaml

# edit a configmap in place
kubectl edit configmap test-config1

# delete a configmap
kubectl delete configmaps test-config1
```

Secrets contain sensitive information that can be injected into a Pod. Note that unless you use an encryption key to
encrypt the data, secrets are stored in clear-text within `etcd` by default and can be accessed by any administrator
of the cluster.

```bash
# create a secret from multiple files
kubectl create secrets some-secret \
    --from-file=some-cert.crt \
    --from-file=some-key.key

# get the secret
kubectl get secrets some-secret

# describe information about a secret
kubectl describe secrets some-secret

# output the raw secret data
kubectl get secret some-secret -o yaml

# update/overwrite a secret with a new value
kubectl replace -f secret-file.yaml

# edit a secret in place
kubectl edit secret some-secret

# delete a secret
kubectl delete secrets some-secret
```

**Note**: Accessing Secrets is done via mounting the Secret as a volume on the Pod, which is mounted as a `tmpfs` volume (RAM only).
This is configured in the `volumeMounts` parameter in the configurations.

You can also configure what are known as Image Pull Secrets which configure credentials for accessing private Docker repositories
and can be utilized in the Pod image definition within configurations.

```bash
# create an image pull secret
kubectl create secret docker-registry some-reg-secret \
    --docker-username=testuser \
    --docker-password=supersecretpassword \
    --docker-email=testuser@testdomain.com
```

#### Deployments

Deployments are a cohesive way to manage upgrades and downgrades of applications in a k8s cluster without service
disruption (rolling updates to versions, health checks, etc.). Most often, deployments are captured in YAML
configuration given the many different options and switches that go into defining a deployment. A Deployment object
manages ReplicaSets (mentioned previously), which in turn manages Pod scaling and sizing. Because of this management
hierarchy, whenever you use a Deployment object, you must manage the scaling (ReplicaSets) configurations through
the Deployment object (not the ReplicaSet object) as the Deployment object will always ensure its configuration is
the current state.

```bash
# create a deployment from a yaml spec
kubectl create -f nginx-deployment.yaml

# list the nginx deployment
kubectl get deployments nginx

# get information about the nginx deployment
kubectl describe deployments nginx

# convert an online deployment spec to a yaml file
# and ensure the deployment is managed by the file
kubectl get deployments nginx --export -o yaml > nginx-deployment.yaml
kubectl replace -f nginx-deployment.yaml --save-config

# get information about past 3 deployment updates
kubectl rollout history deployments nginx --revision=3

# get specific details about a previous rollout
kubectl rollout history deployments nginx --revision=5

# pause and resume rollouts (for troubleshooting)
kubectl rollout pause deployments nginx
kubectl rollout resume deployments nginx

# get information about the current or
# in progress deployment update
kubectl rollout status deployments nginx

# undo the last deployment (roll back to previous)
kubectl rollout undo deployments nginx

# roll back to previous specific revision
kubectl rollout undo deployments nginx --to-revision=1

# delete a deployment and all associated resources
kubectl delete deployments nginx
```

A deployment can follow one of 2 update strategies. The `Recreate` strategy simply updates the ReplicaSet definition
of the container image and destroys all containers, resulting in the ReplicaSet re-creating all instances with the new
container image. The `RollingUpdate` strategy, in contrast, performs a graceful destroy/create on the containers, resulting
in (potential) lack of service disruption. It's important to note, however, that in order for a `RollingUpdate` method
to work effectively, services must have the ability to expose control of service traffic (or handle it automatically via
retry logic), and also be forward/backward compatible as multiple versions of the application running during the
rolling upgrade could cause data or interaction issues.

One final note to mention is that revision history for deployments can grow to be very large. It's likely best to
event source these into a separate event system and limit the number of history revisions in the Deployment object
within k8s to some reasonable number using the `spec.revisionHistoryLimit` configuration.

### Service Discovery

Service discovery is what makes k8s flexible and dynamic. It is a way to provide a "static" IP to expose
the various Pods/Containers running your application.

```bash
# create a service with label selectors and a ClusterIP
# out of the existing 'nginx' deployment
kubectl expose deployments nginx

# show services, selectors, and associated labels
kubectl get services -o wide --show-labels

# at this point, any pods should be able to resolve the
# service name 'nginx' to its pod location/IP address
```

There are a couple ways that services are exposed. Specifically, the following are important to distinguish:

* **ClusterIP**: Service is exposed to an IP address only accessible by the k8s cluster. This is the default if
no other option is specified in the `kubectl expose ...` command.
* **NodePort**: Service is exposed via a port and all nodes in a cluster forward traffic destined to that port
to the target service (from outside the cluster).
* **LoadBalancer**: Expands NodePort by provisioning a dedicated load balancer for the service (to load balance
traffic to the various Pod/Container back-ends). Note this is only possible if an integrated Load Balancer
service is configured (usually common in cloud deployments) but will not work in an out of box configuration
such as the post this blog post expands on/uses for its foundation.

```bash
# create a service with a port that is accessible via the
# port by any request made to any of the cluster nodes
kubectl expose deployments nginx --type=NodePort

# show the NodePort details for the service and capture
# the random port assigned for the service
kubectl get services nginx

# at this point, you should be able to query the nginx
# web server from your local laptop using any of the
# cluster nodes and the randomly-assigned NodePort of
# the service
curl http://<cluster_node_hostname_or_ip>:<random_port>
```

#### Endpoints

If you wish to access your back-end services directly without a static IP (but rather, using the actual IP
addresses of the containers) you can use the `endpoints` concept. Endpoints capture the actual IP and port
information of all containers within a service definition, including dynamically updating them when they
change due to scaling up/down operations.

```bash
# show endpoints for the nginx deployment
# this should show 2 IP/ports reflecting the 2x nginx
# instances that exist in the deployment
kubectl get endpoints nginx
```

In addition to k8s-deployed infrastructure, it's possible to configure Endpoints that point to infrastructure
that is outside of the k8s cluster (such as databases, etc.). There are 2 primary way to configure such Endpoints.

1. **DNS**: Create an Endpoint with a `ExternalAddress` property which specifies the externally-resolvable DNS
entry for where the service exists.
2. **IP**: In the case where DNS is not available or desired, specify a blank Service without a label selector.
This Service will have no associated Endpoints but will allow k8s to allocate a port/IP (depending on your Service
type). Next, create an Endpoint object and associate it with the Service name, and give it the external IP and Port
for the location of your service. At this point, making a call to the Service from within the cluster (or even
externally if the Service is of type LoadBalancer) will treat the external source as though it was located within
the k8s cluster.

### Ingress

Since typical load balancing and service exposure to this point utilize out of box functionality, the
Service object only operates at Layer 4 of the OSI model meaning it cannot do any content/packet inspection
for dynamic routing of requests. This also means each of the endpoints wanting to be exposed needs its own IP
address and load balancing endpoint.

Ingress enhances load balancing by adding OSI Layer 7 capabilities. Traffic sent to a single IP endpoint
is inspected for the Host header and path specified, similar to the way a traditional Virtual IP-based load balancer
functions. This allows for a single IP ingress for multiple service back-ends and can simplify the exposure
of services deployed in k8s.

Ingresses require more complicated configurations which usually cannot/are not managed via the command line.
Some of the configurations created within a `.yaml` file include (not limited to):

* URL Rewriting
* Port Specification
* Path Matching
* Hostname Matching
* Service Discovery

### DaemonSets

Up to this point, discrete pieces of functionality have been described and rolled up to a Deployment. In
addition to Deployments, which manage various components of Pods on a subset of the available k8s nodes, there
is a concept known as a DaemonSet, which is used similar to a Deployment but instead ensures that a Pod is
running on every (or a subset if a node selector is used) node of a k8s cluster. This can be useful for things
like system services, log forwarding, etc.

```bash
# create a DaemonSet
kubectl apply -f sample-daemonset.yaml

# list daemonsets
kubectl get daemonsets

# get information about a specific daemonset
kubectl describe daemonsets kube-proxy --namespace=kube-system

# get the status of a rolling upgrade of a daemonset
kubectl rollout status daemonsets kube-proxy
```

### Storage

There are primarily 2 different types of storage mechanisms in k8s:

1. **PersistentVolume**: This is a statically-allocated storage backend where Volumes are declared in total size
and Claims are declared to allocate some (or all) of the total Volume size and associate it with a Pod.
2. **Dynamic Provisioning**: Although not strictly an Object type in k8s, dynamic provisioning allows creating
Claims with dynamic size whereby the provisioner automatically creates the size (instead of being pre-created). The
catch with dynamic provisioning is that when the Pod is destroyed, the Volume is also destroyed.

In addition to the above, there is the concept of the StatefulSet object type, which allows creation of sequenced,
ordered Pods and allows for things such as database replication to occur. Although StatefulSets are not strictly
a storage type, they are typically used in conjunction with something such as a PersistentVolume.

### Jobs

Most of the previous constructs enforce a state when state drifts (if number of Pod is less than replicas specified,
create additional Pod, etc.). If you want to run one-time operations where a container runs through a task, exits,
and the result is recorded, you would use a job.

```bash
# run a one-shot job
kubectl run -i demo-oneshot --image=custom-image --restart=OnFailure

# get a list of jobs, including completed (-a)
kubectl get jobs -a

# describe information about a job
kubectl describe jobs demo-oneshot

# delete a job (once completed)
kubectl delete jobs demo-oneshot
```

### RBAC

Role-based access control is governed by roles and permissions, and role bindings associate roles with identities or
groups of identities. There are various out of box roles and cluster roles available for use, or you can create your
own.

```bash
# get available roles in a specific namespace
kubectl get roles --namespace=kube-system

# get all cluster roles
kubectl get clusterroles

# check if you can list available pods
kubectl auth can-i get pods
```

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Kubernetes](https://kubernetes.io/docs/home/)
* [Kubernetes - Up and Running (Book)](https://www.amazon.com/Kubernetes-Running-Dive-Future-Infrastructure/dp/1492046531/ref=sr_1_3)

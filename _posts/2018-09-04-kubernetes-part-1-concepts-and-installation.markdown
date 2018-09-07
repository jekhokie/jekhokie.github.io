---
layout: post
title:  "Kubernetes Part 1: Core Concepts and Installation (Minikube)"
date:   2018-09-04 22:05:11 -0400
categories: ubuntu linux docker container kubernetes minikube
---
Although familiar with [Docker](https://www.docker.com/) containers and superficially played
with [Kubernetes](https://kubernetes.io/) in the past, I've not taken the required time to learn the
core concepts of Kubernetes to really grasp all the possibilities the technology offers. This post is
the first in a series of posts geared towards a deeper understanding of the Kubernetes orchestration
application via the use of the [Minikube](https://github.com/kubernetes/minikube) implementation of
Kubernetes to simplify the installation. The series of posts will walk through a full implementation
of a Python Flask application with MySQL backend and explore concepts such as scaling, auto-healing,
service discovery, and other beneficial concepts that Kubernetes offers out of box.

## Series

As noted, this is Part 1 of the 5-part series:

- **Kubernetes Part 1: Core Concepts and Installation (Minikube)**
- [Kubernetes Part 2: Python Flask Application Deployment]({% post_url 2018-09-04-kubernetes-part-2-python-flask-application-deployment %})
- [Kubernetes Part 3: MySQL Database Deployment]({% post_url 2018-09-04-kubernetes-part-3-mysql-database-deployment %})
- [Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)]({% post_url 2018-09-04-kubernetes-part-4-application-deployments-via-yaml %})
- [Kubernetes Part 5: Linking Application with Database (Discovery)]({% post_url 2018-09-04-kubernetes-part-5-linking-application-with-database %})

## Core Concepts

First, let's start with some basic object definitions for the Kubernetes application. I attempted to place
"Equivalent" descriptions next to each core concept for the basic object definitions - these are not
entirely accurate, but are a way of thinking about these new concepts within Kubernetes if you are
familiar with legacy/traditional VM-based load-balanced applications and micro-services:

- **Pod**: Basic building block of deployment. Typically 1:1 with a running process/Container,
but in more complicated scenarios, can be one-to-many.
    - **Equivalent**: In a traditional load-balanced VM-based configuration, this can be thought of as a VM
(or set of VMs) all running the same application.
- **Service**: Abstraction defining a logical set of Pods - typically a single IP address that routes
requests to one or more Pods via balancing mechanisms.
    - **Equivalent**: In a traditional load-balanced VM-based configuration, this can be thought of as the
load balancer in front of the VMs all running the same application, serving requests to each (single
ingress point).
- **Volume**: File system/disk that enables cross-Container storage and reference.
    - **Equivalent**: In legacy load-balanced VM-based deployments, can be thought of as an NFS mount across
all VMs running the same application for persistent storage. The difference is in a
Kubernetes/container-based environment, there is no persistent storage without a Volume, as without this,
all locally stored data is lost when the containers die/are destroyed.
- **Namespace**: Construct to enable large-scale usage of the Kubernetes cluster by many users,
logically grouping clusters.
    - **Equivalent**: Can be thought of as a permissions-based model in a traditional VMware
VCenter-based model where certain groups of users can only see certain types of compute resources.

Next, there are several higher-level abstractions known as Controllers:

- **ReplicaSet**: Component which governs create a destroy Pods dynamically based on scale specifications.
such as Blue/Green deployments, upgrades, rollbacks, pausing/resuming software upgrades, etc.
- **Deployment**: Construct to enable declarative state of a Pod/ReplicaSet. This concept enables features
- **StatefulSet**: Similar to a deployment but maintains stickiness/identity for Pods.
- **DaemonSet**: Similar to a Deployment, and ensures that all Nodes run at least 1 instance of a Pod.
- **Job**: Runs one or more Pods to accomplish a specific task, maintaining state about each Pod utilized.

Finally, some additional terminology that may help:

- **Node**: A worker machine that is available for running applications/servicing various other types of
objects Kubernetes affords the user.

This tutorial will not utilize all of the above concepts/abstractions, but they are useful to understand
the capabilities of Kubernetes.

## Installation (Minikube)

Now that we've learned a few core concepts/terminology, let's get started with implementing and
experimenting with a Kubernetes setup.

### Technology Ecosystem

The steps in this post are executed on an older physical server running the Ubuntu operating system
(specifically, Ubuntu 16.04, 64-bit). Details related to the host machine are as follows:

**Host**

- **Hostname**: kubernetes.localhost
- **OS**: Ubuntu 16.04
- **CPU**: 2
- **RAM**: 3 GB (device used originally 4GB, but failed memory module decreased available memory)
- **Network**: Private Network
- **IP**: 192.168.1.233

Additionally, the following are the versions of software utilized at the time of this post (for
reference):

- **Docker**: 18.06.1-ce
- **Kubectl**: 1.11.2
- **Minikube**: 0.28.2

### Prerequisites

In order to install Minikube, we will first install the following prerequisites:

- Docker
- kubectl

First, let's install Docker (per documentation):

{% highlight bash %}
$ wget -qO- https://get.docker.com/ | sh
# you will need to add your user to the 'docker' group in order
# to successfully interact with the Docker service
$ sudo usermod -aG docker $(whoami)
# next, log out/log back into the host machine in order for group
# settings and environment to align
$ exit
$ ssh 192.168.1.233
# check that the Docker service is functioning by printing
# information about the installation
$ docker version
$ docker info
{% endhighlight %}

Next, we will install kubectl:

{% highlight bash %}
$ sudo snap install kubectl --classic
# verify the version of the kubectl utility
$ kubectl version
{% endhighlight %}

### Install Minikube

We're now ready to install Minikube. This process will download the binary, move it to a location
within the standard `PATH` variable, and make it executable for use:

{% highlight bash %}
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.28.2/minikube-linux-amd64
$ chmod +x minikube
$ sudo mv minikube /usr/local/bin/
{% endhighlight %}

Next, we will start Minikube. There are quite a few options you can provide, but for this tutorial we
will stick to a few of the more common options, such as specifying the maximum memory footprint allowed
for the local install, what type of bootstrap script to use, and the fact that we do not wish to use
a hypervisor for the installation:

{% highlight bash %}
$ sudo minikube start --vm-driver=none --memory 1024 --bootstrapper=localkube
# this step takes several minutes - be patient during the installation
{% endhighlight %}

One item to note - when running the above command, `sudo` was necessary in order to ensure the binaries could
be written to the appropriate locations on the file system. This, however, creates a permission issue with
the kubectl configuration for the current user. To correct this issue, run the following:

{% highlight bash %}
$ sudo chown -R $(whoami) ~/.kube ~/.minikube
{% endhighlight %}

Next, you can verify your Minikube installation to ensure it is running as expected:

{% highlight bash %}
$ minikube ip
{% endhighlight %}

If all went successfully, you should see the IP address of the local host output - you now have a running
Kubernetes instance!

There are some additional items you can play around with - initially, Minikube does not install the heapster
addon. This addon is required if you wish to see nice graphs and metrics associated with resource
consumption of the Kubernetes installation, so let's enable it:

{% highlight bash %}
$ minikube addons enable heapster
{% endhighlight %}

It takes several minutes for the heapster addon (InfluxDB plus Grafana components) to fully install.
There is a graphical/UI dashboard you can view about the Kubernetes installation as well:

{% highlight bash %}
$ minikube dashboard --url
{% endhighlight %}

Note if you do not see graphs/resource information displayed, it is possible the Heapster addon is not
yet complete with its installation - wait several minutes and you should eventually see graphs populate
on various UI screens in the browser.

Additionally, in future posts of this series, service discovery will be designed via the Kubernetes DNS
service, "kube-dns". Ensure this addon is enabled (should be by default) and if not, enable this for the
future service discovery posts:

{% highlight bash %}
# check if kube-dns is enabled
$ minikube addons list

# if not enabled, enable it
$ minikube addons enable kube-dns
{% endhighlight %}

## Next Steps

You now have a fully-functioning Kubernetes instance via the Minikube implementation of Kubernetes and
are ready to start building and deploying applications. Proceed to the [next post]({% post_url 2018-09-04-kubernetes-part-2-python-flask-application-deployment %}) in the series to learn how to create a basic Python "Hello World"
application and deploy it to the cluster.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [How To Configure a Continuous Integration Testing Environment with Docker and Docker Compose on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-configure-a-continuous-integration-testing-environment-with-docker-and-docker-compose-on-ubuntu-14-04)
* [Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
* [Minikube Releases](https://github.com/kubernetes/minikube/releases)
* [Minikube Config](https://darkowlzz.github.io/post/minikube-config/)
* [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
* [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
* [kube-dns](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/kube-dns/README.md)

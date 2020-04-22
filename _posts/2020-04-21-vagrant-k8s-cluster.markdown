---
layout: post
title:  "Vagrant k8s Cluster"
date:   2020-04-21 20:31:00 -0400
categories: k8s vagrant ansible virtualbox linux
---
This post details a different way to experiment with a [Kubernetes](https://kubernetes.io/) cluster that is a bit more
advanced and powerful than utilizing minikube on your local workstation. The tutorial walks you through setting up a
Kubernetes cluster on a suite of VMs hosted on your workstation using VirtualBox and Vagrant, with Ansible for
configuration management of the cluster nodes (to transform them into their identities and form the cluster).

### Jump Right In

If you're wanting to simply pull the trigger on getting the cluster up and running, you can utilize
[this repo project](https://github.com/jekhokie/scriptbox/tree/master/multiple--vagrant-istio-k8s-cluster) which contains
the configuration and instructions for ~3 commands that you can run to get to the finish line. The repo project is the
complement to this tutorial where everything detailed here is automated as a one-click solution for those wanting to
dive right in and start using a cluster.

### Hardware Requirements

This tutorial lays the foundation for future posts that expand on the cluster being created and, as such, specs the nodes in
the k8s cluster to be quite large. It is recommended you have a workstation with at least 32GB RAM and 8 CPU cores available
to support both the cluster and existing operating system and other processes running. You can certainly attempt this tutorial
with 16GB RAM, but if you elect to start all 3 nodes (master plus 2 worker nodes) with the defined specs in the `Vagrantfile`,
you will likely have your workstation fans working quite hard once everything is up and running.

### Software Requirements

Software used in this tutorial is summarized below - there is no need to obtain any of this up front, we will walk through each
of these components throughout the tutorial:

- Vagrant
- VirtualBox
- Ansible
- Docker
- Kubernetes

Components used for the resulting k8s cluster include:

- calico (the linked repository project also has the ability to use flannel, which is commented out in the ansible scripts)

### Prerequisites

It is assumed that you already have a working environment with [Vagrant](https://www.vagrantup.com/) and
[VirtualBox](https://www.virtualbox.org/) at the ready. If not, follow the instructions on the vendor sites to get the components
installed and working together.

In addition, once you have a functioning Vagrant + VirtualBox environment, you'll need to enable a plugin for Vagrant to
automatically install Guest Additions in VMs being provisioned that do not have it pre-installed. This is required in order
to enable local directory mapping which is useful for transferring files, etc. and is the basis for the automation repository
if you get to the end of this tutorial and want to use that functionality:

```bash
vagrant plugin install vagrant-vbguest
```

This is essentially the only local software configuration you will need - the rest will be taken care of in this tutorial.

### Creating the VMs

The cluster in this tutorial will have 3 nodes total - one master, and two worker nodes. We will first create the VMs (all 3)
to get them ready for installing, configuring, and joining the k8s cluster. Create a `Vagrantfile` that defines the spec
for your 3 VMs, each having 4GB RAM and 2x CPUs minimum, sharing a private network IP space. A good reference to use would
be the `Vagrantfile` specified in the automation project [here](https://github.com/jekhokie/scriptbox/blob/master/multiple--vagrant-istio-k8s-cluster/Vagrantfile),
but ensure you remove the `provision` specs for each of the nodes as we will be crafting things by hand in this tutorial.

```bash
vagrant up
```

Once all 3 of your nodes have started, move along to configuration.

### Common/Foundational Configuration

*NOTE*: All commands should be run as the root user unless otherwise specified, so become the root user on each of the nodes
and remain root for the duration of this section:

`sudo su -`

There is a common set of components and configurations that exist on all nodes, regardless of role. We'll start here, meaning
you'll do the following on each (all 3) of the nodes.

First, we'll install some useful packages for debugging and working on the nodes. Note that some items in this list of packages
to install are optional and you can add items to suit your needs/what you expect to need on each node when troubleshooting.

```bash
yum -y install vim \
               device-mapper-persistent-data \
               lvm2 \
               net-tools \
               nc \
               yum-utils
```

Next we'll enable IPv4 forwarding:

```bash
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

Swap is not useful so we'll disable it:

```bash
swapoff -a

vi /etc/fstab
# comment out or remove the line containing
# the swap specification
```

Unfortunately, SELinux is also not supported, so we'll need to disable it as well:

```bash
setenforce 0

vi /etc/selinux/config
# ensure the following is set:
#   SELINUX=disabled
```

A useful item to enable intra-cluster name resolution is to create an `/etc/hosts` file on each of the hosts in the
cluster with static IP addresses for each other host. Edit the `/etc/hosts` file and add the following to the bottom
of each file, leaving out whichever hostname of the host you're already on (given that the loopback adapter will
handle host communication with itself). Note that the IPs specified here correspond to the IP addresses in the sample
`Vagrantfile` linked earlier, so if you're using a different `Vagrantfile` or IP address scheme, update this
section appropriately:

```bash
vi /etc/hosts
# append the following to the bottom, leaving out the
# current host/ip of the host you're currently on
#   master 10.11.12.13
#   node1  10.11.12.14
#   node2  10.11.12.15
```

Now that the OS-level changes are complete, we'll move into the Docker Engine installation and configuration. Add the
Docker repository and then perform the package installation required for the various components (the instructions for
this are taken straight from the Docker site [here](https://docs.docker.com/engine/install/centos/):

```bash
yum install docker-ce \
            docker-ce-cli \
            containerd.io
```

Next, in order to interact with the Docker engine, you'll need to add your user to the `docker` group. We'll do this
for both the root user (which you currently are) as well as the `vagrant` user:

```bash
usermod -aG docker root
usermod -aG docker vagrant
```

Once you've completed adding the users to the group, you'll need to either log out/log back in again, or use the
`newgrp docker` command to reset your terminal to realize the new group configurations.

Start the docker engine and enable it to start on boot of the VM:

```bash
systemctl start docker
systemctl enable docker
```

You should then be able to run the `docker info` command to see details about the Docker environment, confirming that
your install is working as expected.

We're getting closer - next step is to install and configure common components for k8s across all nodes. For this,
we'll be installing 3 core packages on each node related to running the k8s cluster, and starting/enabling the `kubelet`
process which is the primary node agent that handles registering nodes, etc.:

```bash
# configure the k8s repo
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# install the required components
yum install kubelet \
            kubeadm \
            kubectl \
            --disableexcludes=kubernetes

# start and enable the kubelet service
systemctl enable --now kubelet
```

Finally, there are bridge filtering system kernel parameters that need to be set and only show up once Docker and the core
k8s components have been installed and started. These kernel settings enable application of IPTables within scope of bridge
interfaces and are required for k8s to function as expected:

```bash
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
sysctl --system
```

As a note, if the above fails, the `br_netfilter` module may not be loaded. Load this module via `modprobe br_netfilter` and
then re-run the `sysctl --system` command to re-parse the `sysctl.conf` settings with the module available.

This largely concludes the common k8s installation requirements. We'll now move on to the master-specific configs.

### k8s Master Configuration

*NOTE*: All commands should be run as the root user and on the master node unless otherwise specified, so become the root
user on the master node for the duration of this section:

`sudo su -`

The master is the control node for the cluster. Getting the k8s cluster initialized and installing/configuring the pod network
add-on are critical to having a successful cluster join and auto-configuration of the worker nodes. Joining of worker nodes
is done using a token, and we'll generate the token (and capture the token hash, mentioned later) to be used for joining
nodes:

```bash
kubeadm token generate
```

Keep the token accessible - we'll refer to the value of the token using `<TOKEN>` in future sections.

Next, we'll initialize the cluster using the token and specify our pod network. In this case, we're using a `172.x` CIDR
range/private IP range for our pod network, and we're using the master IP as specified in the example `Vagrantfile` mentioned
earlier, with the `<TOKEN>` access token generated just before this step:

```bash
kubeadm init --pod-network-cidr=172.16.0.0/24 \
             --apiserver-advertise-address=10.11.12.13 \
             --apiserver-cert-extra-sans=k8s.cluster.home \
             --token=<TOKEN>
```

The above step will take a minute or two while it initializes the cluster and readies the master. Once complete, we can configure
the pod network, but in order to do so, we need to be able to interact with the k8s master node, which requires the credentials
and configuration to do so. During the init, there was a configuration file created which contains credentials for interacting
with the k8s master as an administrator. While this is generally dangerous and least-priv access should be configured, the admin
access is the starting point for configuring add-ons, so we will configure our `root` user to utilize these credentials:

```bash
mkdir /root/.kube
cp /etc/kubernetes/admin.conf /root/.kube/
```

Once you've copied the admin credentials to the respective directory, you can then query the master node to prove the creds are
working the way you expect. Run `kubectl get nodes` and you should see your master node showing up with a status of `NotReady`
(because we haven't yet added a pod network add-on, which we're about to do).

Now that we can communicate with the master, we'll install the pod network add-on. in this case, we'll use the `calico` network
add-on as it also affords network rules for traffic management (you can use something simpler such as `flannel`, but `flannel`
does not have the ability to control network traffic rules). Installation of `calico` is straightforward, with one catch where
we want to configure the pod network of the default configuration to be the CIDR range of the pod network that we configured
using `kubeadm` above:

```bash
# obtain the spec for the calico configuration
wget https://docs.projectcalico.org/v3.11/manifests/calico.yaml

# edit the calico spec
vi calico.yaml
# search for the key/value pair having name 'CALICO_IPV4POOL_CIDR'
# or search for "192.", and replace the "192.168.0.0/16" default
# with the pod cidr range we specified earlier: 172.16.0.0/24

# apply calico resources
kubectl apply -f calico.yaml
```

It will take several minutes for the `calico` network add-on to configure the network on the master. Check the status of
the master node routinely using `kubectl get nodes` until you see `Ready` showing in the status column for the master node,
indicating your pod network and the resulting master node are ready to move forward.

Prior to moving to the worker nodes, there are 2 things remaining. First, it would be best if you copied the `/etc/kubernetes/admin.conf`
configuration file to the shared/mapped drive on the master so that you can access it from the worker nodes easily for performing
tasks and interacting with the cluster. Second, we'll create a token CA cert hash that will be used in addition to the `<TOKEN>`
parameter to enable joining worker nodes. On the master, discover the token and keep it available (we'll refer to this parameter
in the future with `<CERT_HASH>`):

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
    openssl rsa -pubin -outform der 2>/dev/null | \
    openssl dgst -sha256 -hex | \
    sed 's/^.* //'"
```

Again, copy the value printed above - we'll refer to this as `<CERT_HASH>` in the next section.

### k8s Worker Configuration

*NOTE*: All commands should be run as the root user and on the worker nodes (`node1` and `node2`) unless otherwise specified,
so become the root user on the worker nodes for the duration of this section:

`sudo su -`

Now that our master node is up and available/ready to accept worker join requests, we'll move over to the worker nodes, `node1`
and `node2`. On each of the nodes, first enable communication with the cluster by installing the admin configuration for
the `kubectl` command to function/interact with the cluster:

```bash
mkdir /root/.kube
cp <admin.conf_from_shared_folder> /root/.kube
```

Next, we'll run the `join` command, using the token and hash we captured previously on the master node, including the master
node IP address `10.11.12.13` as specified in the sample `Vagrantfile`:

```bash
kubeadm join --token=<TOKEN> \
             10.11.12.13:6443 \
             --discovery-token-ca-cert-hash=sha256:<CERT_HASH>
```

Once the command is run, you can monitor the status of the node join by watching the `kubectl get nodes` command output.
You should quickly see the `node1` and `node2` nodes join the cluster and show a status of `NotReady` for several minutes while
the `calico` network components are configured/extended to the worker nodes. Once the initialization is completed after several
minutes, you will see all 3 nodes showing `Ready`, indicating your cluster is now available for use!

### Next Steps

Stay tuned for some potential follow-ups where we'll explore adding add-ons to the cluster and eventually building up to configuring
an Istio service mesh on top of the cluster for more advanced traffic and GitOps-like flows with a security-first approach!

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Kubernetes](https://kubernetes.io/docs/home/)
* [Kubernetes - Up and Running (Book)](https://www.amazon.com/Kubernetes-Running-Dive-Future-Infrastructure/dp/1492046531/ref=sr_1_3)
- [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Automating kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#automating-kubeadm)

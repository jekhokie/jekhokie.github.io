---
layout: post
title:  "Understanding k8s DNS"
date:   2020-04-24 18:23:00 -0400
categories: k8s coredns
logo: coredns.jpg
---

Kubernetes by default in its most recent versions utilizes [CoreDNS](https://coredns.io/) for handling DNS for the cluster.
This post is a VERY brief exploration of CoreDNS and interacting with it in the context of a k8s cluster to test local
resolution of names created as part of deploying a Pod and exposing the Pod via a Service. It is not inteded to be a complete
guide to DNS, CoreDNS as a product, or all aspects of DNS as they relate to k8s clusters (it is very pointed at testing and
setting up a workflow to debug CoreDNS on the k8s cluster for private/internal domains).

### Assumptions

As a reminder this is a VERY light exploration of interacting with CoreDNS and exposing it for local resolution from your
workstation using a NodePort for further troubleshooting and exploration. This tutorial assumes you've followed
[this]({% post_url 2020-04-21-vagrant-k8s-cluster %}) previous post and have a local k8s cluster set up using Vagrant
(or any k8s cluster that is reachable from your workstation - but the instructions are geared towards the existing setup
referenced in the previous post).

### Testing CoreDNS Within the Cluster

You can first test that CoreDNS DNS resolution is working as expected on the cluster by deploying a `dnsutils` container
image and testing that local pod name resolution works. First, create a YAML file `dns_pod.yaml` with the following details:

```yaml
# adapted from: https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/admin/dns/dnsutils.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: default
  labels:
    name: dnsutils
spec:
  containers:
  - name: dnsutils
    image: gcr.io/kubernetes-e2e-test-images/dnsutils:1.3
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

Next, deploy your Pod based on the definition created:

```bash
$ kubectl apply -f dns_pod.yaml
```

You can now use the pod to inspect name resolution of the default `kubernetes.local` endpoint, which is a default that should
always be present from a Pod's viewpoint:

```bash
$ kubectl exec -ti dnsutils -- nslookup kubernetes.default
```

If this works, your interaction with CoreDNS is working - let's expose the deployed Pod and validate that it receives an actual
DNS record:

```bash
# expose the pod as a service
$ kubectl expose pod dnsutils --port 22

# check that the pod received a dns name
$ kubectl exec -ti dnsutils -- nslookup dnsutils.default.svc.cluster.local
```

If the `nslookup` command above returned an IP address, CoreDNS is working as expected (assigning names to exposed services)!

### Exposing CoreDNS Resolution

If you want to be able to resolve the local/private DNS zone records on `cluster.local`, you can expose the CoreDNS pod as a Service
and then use `dig` or other tools to point your resolution requests to the exposed service via the NodePort on the cluster. This
isn't a practical implementation generally, but is useful if you want to troubleshoot or explore the various private DNS records created
and managed through CoreDNS.

The below process provides some instructions on how to get the current CoreDNS configuration, expose CoreDNS as an additional service
(given that it's already exposed by default using a ClusterIP, but is not accessible from your workstation with that ClusterIP), and
run a `dig` command to test name resolution of the private zone.

```bash
$ kubectl get pods -n kube-system
# should see coredns pods

# can get coredns configuration (to see domain of cluster, for example)
$ kubectl describe configmap coredns -n kube-system

$ kubectl get services -n kube-system
# should see kube-dns - this is named the same as the legacy kube-dns
# service for interoperability, but is actually the service for coredns

# expose coredns using a NodePort
$ kubectl expose pod <coredns_pod_name> \
                     --name=coredns \
                     --type=NodePort \
                     --port 53 \
                     --protocol='UDP' \
                     -n kube-system

# get the NodePort for the running pod
$ kubectl get service coredns -n kube-system
# capture the second port number after the ':' under PORT(S)

# perform lookup of the dnsutils service created in the last section
# note that the IP used here assumes you deployed the k8s cluster according
# to the previously-referenced tutorial and coredns exists on the master node
$ dig @10.11.12.13 -p <NODE_PORT> dnsutils.default.svc.cluster.local
```

After running the above `dig` command, you should see the same IP address as prevoiusly listed locally on the pod itself. You now have a
way to query the private DNS records internal to the k8s cluster for testing service publication within the cluster scope!

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [ExternalDNS](https://github.com/kubernetes-sigs/external-dns)
* [Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

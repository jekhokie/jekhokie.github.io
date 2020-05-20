---
layout: post
title:  "Concourse on k8s"
date:   2020-04-23 22:48:00 -0400
categories: k8s concourse helm
logo: concourse.jpg
---

[Concourse](https://concourse-ci.org/) is a fantastic orchestration component that can be used in a very pluggable/flexible way. This
tutorial lays the groundwork for the use and exploration of Concourse by installing it on a k8s cluster. The expectation is that this
install can be used to explore Concourse functionality as it relates to the various GitOps-like flows and k8s technologies available for
software delivery and general administration.

### Prerequisites

It's assumed you follwed the [previous tutorial]({% post_url 2020-04-21-vagrant-k8s-cluster %}) and have a local k8s cluster running.

In addition, in order to install Concourse with reasonable functionality, a Persistent Volume needs to be created for storage. Create
a file named `pv.yaml` that will create 3x Persistent Volumes - 2x for worker nodes, and 1x for the PostgreSQL node that Concourse uses:

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: concourse-pv-1
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: concourse-pv-2
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: concourse-pv-3
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
...
```

Next, create the Persistent Volumes and validate they exist:

```bash
# create the persistent volumes
$ kubectl apply -f pv.yaml

# validate the volumes exist
$ kubectl get pv
```

### Concourse Install

We'll now deploy the concourse application using Helm and customize some of the sizes of the Persistent Voluem Claims to be reasonable
for a small cluster deployment (and to fit into our Persistent Volumes that we created in the Prerequisites section above). We'll also
configure the Concourse deployment to use a NodePort since we aren't necessarily using ingress capability:

```bash
$ helm repo add concourse https://concourse-charts.storage.googleapis.com/
$ helm install concourse concourse/concourse \
               --set persistence.worker.size=2Gi \
               --set persistence.worker.storageClass=manual \
               --set postgresql.persistence.size=2Gi \
               --set postgresql.persistence.storageClass=manual \
               --set web.service.type=NodePort
```

You can watch the pods become available for the various components of the deployment using `kubectl get pods`. Once all pods are showing
as Ready/Running, you're safe to proceed.

### Accessing Concourse

The Helm output, when run above, prints some useful information for how to access the Concourse application. Specifically, follow these
steps and access the web page via a browser URL pointing to `http://localhost:8080`:

```bash
$ export POD_NAME=$(kubectl get pods --namespace default -l "app=concourse-web" -o jsonpath="{.items[0].metadata.name}")
$ echo "Visit http://127.0.0.1:8080 to use Concourse"
$ kubectl port-forward --namespace default $POD_NAME 8080:8080
```

You should then be prompted to download the CLI tools for Concourse. You can either download and install via the package links listed,
or if you're using a Mac, you can use Homebrew via the command `brew cask install fly`. This is the command line utility you will use
to deploy pipelines, etc.

In addition, you can also log into the web interface using the default credentials provided with username `test`, password `test`.

### Creating Your First Pipeline

There is a sample you can use to set up some initial functionality found
[here](https://raw.githubusercontent.com/concourse/testflight/8db3bc5680073ec4eb3e36c8a2318297829b9ce0/pipelines/fixtures/simple.yml).
Go ahead and use that file to set up your first pipeline using the `fly` command, making sure to log into a new test team first:

```bash
# log into the test team
$ fly login -t test -u test -p test -c http://localhost:8080

# download the pipeline definition
$ wget https://raw.githubusercontent.com/concourse/testflight/8db3bc5680073ec4eb3e36c8a2318297829b9ce0/pipelines/fixtures/simple.yml

# apply the pipeline definition
$ fly -t test set-pipeline -p test-pipeline simple.yml
```

Now in the UI, you should see a new pipeline named `test-pipeline` with a single job named `simple`. It's highlighted blue indicating
it's in a paused state. Let's unpause it and trigger a build to see what it does:

```bash
# unpause the pipeline
$ fly -t test unpause-pipeline -p test-pipeline

# unpause the job
$ fly -t test unpause-job -j test-pipeline/simple

# trigger a build
$ fly -t test trigger-job -j test-pipeline/simple
```

You should see the UI navbar remove its blue color and the `simple` job start to flash with a yellow outline, indicating the job is
running, and then subside with a green success bar in the UI. Clicking this green bar gives you details about the job specifically.

You now have a Concourse install up and running on your k8s cluster with a sample pipeline - happy testing/experimenting!

### Troubleshooting

You may see the sample job fail with an error like the following:

`runc run: exit status 1: container_linux.go:346: starting container process caused "unknown capability \"CAP_AUDIT_READ\""`

This is likely due to the fact that your cluster is running on a Linux OS whose kernel is too old to support the `CAP_AUDIT_READ`
capability, which was introduced in version 3.16 of the linux kernel (seen [here](http://man7.org/linux/man-pages/man7/capabilities.7.html)).
You can check your Linux kernel version via `uname -a` - note that this is not a Concourse bug but, rather, an incompatibility based
on functionality that Concourse requires in being able to read the audit log via a multicast netlink socket, and the only recourse
is to build your k8s cluster with a new enough kernel that supports this capability (>=3.16).

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Concourse CI](https://concourse-ci.org/)
* [Concourse Helm Chart](https://github.com/concourse/concourse-chart)
* [Linux Capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html)

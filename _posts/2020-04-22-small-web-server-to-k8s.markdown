---
layout: post
title:  "Deploy Tiny Web Server to k8s"
date:   2020-04-22 20:31:00 -0400
categories: k8s busybox helm
---

Simple example deployment of a [busybox](https://busybox.net/) container image to a k8s cluster set up in
[this]({% post_url 2020-04-21-vagrant-k8s-cluster.markdown %}) previous tutorial using a couple of k8s constructs such as Deployments
and Services, as well as using [Helm](https://helm.sh/). This tutorial will lay the groundwork for future exploration in
[GitOps](https://www.weave.works/blog/what-is-gitops-really) flows using tools such as [flux](https://fluxcd.io/).

### Background

[GitOps](https://www.weave.works/blog/what-is-gitops-really) is an interesting concept that captures essential topics around cloud-native
deployments and management. Exploring tools such as [flux](https://fluxcd.io/) to enable GitOps flows requires a workload that can be used
to demonstrate the functionality (as deploying the fluxcd operator is not really valuable/interesting in itself). This tutorial will
summarize the deployment of one of the smallest container image types, the [busybox](https://busybox.net/) image, and have it respond to
web GET requests with a simple "Hello" response.

### Prerequisites

It's assumed you follwed the [previous tutorial]({% post_url 2020-04-21-vagrant-k8s-cluster.markdown %) and have a local k8s cluster running.
In addition, it's also assumed that you have a functioning `docker` environment that you can run locally (if this part is not true, you
can skip the next section "Implementation Using Docker").

### Implementation Using Docker

Let's first test things locally using Docker on your workstation. Implementing a busybox web service is fairly straightforward. Using docker, 
launch an instance of the `busybox` container image as a daemon, mapping local port 8280 to the container port 8280. In addition, have the
container launch with a shell command that will write a simple `Hello` string to an HTML page and use the `httpd` binary start a simple web
server listening on port 8280 and serving contents from the `/var/www/` directory:

```bash
$ docker run --rm -d -p 8280:8280 busybox \
    sh -c "echo 'Hello' > /var/www/index.html && httpd -f -p 8280 -h /var/www/"
```

Once the above launches, you can perform a simple `curl` request locally on your workstation and you should receive a response of `Hello`:

```bash
$ curl http://localhost:8280/index.html
```

If this succeeds, you're on your way and can deploy the service to k8s!

### Deployment to k8s Cluster

There are many different ways to deploy a service to a k8s cluster. We'll go over a couple of them, where Helm charts are the groundwork
we'll lay for a future post dealing with `fluxcd`.

#### Manual via kubectl

To deploy the same functionality as done prior using `docker`, we'll start with a few `kubectl` commands/by hand. First, a namespace will
be created, and then the `busybox` image will be deployed into the namespace using the same command format as before, and we'll also expose
the deployment as a Service to enable accessing the web endpoint:

```bash
# create a test namespace
$ kubectl create ns test-busybox

# deploy the busybox functionality
$ kubectl run busybox --namespace=test-busybox \
                      --port=8280 \
                      --image=busybox \
                      -- sh -c "echo 'Hello' > /var/www/index.html && httpd -f -p 8280 -h /var/www/"

# watch for the pod to show "Running""
$ kubectl get pods -n test-busybox

# expose the deployment
$ kubectl expose pod busybox --type=NodePort \
                             --namespace=test-busybox

# get the node that the particular pod is deployed to
$ kubectl get pods --output=wide -n test-busybox
# based on the "NODE" column, determine the IP address of the node to use
# if you are using the previously configured cluster, this is likely one of the following:
#   - master: 10.11.12.13 (this is only possible if you removed the taint to enable master scheduling of pods)
#   - node1:  10.11.12.14
#   - node2:  10.11.12.14

# get the NodePort for the running pod
$ kubectl get service busybox -n test-busybox
# capture the second port number after the ':' under PORT(S)

# make a GET request to the IP and Port captured in the previous steps
$ curl http://<NODE_IP>:<NODE_PORT>
# the response should print 'Hello'
```

If all goes well and you get the response, feel free to delete the namespace (and all corresponding resources) as we'll be
creating a new fresh namespace in the next section:

```bash
$ kubectl delete ns test-busybox
```

#### Semi-Automatic via Helm Charts

*NOTE*: The descriptions in this section assume a scratch Helm setup and build up contents from there. If you want to get
a Helm directory structure that already has the changes applied in this section and just install the deployment, refer to
and operate all `helm` commands from
[this sample chart director](https://github.com/jekhokie/scriptbox/tree/master/multiple--vagrant-istio-k8s-cluster/charts/busybox-http).

There weren't that many `kubectl` commands to run to get the `busybox` web server deployed, but it's still too manual/error
prone for our liking, not very repeatable, and iterations are slower than a configuration-driven approach. This is where Helm
shines. Helm is a package manager for k8s which affords a more declarative and complete approach to packaging and deploying
services.

First, we'll create a chart using the `helm` binary (can be installed using `brew install helm` if you're on a Mac), which
will create a directory structure for the package:

```bash
$ helm create busybox-http
```

There are a few components that are created, but we'll focus on only the things we need and leave the rest of the discovery
up to the reader via the official Helm docs. We'll need to update a few files to get our deployment aligned.

First, edit the `templates/deployment.yaml` file to insert a couple extra arguments into the template that will allow us to
configure the `busybox` container to run the web server and configure `liveness` and `readiness` probes:

```yaml
...
spec:
...
  template:
  ...
    spec:
    ...
      containers:
      - name: {{ .Chart.Name }}
        ...
        command: ["/bin/sh"]
        args: ["-c", "{{ .Values.bootCommand }}"]
        ...
        ports:
          - name: http
            containerPort: 8280
            ...
        livenessProbe:
          httpGet:
            ...
            port: {{ .Values.healthCheckPort }}
            ...
        readinessProbe:
          httpGet:
            ...
            port: {{ .Values.healthCheckPort }}
            ...
...
```

Next, edit the `values.yaml` file and update the following parameters (the remainder can be left alone for now) as well as
include values for the existing parameters we defined in the deployment template:

```yaml
image:
  repository: busybox
...
nameOverride: "busybox"
fullNameOverride: "busybox"
bootCommand: "echo 'Hello' > /var/www/index.html && httpd -f -p 8280 -h /var/www/"
healthCheckPort: 8280
...
serviceAccount:
  create: false
...
service:
  type: NodePort
  port: 8280
...
```

Next, edit the `Chart.yaml` to ensure the version of `busybox` being deployed is relevant/latest (you can get the latest
version when you run the tutorial from [here](https://hub.docker.com/_/busybox?tab=description)), or simply use `latest`
as the version to get the latest build version available without needing to look it up:

```yaml
...
appVersion: latest
...
```

Next, perform a dry-run of the Helm install to ensure the resources generated using your values look correct.

```bash
$ helm install --dry-run \
               --debug \
               --namespace=test-busybox \
               busybox-http .
```

Finally, if all looks well, create a namespace for the deployment and go ahead and install your service:

```bash
# create new namespace
$ kubectl create ns test-busybox

# install your service
$ helm install --namespace=test-busybox busybox-http .
```

We'll now repeat the process for figuring out how to communicate with the service from the local workstation:

```bash
# get the node that the particular pod is deployed to
$ kubectl get pods --output=wide -n test-busybox
# based on the "NODE" column, determine the IP address of the node to use
# if you are using the previously configured cluster, this is likely one of the following:
#   - master: 10.11.12.13 (this is only possible if you removed the taint to enable master scheduling of pods)
#   - node1:  10.11.12.14
#   - node2:  10.11.12.14

# get the NodePort for the running pod
$ kubectl get service busybox -n test-busybox
# capture the second port number after the ':' under PORT(S)

# make a GET request to the IP and Port captured in the previous steps
$ curl http://<NODE_IP>:<NODE_PORT>
# the response should print 'Hello'
```

If all goes well and you get the response, congratulations - you've successfully deployed your application via Helm!

You're now in a great position to iterate quickly. For example - to demonstrate the fast iteration patterns, let's
make a change to the Helm chart and update our deployment in place. Edit the `values.yaml` file and increase the
value for `replicaCount` to `2` and save the file.

Next, apply the change to the existing deployment (performing an `upgrade`):

```bash
$ helm upgrade --namespace=test-busybox busybox-http .
```

Once the changes are applied, inspect your pods (`kubectl get pods -n test-busybox`) to ensure that you have 2 pods now
showing as `Running`, indicating that the upgrade was successful!

Finally, feel free to delete the namespace (and all corresponding resources) to clean up your k8s cluster, or continue
experimenting with the Helm capabilities!

```bash
$ kubectl delete ns test-busybox
```

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Busybox HTTP Daemon (httpd) webserver](https://openwrt.org/docs/guide-user/services/webserver/http.httpd)

---
layout: post
title:  "k8s GitOps with FluxCD"
date:   2020-04-23 19:58:00 -0400
categories: k8s flux gitops busybox helm
---

GitOps is a highly valuable concept for deploying and maintaining software using machines. This tutorial builds on
[this]({% post_url 2020-04-22-small-web-server-to-k8s %}) previous tutorial by using [flux](https://fluxcd.io/) to monitor
the example tiny busybox web server in [this](https://github.com/jekhokie/gitops-testing) repository for changes (commits,
merges, etc.), and automatically deploys/updates the running instance in the k8s cluster running from
[this previoustutorial]({% post_url 2020-04-21-vagrant-k8s-cluster %}) (although any k8s environment should suffice).

### Prerequisites

It's assumed you follwed the [previous tutorial]({% post_url 2020-04-21-vagrant-k8s-cluster %}) and have a local k8s cluster running.
In addition, you'll need to install the `fluxctl` utility via the Homebrew command `brew install fluxctl`. Finally, fork
[this repository](https://github.com/jekhokie/gitops-testing) as this is the repository we will be configuring flux up to
monitor and support.

### Deploy Flux

To deploy flux to the running cluster, we'll use Helm. Make sure to update `<YOUR_USER_ID>` with the User ID of your GitHub account
where you forked the `gitops-testing` repository to for use:

```bash
# apply custom resource definitions required for flux
$ kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml

# create a flux namespace for install
$ kubectl create ns flux

# install the fluxcd and helm-operator components, ensuring
# only v3 is supported to avoid the need to install tiller
$ helm repo add fluxcd https://charts.fluxcd.io
$ helm upgrade -i flux fluxcd/flux \
               --set git.url=git@github.com:<YOUR_USER_ID>/gitops-testing \
               --set git.pollInterval=60s \
               --set git.branch=master \
               --namespace flux
$ helm upgrade -i helm-operator fluxcd/helm-operator \
               --set git.ssh.secretName=flux-git-deploy \
               --set helm.versions=v3 \
               --namespace flux

# monitor and wait for all pods to be ready and show "Running"
$ kubectl -n flux get pods
```

Next, you'll need to get the public SSH key of flux and add it to your forked version of the repository to enable flux
to communicate with the repo effectively. To get the key contents, use the following command:

```bash
$ fluxctl identity --k8s-fwd-ns flux
```

Copy the output value of the public key and add it to the "Deploy Keys" setting of your forked `gitops-testing` repository,
making sure to grant write access to the repository.

Within a minute, you should have your busybox HTTP service up and running/available in the namespace `test-busybox`. You
can monitor Flux operation logs via the following command:

```bash
$ kubectl -n flux logs deployment/flux -f
```

After seeing updates posted that Flux pulled down the latest commit and applied the changes (created the namespace, applied
the resources, etc.), you can access the service via the previous tutorial method of obtaining the NodePort value and using
the Node IP on which the pods are deployed:

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

If the above worked as expected, then you're in good shape.

### Make a Change, Observe the Results

Now that we have Flux integrated with the forked repository, let's go ahead and make a change and commit it/push it back to
the master branch in the `gitops-testing` repository. Edit the file `charts/busybox-http/values.yaml` and update the
property `replicaCount` to be `2` (or something different than what is already present in that file). Commit your change
and push it to the master branch of your forked repository.

Now watch the flux logs for activity - within 60 seconds (previously configured when installing flux), you should see flux
reach out to the repository and get the latest changes to the repo/apply them to your cluster:

```bash
$ kubectl -n flux logs deployment/flux -f
```

Once you see the change applied, run `kubectl get pods -n test-busybox` and you should see "n" pods deployed, where "n" is
the value you used for the `replicaCount` variable, acknowledging that flux did indeed update the state of your deployment.

That's it - you're now free to use your git repository as your negotiation point with your automated deployment!

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Get Started with Flux](https://docs.fluxcd.io/en/latest/tutorials/get-started/)
* [Get started with Flux using Helm](https://docs.fluxcd.io/en/latest/tutorials/get-started-helm/)
* [Managing Helm releases the GitOps Way](https://github.com/fluxcd/helm-operator-get-started)
* [flux-get-started](https://github.com/fluxcd/flux-get-started)

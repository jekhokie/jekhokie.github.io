---
layout: post
title:  "Configuring Traefik as k8s Ingress Controller"
date:   2019-12-18 21:47:00 -0400
categories: raspberry-pi k8s traefik linux
---

[Kubernetes](https://kubernetes.io/) (k8s) clusters can have LoadBalancer-type services expose deployments over a specific
IP and port to the outside world. However, as the Service object works at Layer 4 of the OSI model, this can fast become
very expensive as each Service would end up with its own LoadBalancer with specific IP and port. In addition, this implementation
requires a dedicated Load Balancer that can support API interaction for configuring the instance. This post details how to set
up an [Traefik](https://docs.traefik.io/) Ingress controller and load balancer that supports OSI Layer 7-based load distribution,
allowing re-use of a dedicated IP address and dynamic path-based routing to back-end services.

### Prerequisites

This post assumes that you have a k8s cluster installed and configured for use. Specifically, it will assume that you have
followed [this previous post]({% post_url 2019-12-14-raspi-k8s-cluster %}) and have a cluster with the same hostnames and
configurations, but this post can likely be adapted to any sufficiently-configured k8s cluster.

### Limitations

As this k8s cluster has been built on "bare metal" without any cloud integrations for load balancers, we will be configuring
the Ingress controller to use a `NodePort` service instead of a `LoadBalancer` service. This will result in a single entry
point via one of the cluster nodes (limiting), but setting up a bare metal load balancer implementation (such as
[MetalLB](https://metallb.universe.tf/)) is complicated based on the routing rules necessary and ARP configurations (or BGP
peering) based on the network layout of the cluster.

It is important to note, however, that if you did have a cloud-deployed cluster or first-class integration with a load balancer
solution, you should utilize it as it mimics a virtual server (via Virtual IP address) ingress method and is more robust/scalable
than simply an Ingress with NodePort Service functionality.

### Deploying the Ingress Controller

Deploying the Traefik ingress controller is fairly straightforward with a couple of changes to the original instructions
to support a smaller workload by using the Alpine container image as the base:

```bash
# apply RBAC permissions
kubectl apply -f https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/traefik-rbac.yaml

# download the daemon set and service configuration
# for deploying the traefik ingress
wget https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/traefik-ds.yaml

# adjust the file to utilize the Alpine version
# of the container image - adjust the following
# property for the DaemonSet object:
#   spec.template.spec.containers[0].image: traefik:v1.7.20-alpine

# then, create the DaemonSet and Service
kubectl apply -f ingress.yml

# wait for the pods to show "Running" prior to proceeding
kubectl get pods --selector="name=traefik-ingress-lb" --namespace=kube-system

# show details about the Service - get the second-listed
# port number after port 80 (for something like "80:30323/TCP",
# copy "30323" as the port):
kubectl get services traefik-ingress-service --namespace=kube-system
```

Now, from your local device, perform a request against the cluster master node on the port captured in the last
step above: `curl http://node1:30323`

If all goes well, you should see a "404 page not found" and at this point, you have your base ingress configured
and ready to serve requests to back-end workloads.

#### Traefik UI

Optionally, you can deploy the Traefik UI to help visualize the Traefik ingress functionality. There is again a
slight adjustment that is needed to the out of box spec, which we will modify below.

```bash
# download the default configuration for the traefik-ui
wget https://raw.githubusercontent.com/containous/traefik/v1.7/examples/k8s/ui.yaml

# edit the ui.yaml file to update the following property
# for the Service definition:
#   spec.type: NodePort

# then, create the Service and Ingress
kubectl apply -f ui.yaml

# show details about the Service - get the second-listed
# port number after port 80 (for something like "80:30323/TCP",
# copy "30323" as the port):
kubectl get services traefik-web-ui --namespace=kube-system
```

After identifying the port number for the UI, open a browser and navigate to the following URL, and you should
see the Traefik UI: [http://node1:30323](http://node1:30323).

### Deploying an Application with Ingress

At this point, the ingress is set up but returning 404 responses because no configurations have been made as of
yet. Let's configure a sample application to be deployed to the cluster and route requests to the application
by way of the ingress. We'll use the [Hypriot Raspberry Pi Busybox HTTP Server](https://hub.docker.com/r/hypriot/rpi-busybox-httpd/)
as a lightweight deployment:

```bash
# create a deployment of the busybox http server
# with 3 replicas
kubectl run hypriot --image=hypriot/rpi-busybox-httpd \
                    --replicas=3 \
                    --port=80

# validate the pods came up to a "Ready" state
# and that the deployment looks correct
kubectl get pods --selector="run=hypriot"
kubectl get deployments hypriot

# next, expose the hypriot web deployment
# as a service and validate the service
kubectl expose deployments hypriot --port 80
kubectl get services hypriot

# using the service CLUSTER-IP, attempt to
# curl the endpoint - you should see an
# HTML response
curl http://<CLUSTER-IP>
```

At this point, you should be able to make a GET request to the endpoint using the `CLUSTER-IP` identified
in the Service and see a HTML response: `curl http://<CLUSTER-IP>`. Note that because the service is
exposed using a `CLUSTER-IP`, you will only be able to make the request/access the Service from one of the
k8s nodes.

Now that we have a Service exposed to the cluster, let's expose the Service to outside the cluster using
the Ingress we configured. Create an Ingress configuration file `expose-hypriot.yaml` with the following
contents:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hypriot
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: hypriot
          servicePort: 80
```

The above configuration will create an Ingress with the root path of the URL pointing to the Hypriot web
server running on port 80. Let's apply the file and create the Ingress:

```bash
# create the ingress
kubectl apply -f expose-hypriot.yaml

# validate the ingress shows up
kubectl get ingress hypriot
```

You can now open a web page and navigate to any of the k8s nodes hostnames with the port captured initially
in this post when deploying the Traefik Ingress controller (the port that shows up when running
`kubectl get services traefik-ingress-service --namespace=kube-system`) and you should see the Hypriot web
page display: `http://node1:<TRAEFIK-INGRESS-CONTROLLER-PORT>`.

#### Advanced URL Rewriting

As mentioned, one of the benefits of an Ingress is the ability to use a single IP and route the requests
to multiple backends based on the hostname and content sent in the request. This sometimes means creating
unique URL paths to query which will allow the Ingress to determine which backend to send the request to.
You can configure a specific `path` for the Hypriot service we deployed, and configure an `annotation` to
ensure that Traefik correctly rewrites the URL to the root path upon sending the request to the backend
(to ensure the web service receives the path request it expects). Delete your existing Ingress
(`kubectl delete -f expose-hypriot.yaml`) and update the `expose-hypriot.yaml` file to include an
`annotation` for URL rewrite and set the path to be something unique such as `/hypriot-web`:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hypriot
  annotations:
    traefik.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /hypriot-web
        backend:
          serviceName: hypriot
          servicePort: 80
```

Create the Ingress:

```bash
kubectl apply -f expose-hypriot.yaml
```

Once created, open your browser or make a curl request to the same URL as before (`http://node1:<TRAEFIK-INGRESS-CONTROLLER-PORT>`)
and you should receive a 404 indicating the root path has no backend to serve content. Append the path
configured `/hypriot-web` and re-submit the request, and you should see the web content response again.

**Note**: Depending on how the web server is written, it may not respond well to the URL rewrites (for instance,
the Hypriot web container image web service does not display image data because the HTML was formatted as such to
use root path references to the image content, which is incompatible with the URL rewriting). Test the functionality
when developing and ensure you get the response content you expect if attempting to use rewriting.


### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Traefik User Guide for Kubernetes](https://docs.traefik.io/v1.7/user-guide/kubernetes/)
* [Traefik Code - Issue #5422](https://github.com/containous/traefik/issues/5422)
* [Hypriot - Kubernetes Raspberry Pi Cluster](https://blog.hypriot.com/post/setup-kubernetes-raspberry-pi-cluster/)
* [Hypriot Raspberry Pi Busybox HTTP Server](https://hub.docker.com/r/hypriot/rpi-busybox-httpd/)

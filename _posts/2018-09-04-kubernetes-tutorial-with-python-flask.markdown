---
layout: post
title:  "Kubernetes Tutorial with Python Flask"
date:   2018-09-04 22:05:11 -0400
categories: ubuntu linux python docker container kubernetes minikube
---
Although familiar with [Docker](https://www.docker.com/) containers and superficially played
with [Kubernetes](https://kubernetes.io/) in the past, I've not taken the required time to learn the
core concepts of Kubernetes to really grasp all the possibilities the technology offers. This post
is a collection of my deper first learnings of the container orchestration technology. In order to
focus on the use of Kubernetes rather than the installation and configuration of it, this tutorial
will be using the [Minikube](https://github.com/kubernetes/minikube) implementation of Kubernetes
to simplify the installation.

## Kubernetes Concepts

First, let's start with some basic object definitions for the Kubernetes application:

- **Pod**: Basic building block of deployment. Typically 1:1 with a running process/Container,
but in more complicated scenarios, can be one-to-many.
- **Service**: Abstraction defining a logical set of Pods - typically a single IP address that routes
requests to one or more Pods via balancing mechanisms.
- **Volume**: File system/disk that enables cross-Container storage and reference.
- **Namespace**: Construct to enable large-scale usage of the Kubernetes cluster by many users,
logically grouping clusters.

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

## Kubernetes Setup

Now that we've learned a few core concepts/terminology, let's get started with implementing and
experimenting with a Kubernetes setup.

### Technology Ecosystem

The steps in this post are executed on an older physical server running the Ubuntu operating system
(specifically, Ubuntu 16.04, 64-bit). Details related to the host machine are as follows:

**Host**

- **Hostname**: kubernetes.localhost
- **OS**: Ubuntu 16.04
- **CPU**: 2
- **RAM**: 3 GB (originally 4GB, but failed memory module decreased available memory)
- **Network**: Private Network
- **IP**: 192.168.1.233

Additionally, the following are the versions of software utilized at the time of this post (for
reference):

- **Docker**: 18.06.1-ce
- **Kubectl**: 1.11.2
- **Minikube**: 

### Prerequisites

In order to install Minikube, we will first install the following prerequisites:

- Docker
- Kubectl

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

Next, we will install Kubectl. Note that this utility is not technically a prerequisite for
installing the Minikube application, but is useful for interacting with the application once it has
been installed (and the Minikube installer looks for the installation of this utility):

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

One item to note - when running the above command, sudo was necessary in order to ensure the binaries could
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

## Application Experimentation

Now that we have a Kubernetes instance to play with, let's create a demo application using Python Flask,
a light-weight Python web server framework, to see how we can deploy and scale an application. This
application will be named "randomizer" and will eventually include a scaled application deployment that
communicates with a MySQL database and prints random words to a user via a web page, sourced from the
MySQL database.

### Create the Flask Application

First, it's important to note that you can utilize the Minikube Docker installation as the Docker
image repository (and should). This will make life easier as it relates to deploying an application from
a Docker image via Minikube as it keeps everything local for Minikube to pull from (does not
require the Minikube installation to source your created images from the web). In order to utilize the
Minikube Docker installation, source the environment variables required for you to interact with Docker:

{% highlight bash %}
$ eval $(minikube docker-env)
{% endhighlight %}

Next, we will need to create a Python Flask application. A good starting point is a Hello World
application I created for re-use during these tutorials and can be found here:

[Hello World Flask App](https://github.com/jekhokie/scriptbox/tree/master/python--flask-hello-world)

Note that this Hello World application requires Python VirtualEnv to be installed in order to manage
dependencies - this can be easily performed via the following:

{% highlight bash %}
$ sudo apt-get install python-pip
$ pip install virtualenv
{% endhighlight %}

Next, clone this repository and then copy the Python Flask tutorial to a directory of your choosing:

{% highlight bash %}
# clone the repository from the link above, then perform the following
$ mv python--flask-hello-world randomizer/
$ cd randomizer/
$ virtualenv .env
$ source .env/bin/activate
$ pip install -r requirements.txt
{% endhighlight %}

You can now test that the application functions as expected - run the following command to launch a
local instance of the Flask application for inspection. Note that this is not at all hooked up to
Docker or Minikube as of yet - this test is just to ensure you have the correct configuration/setup:

{% highlight bash %}
$ python run.env
{% endhighlight %}

Navigate to the following URL (replace the IP address with the IP of your own host) and you should
see a "Hello World" message with the hostname of your host that you are running this application from:

http://192.168.1.233:8000/

### Package the Flask Application as a Docker Image

Now that we have proven the application functions as expected, we can create a Docker image from
the application. For convenience, a Dockerfile has already been created for packaging. You can
inspect the Dockerfile for information about how this image is created. To create the image, perform
the following:

{% highlight bash %}
$ docker build -t randomizer .
# inspect the images to see that your new image was created
$ docker image ls
{% endhighlight %}

Now that we have created the Docker image, let's try it out - note that this test *only* uses Docker
natively and does not interact with the Minikube Kubernetes application:

{% highlight bash %}
$ docker run -p 8080:8000 randomizer
{% endhighlight %}

If all goes well, you should be able to visit the same URL as previously in the above command
with the exception of the port being different since the above `docker` command forwards the Docker
container port 8000 to localhost port 8080. Again, replace the IP address with the IP of your host
instance:

http://192.168.1.233:8080

### Deploy the Flask Application to Minikube

Now that we have a working Docker image, let's have some fun with the Minikube installation. First,
we will deploy a single instance (Pod) of the application to the Minikube application:

{% highlight bash %}
$ kubectl run randomizer --labels="app=flask" --image=randomizer --port=8000 --image-pull-policy=Never
{% endhighlight %}

The above command will deploy your "randomizer" Docker image and apply a label of "app=flask". The
`--port` specifier indicates that port 8000 should be forwarded from the Pod (Container) to the Minikube
Cluster, which is the default port the Hello World Flask application runs on. It is important to
specify `--image-pull-policy=Never` in the above command due to the fact that we stored the Docker
image within the Kubernetes Docker image repository - if you fail to provide this switch, Minikube
will attempt to source the image from the public Docker image repository (where the image obviously
does not exist).

You can get information about your Pod and Deployment via the following:

{% highlight bash %}
# list pods
$ kubectl get pods

# get information about your pod
# (use the "NAME" parameter from the previous command above)
$ kubectl describe pods <NAME>

# information about the deployment
$ kubectl get deployments randomizer
$ kubectl describe deployments randomizer
{% endhighlight %}

Once the above command completes, you will have your single Pod running, but you will not yet be able
to access the application. This is because you need to specify a Service to expose the application
and make it accessible outside of the Minikube cluster:

{% highlight bash %}
$ kubectl expose deployment randomizer --type=LoadBalancer --name=randomizer
{% endhighlight %}

The `expose deployment` command creates a Service and load balances the single "randomizer" Pod
created previously. You can get information about the Service via the following:

{% highlight bash %}
# information about the service
$ kubectl get services randomizer
$ kubectl describe services randomizer

# service URL information
$ minikube service randomizer --url
{% endhighlight %}

The last command in the above list should output a URL to access the application. Type this URL
into a browser that can access your host instance and you should see the Hello World Flask application
web page show up with a "hostname" that is the name of the Docker container inside which the application
is running rather than the host on which your Minikube instance is running.

Congratulations - you now have an application that is deployed to a Kubernetes instance and accessible
outside of the Cluster!

### Scale the Flask Application

Deploying a single Pod is interesting as a proof of concept, but most applications will need to be
scaled to handle high traffic volumes. Kubernetes makes scaling applications incredibly easy via the
ReplicaSet Controller abstraction:

{% highlight bash %}
$ kubectl scale --replicas=3 deployment/randomizer
{% endhighlight %}

Once you run the above command, Minikube will create 2 additional Pods (Containers) running the same
instance of the Python Flask application, automatically placing those Pods in the Service Load Balancer
rotation. An important thing to note is if any of the 3 Pods crashes/is destroyed, the Service will
automatically create new instances to maintain the desired 3 Pods requested (a useful feature for
auto-healing under heavy load or adverse conditions). You can inspect what has happened via the
following:

{% highlight bash %}
# show that the deployment has 3 Pods
$ kubectl get deployments randomizer

# inspect that there are 3 Pods deployed and running
$ kubectl get pods -o wide
{% endhighlight %}

Now that you have verified there are 3 Pods deployed, each with the same application, go ahead and
visit the same Service URL in your browser. This time, refresh the web page several times - you should
see that the "Hello World" text updates with different "hostnames", each being one of the 3 replicas
(Pods) for the deployed application, indicating that not only is the application deployed in a scaled
scenario, it is successfully being load balanced by the Service which exposes it.

### Create the MySQL Instance

**TODO**: FILLMEIN

### Hook the Flask Application Up to the MySQL Instance

**TODO**: FILLMEIN

## Conclusion

There are far more topics and ways to experiment with Kubernetes such as shared storage, secrets,
enhanced Node to Pod mappings, rolling upgrades, blue/green upgrades, etc. This post should lay the
groundwork for exploring these additional topics and may be included in a future post if I end up
getting the time to incorporate them.

## Additional Useful Commands

The below are some additional commands that were not utilized specifically in this tutorial but are
helpful in interacting with the Kubernetes instance:

{% highlight bash %}
# list nodes for a Kubernetes cluster
$ kubectl get nodes

# destroy a service, which will remove the exposure of the application
$ kubectl delete services randomizer

# delete a deployment, which includes the ReplicaSet and Pods
$ kubectl delete deployments randomizer
{% endhighlight %}

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [How To Configure a Continuous Integration Testing Environment with Docker and Docker Compose on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-configure-a-continuous-integration-testing-environment-with-docker-and-docker-compose-on-ubuntu-14-04)
* [Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
* [Minikube Releases](https://github.com/kubernetes/minikube/releases)
* [Minikube Config](https://darkowlzz.github.io/post/minikube-config/)
* [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
* [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)

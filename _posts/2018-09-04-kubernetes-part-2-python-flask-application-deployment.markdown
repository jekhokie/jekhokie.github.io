---
layout: post
title:  "Kubernetes Part 2: Python Flask Application Deployment"
date:   2018-09-04 23:15:48 -0400
categories: ubuntu linux python docker container kubernetes minikube
---
With a functioning Kubernetes cluster, this post is the next step in the series that will go over
how to create, package, and deploy a basic Python Flask Hello World application to the cluster.
The tutorial will include topics such as scaling, service exposure, and other interesting points
when deploying applications to Kubernetes clusters.

## Series

This is Part 2 of the 5-part series:

- [Kubernetes Part 1: Core Concepts and Installation (Minikube)]({% post_url 2018-09-04-kubernetes-part-1-concepts-and-installation %})
- **Kubernetes Part 2: Python Flask Application Deployment**
- [Kubernetes Part 3: MySQL Database Deployment]({% post_url 2018-09-04-kubernetes-part-3-mysql-database-deployment %})
- [Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)]({% post_url 2018-09-04-kubernetes-part-4-application-deployments-via-yaml %})
- [Kubernetes Part 5: Linking Application with Database (Discovery)]({% post_url 2018-09-04-kubernetes-part-5-linking-application-with-database %})

## Applications in Kubernetes

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
$ python run.py
{% endhighlight %}

Navigate to the following URL (replace the IP address with the IP of your own host) and you should
see a "Hello World" message with the hostname of your host that you are running this application from:

[http://192.168.1.233:8000/](http://192.168.1.233:8000/)

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

[http://192.168.1.233:8080](http://192.168.1.233:8080)

### Deploy the Flask Application to Minikube

Now that we have a working Docker image, let's have some fun with the Minikube installation. First,
we will deploy a single instance (Pod) of the application to the Minikube application:

{% highlight bash %}
$ kubectl run randomizer --labels="app=flask" --image=randomizer \
        --port=8000 --image-pull-policy=Never
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

## Next Steps

Head on over to the [next tutorial]({% post_url 2018-09-04-kubernetes-part-3-mysql-database-deployment %})
to learn how to deploy a MySQL instance to Kubernetes.

---
layout: post
title:  "Kubernetes Part 3: MySQL Database Deployment"
date:   2018-09-04 23:31:52 -0400
categories: ubuntu linux python docker container kubernetes minikube
---
A bit backwards, but in Part 2 of the series we created and deployed an application. In this post
we will be deploying a MySQL database for the application to communicate with, and in Part 4 we
will hook this database up to the previously-deployed application.

## Series

This is Part 3 of the 5-part series:

- [Kubernetes Part 1: Core Concepts and Installation (Minikube)]({% post_url 2018-09-04-kubernetes-part-1-concepts-and-installation %})
- [Kubernetes Part 2: Python Flask Application Deployment]({% post_url 2018-09-04-kubernetes-part-2-python-flask-application-deployment %})
- **Kubernetes Part 3: MySQL Database Deployment**
- [Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)]({% post_url 2018-09-04-kubernetes-part-4-application-deployments-via-yaml %})
- [Kubernetes Part 5: Linking Application with Database (Discovery)]({% post_url 2018-09-04-kubernetes-part-5-linking-application-with-database %})

## MySQL 

We will now use an off-the-shelf container image of a MySQL installation to deploy to Kubernetes.
There is one important point in this post which is that we are not going to experiment with
PersistentVolumes and PersistentVolumeClaims. These concepts are required if you wish to maintain
the history of the MySQL instance when the container shuts down, but for simplicity, we are going
to simply deploy the MySQL instance with the assumption that we lose the data within once the
instance shuts down.

First, let's deploy the container - we are going to use an off the shelf instance of MySQL which
can be found [here](https://hub.docker.com/_/mysql/) and pull down an image into our Docker image
repository. You could also simply ignore the `--image-pull-policy=Never` switch in the deployment
command to follow which would automatically obtain the image from the public Docker image
repository:

{% highlight bash %}
$ docker pull mysql
{% endhighlight %}

Next, we will deploy the Docker image to a Pod and expose the instance via a Service. We will also
specify an environment variable for the Container to set the root password in order to enable
access/initialize the instance:

{% highlight bash %}
$ kubectl run mysql --labels="app=mysql" --image=mysql \
        --env="MYSQL_ROOT_PASSWORD=Passw0rd" --port=3306 --image-pull-policy=Never
# verify the pod is running - pod named "mysql" or similar
$ kubectl get pods
$ kubectl get deployments mysql

# expose the mysql pod via a service
$ kubectl expose deployment mysql --type=LoadBalancer --name=mysql

# get the service endpoint for mysql
$ kubectl get services mysql
{% endhighlight %}

The last command above will print the IP and Port for the Service that was created and can be used in
the next section for validation of the instance.

## Validation

You can now use your favorite utility/development tool to connect to the MySQL instance, or a simple
command-line client application, and connect to the IP/Port combination listed in the Service that
was created. If you wish to use a command-line based MySQL client you can install it via the following:

{% highlight bash %}
$ sudo apt-get -y install mysql-client
{% endhighlight %}

You can then connect to the MySQL instance using the IP/Port combination given along with the
password specified when we created the instance:

{% highlight bash %}
$ mysql --user root --host <MYSQL_IP> --port <MYSQL_PORT> -p
# enter the password specified previously as "MYSQL_ROOT_PASSWORD" when prompted

# once logged in, run some commands to get information about the instance
mysql> SELECT VERSION();
# should output something similar:
#   +-----------+
#   | VERSION() |
#   +-----------+
#   | 8.0.12    |
#   +-----------+
#   1 row in set (0.00 sec)
#
{% endhighlight %}

## Next Steps

We now have a fully-functioning MySQL instance without persistent storage. The [next post]({% post_url 2018-09-04-kubernetes-part-4-application-deployments-via-yaml %})
will cover how to create configuration files that define the components of your deployment (much more
manageable)..

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Run a Single Instance Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)

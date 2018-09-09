---
layout: post
title:  "Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)"
date:   2018-09-04 23:50:02 -0400
categories: ubuntu linux python docker container kubernetes minikube
---
Deploying applications via command-line commands is fun and useful for learning the basics
but is not manageable as it relates to maintaining stateful configurations. This post will
start over and re-create the components generated thus far from YAML configuration files that
describe the desired state of the components that will go into this application deployment,
making the solution much more manageable/maintainable.

## Series

This is Part 4 of the 9-part series:

- [Kubernetes Part 1: Core Concepts and Installation (Minikube)]({% post_url 2018-09-04-kubernetes-part-1-concepts-and-installation %})
- [Kubernetes Part 2: Python Flask Application Deployment]({% post_url 2018-09-04-kubernetes-part-2-python-flask-application-deployment %})
- [Kubernetes Part 3: MySQL Database Deployment]({% post_url 2018-09-04-kubernetes-part-3-mysql-database-deployment %})
- **[Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)**
- [Kubernetes Part 5: Linking Application with Database (Discovery)]({% post_url 2018-09-04-kubernetes-part-5-linking-application-with-database %})
- [Kubernetes Part 6: Rolling Updates]({% post_url 2018-09-07-kubernetes-part-6-rolling-updates %})
- [Kubernetes Part 7: Secrets]({% post_url 2018-09-07-kubernetes-part-7-secrets %})
- [Kubernetes Part 8: Persistent Volumes]({% post_url 2018-09-08-kubernetes-part-8-persistent-volumes %})
- [Kubernetes Part 9: ConfigMaps]({% post_url 2018-09-08-kubernetes-part-9-config-maps %})

## Teardown/Clean Slate

First, we will want to destroy all resources previously created to give us a clean slate to start from.
To do so, a list of delete commands are shown below - skip any commands for components you no longer
have in your Kubernetes cluster:

{% highlight bash %}
# destroy all randomizer application resources
$ kubectl delete deployments/randomizer
$ kubectl delete services/randomizer

# delete all mysql resources
$ kubectl delete deployments/mysql
$ kubectl delete services/mysql
{% endhighlight %}

## YAML Files

Now that we have a clean starting point for our Kubernetes cluster, we can start to create the associated
YAML files desired for the deployments. The associated YAML files are included in the repository you cloned
at the beginning of this tutorial but are shown again throughout this post for a more complete explanation
of the directives within.

### Randomizer Application YAML Files

First, we will create YAML configuration files that describe the desired state of 3 Pods for the Randomizer
Python Flask application deployment and associated Service. Create a file named 'app-deployment.yml'
with the content listed below:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: randomizer
  labels:
    app: flask
spec:
  selector:
    matchLabels:
      app: flask
  replicas: 3
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
      - name: randomizer
        image: randomizer
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
          name: flask-container
---
apiVersion: v1
kind: Service
metadata:
  name: randomizer
  labels:
    app: flask
spec:
  ports:
  - port: 8000
    protocol: TCP
    name: flask
  selector:
    app: flask
  type: LoadBalancer
...
```

The content listed above are equivalent to the directives of the `kubectl` command we ran back in
Post 2 when we deployed the application. Next, you will run the `kubectl` command and pass the file
as the argument to indicate to Kubernetes it needs to deploy what is specified in this file:

{% highlight bash %}
$ kubectl apply -f app-deployment.yml
{% endhighlight %}

Once complete, you can inspect the components deployed via the standard kubectl commands as well as
investigate whether your Deployment was successful via opening a browser and inspecting the output
of the application:

{% highlight bash %}
$ kubectl get pods
$ kubectl get deployments/randomizer
$ kubectl get services/randomizer
$ minikube service randomizer --url
# visit the printed URL in a browser to see whether the deployment was successful
{% endhighlight %}

You now have the original application up, running, scaled to 3 Pods, and exposed via a Service. Let's
move on to the MySQL database.

### MySQL Database YAML Files

We will now create YAML configuration files that describe the desired state of the MySQL deployment.
Create a file named 'db-deployment.yml' with the content listed below:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: db
spec:
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: mysql
        image: mysql
        imagePullPolicy: Never
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: Passw0rd
        ports:
        - containerPort: 3306
          name: db-container
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: db
spec:
  ports:
  - port: 3306
    protocol: TCP
    name: mysql
  selector:
    app: db
  type: LoadBalancer
...
```

The content listed above are equivalent to the directives of the `kubectl` command we ran back in
Post 2 when we deployed the application. Next, you will run the `kubectl` command and pass the file
as the argument to indicate to Kubernetes it needs to deploy what is specified in this file:

{% highlight bash %}
$ kubectl apply -f app-deployment.yml
{% endhighlight %}

Once complete, you can inspect the components deployed via the standard kubectl commands as well as
investigate whether your Deployment was successful via opening a browser and inspecting the output
of the application:

{% highlight bash %}
$ kubectl get pods
$ kubectl get deployments/mysql
$ kubectl get services/mysql
{% endhighlight %}

The MySQL database is also now up and running according to the earlier parts of the tutorial. You can use
the local `mysql` client application to access the Service running on the describe Service NodePort to
again validate it is functioning as expected.

## Next Steps

The original components are now back up and running and we have YAML files describing our configurations
that we can utilize next for [Service discovery]({% post_url 2018-09-04-kubernetes-part-5-linking-application-with-database %})
to hook the application up to the database.

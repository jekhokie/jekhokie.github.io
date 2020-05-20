---
layout: post
title:  "Kubernetes Part 9: ConfigMaps"
date:   2018-09-08 22:54:27 -0400
categories: ubuntu linux python docker container kubernetes minikube
logo: k8s-configmap.jpg
---
Now that we have core components ironed out, let's talk about configuration. We somewhat jumped ahead
in our previous post about Secrets before we addressed configuration strategy, so we will revisit the
configuration topic in this post with the use of ConfigMap objects.

## Series

This is Part 9 of the 9-part series:

- [Kubernetes Part 1: Core Concepts and Installation (Minikube)]({% post_url 2018-09-04-kubernetes-part-1-concepts-and-installation %})
- [Kubernetes Part 2: Python Flask Application Deployment]({% post_url 2018-09-04-kubernetes-part-2-python-flask-application-deployment %})
- [Kubernetes Part 3: MySQL Database Deployment]({% post_url 2018-09-04-kubernetes-part-3-mysql-database-deployment %})
- [Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)]({% post_url 2018-09-04-kubernetes-part-4-application-deployments-via-yaml %})
- [Kubernetes Part 5: Linking Application with Database (Discovery)]({% post_url 2018-09-04-kubernetes-part-5-linking-application-with-database %})
- [Kubernetes Part 6: Rolling Updates]({% post_url 2018-09-07-kubernetes-part-6-rolling-updates %})
- [Kubernetes Part 7: Secrets]({% post_url 2018-09-07-kubernetes-part-7-secrets %})
- [Kubernetes Part 8: Persistent Volumes]({% post_url 2018-09-08-kubernetes-part-8-persistent-volumes %})
- **Kubernetes Part 9: ConfigMaps**

## Configuration Maps

Let's revisit a more basic functionality, which is how to manage and organize configurations for your
application. Kubernetes offers another object type known as a ConfigMap for just this task. We somewhat
jumped ahead and addressed Secrets prior to this topic, but ConfigMaps are a nice building block that
every application developer should come to understand in order to ensure Containers are created in a
clean, re-usable fashion.

Let's take our application and put a couple of the remaining hard-coded configuration items into a
ConfigMap instance. Again, there are many ways to use a ConfigMap, but we will demonstrate the method
which exposes environment variables to the respective Pods for reference of the configuration items
in order to make it easy for the application to consume the configurations.

First, let's create a YAML configuration file, named `app-configs.yml`, with the configuration items
we wish to include:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  DB_USER: root
  DB_NAME: randomizer
```

Next, we'll tell Kubernetes to create the ConfigMap from our YAML file:

{% highlight bash %}
$ kubectl create -f app-configs.yml
# inspect the creation
$ kubectl get cm/app-config
$ kubectl describe cm/app-config
{% endhighlight %}

Finally, we'll update our application to use the environment variables and our Deployment YAML file
to pass the variables to the application - first, our application update to the `app/__init__.py`
file to both use the variables as well as print them to the web page to demonstrate the correct
acquisition of the values:

```python
import os
from flask import Flask
import socket
import mysql.connector

app = Flask(__name__)

@app.route('/')
def hello():
    # construct HTML output
    html = "<h3>Hello World from {hostname}!</h3>"
    html += "<h3>Your random word is: {random_word}</h3>"
    html += "<h3>The version of this app is: {version}</h3>"
    html += "<h3>The last host to access the file was: {last_host}</h3>"
    html += "<h3>The database user is: {db_user}</h3>"
    html += "<h3>The database name is: {db_name}</h3>"

    # yes, this is a terrible way to do this, but it works/is simple
    db_user = os.getenv("DB_USER")
    db_name = os.getenv("DB_NAME")
    db = mysql.connector.connect(
              host=os.getenv("MYSQL_SERVICE_HOST"),
              port=os.getenv("MYSQL_SERVICE_PORT"),
              user=db_user,
              passwd=os.getenv("MYSQL_DB_PASSWORD"),
              database=db_name,
              auth_plugin="mysql_native_password"
         )

    cursor = db.cursor()
    cursor.execute("select word from random_words order by rand() limit 1;")
    res = cursor.fetchall()

    # read contents of file, then write hostname to file
    last_host_filename = "/storage-demo/last_access.txt"
    last_host = ""
    if os.path.isfile(last_host_filename):
        last_host = open(last_host_filename, 'r').read()

    last_host_file = open(last_host_filename, 'w')
    last_host_file.write(socket.gethostname())
    last_host_file.close()

    return html.format(random_word=res[0][0],
                       version="7.0",
                       last_host=last_host,
                       db_name=db_name,
                       db_user=db_user,
                       hostname=socket.gethostname())
```

Next, let's build a new Docker image from this with a tagged version of 7 - if desired, you can also
update the `app/__init__.py` file to include version 7 as well:

{% highlight bash %}
$ docker build -t randomizer:v7 .
{% endhighlight %}

We now have our new Docker image - update the deployment file `app-deployment.yml` to use the
new image and specify the use of the ConfigMap items - add the following `envFrom` property to
the .spec.template.spec.containers[0] property at the same indent level as the `env` property,
and obviously update the Docker image tag/version:

```yaml
        ...
        image: randomizer:v7
        ...
        envFrom:
        - configMapRef:
            name: app-config
        ...
```

It may seem strange to specify both an `env` and an `envFrom` property in the YAML configuration but it is
quite natural if you think about the intention - `envFrom` allows creating configurations from a ConfigMap,
while the `env` parameter is still useful for more global parameters that are not application-specific or
if you wish to manage defaults in your Deployments that may be more focused on infrastructure (or wish to use
parameters within the Deployment file as well). There are many reasons to keep both, it just boils down to how
you intend to architect the segregation of configuration from application functionality.

Lastly, to have the changes take effect, perform your rolling update:

{% highlight bash %}
$ kubectl apply -f app-deployment.yml
{% endhighlight %}

Now, refresh your web browser to retrieve a fresh copy of the web page from the version 7 instance we just
deployed. You should see output that includes the environment variables that we created using the ConfigMap
instance, indicating your change was successful!

## Next Steps

This concludes the initial/basics parts of the tutorial. There are likely to be future posts about topics such as
Jobs, Service Accounts, Namespaces, etc. but those will be one-off posts that can build from the output of these
scenarios, so stay tuned, and Happy K8s!

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

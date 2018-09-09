---
layout: post
title:  "Kubernetes Part 6: Rolling Updates"
date:   2018-09-07 20:14:03 -0400
categories: ubuntu linux python docker container kubernetes minikube
---
Now that we have a running application hooked up to a database, we will explore how to go
about rolling updates (zero-downtime updates) of the application.

## Series

This is Part 6 of the 8-part series:

- [Kubernetes Part 1: Core Concepts and Installation (Minikube)]({% post_url 2018-09-04-kubernetes-part-1-concepts-and-installation %})
- [Kubernetes Part 2: Python Flask Application Deployment]({% post_url 2018-09-04-kubernetes-part-2-python-flask-application-deployment %})
- [Kubernetes Part 3: MySQL Database Deployment]({% post_url 2018-09-04-kubernetes-part-3-mysql-database-deployment %})
- [Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)]({% post_url 2018-09-04-kubernetes-part-4-application-deployments-via-yaml %})
- [Kubernetes Part 5: Linking Application with Database (Discovery)]({% post_url 2018-09-04-kubernetes-part-5-linking-application-with-database %})
- **Kubernetes Part 6: Rolling Updates**
- [Kubernetes Part 7: Secrets]({% post_url 2018-09-07-kubernetes-part-7-secrets %})
- [Kubernetes Part 8: Persistent Volumes]({% post_url 2018-09-08-kubernetes-part-8-persistent-volumes %})

## Rolling Updates

Kubernetes has the ability to automatically perform rolling updates of your application. What this means is
if your application is designed to function in a split version scenario (i.e. multiple application
containers behind a load balancer, possibly running different versions of the application) you can utilize
the Deployment construct of Kubernetes to automatically handle rolling updates of your application.

Let's try this out - first, make a change to your application to indicate a difference you can detect in the
version when visiting the web page. For instance, let's update the `app/__init__.py` file to show a version
of the application:

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

    # yes, this is a terrible way to do this, but it works/is simple
    db = mysql.connector.connect(
              host=os.getenv("MYSQL_SERVICE_HOST"),
              port=os.getenv("MYSQL_SERVICE_PORT"),
              user="root",
              passwd="Passw0rd",
              database="randomizer",
              auth_plugin="mysql_native_password"
         )

    cursor = db.cursor()
    cursor.execute("select word from random_words order by rand() limit 1;")
    res = cursor.fetchall()

    return html.format(random_word=res[0][0], version="4.0", hostname=socket.gethostname())
```

Now, let's package this new version of the application as a new Docker image:

{% highlight bash %}
$ docker build -t randomizer:v4 .
# inspect that v3 of the docker image is available
$ docker image ls
{% endhighlight %}

We can now perform a rolling update. This concept previously utilized a Replication Controller as the means for
deploying the new software but the recommendation has since changed to utilize the Deployment object instead.
There are two main ways to accomplish this - via command-line (being explicit to change the image of the deployed
application) or via the `kubectl edit` command, which will allow updating the Deployment YAML config file and then
handling the changes for you. Since we have YAML configuration files driving our application, we will use the
`kubectl edit` command.

First, run the `kubectl edit` command and update the respective parameters for the Docker image:

{% highlight bash %}
$ kubectl edit deployment/randomizer
# edit the .spec.template.spec.container.image
# value to reflect the new docker image
#   image: randomizer:v4
# then save/quit the file

# list the pods and you should see updates to the
# pods being made in a rolling fashion
$ kubectl get pods

# list the rollout status for the update
$ kubectl rollout status deployment/randomizer
{% endhighlight %}

What you'll notice when running the last command (`kubectl get pods`) several times is a rolling update of the
application via a "create 1/remove 1 pod" strategy. This is the default setting for the rolling update - as many
things in Kuberetes, this can also be changed/refined to your liking.

## Rolling Rollbacks

As a note, you can easily revert a deployment in the same fashion (rolling update). First, let's check out the
deployment history:

{% highlight bash %}
# information about deployment history
$ kubectl rollout history deployment/randomizer

# more detail about a specific deployment
$ kubectl rollout history deployment/nginx-deployment --revision 6
{% endhighlight %}

To roll back the application to the last revision previous to the current update, use the following command:

{% highlight bash %}
$ kubectl rollout undo deployment/randomizer
{% endhighlight %}

The above command performs a rolling update of the application back to the revision/version previous to the
current deployment, again in a non-disruptive/non-downtime fashion. You can inspect the pods and the rollout
status similarly to the previous update steps to see what is occurring. There is a major benefit to using the
undo option for the rollout, which is that it is not only a quick way to revert but it also takes care of
reverting the Deployment configuration file changes made as part of that update.

## Next Steps

Now that we have the core functionality built for creating and deploying applications in a rolling fashion,
we will switch gears in our [next post]({% post_url 2018-09-07-kubernetes-part-7-secrets %}) to focus on the
security components of storing passwords and other sensitive information in a secure manner via Kubernetes
Secrets.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Performing Rolling Update Using a Replication Controller](https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/)
* [Deployments -> Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)

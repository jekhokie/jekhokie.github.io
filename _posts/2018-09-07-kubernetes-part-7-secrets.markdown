---
layout: post
title:  "Kubernetes Part 7: Secrets"
date:   2018-09-07 20:52:13 -0400
categories: ubuntu linux python docker container kubernetes minikube
---
Previously we created an application and hard-coded the password for accessing the database (bad). In
this tutorial, we will learn about Kubernetes Secrets and how to use them to store sensitive information
in a safer way.

## Series

This is Part 7 of the 8-part series:

- [Kubernetes Part 1: Core Concepts and Installation (Minikube)]({% post_url 2018-09-04-kubernetes-part-1-concepts-and-installation %})
- [Kubernetes Part 2: Python Flask Application Deployment]({% post_url 2018-09-04-kubernetes-part-2-python-flask-application-deployment %})
- [Kubernetes Part 3: MySQL Database Deployment]({% post_url 2018-09-04-kubernetes-part-3-mysql-database-deployment %})
- [Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)]({% post_url 2018-09-04-kubernetes-part-4-application-deployments-via-yaml %})
- [Kubernetes Part 5: Linking Application with Database (Discovery)]({% post_url 2018-09-04-kubernetes-part-5-linking-application-with-database %})
- [Kubernetes Part 6: Rolling Updates]({% post_url 2018-09-07-kubernetes-part-6-rolling-updates %})
- **Kubernetes Part 7: Secrets**
- [Kubernetes Part 8: Persistent Volumes]({% post_url 2018-09-08-kubernetes-part-8-persistent-volumes %})

## Secrets

Secrets are a nice feature of Kubernetes that enable secure storage and use of sensitive information within
a cluster. First, have a look at the Secrets within Kubernetes, which include some default tokens for access:

{% highlight bash %}
$ kubectl get secrets
{% endhighlight %}

As with many of the Kubernetes objects, there are multiple ways to create secrets, both command-line based as
well as YAML configuration file-based. Given we have learned and are using YAML configuration files for our
application, we will create a YAML configuration file for the database user password Secret. There is one extra
step involved in creating a Secret configuration file which is to base64-encode the sensitive values prior to
placing them into the YAML configuration file:

{% highlight bash %}
$ echo -n 'Passw0rd' | base64
# should output something similar to the following:
#   UGFzc3cwcmQ=
{% endhighlight %}

Next, we will take the base64-encoded value and place it into our Secret YAML file, named `secret.yml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-password
type: Opaque
data:
  mysql_password: UGFzc3cwcmQ=
```

Save the file above and then run the `kubectl` command to create the secret:

{% highlight bash %}
$ kubectl create -f secret.yml

# inspect that the secret was created
$ kubectl get secrets/mysql-password

# decode the secret to validate the contents
$ kubectl get secrets/mysql-password -o yaml

# based on the output of the above command, decode
# the .data.mysql_password value to validate the password:
$ echo 'UGFzc3cwcmQ=' | base64 --decode
# should output the following:
#   Passw0rd
{% endhighlight %}

Now that we have a secret created within the cluster, let's use it in the application. Again, there are
multiple ways you can present a secret to a Pod - either using a file mount or via environment variables.
To keep things clean, we will explore the environment variable option. To do this, we will edit the
Deployment YAML configuration file to replace the clear-text password with a reference to the Secret we
just created. Edit the `app-deployment.yml` file and add the following to the
`.spec.template.spec.containers[0].env` section, right after the `MYSQL_DB_HOST` parameter:

```yaml
...
        - name: MYSQL_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-password
              key: mysql_password
...
```

We will now update our application to use the environment variable to connect to the database. Update
the `app/__init__.py` file to utilize the env var and specify version 5 to ensure we can validate that
the application was deployed easily:

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
              passwd=os.getenv("MYSQL_DB_PASSWORD"),
              database="randomizer",
              auth_plugin="mysql_native_password"
         )

    cursor = db.cursor()
    cursor.execute("select word from random_words order by rand() limit 1;")
    res = cursor.fetchall()

    return html.format(random_word=res[0][0], version="5.0", hostname=socket.gethostname())
```

We will now make the update to our `app-deployment.yml` file to prepare for the rolling update. First, create the
new version of the Docker image. Then, edit the YAML configuration file to use the new image version and perform
the rolling update:

{% highlight bash %}
# create the new image with version 5
$ docker build -t randomizer:v5 .

# update the image in the yaml configuration
$ vim app-deployment.yml
# edit the .spec.template.spec.container.image
# value to reflect the new docker image
#   image: randomizer:v5

# apply the changes/perform the rolling update
$ kubectl apply -f app-deployment.yml
{% endhighlight %}

You'll note we added yet another way to perform a rolling update via the `kubectl apply` command - this is probably
one of the more useful ways to perform this activity due to this method maintaining configuration state in the local
files and not having to remember which configurations have made their way into the cluster yet.

You can first check that the environment variables are present in the Pods via connecting to a shell for one of the
deployed Pods and then using the `printenv` command:

{% highlight bash %}
$ kubectl get pods
# using one of the pod names from the above command:
$ kubectl exec -it <POD_NAME> -- /bin/bash
root@<POD_NAME>:/app# printenv | grep MYSQL_DB_PASSWORD
# should return the following:
#   MYSQL_DB_PASSWORD=Passw0rd
{% endhighlight %}

Next, refresh your web page - you should see the new version of the application and a random word indicating that
the application successfully loaded and used the MySQL root password from the environment variable loaded via
the Kubernetes Secret!

## Next Steps

We now have a way to store sensitive information - in the [next post]({% post_url 2018-09-08-kubernetes-part-8-persistent-volumes %})
we'll explore how to create storage for persisting information so when Pods/Containers are cycled the data can
persist. Additionally, the volumes can be used for shared state information across a horizontally-scaled
application.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

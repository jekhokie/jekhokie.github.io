---
layout: post
title:  "Kubernetes Part 5: Linking Application with Database (Discovery)"
date:   2018-09-04 23:55:22 -0400
categories: ubuntu linux python docker container kubernetes minikube
---
In this post, we will hook the previously-created application up to the now deployed MySQL
database. This post will utilize DNS for discovery and hard-coded credential information,
and in potential future posts we will go over a better way to store confidential information
in the form of Kubernetes secrets.

## Series

This is Part 5 of the 9-part series:

- [Kubernetes Part 1: Core Concepts and Installation (Minikube)]({% post_url 2018-09-04-kubernetes-part-1-concepts-and-installation %})
- [Kubernetes Part 2: Python Flask Application Deployment]({% post_url 2018-09-04-kubernetes-part-2-python-flask-application-deployment %})
- [Kubernetes Part 3: MySQL Database Deployment]({% post_url 2018-09-04-kubernetes-part-3-mysql-database-deployment %})
- [Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)]({% post_url 2018-09-04-kubernetes-part-4-application-deployments-via-yaml %})
- **Kubernetes Part 5: Linking Application with Database (Discovery)**
- [Kubernetes Part 6: Rolling Updates]({% post_url 2018-09-07-kubernetes-part-6-rolling-updates %})
- [Kubernetes Part 7: Secrets]({% post_url 2018-09-07-kubernetes-part-7-secrets %})
- [Kubernetes Part 8: Persistent Volumes]({% post_url 2018-09-08-kubernetes-part-8-persistent-volumes %})
- [Kubernetes Part 9: ConfigMaps]({% post_url 2018-09-08-kubernetes-part-9-config-maps %})

## Service Discovery

In Kubernetes, there are a couple of different ways to link applications via discovery - the
first is via environment variables, and the second is via the Kubernetes DNS. The benefit in having YAML
files drive our Deployments is that it allows us to utilize selectors in attempting to link applications
together. 

As a note, when you have "kube-dns" enabled and have Services you deployed, you can resolve the IP address of
the Service via simple `nslookup` commands utilizing the name of the Service you created:

{% highlight bash %}
$ nslookup randomizer
# should output something similar:
#   Server:    127.0.1.1
#   Address:  127.0.1.1#53
#
#   Non-authoritative answer:
#   Name:  randomizer
#   Address: 92.242.140.21
{% endhighlight %}

The output of the IP address is an internally (within the namespace of the Pods) routable IP address that can
be used for linking Pods for communication. DNS is dynamically updated when Services are created, which eliminates
the ordering problem that environment variables have where Pods wishing to use other Pods must be created
after the Pods they depend on in order for them to obtain the environment variables required.

Given we are using YAML files to drive our configurations, we can use the selector/match capability of the
Kubernetes YAML configuration directives to define the Services and associated information for discovery.

The update to enable discovery is quite simple - it is simply an update to the `app-deployment.yml` file created
in the previous post to call out the environment variable as referencing the database Service information. Add the
following property to the Application Deployment .spec.template.spec.containers[0] property in your `app-deployment.yml`
file:

```yaml
...
        env:
        - name: MYSQL_DB_HOST
          value: mysql
...
```

What the above does is create a match reference to the Service declared in the `db-deployment.yml` file and will pull
the connection information for the database Service once it has been created and inject it into the application
Deployment for use.

Now that you have updated the configuration file, run the `apply` command to apply the changes/inject the environment
variable into the randomizer application Deployment:

{% highlight bash %}
$ kubectl apply -f app-deployment.yml
{% endhighlight %}

You can now inspect the variables that were introduced into the application. First, get the name of one of the Pods
within the Deployment. Then, using this name, connect to the Container and inspect the environment variables available
to see what types of vars can be used within the application:

{% highlight bash %}
$ kubectl get pods

# copy one of the "Name"s of a randomizer Pod
$ kubectl exec -it <RANDOMIZER_POD_NAME> -- /bin/bash
root@<RANDOMIZER_POD_NAME>:/app# printenv | grep MYSQL
# should output something similar to the following:
#   MYSQL_DB_HOST=mysql
#   MYSQL_PORT_3306_TCP_PROTO=tcp
#   MYSQL_SERVICE_PORT_MYSQL=3306
#   MYSQL_SERVICE_PORT=3306
#   MYSQL_PORT=tcp://10.102.104.134:3306
#   MYSQL_PORT_3306_TCP=tcp://10.102.104.134:3306
#   MYSQL_PORT_3306_TCP_PORT=3306
#   MYSQL_PORT_3306_TCP_ADDR=10.102.104.134
#   MYSQL_SERVICE_HOST=10.102.104.134
{% endhighlight %}

As you can see, there are quite a few variables available to the Pod for Service discovery at this point.

## Populate Database

Before we update the application to communicate with the database, we first want to create a database schema and
populate some dummy data. Using the mysql client application, connect to the MySQL database and run the following
commands to create a base schema and put some dummy data into the database:

{% highlight bash %}
$ mysql -u root --host <HOST_IP> --port <SERVICE_PORT> -p

mysql> create database randomizer;
mysql> use randomizer;
mysql> create table random_words(word_id int auto_increment, word varchar(255) not null, primary key(word_id));
mysql> insert into random_words (word) values ('randword1'), ('randword2'), ('randword3');

# check that the inserts succeeded
mysql> select * from random_words;
# should output something similar to:
#   +---------+-----------+
#   | word_id | word      |
#   +---------+-----------+
#   |       1 | randword1 |
#   |       2 | randword2 |
#   |       3 | randword3 |
#   +---------+-----------+
#   3 rows in set (0.00 sec)
{% endhighlight %}

Obviously it would be better to use a Kubernetes Job for this task, but for simplicity, we will use brute force
for this part of the tutorial. Additionally, you would ideally want to inject the schema name, user name, etc.
as environment variables for use by the application as well, but in the interest of time and scope, we will also
brute-force this activity.

## Adding Service Discovery and Updating

Now that we have the database created, table created and populated, and Service exposed, we'll update the application
to use the MySQL database. First, let's update the application to print out the connection information to ensure
we are accurately pulling the environment variables needed. Update the `app/__init__.py` file to read as follows:

```python
import os
from flask import Flask
import socket

app = Flask(__name__)

@app.route('/')
def hello():
    # construct HTML output
    html = "<h3>Hello World from {hostname}!</h3>"
    html += "<h4>MySQL Service Host: {mysql_host}</h4>"
    html += "<h4>MySQL Service Port: {mysql_port}</h4>"

    # get environment variables
    mysql_service_host = os.getenv("MYSQL_SERVICE_HOST")
    mysql_service_port = os.getenv("MYSQL_SERVICE_PORT")

    return html.format(mysql_host=mysql_service_host,
                       mysql_port=mysql_service_port,
                       hostname=socket.gethostname())
```

Once the above file is updated, create a new Docker image from this file tagged with `v2` to indicate an incremental
revision to the application:

{% highlight bash %}
$ docker build -t randomizer:v2 .
{% endhighlight %}

Now we'll update the Pod images to use the new image we have created. This can be done all at once or in a rolling
fashion. Since we did not explicitly create a Replication Controller when creating the various resources, we cannot
perform a Rolling Upgrade and therefore will perform a complete swap-out of the images instead (destructive/downtime-
driving activity) and save Rolling Upgrades for a future post:

{% highlight bash %}
kubectl set image deployments/randomizer randomizer=randomizer:v2
{% endhighlight %}

Once the command completes, it will take several minutes for the existing Pods to terminate and new Pods to be
created. You can check the status of the operation using the standard `kubectl get pods` command. Once the new Pods
are available, re-visit the web page for the Service to see the MySQL database IP and port listed as part of the
rendering, indicating we are making good progress to fully linking the application with the database.

Next we will install the MySQL Python library and pull random words from the database as part of the rendering of
the page. Update the `requirements.txt` file in the `randomizer/` folder to read as follows:

```python
Flask
mysql-connector
mysql-connector-python
```

Then, install the additional packages:

{% highlight bash %}
$ . .env/bin/activate
$ pip install -r requirements.txt
{% endhighlight %}

Now that the library is available, update the `app/__init__.py` file to look like the following:

```python
from flask import Flask
import socket
import mysql.connector

app = Flask(__name__)

@app.route('/')
def hello():
    # construct HTML output
    html = "<h3>Hello World from {hostname}!</h3>"
    html += "<h3>Your random word is: {random_word}</h3>"

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

    return html.format(random_word=res[0][0], hostname=socket.gethostname())
```

**Note**: Yes, yes, I know, terrible code and lack of error checking/boundary condition checks. This is
not intended to be a defensive coding tutorial, so brute-force it is. Additionally, the password contained within
is better suited to be passed via a Kubernetes Secret type (topic for another day) and other configurations sent
via configurations within Kubernetes as well.

Once the file is updated, build another image (v3) for use and deploy it:

{% highlight bash %}
$ docker build -t randomizer:v3 .
$ kubectl set image deployments/randomizer randomizer=randomizer:v3
{% endhighlight %}

And again, update your Pods to use the latest (v3) version of the image. Once fully updated and the Pods have come
back online, re-visit your web page and refresh the page repeatedly. You should see the 3 random words appearing
somewhat randomly when you refresh, indicating the application is communicating effectively with the database and
each Pod is successfully getting a random word from the MySQL instance!

## Next Steps

We now have a full-fledged application hooked up to a database. Applications change over time, and with change
comes deployments of the new software versions. Visit the [next post]({% post_url 2018-09-07-kubernetes-part-6-rolling-updates %})
to see how you can do rolling updates of the application in a zero-downtime fashion.

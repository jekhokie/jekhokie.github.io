---
layout: post
title:  "Kubernetes Part 8: Persistent Volumes"
date:   2018-09-08 20:21:13 -0400
categories: ubuntu linux python docker container kubernetes minikube
logo: k8s-persistentvolume.jpg
---
Up to this point, any storage utilized by Pods will disappear if and when the Pod/Container is destroyed.
This is acceptable for stateless applications but less desirable for stateful applications and databases.
This post will go over how to create and use a Persistent Volume across multiple Pods to demonstrate how
to create stateful application deployments.

## Series

This is Part 8 of the 9-part series:

- [Kubernetes Part 1: Core Concepts and Installation (Minikube)]({% post_url 2018-09-04-kubernetes-part-1-concepts-and-installation %})
- [Kubernetes Part 2: Python Flask Application Deployment]({% post_url 2018-09-04-kubernetes-part-2-python-flask-application-deployment %})
- [Kubernetes Part 3: MySQL Database Deployment]({% post_url 2018-09-04-kubernetes-part-3-mysql-database-deployment %})
- [Kubernetes Part 4: Application Deployments (The Smart Way - YAML Files)]({% post_url 2018-09-04-kubernetes-part-4-application-deployments-via-yaml %})
- [Kubernetes Part 5: Linking Application with Database (Discovery)]({% post_url 2018-09-04-kubernetes-part-5-linking-application-with-database %})
- [Kubernetes Part 6: Rolling Updates]({% post_url 2018-09-07-kubernetes-part-6-rolling-updates %})
- [Kubernetes Part 7: Secrets]({% post_url 2018-09-07-kubernetes-part-7-secrets %})
- **Kubernetes Part 8: Persistent Volumes**
- [Kubernetes Part 9: ConfigMaps]({% post_url 2018-09-08-kubernetes-part-9-config-maps %})

## Persistent Volumes and Claims

Up to this point, we have created an application architecture that has no persistent storage. We will now
explore how to create a Persistent Volume that can be used to present persistent storage across one or
multiple Pods.

There are two main concepts when it comes to Persistent Volumes. First, a Persistent Volume is created by
an Administrator-like individual and serves as a mass of storage that can be consumed by one or more
Pods. Consider this a "pool" of storage.

Second, a Persistent Volume Claim is created by a user/application and consumes part or all of a Persistent
Volume. The Claim is what slices off a piece of the Persistent Volume and presents it to the Pod(s) that
wish to consume it.

## Creating and Consuming a Persistent Volume

Now that we understand the concepts, we'll expand our application to have a mount on every Pod that is a
shared storage volume to simulate both the persistence of the storage as well as the shared nature across
multiple Pods. Again, since we are now utilizing YAML configuration files to create objects, we'll
create a YAML file to construct the Persistent Volume.

Create a file named `create-storage.yml` with the following contents to create our Persistent Volume:

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: app-pvolume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

We will then create a Persistent Volume Claim that can be consumed by the Pod(s) - create another YAML
file named `create-storage-claim.yml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvclaim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Let's now create the objects, starting with the Persistent Volume, and then ending with the Persistent
Volume Claim:

{% highlight bash %}
# create the pv
$ kubectl create -f create-storage.yml

# inspect details about the new volume
$ kubectl get pv/app-pvolume
$ kubectl describe pv/app-pvolume

---

# create the pvc
$ kubectl create -f create-storage-claim.yml

# inspect details about the new claim
$ kubectl get pvc/app-pvclaim
$ kubectl describe pvc/app-pvclaim
{% endhighlight %}

We'll now update the application YAML to indicate that the volume should be consumed by the Deployment.
Update the `app-deployment.yml` file to consume the Volume by specifying the .spec.template.spec contents
reflected below:

```yaml
...
    spec:
      containers:
      - name: randomizer
        image: randomizer:v5
        imagePullPolicy: Never
        env:
        - name: MYSQL_DB_HOST
          value: mysql
        - name: MYSQL_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-password
              key: mysql_password
        ports:
        - containerPort: 8000
          name: flask-container
        volumeMounts:
        - name: shared-storage
          mountPath: /storage-demo
      volumes:
      - name: shared-storage
        persistentVolumeClaim:
          claimName: app-pvclaim
...
```

Next, let's ensure that the directory `/storage-demo` does not yet exist and create it in preparation
for the mount:

{% highlight bash %}
# copy one of the application pod names
$ get pods

# open a shell to the pod
$ kubectl exec -it <POD_NAME> -- /bin/bash

# inspect to ensure no /storage-demo directory exists
root@<POD_NAME>:/app# ls -l /storage-demo
# should output:
#   ls: cannot access '/storage-demo': No such file or directory

# create the directory in preparation for the mount
root@<POD_NAME>:/app# mkdir /storage-demo

# check that the directory is under the root volume
root@<POD_NAME>:/app# df -h /storage-demo
# output should be similar to the following:
Filesystem      Size  Used Avail Use% Mounted on
overlay         457G   22G  413G   5% /
{% endhighlight %}

You'll now see we have a directory created that sits on the root volume. Let's now create the new
mount point using the new volume. When applying the changes, Kubernetes will perform a rolling update
of the Pods (destroy old, create new) to apply the new volume:

{% highlight bash %}
$ kubectl apply -f app-deployment.yml

# inspect the new pods being created
$ kubectl get pods
{% endhighlight %}

Once the new Pods finish startup, inspect the deployment to see that the new Volume is present:

{% highlight bash %}
# note the mount in the deployment
$ kubectl describe deployments/randomizer

# inspect that the volume is consumed/bound
$ kubectl describe pvc app-pvclaim
{% endhighlight %}

From the `describe` command you should see reference to the `/storage-demo` mount and the Persistent
Volume Claim named "app-pvclaim" - your volume is now mounted and available to the Pods!

## Using the Persistent Volume

We'll now update our application to use the Persistent Volume and demonstrate both the shared storage aspect
across multiple Pods plus the persistence mechanism related to new Pods being able to consume the storage and
respective data within.

### Shared Storage Across Application Instances

Let's first update our `app/__init__.py` file to read/write from the volume. The application will create a
file in the directory if it does not exist. The app will then read the contents of the file and store the
contents to a variable, which should be the last host that accessed the file. It will then write its own
hostname to the file for the next request to parse. This could obviously create read/write contention for
an application of *any* concurrent request types, but is simple and will demonstrate the point of shared
storage. Let's update the file to reflect the following contents:

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

    # read contents of file, then write hostname to file
    last_host_filename = "/storage-demo/last_access.txt"
    last_host = ""
    if os.path.isfile(last_host_filename):
        last_host = open(last_host_filename, 'r').read()

    last_host_file = open(last_host_filename, 'w')
    last_host_file.write(socket.gethostname())
    last_host_file.close()

    return html.format(random_word=res[0][0], version="6.0", last_host=last_host, hostname=socket.gethostname())
```

We can now create a new Docker image (v6) with the updated application:

{% highlight bash %}
$ docker build -t randomizer:v6 .
{% endhighlight %}

Update your `app-deployment.yml` file to use the new "v6" image (property should read as `image: randomizer:v6`
in the file), and we're now ready to use our rolling deploy (zero downtime) skills to update the Deployment:

{% highlight bash %}
$ kubectl apply -f app-deployment.yml

# wait for new pods to come online
$ kubectl get pods
{% endhighlight %}

With any luck, you should now be able to visit the application web page and see the newly-versioned application
(version 6). If you refresh multiple times, you should see the message indicating "last access" updating to be
the last host that accessed the file, and if the current hostname (source) of the web page does not match the
last access hostname, it is indicative that each Pod is using the same shared storage/source file!

### Persistent Storage for New Application Instances

Since we now have a persistent storage volume, we'll terminate one of our Pods which should then result in the
controller spinning up a new one due to our replication factor being "3". This will allow us to inspect that the
storage mount not only shows up on the new Pod but also has the file with the same contents as the other Pods,
indicating that the state was preserved.

We will first inspect the file contents on one of the Pods, followed by terminating the Pod:

{% highlight bash %}
$ kubectl get pods
# copy the name of one of the pods

# open a shell to the pod
$ kubectl exec -it <POD_NAME> -- /bin/bash
root@<POD_NAME>:/app# ls -l /storage-demo
root@<POD_NAME>:/app# cat /storage-demo/last_access.txt
# take note of the contents of the file and exit
# the pod shell session
root@<POD_NAME>:/app# exit

# destroy the pod
$ kubectl delete pods/<POD_NAME>
{% endhighlight %}

At this point, the Pod having name "\<POD_NAME\>" should be terminating, and the replication factor should result
in the controller spinning up a brand new Pod in its place to maintain the replica status of 3 instances.
Let's inspect the new Pod and ensure that both the mount point exists and the file contents are the same:

{% highlight bash %}
$ kubectl get pods
# take note of the name of the *new* pod
$ kubectl exec -it <POD_NAME> -- /bin/bash
root@<POD_NAME>:/app# ls -l /storage-demo
root@<POD_NAME>:/app# cat /storage-demo/last_access.txt
{% endhighlight %}

If the mount point exists and the file contents match what was output in the first Pod before it was
terminated, your setup is complete and you now have a fully-functioning Persistent Volume for shared,
persistent storage!

## Next Steps

We will move forward with a more basic task focused on configuration strategy. In this [next post]({% post_url 2018-09-08-kubernetes-part-9-config-maps %})
the objective will be to define how to separate configuration from the application in a way that scales and is
organized.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Run a Single-Instance Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)
* [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

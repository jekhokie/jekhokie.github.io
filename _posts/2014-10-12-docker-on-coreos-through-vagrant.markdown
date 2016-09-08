---
layout: post
title:  "Docker on CoreOS using Vagrant"
date:   2014-10-12 19:12:18 -0400
categories: vagrant coreos docker container
---
Tutorial on how to configure and run a [Docker](https://www.docker.com/) container within a
host that is running the [CoreOS](https://coreos.com/) operating system. The tutorial uses
[Vagrant](https://www.vagrantup.com/) to launch a VM capable of deploying and running Docker
containers.

### Note

CentOS hosts are known to have filesystem/mapper issues when run on Vagrant and attempting to
launch a container through Docker. It's better to run the Linux host natively, or use something
else like CoreOS, which is why this tutorial was put together.

### Process

Install VirtualBox and Vagrant.

Install the CoreOS plugin for Vagrant.

Boot the CoreOS host via Vagrant (Docker should already be installed). After creating the VM, SSH
into the host using the native Vagrant commands:

{% highlight bash %}
$ vagrant ssh <COREOS_VM>
{% endhighlight %}

**Note**: All following commands will be run from within the CoreOS host.

Create a Dockerfile on the CoreOS VM containing the following:

{% highlight bash %}
# BASE EXAMPLE 1: BASE IMAGE INSTALL OF PUPPET USING EPEL REPO ON CENTOS 5
FROM centos:centos5

MAINTAINER Some Person "someperson@someplace.com"

RUN yum -y install wget && \
    wget https://yum.puppetlabs.com/RPM-GPG-KEY-puppetlabs && \
    rpm --import RPM-GPG-KEY-puppetlabs && \
    rpm -ivh http://yum.puppetlabs.com/puppetlabs-release-el-5.noarch.rpm && \
    yum -y install puppet
{% endhighlight %}

Create the Docker image (assuming you are in the same directory as the previously-created Dockerfile):

{% highlight bash %}
$ sudo docker build -t="someuser/centos5_puppet" .
{% endhighlight %}

Inspect the images repository to see that the newly-created image exists:

{% highlight bash %}
$ docker images
# should return the previously-created 'someuser/centos5_puppet' image
# copy the REPOSITORY value
{% endhighlight %}

(Optional) You can also test the image quickly to ensure that it functions as expected:

{% highlight bash %}
$ sudo docker run -i -t someuser/centos5_puppet /bin/bash
# this will launch the CentOS 5 image and present a login shell
{% endhighlight %}

Now that we have a fully-functioning base CentOS 5 image, create a directory structure for the
required/desired containers to be run and hosting the respective software:

{% highlight bash %}
$ mkdir ActiveMQ
$ mkdir PostgreSQL
$ mkdir ...
{% endhighlight %}

Next, create a `Dockerfile` in each previously-created software directory. This is an example
of one corresponding to the ActiveMQ container desired (file is `ActiveMQ/Dockerfile`). The
instructions in the Dockerfile assume that there is a directory `shared/` which contains the
configuration files and packages desired/to be copied to the container:

{% highlight bash %}
# EXAMPLE 1: INSTALL AND RUN ACTIVEMQ EXPOSED ON PORT 61616
FROM someuser/centos5_puppet:latest

MAINTAINER Some Person "someperson@someplace.com"

EXPOSE 61616

ADD shared/jboss-a-mq-6.0.0.redhat-024.zip jboss-a-mq-6.0.0.redhat-024.zip

RUN yum -y install unzip java && \
useradd activemq && \
mkdir -p /data/activemq/kahadb && \
mkdir -p /data/activemq/localhost/tmp_storage && \
chown -R activemq /data/activemq && \
mkdir -p /opt/apache/activemq/dtds/jboss-a-mq-6.0.0.redhat-024/ && \
unzip -d /opt/apache/activemq/ jboss-a-mq-6.0.0.redhat-024.zip && \
ln -s /opt/apache/activemq/jboss-a-mq-6.0.0.redhat-024 /opt/apache/activemq/current && \
echo -e "monitorRole abc123\ncontrolRole abc1234" > /opt/apache/activemq/current/etc/jmx.password && \
echo-e "monitorRole readonly\ncontrolRole readwrite" > /opt/apache/activemq/current/etc/jmx.access && \
chmod 600 /opt/apache/activemq/current/etc/jmx.* && \
mkdir /var/log/activemq && chown activemq /var/log/activemq

ADD shared/activemq.xml /opt/apache/activemq/current/etc/
ADD shared/karaf /opt/apache/activemq/current/bin/
ADD shared/setenv /opt/apache/activemq/current/bin/
ADD shared/org.apache.karaf.management.cfg /opt/apache/activemq/current/etc/
ADD shared/system.properties /opt/apache/activemq/current/etc/
ADD shared/jetty.xml /opt/apache/activemq/current/etc/
ADD shared/users.properties /opt/apache/activemq/current/etc/
ADD shared/org.apache.activemq.webconsole.cfg /opt/apache/activemq/current/etc/
ADD shared/org.ops4j.pax.logging.cfg /opt/apache/activemq/current/etc/
ADD shared/configure.dtd /opt/apache/activemq/dtds/jboss-a-mq-6.0.0.redhat-024/

RUN chown -R activemq /opt/apache
{% endhighlight %}

Next, we will use the Dockerfile to create a new image using the specified `ActiveMQ/Dockerfile`
file, which uses the base CentOS 5 image we created earlier:

{% highlight bash %}
$ cd ActiveMQ
$ docker build -t amq .
# the build step will take some time to complete - once done, verify the image is present
$ docker images
{% endhighlight %}

Now that the ActiveMQ image has been created, you can create as many containers using the image
as you would like. These are examples of different ways to launch a container using the `amq` image:

{% highlight bash %}
# create and log into using interactive shell
$ docker run -t -i --name amq amq /bin/bash

# create and daemonize
$ docker run -d --name amq amq
{% endhighlight %}

### Extra Commands

This is a list of useful commands to interact with Docker:

{% highlight bash %}
# show the running containers, including each corresponding CONTAINER_ID
$ docker ps

# stop a running container
$ docker stop <CONTAINER_ID>

# remove a stopped container
$ docker rm <CONTAINER_ID>

# list images, including each corresponding IMAGE_ID
$ docker images

# remove an image
$ docker rmi <IMAGE_ID>
{% endhighlight %}

---
layout: post
title:  "Jenkins - Docker Slaves Tutorial"
date:   2017-03-11 16:51:37 -0400
categories: ubuntu linux jenkins ci devops docker
---
Expanding on the [previous Jenkins tutorial]({% post_url 2017-03-11-jenkins-tutorial %}), this tutorial
will configure Jenkins to utilize Docker containers as build slaves - spinning up Docker containers
for builds, running the build, and then destroying the container following the build.

### Background

Often, people will configure virtual machines or physical machines as build slaves for a Jenkins
instance. While this is often necessary given the compute power, storage space, or RAM requirements
of jobs, for lighter applications that simply want to run tests and are not compiled, it can be
overkill. This tutorial will explore how to configure Jenkins to spin up Docker containers as build
slaves, run the "build" (in our case it will simply be test cases), and then destroy the container,
giving us much more flexibility in the overall approach to build instances. While this effort still
requires a virtual machine as a "host" for the build slave containers, it allows a bit of
multiplexing in our approach.

### Technology Ecosystem

This tutorial assumes that you have completed the
[previous Jenkins tutorial]({% post_url 2017-03-11-jenkins-tutorial %}) and have a functioning Jenkins
instance as well as source repository that contains Ruby code with an associated test case. We will
in addition be adding 1 virtual machine to the stack that will serve as the Docker container host, and
your ecosystem will result in the following (including previous infrastructure):

**Jenkins VM**

- **Hostname**: jenkins.localhost
- **OS**: Ubuntu 16.04
- **CPU**: 2
- **RAM**: 2048MB
- **Disk**: 50GB
- **Network**: Private Network
- **IP**: 10.11.13.15

**Docker VM**

- **Hostname**: dkr.localhost
- **OS**: Ubuntu 16.04
- **CPU**: 2
- **RAM**: 2048MB
- **Disk**: 50GB
- **Network**: Private Network
- **IP**: 10.11.13.16

In addition, we will assume the following software versions:

- **Jenkins**: 2.7.4
- **Ruby**: 2.3.1
- **RVM**: 1.29.1
- **Docker**: 17.03.0-ce, build 3a232c8

#### WARNING

As before, this tutorial is a vanilla/base setup and is not recommended for production use. In
addition, this is somewhat experimental in terms of the build capabilities of Docker containers
and, as such, your mileage may vary as it pertains to success in using this configuration.

### Notes

The code blocks included in this tutorial will list, as a comment, the node(s) that the commands
following need to be run on. For instance, if required on all nodes, the code will include a comment
like so:

{% highlight bash %}
# Node: jenkins.localhost, dkr.localhost
{% endhighlight %}

If the code block only applies to one of the nodes, it will list the specific node it applies to
like so:

{% highlight bash %}
# Node: dkr.localhost
{% endhighlight %}

All commands assume that your user is `vagrant` and the user's corresponding home directory is
`/home/vagrant` for the purposes of running sudo commands.

### Prerequisites

As previously stated, first, follow [this Jenkins tutorial]({% post_url 2017-03-11-jenkins-tutorial %})
and ensure you have a fully-functioning Jenkins instance, Ruby source sample repository, and have had
at least 1 successful build triggered.

### Installing Docker

Since we have Jenkins and the source repository configured, we will now focus on the Docker
installation. First, install Docker on the Docker host:

{% highlight bash %}
# Host: dkr.localhost
$ sudo apt-get install apt-transport-https \
                       ca-certificates \
                       curl \
                       software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
                          $(lsb_release -cs) \
                          stable"
$ sudo apt-get update
$ sudo apt-get install docker-ce
{% endhighlight %}

To test that Docker has been successfully installed, run the "Hello World" docker image:

{% highlight bash %}
# Host: dkr.localhost
$ sudo docker run hello-world
{% endhighlight %}

If you see "Hello World" with a corresponding message once the container is launched, your Docker
installation is now complete!

### Configure Docker

Now that the Docker service is installed, we need to configure it to allow remote API access.
This step is important to ensure that Jenkins can interact with the remote Docker service to launch
containers for building. Although the official documentation states that the file `/etc/default/docker`
should be updated to include the bind parameter for the API, I attempted several times to use the
specified `DOCKER_OPTS` variable and restart the Docker service without any success in exposing the
API on the specified port. As such, I followed the instructions
[here](https://www.ivankrizsan.se/2016/05/18/enabling-docker-remote-api-on-ubuntu-16-04/) which
essentially update the installed service directly:

{% highlight bash %}
# Host: dkr.localhost
$ sudo vim /lib/systemd/system/docker.service
# add the following to the end of the "ExecStart" variable:
#   -H tcp://0.0.0.0:4243
$ sudo systemctl daemon-reload
$ sudo service docker restart
{% endhighlight %}

Once you've restarted Docker, test that you can retrieve a list of images from the **Jenkins** host:

{% highlight bash %}
# Host: jenkins.localhost
$ curl http://10.11.13.16:4243/images/json
{% endhighlight %}

If successful, JSON will be returned containing (at a minimum if you ran the test command specified
following the installation of Docker) the "Hello World" image, indicating your exposure of the
REST API was successful.

### Create a Docker Image

In order for Docker to spin up a container to act as a build slave, the container needs to have the
appropriate developer tools, libraries, etc. The process we are going to perform next will construct
an image that Jenkins can trigger Docker to launch and will handle the build for the Ruby sample
application we developed in the previous tutorial. It is certainly possible to have the environment
set up through the build definition in Jenkins on every build, but this will slow down the build
cycle as it will require re-installing all developer packages, libraries, etc. on every build run.

**Pro Tip**: It is likely unreasonable to expect every team working with the build instance to register
a Docker image with the Jenkins systems administrators each and every time they wish to have a new
build environment set up. Instead, have a process by which your developers can automatically register
their images for builds - this self-service is much more streamlined and provides a better experience
for the engineers using the service.

On the Docker instance, we will construct a `Dockerfile` that will define our image:

{% highlight bash %}
# Host: dkr.localhost
$ mkdir ~/jenkins-slave
$ cd ~/jenkins-slave
$ vim Dockerfile
# ensure the file contains the following:
#   FROM ubuntu:16.04
#   RUN apt-get update && \
#       apt-get install -y software-properties-common python-software-properties sudo && \
#       add-apt-repository ppa:openjdk-r/ppa && \
#       apt-get update && \
#       apt-get install -y git openssh-server curl openjdk-7-jdk && \
#       mkdir -p /var/run/sshd && \
#       sed -i 's|session    required     pam_loginuid.so|session    optional     pam_loginuid.so|g' /etc/pam.d/sshd && \
#       useradd -m -d /home/jenkins -s /bin/bash jenkins && \
#       echo "jenkins:jenkins123" | chpasswd && \
#       apt-get install -y build-essential patch gawk g++ gcc make libc6-dev libgmp-dev \
#                          patch libreadline6-dev zlib1g-dev libssl-dev libyaml-dev \
#                          libsqlite3-dev sqlite3 autoconf libgdbm-dev libncurses5-dev \
#                          automake libtool bison pkg-config libffi-dev
#
#   USER jenkins
#   RUN /bin/bash -l -c "curl -sSL https://get.rvm.io | bash -s stable"
#   RUN echo 'source $HOME/.rvm/scripts/rvm' >> $HOME/.bashrc
#
#   USER root
#   EXPOSE 22
#   CMD ["/usr/sbin/sshd", "-D"]
{% endhighlight %}

The Dockerfile defined above sets up the build image as required:

1. Start with a base Ubuntu 16.04 image.
2. Install dependencies to add the OpenJDK repository since it is not included by default in Ubuntu 16.04.
3. Update the OS repository definitions, and install git, SSH server, curl, and Java JDK.
4. Create a directory for the SSH daemon PID file.
5. Configure the PAM settings to allow SSH login.
6. Create a jenkins user with a password 'jenkins123' belonging to group RVM.
7. Install RVM dependencies.
8. Install RVM.
9. Expose port 22 (default SSH port).
10. Start the SSH daemon.

**Note**: The instructions for RVM are accurate at the time of this post and you should reference
the official RVM documents for the exact commands. The list of dependencies for RVM is quite long and
likely to change as time goes on. If you wish to see a list of dependencies for RVM, add `rvm autolibs
fail` right before the `rvm install ruby-2.3.1` command and it will automatically output a list of
dependent packages that need to be installed - add these to the `apt-get install` command above and
you should be good to continue.

We will now use this Dockerfile to create our base build image:

{% highlight bash %}
# Host: dkr.localhost
$ cd ~/jenkins-slave
$ sudo docker build -t jenkins-slave .
{% endhighlight %}

At this point, you should see some output corresponding to downloading the base image followed by
the layers of your current image being built. Once the Docker image has been successfully built,
you can check that it registered correctly like so:

{% highlight bash %}
# Host: dkr.localhost
$ sudo docker image list
# should output something that contains the 'jenkins-slave' image:
#   REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
#   jenkins-slave       latest              618d98662e18        27 seconds ago      317 MB
#   ...
{% endhighlight %}

Now we will start a container using this base image to test our configuration and make sure that
the `jenkins` user can log in (required for Jenkins to be able to launch and run a build on a
Docker slave container):

{% highlight bash %}
# Host: dkr.localhost
$ sudo docker run -d -P --name test-jenkins-slave jenkins-slave
$ sudo docker port test-jenkins-slave 22
# should output something like the following - note the port number:
#   0.0.0.0:32770
$ ssh jenkins@localhost -p 32770
# enter the jenkins user password when prompted - if using the above Dockerfile,
# the password 'jenkins123' should work.
# once you are in, you might want to try "rvm --version" to ensure the jenkins
# user has the rvm command it its path
{% endhighlight %}

If you were able to log into the container above, your build image is configured correctly for
Jenkins to utilize. You can now exit and clean up the running container:

{% highlight bash %}
# Host: dkr.localhost
$ sudo docker container list --all
# get the ID of the container listed as utilizing 'jenkins-slave' created most recently
$ sudo docker container rm <CONTAINER_ID>
{% endhighlight %}

### Add Container as Jenkins Build Slave

Now that we have a functioning Docker image, we can configure Jenkins to utilize the image for
build slave instances when requested. Log into the Jenkins service as the administrator and
select "Manage Jenkins" from the left menu. Navigate to the "Manage Plugins" option, and under
the "Available" tab, search for the "Docker plugin" plugin under "Cloud Providers" - select it and
click on "Install without restart".

Once the plugin has been installed, navigate back to the home screen and again select "Manage Jenkins"
from the left menu. Then, click on "Configure system". Navigate to the bottom under "Cloud" and select
"Docker" from the "Add a new cloud" drop-down. When the form populates, fill in the fields as follows:

- **Name**: Test Jenkins Containers
- **Docker URL**: tcp://10.11.13.16:4243
- **Docker API Version**: (Leave Blank)
- **Credentials**: (Leave Blank)
- **Connection Timeout**: 5
- **Read Timeout**: 15
- **Container Cap**: 100

Once you have filled out the above, click "Test Connection" to ensure that Jenkins is able to reach
and communicate with the Docker API. If all goes well, you will be presented with a "Version" and
"API Version" parameters on your screen. Next, select the "Add Docker Template" drop-down and click
"Docker Template". When presented with the next form, fill in the fields as follows:

- **Docker Image**: jenkins-slave
- **Instance Capacity**: 2
- **Remote Filing System Root**: /home/jenkins
- **Labels**: container-slave
- **Usage**: Only build jobs with label expressions matching this node
- **Launch Method**: Docker SSH computer launcher
- **Credentials**: (Enter "jenkins" and "jenkins123" if you followed this tutorial exactly)
- **Remote FS Root Mapping**: /var/lib/jenkins
- **Remove Volumes**: (Leave Un-Checked)
- **Pull Strategy**: Pull once and update latest

Once the above are filled out, click the "Save" button near the bottom of the screen - your Docker
configuration is now complete!

### Use Container as Slave for Ruby Job

Now that there is a Docker template definition, we can re-configure our existing Ruby job to utilize
the container service. Navigate to the Jenkins UI and click the drop-down arrow next to the "test-job"
project, and select "Configure". When brought to the configuration page, place a check mark next to the
option "Restrict where this project can be run" and specify "container-slave" for the "Label
Expression" parameter. This tells the job to utilize the existing container template as the build
slave to run the job on. Next, prior to clicking "Save" in the job configuration, navigate down to the
"Build" section. Update the "Command" parameter to list "rvm install ruby-2.3.1" as the first command
to run after the shebang in the script - your complete build command should now look like this:

{% highlight bash %}
#!/bin/bash

rvm install ruby-2.3.1
gem install bundler
bundle install
rspec spec
{% endhighlight %}

Once you have updated the "Command" property you can click "Save". When brought back to the job
screen, click "Build Now" and wait for the build to complete. You may be surprised to see that the
build will likely fail - what happened? If you click the build and then click "Console Output" in
the left menu, you'll likely notice in the output a line similar to the following:

```
stderr: fatal: '/home/vagrant/test-repo' does not appear to be a git repository
```

If you recall, we initially set up Jenkins using a local git file system. This will obviously not
work as the slave itself does not have the git content on it by default, so we will need to host the
code and serve it via some method for the slave instance to retrieve it.

### Set up Apache to Serve Git

In order for our build slaves to be able to obtain the git code, we need to set up a simple server
that will serve the content for the slaves to retrieve. We will use Apache for this, and since the
code is on the Jenkins master instance, we will install Apache on that host:

{% highlight bash %}
# Host: jenkins.localhost
$ sudo apt-get install apache2
$ cd /var/www/html
$ sudo git clone --bare /home/vagrant/test-repo test-repo.git
$ cd test-repo.git
$ sudo git --bare update-server-info
{% endhighlight %}

The above commands set up a directory structure, permissions, etc. that allow anyone to clone the
git repository through the Apache interface. Now, we will need to update the Jenkins build
configuration to utilize the Apache server endpoint. Navigate to the Jenkins UI, click the "test-job"
project drop-down, and select "Configure". Under "Source Code Management" update the "Repository URL"
property to reflect "tcp://10.11.13.15/test-repo.git", then click "Save".

At this point, you should be brought back to the "test-job" project. Kick off another build via the
"Build Now" menu item in the left hand menu. Wait for the build to complete and, once complete,
inspect the results. The build should have completed successfully, indicating your Docker container
setup for Jenkins build slaves is successful!

### Summary

This tutorial has posed ways to utilize Docker containers as Jenkins build slaves. There are tangible
benefits to doing this (no need to keep single build slaves around indefinitely, small footprint,
etc.). However, there are limitations to this implementation and this tutorial is by no means a
complete and comprehensive setup for utilizing Docker containers as build slaves. There are many
security implications to consider in addition to architectural efficiencies missing from this setup
that are required for a production-level CI/CD pipeline using Jenkins with containers. However, the
tutorial serves as a starting point for exploring how one might consider their build server environment
in a slightly different and non-conventional light.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Jenkins Documentation](https://jenkins.io/doc/)
* [Get Docker for Ubuntu](https://docs.docker.com/engine/installation/linux/ubuntu)
* [Enabling Docker Remote API on Ubuntu 16.04](https://www.ivankrizsan.se/2016/05/18/enabling-docker-remote-api-on-ubuntu-16-04/)
* [Docker Containers as Build Slaves in Jenkins](http://devopscube.com/docker-containers-as-build-slaves-jenkins/)

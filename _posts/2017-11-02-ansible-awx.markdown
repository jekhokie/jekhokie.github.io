---
layout: post
title:  "Ansible AWX"
#date:   2017-11-02 20:24:13 -0400
categories: ubuntu linux ansible devops awx
---
Installation and configuration of the [Ansible AWX](https://github.com/ansible/awx) product, the open-source
version of the RedHat Enterprise [Ansible Tower](https://www.ansible.com/tower) product.

### Background

This post is a simple explanation of how to install and configure the Ansible AWX platform on a VirtualBox
VM. The Ansible AWX software is installed and configured to run on a sequence of Docker containers running
on a CentOS 7 VM.

### Technology Ecosystem

There is 1 host for this tutorial. As a warning, make *sure* you follow the specs for the host as minimum
requirements. If you do not allocate at least 4GB of memory and 2x vCPUs, your AWX web UI will likely be
quite laggy and un-usable. Additionally, at the time of this post, you will need to use a CentOS 7 instance
in order for this project to be successful.

**Ansible AWX VM**

- **Hostname**: awx.localhost
- **OS**: CentOS 7
- **CPU**: 2
- **RAM**: 4096MB
- **Disk**: 20GB
- **Network**: Private Network
- **IP**: 10.11.13.15

In addition, we will assume the following software versions:

- **Ansible AWX**: 'devel' branch
- **Docker**: 17.09.0-ce, build afdb6d4

### Prerequisites

First, perform the prerequisite installations:

{% highlight bash %}
# Host: awx.localhost
# install dependencies
$ sudo yum -y update
$ sudo yum -y install ansible git python-pip
$ sudo yum -y install yum-utils device-mapper-persistent-data lvm2
{% endhighlight %}

Next, install the Docker service and verify the Docker service is running as expected:

{% highlight bash %}
# Host: awx.localhost
# install docker
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum -y install docker-ce
$ sudo systemctl start docker

# verify that docker is running
$ sudo docker run hello-world
# should output something similar to the following:
#   Hello from Docker!
#   This message shows that your installation appears to be working correctly.
#   ...

# install the Docker Python dependencies
$ sudo pip -y install docker
{% endhighlight %}

### Installing AWX

Next, we will actually install and configure AWX using Docker containers via the installer.

{% highlight bash %}
# Host: awx.localhost
# clone the repository and install
$ git clone https://github.com/ansible/awx.git
$ cd awx/installer/
$ sudo ansible-playbook -i inventory install.yml
{% endhighlight %}

At this point, the installer will use Ansible locally to kick off Docker containers and configure
the AWX service. To verify that AWS and its dependencies were installed into Docker containers, use
the Docker `ps` command to verify there are 5 containers having names corresponding to each of the
required services.

{% highlight bash %}
# Host: awx.localhost
$ sudo docker ps
# should output the following:
#   CONTAINER ID  IMAGE                   COMMAND                 CREATED             STATUS            PORTS                                NAMES
#   8104bf3f940c  ansible/awx_task:latest "/tini -- /bin/sh ..."  53 seconds ago      Up 50 seconds     8052/tcp                             awx_task
#   cdf818a2fffb  ansible/awx_web:latest  "/tini -- /bin/sh ..."  About a minute ago  Up About a minute 0.0.0.0:80->8052/tcp                 awx_web
#   b9bb8295e643  memcached:alpine        "docker-entrypoint..."  3 minutes ago       Up 3 minutes      11211/tcp                            memcached
#   eccbe7efd48d  rabbitmq:3              "docker-entrypoint..."  3 minutes ago       Up 3 minutes      4369/tcp, 5671-5672/tcp, 25672/tcp   rabbitmq
#   a9fc7ea3b3cb  postgres:9.6            "docker-entrypoint..."  4 minutes ago       Up 4 minutes      5432/tcp                             postgres
{% endhighlight %}

At this point, your installer is likely complete locally, but there is database bootstrapping
occurring. You can watch the database output progress by running the following docker command.
Note that it takes several minutes for any progress to appear to be made, as well as several
minutes in general to get to the point of finishing.

{% highlight bash %}
# Host: awx.localhost
$ sudo docker logs -f awx_task
# once you see something like the following, you're all set (this takes several minutes to complete):
#   ...
#   >>> <User: admin>
#   >>> Default organization added.
#   Demo Credential, Inventory, and Job Template added.
#   Successfully registered instance awx
#   (changed: True)
#   Creating instance group tower
#   Added instance awx to tower
#   (changed: True)
#   ...
{% endhighlight %}

Once the above is complete, you can visit the AWX interface via navigating to the following URL
in your browser (again, assuming the IP address of the VM this software was installed on is
10.11.13.15):

http://10.11.13.15/

The default login username is 'admin', and corresponding password is 'password'.

### Stopping/Starting AWX

There isn't great documentation on how to stop/start the AWX instances other than brute-force
stopping/starting the Docker service. In addition, I did run into an issue where one of my
containers was lost/shut down by accident. In this case, it's best to leverage the idempotency
of the Ansible framework and re-use the installer to re-create any missing Docker containers
for you. More specifically, if you lose any of your containers, re-run the installer command:

{% highlight bash %}
# Host: awx.localhost
$ cd awx/installer/
$ sudo ansible-playbook -i inventory install.yml
{% endhighlight %}

As the playbook is run, you'll notice a lot of "no changes" type of activity, but it should
output changes corresponding to the start of any missing containers. At the end of the run,
you should be up and going again.

Also, if you restart your VM at all and don't have Docker configured to start at boot, simply
start the Docker service once the VM has booted and your AWX containers should come back up
nicely (again, if not, just re-run the installer command above).

### Troubleshooting

After installing I did run into an issue when attempting to configure SSH keys and run jobs on
remote hosts from the Ansible AWX instance. The specific bug I encountered is documented
[here](https://github.com/ansible/awx/issues/130). To work around this issue, I followed the
recommended course of action in the issue, but to be explicit, the following are the steps I took.

The error message I received when attempting to run jobs on remote hosts was the following:

```
fatal: Unreachable! => failed to connect to the host via ssh: Control Socket Connect:
Connect Refused. Failed to connect to new control master.
```

To correct the issue, you need to connect to the Docker container labeled "ansible/aws_task:latest"
and update the `ansible.cfg` file to include a ControlPath specification.

{% highlight bash %}
# Host: awx.localhost
$ docker ps
$ docker exec -it <DOCKER_TASK_CONTAINER_ID> sh
$ vi /etc/ansible/ansible.cfg
# uncomment the "ssh_args" line and add the ControlPath=/dev/shm/cp%h-%p-%r part:
#   ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s -o ControlPath=/dev/shm/cp%h-%p-%r
{% endhighlight %}

For good measure, I restarted the containers using the Docker service restart command:

{% highlight bash %}
# Host: awx.localhost
$ sudo systemctl restart docker
{% endhighlight %}

Following the restart, I was able to successfully run commands on remote instances.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Ansible AWX](https://github.com/ansible/awx)
* [Ansible Tower](https://www.ansible.com/tower)
* [Ansible Tower Documentation](http://docs.ansible.com/ansible-tower/)
* [Docker](https://www.docker.com/)
* [Docker Documentation](https://docs.docker.com/)

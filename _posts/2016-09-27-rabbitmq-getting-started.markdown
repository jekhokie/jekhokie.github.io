---
layout: post
title:  "RabbitMQ Getting Started"
date:   2016-09-27 18:53:11 -0400
categories: rabbitmq ha clustering installation configuration
---
Tutorial related to installing and configuring [RabbitMQ](https://www.rabbitmq.com/), including
clustering, high-availability, and testing.

### Background

This tutorial includes steps to install, configure, cluster, and test a
[RabbitMQ](https://www.rabbitmq.com/) messaging service on a Linux system. The steps involved were
performed on an Ubuntu 16.04 instance. Note that the steps are very much manual and I would always recommend
using a software management solution such as Chef or Puppet, but for the purposes of understanding some
aspects of RabbitMQ configuration/operation, manual steps are an important first-step to automating.

Note that for clustering, a minimum of 3 nodes is not only recommended but most likely required in order to
ensure high-availability. See the section near the bottom of this post related to testing for details about
why less than 3 node clusters are generally a bad idea/will introduce failures that will keep your administrators
up all night.

### Installation

The installation instructions for RabbitMQ can be found [here](https://www.rabbitmq.com/admin-guide.html).
The steps below are published from the guide in a consolidated fashion to help identify quick installation
and configuration of the service, as well as my understanding of the capabilities through testing and
exploration.

Note that the steps below were performed on an Ubuntu 16.04 instance with specific versions of each package
in order to reduce complexities related to dependencies. In addition, in order to achieve high availability
and clustering, the steps below for "Installation" must be performed on all N instances where you wish to
have RabbitMQ running.

#### Software Installation

***The following steps are required for all RabbitMQ instances that you wish to cluster.***

There are a couple dependencies involved with installing and configuring a RabbitMQ service. Specifically,
Erlang and socat are required for RabbitMQ to work correctly:

{% highlight bash %}
# install the erlang requirements
$ sudo wget http://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb
$ sudo dpkg -i erlang-solutions_1.0_all.deb
$ sudo apt-get update
$ sudo apt-get -y install erlang-nox=1:18.2

# install the socat requirement
$ sudo apt-get install socat
{% endhighlight %}

Once the required dependencies are installed, the RabbitMQ server can be installed on the instance:

{% highlight bash %}
# install rabbitmq
$ sudo wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.5/rabbitmq-server_3.6.5-1_all.deb
$ sudo dpkg -i rabbitmq-server_3.6.5-1_all.deb
{% endhighlight %}

#### Software Configuration - Clustering

***The following steps are required for all RabbitMQ instances that you wish to cluster.***

In order to cluster multiple RabbitMQ instances to operate as a single unit, there are several configuration
steps followed by service adjustments necessary. First, specify the network partition handling that should
be used for each instance (more information can be found [here](https://www.rabbitmq.com/partitions.html)):

{% highlight bash %}
$ sudo vim /etc/rabbitmq/rabbitmq.config
# ensure the following configuration exists, among any other configurations that are already present
#   [
#       {rabbit, [
#           {cluster_partition_handling, pause_minority}
#       ]}
#   ].
{% endhighlight %}

Next set a consistent Erlang cookie on each host - this cookie must be exactly the same across all cluster
instances in order to ensure successful communication in cluster mode:

{% highlight bash %}
$ sudo vim /var/lib/rabbitmq/.erlang.cookie
# insert something crafty here - "mysupersecretcookie" or the like
{% endhighlight %}

On all cluster instances, run the following to prepare for clustering:

{% highlight bash %}
# restart the rabbitmq service to read new conifguration data
$ sudo rabbitmq-server restart
# stop the application and reset it to prepare for clustering
$ sudo rabbitmqctl stop_app
$ sudo rabbitmqctl reset
{% endhighlight %}

Before proceeding, ensure that all RabbitMQ instance nodes can resolve the ***hostname*** of all other nodes
that you wish to cluster. If no DNS solution is in place that provides resolution, simply editing the
`/etc/hosts` file (although a nightmare to manage) to include hostnames will suffice for this tutorial. Failure
to address the hostname resolution issue could result in this type of failure message when attempting to
cluster the instances:

{% highlight bash %}
#   TCP connection succeeded but Erlang distribution failed
#   ...
#     * suggestion: hostname mismatch?
#   ...
{% endhighlight %}

***On a single RabbitMQ instance/leader - node 1***

Perform the following step on a single RabbitMQ instance only - this will ensure that all other instances
can establish a connection with/join the cluster:

{% highlight bash %}
$ sudo rabbitmqctl start_app
{% endhighlight %}

***On all other RabbitMQ instances except the leader - node(s) 1+n***

On all remaining RabbitMQ instances (***not*** the single/leader instance previously specified), 

{% highlight bash %}
$ sudo rabbitmqctl join_cluster rabbit@<FIRST_INSTANCE_HOSTNAME>
{% endhighlight %}

Note that for persistence, the above command could be replaced with editing and adding the cluster nodes
into the `rabbitmq.config` configuration file instead under the `rabbit` key (would require restart of
the RabbitMQ service for configuration directive to take effect):

{% highlight bash %}
#   ...
#   {cluster_nodes, ['<INSTANCE_1_HOSTNAME>', '<INSTANCE_2_HOSTNAME>', ...]},
#   ...
{% endhighlight %}

Once the cluster command has been run, start the application so that the node becomes an active member of
the cluster:

{% highlight bash %}
$ sudo rabbitmqctl start_app
{% endhighlight %}

Once the above have been performed on each node, you can run the following command from any node in the cluster
to verify that the cluster has been successfully formed:

{% highlight bash %}
$ sudo rabbitmqctl cluster_status
# should output something similar to:
#  Cluster status of node 'rabbit@ip-10-11-13-11' ...
#    [{nodes,[{disc,['rabbit@ip-10-11-13-10','rabbit@ip-10-11-13-11','rabbit@ip-10-11-13-12']}]},
#    {running_nodes,['rabbit@ip-10-11-13-10','rabbit@ip-10-11-13-11','rabbit@ip-10-11-13-12']},
#    {cluster_name,<<"rabbit@localhost">>},
#    {partitions,[]}]
{% endhighlight %}

#### Virtual Host, User, Permissions

Now that you have a fully-functioning cluster, you can configure a virtual host, user, and permissions to start
utilizing the cluster. It is recommended that these steps be performed prior to use in order to ensure security
of the data you are sending to/from the cluster (rather than using the "guest" account that is configured by
default).

Adding a virtual host is simple and defines a 'folder'-like hierarchy for queues to be created in. Users are
defined and granted permissions to a virtual host:

{% highlight bash %}
# add a virtual host
$ sudo rabbitmqctl add_vhost /testing

# add a user and grant all permissions to operate within the virtual host
$ sudo rabbitmqctl add_user test_user superTestUserPassword
$ sudo rabbitmqctl set_permissions -p /testing test_user ".*" ".*" ".*"
{% endhighlight %}

#### High Availability

By default, certain aspects of the RabbitMQ service are replicated across cluster nodes. However, queues are
not by default replicated and, therefore, failure of the source node which holds queue data will result in a
loss of information. In order to avoid this, you can configure high availability of queues via the following
command (this can, again, also be configured in the `rabbitmq.config` configuration file for persistence).
The following command will enable HA for the queue named `<QUEUE_NAME>` in the command under the `/testing`
virtual host created in earlier steps:

{% highlight bash %}
$ sudo rabbitmqctl set_policy ha-testing "^(<QUEUE_NAME>$)" '{"ha-mode":"all", "ha-sync-mode":"automatic"}' -p /testing
{% endhighlight %}

### Monitoring/Metrics

RabbitMQ provides a [management plugin](https://www.rabbitmq.com/management.html) which can be used for
various monitoring and administrative capabilities. I'm including steps here to install/configure the plugin
based on it being helpful/useful for troubleshooting and investigating overall performance of the RabbitMQ
cluster. In addition, there are several Collectd RabbitMQ plugins available for sending the same type of
metrics/data from a RabbitMQ cluster/instance to a Graphite or other endpoint that may prove to be more
useful and/or useful in conjunction with the management plugin.

To install the management plugin, perform the following command on one of the nodes. Note that you will likely
want to install and configure the plugin in such a way that high availability is also provided (as in this
configuration, if the node on which the management plugin is installed goes down, the management API will
be unavailable), but the process for this is out of scope for this article:

{% highlight bash %}
# install/enable the management plugin
$ sudo rabbitmq-plugins enable rabbitmq_management
{% endhighlight %}

By default, there is a 'guest' user defined that is only able to access the web interface via the loopback
adapter (locally on the RabbitMQ instance). This is likely insufficient for 2 reasons - 1, the guest account
is well-known and anyone could access your cluster if they knew the IP/endpoint and had SSH access, and 2,
you will likely wish to access the web interface from you local machine, which is typically different than
the RabbitMQ instance.

It is likely a better idea to configure a separate user and disable the guest account. However, if you wish
to access the web interface using the guest account credentials (user: `guest`, password: `guest`), you can
clear the `loopback_users` configuration in the `rabbitmq.config` so that the `guest` user is able to
log into the web interface from a remote machine:

{% highlight bash %}
$ sudo vim /etc/rabbitmq/rabbitmq.config
# add the following:
#   [
#     [{rabbit, [
#       ...
#       {loopback_users, []}
#       ...
#     ]}
#   ].
# restart the rabbitmq-server instance for new configuration to take effect
$ sudo rabbitmq-server restart
{% endhighlight %}

Once the above has been performed, you can access the management web interface via the following URL and
log in via the guest account (`guest:guest`):

* http://<INSTANCE_HOSTNAME>:15672

From the interface, you can add the ability for users to log in/use the management interface under the
"Admin" tab, and creating/selecting a user. To grant access to the interface, add a tag "management" to
the user account, which will grant them the role they need to access the interface.

As a note, statistics are collected every 5 seconds by default. You can tune this interval (i.e. in the
case of wishing to manage the memory footprint of the service) by placing the `collect_statistics_interval`
parameter in the `rabbitmq.conf` file and restarting the RabbitMQ service (parameter is defined in
milliseconds).

If you wish to clear the statistics database (since everything is stored in memory) for instance, in the
case of testing, you can run the following command on the node which is currently running the statistics
database:

{% highlight bash %}
$ sudo rabbitmqctl eval 'supervisor2:terminate_child(rabbit_mgmt_sup_sup, rabbit_mgmt_sup), rabbit_mgmt_sup_sup:start_child().'
{% endhighlight %}

### Testing

In order to test your cluster, you can develop a publisher and/or consumer in many various languages that
have libraries for AMQP interaction (the protocol used by RabbitMQ). For the purposes of this post, you can
also utilize some demonstration scripts developed in my personal repository `scriptbox` located
[here](https://github.com/jekhokie/scriptbox.git). The folder `ruby--rabbitmq-producer-consumer` contains a
README and scripts which allow interaction with a RabbitMQ instance/cluster, and the instructions for
working with the scripts are in the repository so I won't touch on any of that here.

One note to call out is related to clustering. When testing, I found that having an "HA" cluster of 2 nodes
is not sufficient and just plain did not work when failures occurred on a single node. If clustered with 2
nodes, taking down one node resulted in the secondary node completely rejecting any requests to communicate
(send or receive information to/from queues). I wasn't able to ascertain whether this was an environmental
configuration associated with my setup or one of the many unique failure modes listed in the documentation
for clusters of less than 3 nodes in size, but the point is that the documentation does not recommend cluster
sizes less than 3 so further investigation was not warranted (I will simply use 3 as the minimum cluster
size for any implementation moving forward).

### Final Notes - Production Use

There are several additional steps involved with configuring your RabbitMQ cluster for production use. Some
of the details were left out of this post for brevity, but can be found in the official RabbitMQ
documentation/site listed at the beginning of this article.

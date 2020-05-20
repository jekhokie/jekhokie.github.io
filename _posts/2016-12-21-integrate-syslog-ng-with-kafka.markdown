---
layout: post
title:  "Integrate syslog-ng with Kafka"
date:   2016-12-21 19:14:01 -0400
categories: syslog syslog-ng kafka logging linux
logo: kafka.jpg
---
Tutorial on how to integrate [syslog-ng](https://syslog-ng.org/) with [Apache Kafka](https://kafka.apache.org/)
to allow forward and store of syslog logging for future stream and event processing of the logs.

### Background

syslog-ng is a lightweight logging agent that is based on the classic syslog framework. In this tutorial,
we will demonstrate how to configure syslog-ng as a relay to receive logging and events from any syslog-ng
client agent and forward those events and logs to a downstream Kafka instance. The benefit in doing this
allows for a building block in larger-scale data processing pipelines since Kafka (and similar technologies)
is often used as a queueing system for data ingest into the "big data" processing stream.

### Underlying Compute Technology/Ecosystem

The following assumptions are made about your infrastructure. Although the instructions can be extended
to accommodate cloud-based and/or other infrastructure environments quite easily, all details are
explained with respect to local development for the purposes of simplicity:

- **Hypervisor Technology**: VirtualBox
- **Provisioner**: Vagrant
- **Number of VMs**: 2
- **Operating System**: Ubuntu 16.04
- **Arch**: 64-bit
- **CPUs**: 1
- **Mem**: 2GB
- **Disk**: 20GB

This tutorial is built with 2 virtual machines - one of the resources will be the Apache Kafka/Apache
ZooKeeper instance while the other will be a "remote" syslog-ng instance for log forwarding. In addition,
a syslog-ng agent (client) will be installed on the same node as the Apache Kafka instance to demonstrate
and test the interaction of the architecture. The following hostnames and corresponding IP addresses
will be used for the 2 virtual machine instances:

- **node1.localhost**: 10.11.13.15
- **node2.localhost**: 10.11.13.16

The respective versions of software used in this tutorial are as follows. Other versions **may** work
using these instructions but, as usual, your mileage may vary:

- **syslog-ng**: 3.7.3
- **Apache ZooKeeper**: 3.4.6-1569965
- **Apache Kafka**: 2.11-0.10.0.1

Finally, the code blocks included in this tutorial will list, as a comment, the node(s) that the commands
following need to be run on. For instance, if required on both nodes, the code will include a comment like
so:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
{% endhighlight %}

If the code block only applies to one of the nodes, it will list the specific node it applies to
like so:

{% highlight bash %}
# Node: node2.localhost
{% endhighlight %}

All commands assume that your user is `vagrant` and the user's corresponding home directory is
`/home/vagrant` for the purposes of running sudo commands.

### End State Architecture

At the end of this tutorial, the following end-state architecture will be realized (where "Server"
is the remote instance of syslog-ng where the Client will forward logs to). Obviously in an ideal
scenario the Kafka/ZooKeeper instance would be a cluster on its own hosts, but for simplicity and
resource sake, we will install it on the client node as though it were its own standalone instance:

{% highlight bash %}
|---------------------|               |---------------------|
|    Client/Kafka     |               |       Server        |
|   node1.localhost   |               |   node2.localhost   |
|    (10.11.13.15)    |               |    (10.11.13.16)    |
|                     |     Log       |                     |
|     |-----------|   |  Forwarding   |     |-----------|   |
|     | syslog-ng |------------------------>|           |   |
|     |-----------|   |               |     |           |   |
|                     |               |     | syslog-ng |   |
|     |-----------|   |               |     |           |   |
|     | Kafka/ZK  |<------------------------|           |   |
|     |-----------|   |     Log       |     |-----------|   |
|---------------------|  Forwarding   |---------------------|
{% endhighlight %}

### Installing Kafka

To install ZooKeeper and Kafka on the host "node1.localhost" (IP 10.11.13.15), please follow the
instructions in the previous post [Installing Kafka]({% post_url 2016-08-23-installing-kafka %}).

### Installing syslog-ng

Installing syslog-ng is based on the previous tutorial
[syslog-ng Tutorial]({% post_url 2016-12-20-syslog-ng-tutorial %}). However, this section will focus
on installing a more recent version of the syslog-ng service that is compatible with forwarding logs
to Kafka (versions >=3.7), and will **exclude** the SSL/security components that were discussed in the
previous post for brevity sake.

**Note**: Most commands in this section are applicable to both nodes - the installation is fairly
straightforward and can be done in parallel on both nodes. Note that there is no configuration
occurring in these steps - configuration is performed at a later step in this tutorial.

#### Prerequisites

Since we will be compiling the syslog-ng software from source, several dependencies are required in
order to ensure a successful compilation and installation:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo apt-get update
$ sudo apt-get install default-jre \
                       default-jdk \
                       gradle \
                       gcc \
                       flex \
                       bison \
                       pkg-config \
                       libglib2.0 \
                       openssl \
                       libssl-dev \
                       python-dev \
                       libjson0-dev
{% endhighlight %}

Once the above dependencies have been installed, compile and install a required version of the
eventlog software:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ cd ~
$ wget https://my.balabit.com/downloads/eventlog/0.2/eventlog_0.2.12.tar.gz
$ tar -xzf eventlog_0.2.12.tar.gz
$ cd eventlog_0.2.12/
$ PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH
$ sudo ./configure
$ sudo make
$ sudo make install
{% endhighlight %}

Finally, since we are going to be forwarding logs to the Kafka instance, we need to place the relative
Kafka libraries on the system somewhere that syslog-ng can be configured to access them. Note that this
step is **ONLY** required for the "node2.localhost" (server) instance as the libraries have already been
installed on "node1.localhost" (the Kafka/ZooKeeper instance):

{% highlight bash %}
# Node: node2.localhost
$ cd ~
$ wget http://apache.claz.org/kafka/0.10.0.1/kafka_2.11-0.10.0.1.tgz
$ sudo tar -xzf kafka_2.11-0.10.0.1.tgz -C /opt/
{% endhighlight %}

#### Software Install

Next we will install the syslog-ng version 3.7.3 software and libraries. Note that this package version
is not available via the native package management system of the operating system at this time and, as
such, the software will be installed from source:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ cd ~
$ wget https://github.com/balabit/syslog-ng/releases/download/syslog-ng-3.7.3/syslog-ng-3.7.3.tar.gz
$ sudo tar -xzvf syslog-ng-3.7.3.tar.gz
$ cd syslog-ng-3.7.3/
$ export LD_LIBRARY_PATH=/usr/lib/jvm/default-java/jre/lib/amd64/server:$LD_LIBRARY_PATH
$ sudo ./configure --enable-java \
                   --enable-java-modules \
                   --enable-json \
                   --with-module-dir=/opt/syslog-ng/modules \
                   --prefix=/opt/syslog-ng
$ sudo make
$ sudo make install
{% endhighlight %}

### Configuration

Now that syslog-ng is installed and the libraries for Kafka are in place, we can configure the
syslog-ng client to send logs to the server, and then further configure the syslog-ng server to
forward the logs to the Kafka instance.

#### Server - Receive and Forward

First, let's set up the server instance to accept connections and forward log data to the Kafka
endpoint. We need to ensure that the Kafka instance is reachable via hostname since the original
Kafka tutorial utilizes the canonical hostname to bind to the respective port - this method is
used in favor of proper DNS records due to simplicity:

{% highlight bash %}
# Node: node2.localhost
$ sudo vim /etc/hosts
# ensure the following line is present somewhere in the file
#   10.11.13.15   node1.localhost   node1
{% endhighlight %}

Once we've made the Kafka host name resolution possible, we can update the syslog-ng configurations
to receive and forward logs to the Kafka endpoint:

{% highlight bash %}
# Node: node2.localhost
$ cd /opt/syslog-ng/etc/

$ sudo vim syslog-ng.conf
# add the following to the very end of the file:
#   @include "/opt/syslog-ng/etc/conf.d/*.conf"

$ sudo mkdir conf.d/
$ sudo vim conf.d/myapp.conf
# ensure contains the following - WARNING - "client_lib_dir" and "kafka_bootstrap_servers"
# MUST contain underscores and not dashes '-' as the tutorial attempts to detail:
#   @module mod-java
#
#   destination myapp_kafka {
#     kafka(
#       client_lib_dir("/opt/syslog-ng/modules/java-modules/:/opt/kafka_2.11-0.10.0.1/libs/")
#       kafka_bootstrap_servers("node1.localhost:9092")
#       topic("${HOST}")
#     );
#   };
#   source myapp_network {
#     tcp(ip(0.0.0.0) port(514));
#   };
#   log {
#     source(myapp_network);
#     destination(myapp_kafka);
#   };
{% endhighlight %}

Finally, we can start the syslog-ng service using the configurations created previously - this step
starts the service in the foreground with verbose debug logging to assist with troubleshooting. Note
that in "production" mode, it would be best to place the environment variable in a service script and
start/manage the process as a daemon process with sufficient logging for troubleshooting:

{% highlight bash %}
# Node: node2.localhost
$ sudo LD_LIBRARY_PATH=/usr/lib/jvm/default-java/jre/lib/amd64/server \
       /opt/syslog-ng/sbin/syslog-ng -Fvd
{% endhighlight %}

As a note, if you happened to start the service in background/daemon mode and wish to stop it
gracefully (without an explicit `kill` operation), the following command can be used:

{% highlight bash %}
# Node: node2.localhost
$ sudo LD_LIBRARY_PATH=/usr/lib/jvm/default-java/jre/lib/amd64/server \
       /opt/syslog-ng/sbin/syslog-ng-ctl stop
{% endhighlight %}

#### Client - Forwarding Configuration

Now that the server is set up to receive and forward logs to the Kafka endpoint, let's set up the
client configuration to instruct the client syslog-ng service to forward logs to the server
syslog-ng instance:

{% highlight bash %}
# Node: node1.localhost
$ cd /opt/syslog-ng/etc/

$ sudo vim syslog-ng.conf
# add the following to the very end of the file:
#   @include "/opt/syslog-ng/etc/conf.d/*.conf"

$ sudo mkdir conf.d/
$ sudo vim conf.d/myapp.conf
# ensure contains the following
#   source myapp_log {
#     file("/var/log/myapp.log");
#   };
#   destination myapp_remote {
#     tcp("10.11.13.16" port(514));
#   };
#   log {
#     source(myapp_log);
#     destination(myapp_remote);
#   };
{% endhighlight %}

Once the configuration is in place, you can start the syslog-ng service - we will start this service
as a background worker as there is lower risk that the configurations are incorrect at this point:

{% highlight bash %}
# Node: node1.localhost
$ sudo /opt/syslog-ng/sbin/syslog-ng
{% endhighlight %}

### Testing

Once the above configurations have been completed and all services started, we can test the operation
of our technology stack. Open 2 terminal windows, both with SSH sessions to "node1.localhost" (where
both the syslog-ng client and Kafka services reside).

In the first window, check the available topics in the Kafka instance:

{% highlight bash %}
# Node: node1.localhost
$ /opt/kafka_2.11-0.10.0.1/bin/kafka-topics.sh --zookeeper localhost:2181 --list
# should output something *similar* to the following:
#   node2
{% endhighlight %}

Once you've identified the topic which corresponds to the hostname (canonical) of the server instance,
you can start to monitor the topic for incoming messages. In the same window, start consuming from the
topic listed:

{% highlight bash %}
# Node: node1.localhost
$ /opt/kafka_2.11-0.10.0.1/bin/kafka-console-consumer.sh --zookeeper localhost:2181 \
                                                         --topic node2 \
                                                         --from-beginning
{% endhighlight %}

Leave the above command running and navigate to the second SSH terminal - we will now generate a
simulated log message to the monitored log file:

{% highlight bash %}
# Node: node1.localhost
$ echo "Testing this out" | sudo tee -a /var/log/myapp.log
{% endhighlight %}

Once you execute the above command, you should see in your first SSH terminal (monitoring the topic
in Kafka) populate the log message with a corresponding timestamp, indicating that your technology
pipeline is functioning as expected!

### Troubleshooting

If you were not successful in realizing the log through the Kafka topic when testing, attempt to isolate
the issue to a specific point in the processing pipeline. The following is the sequence of events through
the pipeline that should help in ensuring that the message makes it to each respective location. Once the
failure point has been identified, work backwards to determine if there is a misconfiguration or other
issue causing problems, such as name resolution, port conflicts, etc.

1. Write log message to end/append to `/var/log/myapp.log` file on node1.localhost.
2. syslog-ng on node1.localhost identifies log and forwards to syslog-ng on node2.localhost port 6514.
3. syslog-ng on node2.localhost receives forwarded log and forwards to Kafka on node1.localhost port 9092.
4. Kafka on node1.localhost receives log on port 9092 and writes to topic 'node2'.
5. Kafka consumer script recognizes new item in topic and outputs to console.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Compiling syslog-ng](https://www.balabit.com/sites/default/files/documents/syslog-ng-ose-latest-guides/en/syslog-ng-ose-guide-admin/html/compiling-syslog-ng.html)
* [syslog-ng Kafka Driver](http://syslogng-kafka.readthedocs.io/en/latest/readme.html)
* [syslog-ng Kafka](https://github.com/ilanddev/syslogng_kafka)
* [Publishing Messages to Apache Kafka](https://www.balabit.com/documents/syslog-ng-ose-latest-guides/en/syslog-ng-ose-guide-admin/html/configuring-destinations-kafka.html)

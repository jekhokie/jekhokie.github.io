---
layout: post
title:  "Journey to Near-Real-Time Processing"
date:   2016-11-07 20:32:46 -0400
categories: ubuntu spark spark-streaming kafka zookeeper scala
---
Tutorial and steps related to the installation, configuration, and use of a technology stack that provides
near-real-time data processing capabilities. Given some of the technologies are fairly new to me, this
tutorial is an explanation of my journey into [Apache Kafka](https://kafka.apache.org/),
[Apache ZooKeeper](https://zookeeper.apache.org/), and [Apache Spark](http://spark.apache.org/) (more
specifically, [Spark Streaming](http://spark.apache.org/streaming/)).

### Background

Real-time data processing seems to be a wave that exposes itself every so often, and currently, that wave
appears to be at its peak. Businesses are wanting to get their heads wrapped around the "Big Data" craze
that exists today, allowing them to make split decisions based on massive amounts of data in real-time.
There's only one problem with this - making near-real-time decisions based on massive amounts of complicated
data requires an architecture that can retrieve and process such massive amounts of complicated data.

In exploring various topologies and technologies that exist for data processing and how many businesses
accomplish such a feat, it appears that there is generally consensus that a technology stack of Apache
services seems to fit the bill:

* [Apache Kafka](https://kafka.apache.org/): Publish/subscribe distributed streaming platform that provides
clustering and data replication for high availability. The technology stack also provides libraries for
integrating data sources (connectors) as well as processing data near-real-time (stream processors).
* [ZooKeeper](https://zookeeper.apache.org/): A dependency of the Kafka product listed above, ZooKeeper is
a key/value store providing storage of data along with ephemeral data points for dynamic registration and
de-registration of services.
* [Spark](http://spark.apache.org/): Engine that allows for large-scale data processing and computation.
The engine can run on many various data stores (Hadoop, Cassandra, Mesos, HBase, etc) and provides the
ability to write applications in Java, Scala, Python, and R. Performance of the Spark processing engine
at the time of this post is advertised to be 100x faster than Hadoop MapReduce in memory, and 10x faster
than Hadoop MapReduce on disk for batch processing.
* [Spark Streaming](http://spark.apache.org/streaming/): Spark Streaming is an extension of the Spark
data processing engine and provides the ability to create fault-tolerant streaming applications. It takes
the capbabilities of the Spark processing engine and extends them to perform near-real-time data stream
processing and computations.

The sections in this post will be broken into separate technology components, each of which builds on
the prior section(s) and integrates a new piece of technology to the overall stack. It is done in this
way to help understand the basics behind each component as well as how each fits into the previous.

***WARNING***: This tutorial is entirely experimental and intended as a learning exercise for how the
various technologies are installed, configured, and can be integrated and used. This tutorial is in no
way intended to represent a production-ready implementation of a real-time streaming platform (it is by
no means tuned appropriately for such a scale). In addition, configuration files, scripts, temporary
files, and other components of the setup are not optimized for the "correct" locations they should
be located in. IP addresses are also used in most configurations in order to simplify the environment
expectations (no DNS requirements), but if DNS is correctly set up and each host can resolve the other,
the IP addresses listed can easily be swapped for FQDN-based configurations. For the purposes of
configuring nodes, however, Kafka/ZooKeeper requires some semblence of name-based resolution, so the
host file (`/etc/hosts`) on each node will be updated to include resolution of each node.

### Underlying Compute Technology/Ecosystem

This tutorial utilizes/makes the following assumptions about your compute infrastructure - although the
instructions can be extended to accomodate cloud-based and/or other infrastructure environments quite
easily, all details are explained with respect to local development for the purposes of simplicity:

- **Hypervisor Technology**: VirtualBox
- **Provisioner**: Vagrant
- **Number of VMs**: 2
- **Operating System**: Ubuntu 16.04
- **Arch**: 64-bit
- **CPUs**: 2
- **Mem**: 4GB
- **Disk**: 50GB

In addition, most services are best installed and configured as a cluster - this tutorial is built with
2 virtual machines serving as the compute resources for such clusters, but it should be noted that most
cluster documentation for these products lists a minimum cluster size of 3, so you may run into unexpected
and strange failure conditions (again, cluster size of 2 is intentionally done as a way to make the
instructions contained within simpler to understand/follow). The following hostnames and corresponding IP
addresses will be used for the 2 virtual machine instances:

- **node1.localhost**: 10.11.13.15
- **node2.localhost**: 10.11.13.16

The following software versions are used as part of this tutorial - again, versions can/may be changed,
but there is no guarantee that using different versions will result in a working/functional technology
stack. The versions listed are such that they represent a fully-functional technology ecosystem:

- **Scala**: 2.11.6 (dependency for building applications)
- **Apache ZooKeeper**: (bundled with Kafka)
- **Apache Kafka**: 0.10.1.0
- **Apache Spark/Spark Streaming**: 2.0.1

Additionally, the steps contained within are a VERY manual way to install and configure the respective
services - it is and has always been recommended that all of these steps make their way into a software
management system of some type (Puppet/Chef) rather than hand-provisioning and managing the stack manually
(which would be ridiculous for a system of any complexity or scale).

Finally, the code blocks included in this tutorial will list, as a comment, the node(s) that the commands
following need to be run on. For instance, if required on both nodes, the code will include a comment like
so:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
{% endhighlight %}

If the code block only applies to one of the nodes, it will list the specific node it applies to like so:

{% highlight bash %}
# Node: node2.localhost
{% endhighlight %}

Listing both nodes makes the tutorial easier to follow as you can execute the same command in both places
rather than having to re-visit all the steps applicable to the second node after the fact. In some cases,
sequencing is important and, as such, those scenarios will be called out specifically.

All commands assume that your user is `vagrant` and the user's corresponding home directory is `/home/vagrant`
for the purposes of running sudo commands.

### End State Architecture

At the end of this tutorial, the following end-state architecture will be realized:

{% highlight bash %}

|-----------------------|               |-----------------------|
|    node1.localhost    |               |    node2.localhost    |
|     (10.11.13.15)     |               |     (10.11.13.16)     |
|                       |               |                       |
|     |-----------|     |   Clustered   |     |-----------|     |
|     | ZooKeeper |<------------------------->| ZooKeeper |     |
|     | Kafka     |     |               |     | Kafka     |     |
|     |-----------|     |               |     |-----------|     |
|           ^           |               |           ^           |
|           |           |               |           |           |
|           v           |               |           v           |
|  |-----------------|  |     Linked    |  |-----------------|  |
|  |  Spark Master   |<------------------->|   Spark Worker  |  |
|  |    Status UI    |  |               |  |                 |  |
|  |-----------------|  |               |  |-----------------|  |
|                       |               |                       |
|-----------------------|               |-----------------------|

{% endhighlight %}

### Prerequisites

The following are some prerequisites required to be performed prior to proceeding with the rest of the
tutorial. Note that failure to accommodate these will likely end up in some strange behavior/failure
conditions that are difficult to debug in some cases.

#### Java

Almost all products require the Java Runtime Environment (minimum) to be installed in order to successfully
configure and run them. Start by installing the default JRE for the Ubuntu-based system:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo apt-get update
$ sudo apt-get -y install default-jre
{% endhighlight %}

#### Hosts File

As stated in the introduction, IP addresses will mostly be used for configurations in order to eliminate
the complexity of setting up a DNS-based environment. However, in some configuration cases (such as
ZooKeeper/Kafka), being able to resolve the hostnames of either host is required and, therefore, we will
place static names/IPs into each hosts' `/etc/hosts` file to accommodate such a scenario:

{% highlight bash %}
# Node: node1.localhost
$ echo "10.11.13.16   node2.localhost   node2" | sudo tee -a /etc/hosts

# Node: node2.localhost
$ echo "10.11.13.15   node1.localhost   node1" | sudo tee -a /etc/hosts
{% endhighlight %}

### Apache ZooKeeper

This section will detail the steps involved with installing and configuring an Apache ZooKeeper
cluster. The Zookeeper cluster will be instantiated between the 2 VMs and, as such, the software
and configurations will live on both instances.

#### Installation and Configuration

First, download and extract the Kafka package, which will provide an out of box ZooKeeper
software implementation corresponding to the release of Kafka downloaded:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ cd ~
$ wget http://apache.claz.org/kafka/0.10.1.0/kafka_2.11-0.10.1.0.tgz
$ tar xzf kafka_2.11-0.10.1.0.tgz
$ cd kafka_2.11-0.10.1.0/
{% endhighlight %}

Create a data directory for use by ZooKeeper:

{% highlight bash %}
$ sudo mkdir -p /data/zookeeper
{% endhighlight %}

Configure each instance ID for ZooKeeper:

{% highlight bash %}
# Node: node1.localhost
$ echo 1 | sudo tee /data/zookeeper/myid
# Node: node2.localhost
$ echo 2 | sudo tee /data/zookeeper/myid
{% endhighlight %}

Configure the ZooKeeper instance(s) via the configuration provided:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ vim config/zookeeper.properties
# ensure the following properties are set in the configuration:
#   dataDir=/data/zookeeper
#   initLimit=5
#   syncLimit=5
#   server.1=10.11.13.15:2888:3888
#   server.2=10.11.13.16:2888:3888
{% endhighlight %}

Next, start the ZooKeeper service and background the job/redirect logging output:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo -s /bin/bash -c 'nohup /home/vagrant/kafka_2.11-0.10.1.0/bin/zookeeper-server-start.sh \
                        /home/vagrant/kafka_2.11-0.10.1.0/config/zookeeper.properties >/var/log/zookeeper.out 2>&1 &'
{% endhighlight %}

#### Validation

Checking that the process is running is a simple way to ensure the ZooKeeper service did not immediately
crash on start:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ ps ef | grep zookeeper
# should output a process and corresponding process ID - output truncated for brevity:
#   17216 pts/0    R+     0:00  \_ grep --color=auto zookeeper XDG_SESSION_ID=3 SSH...
{% endhighlight %}

Inspect the `zookeeper.out` log file for more specific details on the processes:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo tail -f /var/log/zookeeper.out
# inspect the logs to ensure that the nodes have clustered and nothing out of the ordinary has happened
{% endhighlight %}

In addition, you can explicitly request status information from the ZooKeeper service on each node:

{% highlight bash %}
# Node: node1.localhost
$ echo status | nc localhost 2181
# should output something like the following:
#   Zookeeper version: 3.4.8--1, built on 02/06/2016 03:18 GMT
#   Clients:
#     /0:0:0:0:0:0:0:1:53334[0](queued=0,recved=1,sent=0)
#   Latency min/avg/max: 0/0/0
#   Received: 1
#   Sent: 0
#   Connections: 1
#   Outstanding: 0
#   Zxid: 0x0
#   Mode: follower
#   Node count: 4

# Node: node2.localhost
$ echo status | nc localhost 2181
# should output something like the following:
#   Zookeeper version: 3.4.8--1, built on 02/06/2016 03:18 GMT
#   Clients:
#     /0:0:0:0:0:0:0:1:48048[0](queued=0,recved=1,sent=0)
#   Latency min/avg/max: 0/0/0
#   Received: 1
#   Sent: 0
#   Connections: 1
#   Outstanding: 0
#   Zxid: 0x100000000
#   Mode: leader
#   Node count: 4

# Note: if the 'leader' and 'follower' status commands above are flipped between the two nodes,
#       that is acceptable/ok - so long as there are exactly 1 leader, and 1 follower total.
{% endhighlight %}

### Apache Kafka

This section will detail the steps involved with installing and configuring an Apache Kafka cluster.
The Kafka cluster will be instantiated between the 2 VMs and, as such, the software and
configurations will live on both instances.

#### Installation and Configuration

The previously-downloaded Kafka tar file will be used for the installation and configuration of the
Kafka cluster. Simply switch to the respective directory if you are no longer in the directory:

{% highlight bash %}
$ cd /home/vagrant/kafka_2.11-0.10.1.0
{% endhighlight %}

Create a log directory under which the Kafka logs can be stored:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo mkdir /var/log/kafka
{% endhighlight %}

Next, configure each Kafka node to ensure clustering can be supported/enabled:

{% highlight bash %}
# Node: node1.localhost
$ vim config/server.properties
# ensure the following properties are set in the configuration - feel free to leave
# the rest of the 'out of box' configurations alone for now:
#   broker.id=1
#   listeners=PLAINTEXT://10.11.13.15:9092
#   log.dirs=/var/log/kafka
#   zookeeper.connect=10.11.13.15:2181,10.11.13.16:2181

# Node: node2.localhost
$ vim config/server.properties
# ensure the following properties are set in the configuration - feel free to leave
# the rest of the 'out of box' configurations alone for now:
#   broker.id=2
#   listeners=PLAINTEXT://10.11.13.16:9092
#   log.dirs=/var/log/kafka
#   zookeeper.connect=10.11.13.15:2181,10.11.13.16:2181
{% endhighlight %}

Once the configurations are in place, start the Kafka service:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo -s /bin/bash -c '/home/vagrant/kafka_2.11-0.10.1.0/bin/kafka-server-start.sh \
                        /home/vagrant/kafka_2.11-0.10.1.0/config/server.properties >/var/log/kafka/kafka.out 2>&1 &'
{% endhighlight %}

#### Validation

Checking that the process is running is a simple way to ensure the Kafka service did not immediately
crash on start:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ ps ef | grep kafka
# should output a process and corresponding process ID - output truncated for brevity:
#   17216 pts/0    R+     0:00  \_ grep --color=auto zookeeper XDG_SESSION_ID=3 SSH...
{% endhighlight %}

Inspect the `kafka.out` log file for more specific details related to the process:

{% highlight bash %}
# Node: node1.localhost
$ sudo tail -f /var/log/kafka/kafka.out
# inspect the logs to ensure that, at a minimum, the following lines are included:
#   ...
#   INFO Socket connection established to 10.11.13.16/10.11.13.16:2181, initiating session (org.apache.zookeeper.ClientCnxn)
#   ..
#   INFO zookeeper state changed (SyncConnected) (org.I0Itec.zkclient.ZkClient)
#   INFO Cluster ID = d9kYPlV0SE6KPaM1afPvow (kafka.server.KafkaServer)
#   ...
#   INFO 1 successfully elected as leader (kafka.server.ZookeeperLeaderElector)
#   INFO New leader is 1 (kafka.server.ZookeeperLeaderElector$LeaderChangeListener)
#   ...
#   INFO Creating /brokers/ids/1 (is it secure? false) (kafka.utils.ZKCheckedEphemeral)
#   ...
#   INFO [Kafka Server 1], started (kafka.server.KafkaServer)
#   ...

# Node: node2.localhost
$ sudo tail -f /var/log/kafka/kafka.out
# inspect the logs to ensure that, at a minimum, the following lines are included:
#   ...
#   INFO Socket connection established to 10.11.13.16/10.11.13.16:2181, initiating session (org.apache.zookeeper.ClientCnxn)
#   ..
#   INFO zookeeper state changed (SyncConnected) (org.I0Itec.zkclient.ZkClient)
#   INFO Cluster ID = d9kYPlV0SE6KPaM1afPvow (kafka.server.KafkaServer)
#   ...
#   INFO Creating /brokers/ids/2 (is it secure? false) (kafka.utils.ZKCheckedEphemeral)
#   ...
#   INFO [Kafka Server 2], started (kafka.server.KafkaServer)
#   ...

# Note: If the above logs are flipped in terms of the leader being on node 2 versus node 1, that
#       is acceptable so long as there is only 1 leader and the 'Cluster ID' is the same between
#       the two nodes.
{% endhighlight %}

To test that the cluster is operating as expected, you can create a test topic named `test` with a
replication factor of 2, indicating that the topic should reside on both nodes. Following the creation
of the topic, ask Kafka to describe the topic to ensure that both brokers (nodes) have visibility of
the topic:

{% highlight bash %}
# Node: node1.localhost
$ bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic test --replication-factor 2 --partitions 1
# should see the following output:
#   Created topic "test".

# Node: node1.localhost
$ bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test
# should see something like the following:
#   Topic:test  PartitionCount:1  ReplicationFactor:2  Configs:
#      Topic: test  Partition: 0  Leader: 2  Replicas: 2,1  Isr: 2,1
# Some of the data may not be exactly as above, but the important parts:
# 'Leader' indicates that node 2 is responsible for all reads/writes for the given partition
# 'Replicas' indicates that both brokers have knowledge of the topic (good - clustered)
# 'Isr' are the in-sync replicas, which indicates that both brokers are in sync (good - clustered)
{% endhighlight %}

### Apache Spark

This section will detail the steps involved with installing and configuring the Apache Spark data
processing engine. Spark will be installed on both nodes where the Kafka/ZooKeeper services are
installed, but does not necessarily have to be (set up this way for simplicity/reduced hardware
requirements). This cluster installation will result in an installation of Spark in a cluster
using the standalone resource scheduler (as opposed to something such as YARN).

#### Installation and Configuration

Download the Spark package:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ cd ~
$ wget http://d3kbcqa49mib13.cloudfront.net/spark-2.0.1-bin-hadoop2.7.tgz
$ tar xzf spark-2.0.1-bin-hadoop2.7.tgz
$ cd spark-2.0.1-bin-hadoop2.7/
{% endhighlight %}

Create a logging directory for the Spark process:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo mkdir /var/log/spark
{% endhighlight %}

Set some configurations for the Spark processes - note that some memory tuning is done in order
to ensure that jobs do not over-commit the small amount of VM resources, but the memory specified
in these configurations are way too low for any kind of complicated processing:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ cp conf/spark-defaults.conf.template conf/spark-defaults.conf
$ vim conf/spark-defaults.conf
# add the following configuration directives to the end of the file:
#   spark.driver.memory=512m
#   spark.executor.memory=512m
#   spark.executor.cores=1
$ cp conf/spark-env.sh.template conf/spark-env.sh
$ chmod 775 conf/spark-env.sh

# Node: node1.localhost
$ vim conf/spark-env.sh
# add the following configuration directives to the end of the file:
#   SPARK_LOG_DIR=/var/log/spark
#   SPARK_MASTER_HOST=10.11.13.15

# Node: node2.localhost
$ vim conf/spark-env.sh
# add the following configuration directives to the end of the file:
#   SPARK_LOG_DIR=/var/log/spark
#   SPARK_LOCAL_IP=10.11.13.16
{% endhighlight %}

Once the above configuration files are in place, you can start the Spark processes. Ensure that the
processes are started in this order to ensure that the worker can connect to the master instance:

{% highlight bash %}
# Node: node1.localhost
$ sudo /home/vagrant/spark-2.0.1-bin-hadoop2.7/sbin/start-master.sh
# you should see some output along the lines of:
#   starting org.apache.spark.deploy.master.Master, logging to /var/log/spark/spark-root-org.apache.spark.deploy.master.Master-1-node1.out

# Node: node2.localhost
$ sudo /home/vagrant/spark-2.0.1-bin-hadoop2.7/sbin/start-slave.sh spark://10.11.13.15:7077
# you should see some output along the lines of:
#   starting org.apache.spark.deploy.worker.Worker, logging to /var/log/spark/spark-root-org.apache.spark.deploy.worker.Worker-1-node2.out
{% endhighlight %}

#### Validation

Once the above have been completed, you theoretically have a working installation of a Spark ecosystem.
To validate, check that the Spark process is functioning on each node:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ ps ef | grep spark
# should output a process and corresponding process ID - output truncated for brevity:
#   24676 pts/0    S+     0:00  \_ grep --color=auto spark XDG_SESSION_ID=3...
{% endhighlight %}

Inspect the log files that were returned following the output of the `start` commands:

{% highlight bash %}
# Node: node1.localhost
$ sudo tail -f /var/log/spark/spark-root-org.apache.spark.deploy.master.Master-1-node1.out
# inspect the logs to ensure that, at a minimum, the following lines are included:
#   ...
#   INFO Master: Started daemon with process name: 24612@node1
#   ..
#   INFO Utils: Successfully started service 'sparkMaster' on port 7077.
#   INFO Master: Starting Spark master at spark://10.11.13.15:7077
#   INFO Master: Running Spark version 2.0.1
#   INFO Utils: Successfully started service 'MasterUI' on port 8080.
#   INFO MasterWebUI: Bound MasterWebUI to 10.11.13.15, and started at http://10.11.13.15:8080
#   INFO Utils: Successfully started service on port 6066.
#   INFO StandaloneRestServer: Started REST server for submitting applications on port 6066
#   INFO Master: I have been elected leader! New state: ALIVE
#   INFO Master: Registering worker 10.11.13.16:43675 with 2 cores, 2.9 GB RAM

# Node: node2.localhost
$ sudo tail -f /var/log/spark/spark-root-org.apache.spark.deploy.worker.Worker-1-node2.out
# inspect the logs to ensure that, at a minimum, the following lines are included:
#   ...
#   INFO Worker: Started daemon with process name: 16578@node2
#   ...
#   INFO Utils: Successfully started service 'sparkWorker' on port 43675.
#   INFO Worker: Starting Spark worker 10.11.13.16:43675 with 2 cores, 2.9 GB RAM
#   INFO Worker: Running Spark version 2.0.1
#   INFO Worker: Spark home: /home/vagrant/spark-2.0.1-bin-hadoop2.7
#   INFO Utils: Successfully started service 'WorkerUI' on port 8081.
#   INFO WorkerWebUI: Bound WorkerWebUI to 10.11.13.16, and started at http://10.11.13.16:8081
#   INFO Worker: Connecting to master 10.11.13.15:7077...
#   INFO TransportClientFactory: Successfully created connection to /10.11.13.15:7077 after 26 ms (0 ms spent in bootstraps)
#   INFO Worker: Successfully registered with master spark://10.11.13.15:7077
{% endhighlight %}

Once the logs have been verified, you can view the Spark infrastructure stack visually via browsing
to the following URL in your browser:

* [http://10.11.13.15:8080/](http://10.11.13.15:8080)

Once loaded, you should see a webpage indicating that the node queried is the Spark master at location
10.11.13.15, and the "Workers" section should include the worker configured on node 10.11.13.16.

If the above validation steps line up, you are likely in a good position to submit a test job to test
the worker interaction in your Spark ecosystem. Perform the following from one of the two nodes - this
action will submit an example job that calculates the value of Pi:

{% highlight bash %}
# Node: node1.localhost
$ ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
                     --master spark://10.11.13.15:6066 \
                     --deploy-mode cluster \
                     examples/jars/spark-examples_2.11-2.0.1.jar 1000
# if all goes well, you will likely see a bit of output - within the output, something like the
# following output should appear, indicating successful scheduling of the job:
#   ...
#   {
#     "action" : "CreateSubmissionResponse",
#     "message" : "Driver successfully submitted as driver-20161107210027-0000",
#     "serverSparkVersion" : "2.0.1",
#     "submissionId" : "driver-20161106210027-0000",
#     "success" : true
#   }
{% endhighlight %}

As stated in the comments, if the job was successful you should see a response from the master indicating
that the driver submission was successful. In order to check the outcome of the driver, navigate back to
the web UI located at [http://10.11.13.15:8080/](http://10.11.13.15:8080) and refresh the page if
necessary. You should then see a driver submission with the driver ID listed above for the date/time stamp
of the submission. Once the job finishes, you should see it move into the "Completed Drivers" category
towards the bottom of the page. If all went well, the "State" should be listed as "FINISHED".

Click the "worker" link associated with the job (under the "Worker" column) to see details about the job -
this will bring you to a page associated with the IP address of the worker node (note the IP address in the
URL updated to "10.11.13.16"). Scroll to the bottom of the page and in the row for the worker ID under the
"Logs" column, select the "stdout" link and inspect the log output - you should see something similar to the
following in the output:

```
Pi is roughly 3.1414870314148704
```

If you see the above in the "stdout" log output, all is well with your technology stack and the job
submission succeeded - congratulations! If this is not the case, or the "State" of the job indicates
some kind of failure, inspect the output of the "stderr" logs to investigate what may have gone wrong.

One item to note before moving along - the driver listed shows some details on the main/master page.
The worker link is named with the actual worker node that completed the computation, along with the
associated details related to how many resources were utilized to perform the computation (CPU, memory,
etc.). These types of resource details can also be configured at the job-submission command prompt via
extra parameters such as `--executor-memory`, `--total-executor-cores`, and various other tuning or
constraint parameters.

### Apache Spark Streaming

This section will detail the steps involved with installing and configuring the Apache Spark Streaming
stream-processing engine. No further installation requirements are necessary due to the Spark packages
already being installed/configured - this section will focus on configuration of Spark streaming,
validation, and testing to demonstrate a very basic capability.

Spark Streaming is an extension of the Spark processing engine and allows taking streams of information,
breaking them into batches, passing them to the Spark processing engine to perform computation, and then
passes the batches along to the consumer/downstream process. As we have already set up the Master and
corresponding worker node, we will in this section be designing a Spark Streaming application that will
simply read from the previously-created `test` Kafka topic and output data as it is received.

We will be performing the build of the streaming application on the master node (node1.localhost) and,
as such, you will only see commands run from this node.

#### Project Prerequisites

**NOTE**: *ALL* of the following commands should only be run on/from the Master node instance (node1.localhost).
As such, the `Node` comment in the command sections will be omitted for each step.

In order to successfully create and package a Spark job, a few items are required. First, we will need to
install the Scala language engine, along with the corresponding Scala build tool. If you are doing the
installation on an Ubuntu-16 operating system, the following commands will suffice:

{% highlight bash %}
# install scala 2.11
$ sudo apt-get -y install scala

# install sbt
$ echo "deb https://dl.bintray.com/sbt/debian /" | sudo tee -a /etc/apt/sources.list.d/sbt.list
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2EE0EA64E40A89B84B2DF73499E82A75642AC823
$ sudo apt-get update
$ sudo apt-get -y install sbt
{% endhighlight %}

#### Driver/Application Development

Now that your dependencies have been installed, you are ready to start coding your driver/application. First,
a project directory needs to be built for the streaming application to be built within - the Scala process
follows the standard Maven-based directory naming conventions:

{% highlight bash %}
# create project dir in your home directory
$ cd ~
$ mkdir TestApp
$ cd TestApp/
$ mkdir -p lib \
           project \
           src/main/scala
{% endhighlight %}

Once the directories are in place, we can start setting up the build environment. The Spark ecosystem
likes "uber" jar files, meaning dependencies are included as part of the application packaging rather
than listed as external dependencies. Although the `submit-job` command allows for indicating other
external libraries/dependencies which the worker node(s) will automatically download (from the Master
node, public internet, or otherwise), we will be packaging the dependencies into our application and
creating the "uber" jar file using the "sbt-assembly" plugin - so let's add the plugin to our project:

{% highlight bash %}
$ vim project/assembly.sbt
# ensure contains the following:
#   addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.2")
{% endhighlight %}

Once you have configured the environment for the assembly plugin, construct the Scala build tool
project metadata file as follows:

{% highlight bash %}
$ vim build.sbt
# ensure contains the following:
#   name := "TestApp"
#   version := "0.0.1"
#   scalaVersion := "2.11.6"
#
#   libraryDependencies += "org.apache.spark" % "spark-core_2.11" % "2.0.1" % "provided"
#   libraryDependencies += "org.apache.spark" % "spark-streaming_2.11" % "2.0.1" % "provided"
#   libraryDependencies += "org.apache.spark" % "spark-streaming-kafka-0-10_2.11" % "2.0.1"
#
#   // override any 'deduplicate' strategy with a 'pick last to avoid errors'
#   assemblyMergeStrategy in assembly := {
#       case PathList("META-INF", "MANIFEST.MF") => MergeStrategy.discard
#       case x => MergeStrategy.last
#   }
{% endhighlight %}

The above build definition will tell the build tool to ensure that "last" is always the merge
strategy for scenarios where merges are required. This is likely not desirable and finer-grain
controls should likely be put in place, but for the sake of brevity, this file will be used to
use the "last" instance of any merge conflict as it has been tested and works successfully.

Next, we will construct our application code:

{% highlight bash %}
$ vim src/main/scala/TestApp.scala
# ensure contains the following:
#   // This application is based off the "KafkaWordCount" example
#   // application included in the Spark examples.
#
#   import org.apache.kafka.clients.consumer.ConsumerRecord
#   import org.apache.kafka.common.serialization.StringDeserializer
#   import org.apache.spark.streaming._
#   import org.apache.spark.SparkConf
#   import org.apache.spark.streaming.kafka010._
#   import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
#   import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe
#
#   object TestApp {
#       def main(args: Array[String]) {
#           // handle the input arguments
#           if (args.length < 2) {
#               System.err.println("Usage: TestApp <zkInstances> <topicName> <frequency>")
#               System.exit(1)
#           }
#           val Array(zkInstances, topicName, frequency) = args
#
#
#           // create a configuration and new streaming context
#           val sparkConf = new SparkConf().setAppName("TestApp")
#           val ssc = new StreamingContext(sparkConf, Seconds(frequency.toInt))
#
#           // set up the configuration for the stream processing
#           val kafkaParams = Map[String, Object](
#               "bootstrap.servers"  -> zkInstances,
#               "key.deserializer"   -> classOf[StringDeserializer],
#               "value.deserializer" -> classOf[StringDeserializer],
#               "group.id"           -> "TestAppGroup",
#               "auto.offset.reset"  -> "latest",
#               "enable.auto.commit" -> (false: java.lang.Boolean)
#           )
#
#           // create the Direct Stream integration for processing
#           val stream = KafkaUtils.createDirectStream[String, String](
#                            ssc,
#                            PreferConsistent,
#                            Subscribe[String, String](Array(topicName), kafkaParams)
#           )
#
#           // perform the map operation and print the output
#           val out = stream.map(record => (record.key, record.value))
#           out.print()
#
#           // take action based on the above configurations
#           ssc.start()
#           ssc.awaitTermination()
#       }
#   }
{% endhighlight %}

As demonstrated via the comments within the application code, this application will create a Direct
Kafka stream connection (see [here](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)
for details related to this implementation) and prints the data received to the console.

Once you have created the directory structure and respective application files/code, run the sbt
command to assemble your "uber" jar:

{% highlight bash %}
$ sbt assembly
# should see a bit of output, something along the lines of:
#   [info] Loading project definition from /home/vagrant/TestApp/project
#   [info] Set current project to test (in build file:/home/vagrant/TestApp/)
#   [info] Including from cache: commons-configuration-1.6.jar
#   [info] Including from cache: commons-lang3-3.3.2.jar
#   [info] Including from cache: apache-log4j-extras-1.2.17.jar
#   [info] Including from cache: kryo-shaded-3.0.3.jar
#   ...
#   [warn] Strategy 'discard' was applied to 69 files
#   [warn] Strategy 'first' was applied to 3761 files
#   [info] Assembly up to date: /home/vagrant/TestApp/target/scala-2.11/TestApp-assembly-0.0.1.jar
#   [success] Total time: 4 s, completed Nov 5, 2016 9:15:33 PM
{% endhighlight %}

If the `[success]` line shows up at the bottom of the output generated, your uber jar has been
successfully assembled and is ready for use.

Start the application via the following command line - note that this command sequence again assumes
that you are running the command from node1.localhost (note the IP address listed):

{% highlight bash %}
$ SPARK_LOCAL_IP=10.11.13.15 ../spark-2.0.1-bin-hadoop2.7/bin/spark-submit \
                             --driver-memory 512m \
                             --master local[2] \
                             /home/vagrant/TestApp/target/scala-2.11/TestApp-assembly-0.0.1.jar \
                             10.11.13.15:9092,10.11.13.16:9092 test 5
{% endhighlight %}

The above command will run the application you just created on the local master instance with up to
2 processors allocated. Driver memory has been pinned to a maximum of 512MB of RAM to ensure that
the driver does not consume too much memory. Once launched, the driver will create a direct stream
connection to the Kafka instances on node1.localhost and node2.localhost and gather any data sent
to the 'test' topic at intervals of 5 seconds, writing the contents of the 'test' topic to the
console output.

Now that the application is running, move over to the node2.localhost instance and launch the Kafka
console producer script to generate some data into the Kafka instance(s) so that your application
can read and print out the data:

{% highlight bash %}
# Note: This is run from node2.localhost:
$ cd /home/vagrant/kafka_2.11-0.10.1.0/bin/
$ ./kafka-console-producer.sh --broker-list 10.11.13.15:9092,10.11.13.16:9092 --topic test
{% endhighlight %}

Once the above command is typed and you press \<Enter\>, it will appear as though nothing has happened.
However, if there are no errors, you are likely now connected to the Kafka brokers/listeners on each
node, and are ready to start sending data. Keep the window open that contains the Spark Streaming
driver output and visible - then, start to type a few lines of text into the Kafka console producer
application that is running - something like "this is a test 1", followed by \<Enter\>, then "this is
a secondary test to see if this works", followed by \<Enter\>. In your driver application window, you
should see some output similar to the following, indicating that the driver application pulled and
parsed the data from the Kafka topic:

```
...
(null,this is a test)
(null,this is a secondary test to see if this works)
...
```

If the above output is printed, your driver application is working as expected, congratulations!

You will note that the above script was run with a 5 second integration frequency - you can obviously
decrease this for faster/near-real-time processing, and also disperse this application across multiple
worker nodes, leveraging a full worker cluster for more concurrent processing. The application and
demonstration lists just a very basic example of how to develop within the Spark Streaming technology
and is intended to be a foundation for you to do more interesting and complicated.

### Some Useful Automation

Much of the above was pieced together following a very quick/rough automation effort to install and
configure the software on a local VM. There are some scripts pieced together to detail the installation
efforts and are slightly different than what was explained above, but serve as a decent starting point
for automating the effort - take these with a small grain of salt as many learnings have occurred via
the construction of this post/tutorial and, as such, these scripts may be slightly out of date and/or
incorrect based on the larger effort of learning this technology stack:

* [Kafka Installation](https://github.com/jekhokie/scriptbox/tree/master/bash--install-configure-kafka-ubuntu)
* [Spark Installation](https://github.com/jekhokie/scriptbox/tree/master/bash--install-configure-spark-ubuntu)

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Apache ZooKeeper - Clustered (Multi-Server) Setup](https://zookeeper.apache.org/doc/r3.3.2/zookeeperAdmin.html#sc_zkMulitServerSetup)
* [Apache Kafka - Quickstart](https://kafka.apache.org/quickstart)
* [Apache Spark - Quickstart](http://spark.apache.org/docs/latest/quick-start.html)
* [Apache Spark - Cluster Overview](https://spark.apache.org/docs/latest/cluster-overview.html)
* [Spark Streaming + Kafka Integration Guide](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)
* [SBT - The Missing Tutorial](https://github.com/shekhargulati/52-technologies-in-2016/blob/master/02-sbt/README.md)
* [Scala Fat Jars for Spark on SBT with SBT Assembly Plugin](http://queirozf.com/entries/creating-scala-fat-jars-for-spark-on-sbt-with-sbt-assembly-plugin)
* [SBT Assembly](https://github.com/sbt/sbt-assembly)
* [Spark Scala Examples](https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples)

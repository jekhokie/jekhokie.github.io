---
layout: post
title:  "Installing Kafka"
date:   2016-08-23 07:16:14 -0400
categories: ubuntu kafka
---
Tutorial related to the installation and configuration of [Kafka](http://kafka.apache.org/) on an Ubuntu 16.04
virtual machine. The instructions and tutorial include setting up the dependent components (ZooKeeper, etc) as
well as testing and interacting with the Kafka instance.

### Warning

Note that the resulting Kafka setup from these instructions is **NOT** for production use. These instructions
are for setting up an initial test/dev setup of Kafka to understand how the application functions.

### Installation/Setup Process

The following steps are performed on a VirtualBox-hosted Ubuntu virtual machine with 1CPU and 6GB RAM.

#### Prerequisites

Install the required prerequisites (Java):

{% highlight bash %}
$ sudo apt-get update
$ sudo apt-get install default-jre default-jdk
{% endhighlight %}

#### Setup

Retrieve the Kafka package from the [Downloads](http://kafka.apache.org/downloads.html) page on the Apache
Kafka website. For the purposes of these instructions, use the **Binary** package download.

At the time of this post, the version used is 0.10.0.1 for Scala version 2.11.

Copy the downloaded package to your home directory, and unpack it:

{% highlight bash %}
$ tar -xzvf kafka_2.11-0.10.0.1.tgz
$ cd kafka_2.11-0.10.0.1/
{% endhighlight %}

#### Zookeeper

ZooKeeper is used for internal tracking and status information for Kafka.

##### Configuration/Setup

The configuration file for ZooKeeper is in config/zookeeper.properties. One item of note in the configuration
file is the `clientPort` parameter, which defaults to 2181.

To start the ZooKeeper instance, run the following command:

{% highlight bash %}
$ bin/zookeeper-server-start.sh config/zookeeper.properties
{% endhighlight %}

#### Kafka

Kafka is the main event processing application.

##### Configuration/Setup

The configuration file for Kafka is in config/server.properties. There are quite a few tunable parameters in
this file, but one of note is `zookeeper.connect` which tells Kafka where its ZooKeeper instance is
located (defaults to 'localhost:2181', which is fine for this exercise). By default, the single Kafka broker
listens on port 9092.

Kafka can be a memory hog (part of the benefits it provides related to speed). If developing on a local
VM it is likely a good idea to memory constrain the Kafka process, which is the KAFKA_HEAP_OPTS parameter
included in the commands below.

To start the Kafka instance, run the following command(s):

{% highlight bash %}
# if running on a VM and are memory-strapped, include the memory restriction env variable
$ export KAFKA_HEAP_OPTS="-Xmx256M -Xms256M"
$ bin/kafka-server-start.sh config/server.properties
{% endhighlight %}

### Interaction and Testing

Now that you have a ZooKeeper and Kafka instance set up, let's test it. Kafka comes pre-packaged with scripts
to create, modify, delete, etc. topics and messages within.

#### Create a Topic

The first step to interact with Kafka would be to create a topic:

{% highlight bash %}
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 \
                               --replication-factor 1 \
                               --partitions 1 \
                               --topic test
# Created topic "test"
{% endhighlight %}

Once the topic is created, verify using the list operation:

{% highlight bash %}
$ bin/kafka-topics.sh --list --zookeeper localhost:2181
# test
{% endhighlight %}

#### Send Test Messages

Now that the topic is created, let's send some messages to it. Running the console producer places your
shell in input mode. Each new line (enter key) will result in the message being sent to the specified topic
within Kafka. When you are done, terminate the script using the CTRL+C command sequence.

{% highlight bash %}
$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
Some test message
Another test message
{% endhighlight %}

#### Retrieve Topic Messages

The topic now has messages on it from the previous step. Let's inspect to verify that a consumer can see
the messages. The console consumer runs until the CTRL+C command sequence is executed to terminate the script.

{% highlight bash %}
$ bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
Some test message
Another test message
{% endhighlight %}

### Credit

Contributions to some of the above were gleaned from:

* [Apache Kafka](http://kafka.apache.org/)
* [Apache Kafka Quickstart](http://kafka.apache.org/documentation.html#quickstart)

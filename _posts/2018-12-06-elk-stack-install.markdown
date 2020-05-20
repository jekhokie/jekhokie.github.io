---
layout: post
title:  "ELK Stack Install"
date:   2018-12-06 19:24:03 -0400
categories: elk linux elasticsearch kibana filebeat beats ubuntu ansible
logo: elastic.jpg
---
This is a quick refresher on installing and configuring an ELK stack in an Ubuntu multi-VM cluster.
For simplicity, we'll use Ansible to configure the target instances and the Ansible code can be found
in my Scriptbox Github project [here](https://github.com/jekhokie/scriptbox/tree/master/ansible--elk-stack).
In addition, instead of installing Logstash, we will install the Filebeat capability.

## Technology Layout

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

For starters, we'll use 2 VMs created through Vagrant, although it is better practice to create 3+
nodes for failover of indexes and such. Below is the layout of the instances and their respective target
roles:

{% highlight bash %}

|-----------------------|               |-----------------------|
|    node1.localhost    |               |    node2.localhost    |
|     (10.11.13.15)     |               |     (10.11.13.16)     |
|                       |               |                       |
|   |---------------|   |   Clustered   |   |---------------|   |
|   | Elasticsearch |<--------------------->| Elasticsearch |   |
|   |---------------|   |               |   |---------------|   |
|        ^       ^      |               |           ^           |
|        |       |      |               |           |           |
|        v       |      |               |           |           |
|   |--------|   |      |               |           |           |
|   | Kibana |   |      |               |           |           |
|   |--------|   |      |               |           |           |
|                |      |               |           |           |
|                |      |               |           |           |
|                v      |               |           v           |
|   |---------------|   |               |  |-----------------|  |
|   |    Filebeat   |   |               |  |     Filebeat    |  |
|   |---------------|   |               |  |-----------------|  |
|-----------------------|               |-----------------------|

{% endhighlight %}

Note that the diagram above is slightly different than the way Ansible will configure things. In
the Ansible code, the Filebeat modules are set up to utilize both Elasticsearch instances in the
cluster for better fault-tolerance.

The following versions of the Elastic stack are used in this tutorial:

- **Elasticsearch**: 6.5.2
- **Kibana**: 6.5.2
- **Filebeat**: 6.5.2

## SSH-Key Exchange

In order for Ansible to work appropriately from the source laptop (localhost) to the target node1
and node2 instances, SSH keys must be created and exchanged. Perform the following:

{% highlight bash %}
$ ssh-keygen -t rsa -b 2048
# use a naming such as "ansible-elk" and no password
{% endhighlight %}

Next, copy the public keys to the `authorized_keys` file on each of the node1 and node2 instances.
You can do this as the `vagrant` user or some other user that you will run the Ansible playbook as:

{% highlight bash %}
# Host: node1.localhost
$ echo "<SSH_PUB_KEY>" >> .ssh/authorized_keys

# Host: node1.localhost
$ echo "<SSH_PUB_KEY>" >> .ssh/authorized_keys
{% endhighlight %}

Then test login using password-less SSH keys to ensure you can log into each host with the respective
SSH keys and no password - in this case, I'm assuming that the pub key contents were added to the
`vagrant` users' `authorized_keys` file on each node:

{% highlight bash %}
$ ssh -i /path/to/ansible-elk vagrant@10.11.13.15
$ ssh -i /path/to/ansible-elk vagrant@10.11.13.16
{% endhighlight %}

## Create Stack Using Ansible

Let's now set up the Ansible roles to create the respective components on each node. You can obtain
the Ansible project used for this activity from my GitHub "Scriptbox" repository [here](https://github.com/jekhokie/scriptbox/tree/master/ansible--elk-stack).
Note that the Ansible scripts and Ansible execution will exist and be executed from the local laptop
which is running Vagrant and can reach the node1/node2 hosts. We'll simply clone the project and run
the Ansible code to apply the software such that the nodes mirror the architecture diagram above:

{% highlight bash %}
# Host: localhost
$ git clone https://github.com/jekhokie/scriptbox.git
$ cd scriptbox/ansible--elk-stack/
$ ansible-playbook -i hosts site.yml
{% endhighlight %}

You should see many lines of information scrolling by as Ansible starts to build out the infrastructure
as it should be configured according to the diagram above. Once complete, you should have a fully
functioning Elasticsearch, Kibana, and Filebeat setup.

### Inspection

There are some basic things you can do to inspect how things are operating/whether things are configured
appropriately. Below are some URLs that can be visited:

- **[http://10.11.13.16:9200/_cat](http://10.11.13.16:9200/_cat)**: Listing of interesting endpoints to inspect.
- **[http://10.11.13.15:9200/_cat/nodes?v](http://10.11.13.15:9200/_cat/nodes?v)**: Listing of the nodes
in the cluster and corresponding status.
- **[http://10.11.13.15:9200/_cat/indices?v](http://10.11.13.15:9200/_cat/indices?v)**: Listing of indices
in the Elasticsearch cluster and corresponding documents for each. You should see the filebeat-XXX index
and associated number of documents growing over time as the Filebeat application pushes information from the
audit and system logs.
- **[http://10.11.13.16:9200/_cat/shards](http://10.11.13.16:9200/_cat/shards)**: Listing of shards and their
respective locations on each node.
- **[http://10.11.13.16:9200/_cat/nodeattrs](http://10.11.13.16:9200/_cat/nodeattrs)**: Attributes about each
node in the ES cluster.
- **[http://10.11.13.15:5601](http://10.11.13.15:5601)**: URL for the Kibana instance.

## Kibana

Now that we have some data flowing into Elasticsearch, let's log into Kibana and start looking around. The
URL to access Kibana is [http://10.11.13.15:5601](http://10.11.13.15:5601). When you enter, you should navigate
to the "Management" page if you aren't automatically brought there and navigate to "Kibana" -> "Index Patterns".
You will need to create your first index pattern and we will do so in order to capture the Filebeat index.
Select "Create Index Pattern", and follow the steps:

1. **Define index pattern**: Specify `filebeat-*` for the "Index pattern" field, then select "Next step".
2. **Tile Filter field name**: Specify `@timestamp` from the drop-down menu and select "Create index pattern".

Once the above are completed, navigate back to the "Discover" option in the left menu bar and start searching.
Make sure your timeline goes back far enough (in the upper right corner of the screen), select your index pattern
from the drop-down right beneath the filter/search field, and search away!

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Elasticsearch](https://www.elastic.co/products/elasticsearch)
* [Kibana](https://www.elastic.co/products/kibana)
* [Beats](https://www.elastic.co/products/beats)
* [Get System Logs and Metrics into Elasticsearch with Beats System Modules](https://www.elastic.co/blog/get-system-logs-and-metrics-into-elasticsearch-with-beats-system-modules)

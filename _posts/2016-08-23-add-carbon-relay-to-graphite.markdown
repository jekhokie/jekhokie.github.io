---
layout: post
title:  "Add Carbon Relay to Graphite"
date:   2016-08-23 12:43:14 -0400
categories: ubuntu graphite carbon whisper relay
---
Tutorial related to expanding on on the initial Graphite installation approach
[here]({% post_url 2016-08-22-installing-graphite-bleeding-edge %}) and adding a Carbon Relay instance
to send metrics to multiple Carbon Cache instances.

### Warning

Note that the resulting Carbon Relay setup from these instructions is **NOT** for production use. These
instructions are for setting up an initial test/dev setup of Carbon Relay to understand how the application
functions.

### Prerequisites

This tutorial assumes that multiple Carbon Cache instances are already provisioned and configured to
accept metrics on a local or remote host. The tutorial is scoped to installing and configuring a Carbon
Relay agent to send metrics downstream to multiple Carbon Cache instances.

### Dependencies

Install the required dependencies:

{% highlight bash %}
$ sudo apt-get install -y pkg-config \
                          python-dev \
                          python-pip \
                          python-virtualenv \
                          librrd-dev \
                          git-core \
                          gcc
{% endhighlight %}

### Process

#### Installation/Configuration

Create the Python virtual environment and set up the path settings:

{% highlight bash %}
$ sudo virtualenv /opt/graphite
$ export PATH=/opt/graphite/bin:$PATH
{% endhighlight %}

Clone and build/install the required components:

{% highlight bash %}
# carbon relay - same as carbon cache package
$ cd /usr/local/src
$ sudo git clone https://github.com/graphite-project/carbon.git
$ cd carbon
$ sudo -E pip install -r requirements.txt
$ sudo -E python setup.py install
{% endhighlight %}

Create the configuration (default) files:

{% highlight bash %}
$ cd /opt/graphite/conf

# copy the storage schema and edit for purposes - this should probably match the carbon caches
$ sudo cp storage-schemas.conf.example storage-schemas.conf

# carbon
$ sudo cp carbon.conf.example carbon.conf
# update the following configuration parameter in the carbon.conf file to ensure not running as superuser
#   USER = carbonusr
# also, ensure that the following relay settings are specified
#   RELAY_METHOD = rules
#   DESTINATIONS = <IP1>:<PICKLE_PORT1>:a, <IP2>:<PICKLE_PORT2>:b

# carbon relay
$ sudo cp relay-rules.conf.example relay-rules.conf
# update the [default] section to reflect the carbon cache instance IP:Port combinations to route to
# note that the port numbers must be the PICKLE port numbers
#   [default]
#   default = true
#   destinations = <IP1>:<PICKLE_PORT1>:a, <IP2>:<PICKLE_PORT2>:b
{% endhighlight %}

Create the carbon group and user for the carbon service:

{% highlight bash %}
$ sudo groupadd carbon
$ sudo useradd -c "User for the carbon service" -g carbon -s /dev/null carbonusr
{% endhighlight %}

Adjust permissions for files, binaries and directories for the Carbon user:

{% highlight bash %}
$ sudo mkdir -p /opt/graphite/storage/log/carbon-relay
$ sudo chown -R carbonusr:carbon /opt/graphite/storage/log
$ sudo chmod 775 /opt/graphite/storage
$ sudo chgrp carbon /opt/graphite/storage
{% endhighlight %}

Start the carbon relay, and ensure it is running:

{% highlight bash %}
$ sudo -E /opt/graphite/bin/carbon-relay.py \
          --config=/opt/graphite/conf/carbon.conf \
          --rules=/opt/graphite/conf/relay-rules.conf \
          start
# to check the status of the carbon-relay daemon:
$ sudo -E /opt/graphite/bin/carbon-relay.py status
#   carbon-relay (instance a) is running with pid 15225
{% endhighlight %}

#### Validation

Ensure the carbon relay process is running/listening on correct ports:

{% highlight bash %}
$ cat /opt/graphite/storage/carbon-relay-a.pid
# 15225
$ pgrep -fa carbon
# 15225 /usr/bin/python /opt/graphite/bin/carbon-relay.py
#       --config=/opt/graphite/conf/carbon.conf
#       --rules=/opt/graphite/conf/relay-rules.conf start
$ netstat -vant | grep LISTEN | grep '201[34]'
# tcp        0      0 0.0.0.0:2013            0.0.0.0:*               LISTEN
# tcp        0      0 0.0.0.0:2014            0.0.0.0:*               LISTEN
{% endhighlight %}

Send some new data and validate the data is received and stored by each carbon cache. For instance, 
run the netcat (nc) command on a local host and point it at the relay instance IP address or hostname
to send the test metric to the relay. Once performed, navigate to the instance running each carbon
cache service and run the whisper-fetch.py script to ensure that the metric made it from the relay
down to the whisper database.

{% highlight bash %}
# send a metric "test.metric.foo" with value 2 at current time to relay on port 2013
$ echo "test.something.foo 2 `date +%s`" | nc <CARBON_RELAY_IP> 2013

# check to ensure that the metric shows up on each of the carbon cache instances
# this command should be run locally on each carbon cache to ensure the metric was routed correctly
$ /usr/local/bin/whisper-fetch.py /opt/graphite/storage/whisper/test/something/foo.wsp
# should output a bunch of metrics for time series, including value sent to the relay above
{% endhighlight %}

### Credit

Contributions to some of the above were gleaned from:

* [GraphiteApp](http://graphite.readthedocs.io/en/latest/)

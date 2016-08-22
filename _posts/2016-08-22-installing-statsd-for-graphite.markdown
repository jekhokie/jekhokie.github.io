---
layout: post
title:  "Installing Statsd for Graphite"
date:   2016-08-22 07:26:04 -0400
categories: ubuntu graphite statsd
---
Instructions related to the installation and configuration of [Statsd](https://github.com/etsy/statsd/wiki) on an Ubuntu 16.04 virtual machine.
Note that these instructions follow the "Install from Source" process from the documentation in order to facilitate the highest
level of configurability.

### Warning

Note that the resulting Statsd setup from these instructions is **NOT** for production use. These instructions are for setting up
an initial test/dev setup of Statsd to understand how the application functions.

### Prerequisites

Assumes the following are already installed/ready for use on a network-accessible Ubuntu instance:

* Graphite (including whisper, carbon-cache, and graphite-web)

### Dependencies

Install the required dependencies:

{% highlight bash %}
$ sudo apt-get install -y git nodejs npm
{% endhighlight %}

### Process

#### Installation/Configuration

Clone the project into the /opt directory:

{% highlight bash %}
$ sudo git clone https://github.com/etsy/statsd.git /opt/statsd
{% endhighlight %}

Install the required packages:

{% highlight bash %}
$ cd /opt/statsd
$ sudo npm install
{% endhighlight %}

Edit the configuration file for your environment:

{% highlight bash %}
$ cd /opt/statsd
$ sudo vim exampleConfig.js
# ensure at least the following values are specified
# assumes graphite is running on the same host, grants 1 second resolution, and sets log levels to DEBUG
#   {
#       graphitePort: 2003,
#     , graphiteHost: "localhost"
#     , port: 8125
#     , backends: [ "./backends/graphite" ]
#     , dumpMessages: true
#     , log: { level: "LOG_DEBUG" }
#     , flushInterval: 1000
#   }
{% endhighlight %}

Start the daemon:

{% highlight bash %}
$ nodejs /opt/statsd/stats.js /opt/statsd/exampleConfig.js
{% endhighlight %}

#### Validation

To ensure that the statsd instance is correctly receiving metrics and forwarding to the Graphite instance, send
a test metric to the statsd instance:

{% highlight bash %}
# send a counter metric "statsd.test.foo" with value 2 at current time to local statsd instance on port 8125
$ echo "statsd.test.foo:2|c" | nc -u -w0 localhost 8125
# ensure that the counter metric made it to graphite - this command is run on the whisper storage instance
# which should be localhost (same as statsd) if these instructions were followed
whisper-fetch.py /opt/graphite/storage/whisper/stats_counts/statsd/test/foo.wsp
# should output a bunch of metrics for time series, including value sent above
{% endhighlight %}

### Credit

Contributions to some of the above were gleaned from:

* [Statsd](https://github.com/etsy/statsd/wiki)

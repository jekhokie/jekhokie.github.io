---
layout: post
title:  "Collectd to Graphite/Statsd"
date:   2016-09-26 17:31:41 -0400
categories: collectd graphite statsd installation configuration
logo: collectd.jpg
---
Simple installation/configuration of [collectd](https://collectd.org/) to send metrics/data to a
[statsd](https://github.com/etsy/statsd/wiki) endpoint for visualization.

### Background

In many cases, it is useful to collect system metrics to performance baseline the system on which
your software is running. With it's multitude of plugins (and ability to extend via writing new
plugins), [collectd](https://collectd.org/) is a great daemon to configure and run on your instance,
and through the [write_graphite](https://collectd.org/wiki/index.php/Plugin:Write_Graphite) plugin
can be configured to send its data to a [Graphite](http://graphiteapp.org/) or
[statsd](https://github.com/etsy/statsd/wiki) endpoint.

### Installation and Configuration

The following steps were performed on an Ubuntu 16.04 instance.

Use the native package management system to install the collectd daemon:

{% highlight bash %}
$ sudo apt-get update
$ sudo apt-get install collectd
{% endhighlight %}

Once the package is installed, edit the configuration file to enable and configure the Graphite/statsd
connectivity:

{% highlight bash %}
$ sudo vim /etc/collectd/collectd.conf
# note that there are likely many items enabled/configured, but ensure the following are configured
# for the best experience when visualizing metrics/data:
#   ...
#   Hostname 'some_host'
#   FQDNLookup false
#   ...
#
# Un-comment (enable) the following plugins as a good starting point - note that 'write_graphite' is required:
#   ...
#   LoadPlugin cpu
#   LoadPlugin load
#   LoadPlugin memory
#   LoadPlugin processes
#   LoadPlugin write_graphite
#   ...
#
# Set the respective configuration for the write_graphite plugin - note "tcp" in this case (statsd)
#   <Plugin write_graphite>
#       <Node "some_host">
#           Host "graphite_or_statsd_hostname_or_ip"
#           Port "2003"
#           Protocol "tcp"
#           LogSendErrors true
#           Prefix "collectd"
#           Postfix "collectd"
#           StoreRates true
#           AlwaysAppendDS false
#           EscapeCharacter "_"
#       </Node>
#   </Plugin>
{% endhighlight %}

Once the configuration has been set, start or restart the collectd service (if already running):

{% highlight bash %}
$ sudo service collectd start
{% endhighlight %}

If you suspect errors/issues with the configuration or service (i.e. if data is not showing up at your
graphite/statsd endpoint for visualization), you can investigate the instance `syslog` which contains
any errors that might occur with the collectd service/configuration:

{% highlight bash %}
$ sudo tail -f /var/log/syslog
{% endhighlight %}

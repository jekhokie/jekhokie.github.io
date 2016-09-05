---
layout: post
title:  "Useful JMX/JConsole Information"
date:   2013-04-21 13:16:48 -0400
categories: linux osx java jmx jconsole
---
Somewhat related to an earlier Java post, this post is dedicated to useful JConsole information
that does not warrant their own posts (too short, not enough information, etc). As before,
**Platform** is specified to indicate the platform the process was tested on, but the process will
very likely work on other platforms as well.

## Enable Secure JMX Monitoring

**Platform**: Linux

JMX monitoring is useful in troubleshooting and profiling applications. However, it can be dangerous
to do so without securing the port the JMX interface is listening on. These instructions detail how
to expose JMX monitoring and secure it using a simple username/password scheme. The instructions assume
that you have deployed a Tomcat application to the directory /opt/tomcat and the contents of the Tomcat
directory are default/out of box. The setup will enable JMX monitoring according to the following setup:

* Port: 1099
* Protocol: Non-SSL
* Authentication: Basic Username/Password
* Username: Stored in file /opt/tomcat/current/conf/jmx.access
* Password: Stored in file /opt/tomcat/current/conf/jmx.password
* Roles: Monitor (read-only), Control (read/write)

1. Edit the JAVA environment settings to add/adjust the required settings for secure JMX monitoring:

{% highlight bash %}
$ sudo vim /opt/tomcat/current/bin/setenv.sh
# add/edit the following JMX-related settings
#   JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote"
#   JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.port=1099"
#   JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.ssl=false"
#   JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.authenticate=true"
#   JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.access.file=/opt/tomcat/current/conf/jmx.access"
#   JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote.password.file=/opt/tomcat/current/conf/jmx.password"
{% endhighlight %}

2. Add/edit a JMX Access file with the following content:

{% highlight bash %}
$ sudo vim /opt/tomcat/current/conf/jmx.access
# add/edit the following JMX access settings
#   monitorRole readonly
#   controlRole readwrite
{% endhighlight %}

3. Add/edit a JMX Password file with the following content (obviously replacing `<MONITOR_PASSWORD>` and
`<CONTROL_PASSWORD>` with your respective settings):

{% highlight bash %}
$ sudo vim /opt/tomcat/current/conf/jmx.password
# add/edit the following JMX password settings
#   monitorRole <MONITOR_PASSWORD>
#   controlRole <CONTROL_PASSWORD>
{% endhighlight %}

4. Modify the permissions for the respective JMS access/password files. This is required in order for the JVM
to respect usage of the files (failure to do this will result in a Java Error on Tomcat start):

{% highlight bash %}
$ sudo chmod 600 /opt/tomcat/current/conf/jmx.access
$ sudo chmod 600 /opt/tomcat/current/conf/jmx.password
{% endhighlight %}

5. Restart the Tomcat process to enable the JMX monitoring according to the configuration files above:

{% highlight bash %}
$ sudo /etc/init.d/tomcat restart
{% endhighlight %}

## Tomcat JMX in Vm - Connection Refused Error

**Platform**: Linux

When configuring JMX for Tomcat, attempting to access JMX using JConsole on port 4231 (default) may fail due
to a connection error. If this happens, it is possible that Tomcat has bound the JMX interface to the loopback
adapter. To resolve this, update the configuration to bind the JMX interface to either all interfaces or the
IP address of the host instance (commands below assume your Tomcat is installed to `/opt/tomcat`):

{% highlight bash %}
$ sudo vim /opt/tomcat/current/bin/setenv.sh
# ensure the following line is specified - this configuration binds JMX as available via the VM's IP address
#   CATALINA_OPTS="${CATALINA_OPTS} -Djava.rmi.server.hostname=10.11.13.14"
$ sudo /etc/init.d/tomcat restart
{% endhighlight %}

Following the aboe command, JMX will be accessible (assuming the default port) via `10.11.13.14:4231`.

## JConsole Logging

**Platform**: Linux

To enable more verbose debug logging for JConsole, create a `logging.properties` file:

{% highlight bash %}
$ sudo vim logging.properties
# add/adjust the following, as an example
#    handlers = java.util.logging.ConsoleHandler
#    .level = INFO
#    ...
#    java.util.logging.FileHandler.pattern = %h/java%u.log
#    java.util.logging.FileHandler.limit = 50000
#    java.util.logging.FileHandler.count = 1
#    java.util.logging.FileHandler.formatter = java.util.logging.XMLFormatter
#    ...
#    java.util.logging.ConsoleHandler.level = FINEST
#    java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter
#    ...
#    javax.management.level = FINEST
#    javax.management.remote.level = FINEST
{% endhighlight %}

Once the above file has been created, reference it when launching JConsole to have log files created with
respect to the operation of JConsole:

{% highlight bash %}
$ jconsole -J-Djava.util.logging.config.file=logging.properties
{% endhighlight %}

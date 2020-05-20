---
layout: post
title:  "Useful Java Information"
date:   2013-03-22 11:02:18 -0400
categories: linux osx java jnlp
logo: java.jpg
---
A collection of useful information related to Java debugging, troubleshooting, and general use of
products that run in a JVM. None of the commands/procedures warrant their own post (too short), so
this simply aggregates a bunch of useful blips of information into a single post. **Platform** is
specified to indicate the platform the process was tested on, but the process will very likely work
on other platforms as well.

### Enable Logging/Tracing for Java Web Start Application

**Platform**: OSX, Linux

To enable the logging/tracing capability of a Java Web Start application, two different procedures
can be used (may be more, these are the most useful I've found). After performing one of the following
procedures and starting the application, log files and trace logging should be able to be found in
the following directory: `~/.java/deployment/log/*`

#### Option 1: Control Panel (System-Level Enable)

This procedure will enable logging/trace for the system Java (all applications that are run):

{% highlight bash %}
$ cd /usr/local/java/jdk/bin/
$ ./ControlPanel
# navigate to Advanced -> Debugging
#   select "Enable Tracing" and "Enable Logging"
{% endhighlight %}

#### Option 2: Application Configuration (Application-Specific Enable)

This procedure will enable logging/trace for a specific application:

{% highlight bash %}
$ vim <your_app>/.java/deployment/deployment.properties
# add or edit the following properties:
#   deployment.javapi.lifecycle.exception=true
#   deployment.trace=true
#   deployment.log=true
{% endhighlight %}

### Error Attempting to Run JNLP

**Platform**: OSX

When attempting to run a JNLP file, if clicking the file opens the Apple Store or displays messages
such as "Bad Installation. No JRE found in configuration file", then run the following command to
fix the issue:

{% highlight bash %}
$ sudo /usr/libexec/PlistBuddy -c "Delete :JavaWebComponentVersionMinimum" /System/Library/CoreServices/CoreTypes.bundle/Contents/Resources/XProtect.meta.plist
{% endhighlight %}

### Switching Java Version on Mac OSX

**Platform**: OSX

Upgrading Java on Mac OSX can be somewhat cumbersome. Upon one attempt, I installed a newer version of the
Java SE and changed the symlink in `/System/Library/Frameworks/JavaVM.framework/Versions` to point to the
newer version (rather than `A`):

{% highlight bash %}
$ sudo rm /System/Library/Frameworks/JavaVM.framework/Versions/Current
$ sudo ln -s /System/Library/Frameworks/JavaVM.framework/Versions/1.7/System/Library/Frameworks/JavaVM.framework/Versions/Current
{% endhighlight %}

After doing so and attempting to run an application, I received the following response from Java:

{% highlight bash %}
$ java MyExample
#   2012-08-03 11:40:12.051 java[1035:10b] Apple AWT Startup Exception : *** - [NSCFArray insertObject:atIndex:]: attempt to insert nil
#   2012-08-03 11:40:12.072 java[1035:10b] Apple AWT Restarting Native Event Thread
{% endhighlight %}

To address this, I needed to re-configure the symlink contained within the Java VM frameworks directory such
that it would continue to report to the `A` version:

{% highlight bash %}
$ sudo rm /System/Library/Frameworks/JavaVM.framework/Versions/Current
$ sudo ln -s /System/Library/Frameworks/JavaVM.framework/Versions/A /System/Library/Frameworks/JavaVM.framework/Versions/Current
{% endhighlight %}

I then used the Java Preferences Application to update the current version of Java, and updated my .bash_profile
to include the following:

{% highlight bash %}
$ sudo vim ~/.bash_profile
# include the following lines:
#   export JAVA_HOME=/System/Library/Frameworks/JavaVM.framework/Versions/1.7/Home
#   export PATH=$JAVA_HOME/bin:$PATH
{% endhighlight %}

Case and point - don't update your Java version by hand on OSX - the system Java installed and managed is not
as flexible to manipulate as it would be on a *nix system.

### Keystore for Java - Re-import with Updated Key Password

**Platform**: Linux

To manage Java keystores, the following commands are useful for getting certificates/keys and updating passwords:

{% highlight bash %}
$ updatedb
# update the cert store
$ locate cacerts
# inspect to figure out which Java version is being used (copy the cacerts directory)
$ keytool -list -keystore "<CACERTS_DIR>"
# list all certs using the CACERTS_DIR from prior command
# take the key desired and store in a file <KEY_NAME>.asc
$ keytool -import -noprompt -trustcacerts -alias 'My Cert' -file <KEY_NAME>.asc -keystore "<CACERTS_DIR>" -storepass "<NEW_PASS>"
# re-import the key/certificate with new password
{% endhighlight %}

### Java Heap Dump

**Platform**: Linux

To perform a Java Heap dump (for investigating memory):

{% highlight bash %}
$ sudo -u <USER> jmap -dump:format=b,file=/tmp/heapdump.bin <PROCESS_ID>
# create a heap dump binary file using the user <USER> as /tmp/heapdump.bin for the <PROCESS_ID> specified
{% endhighlight %}

---
layout: post
title:  "X11 Forwarding from Mac OSX to Linux"
date:   2013-02-22 11:14:01 -0400
categories: linux x11 osx ssh
logo: x11.jpg
---
To enable GUI interaction with a Linux VM over SSH from a Mac OSX instance, X11 forwarding can be used.
The following are steps to install, configure, and run tools/commands related to establishing GUI sessions
with remote Linux instances (assuming they allow it/X11 forwarding is not explicitly disallowed by the SSH
settings on the VM or a remote firewall).

### OSX Software Installation

X sessions can be displayed on a Mac OSX instance using [XQuartz](https://www.xquartz.org/). First,
download and install XQuartz using the supplied method [here](https://www.xquartz.org/).

### Remote Linux SSH Configuration

To enable X11 forwarding, the remote Linux system must have its SSH configurations adjusted to
explicitly allow it (if not already configured). Edit the file /etc/ssh/ssh_config to ensure the
following lines are specified/set as follows:

{% highlight bash %}
$ sudo vim /etc/ssh/ssh_config
# Ensure the file contains the following:
# Allow any host to connect (can be refined for security)
#   Host *
# Allow X11 forwarding
#   ForwardX11 yes
# Ensure trust is established
#   ForwardX11Trusted yes
{% endhighlight %}

Following the configuration update, restart the SSH daemon:

{% highlight bash %}
$ sudo service sshd restart
{% endhighlight %}

### Establish the Connection

Once the above SSH configurations are set, a session can be established from the Mac OSX instance
to the remote Linux instance. Run the following from the Mac OSX instance:

{% highlight bash %}
$ ssh -X <user>@<remote_host>
{% endhighlight %}

At this point, the XQuartz application should automatically launch (if not already running) and
a remote session should be established/you should see the desktop of the Linux instance. If no
GUI was pre-installed on the remote Linux system, the session may just be established and waiting
for an application with GUI capability to open. Open any application (such as 'xclock') and the
GUI application should appear on your OSX desktop.

### Optional/Troubleshooting

The following are potential optional components, depending on what the distro comes pre-installed with.

{% highlight bash %}
# additional auth options
$ sudo yum -y install xorg-x11-xauth

# if fonts are missing when the session is established
$ sudo yum -y install xorg-x11*
{% endhighlight %}

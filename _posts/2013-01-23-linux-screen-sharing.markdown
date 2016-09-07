---
layout: post
title:  "Linux Screen Sharing Using Screen"
date:   2013-01-23 13:28:31 -0400
categories: linux screen
---
To share a screen session (screen is a Linux-based software package which allows multiple TTY sessions
under the same console window) with a fellow developer/other, the following commands can be useful.

## Software Installation/Configuration

Install the software package:

{% highlight bash %}
$ sudo yum install screen
{% endhighlight %}

Set up appropriate permissions:

{% highlight bash %}
$ sudo chmod u+s /usr/bin/screen
$ sudo chmod 755 /var/run/screen
{% endhighlight %}

## Screen Session Initialization

As a user wishing to start the screen session, create a screen session with a given name:

{% highlight bash %}
$ screen -S <screen_name>
{% endhighlight %}

Configure the share/join options for the joining user(s), which include enabling multiuser mode
and granting access to the screen session to the joining user explicitly:

{% highlight bash %}
# press the 'CRTL+a', ':' keys to enter command mode, then type:
multiuser on <enter>
# press the 'CRTL+a', ':' keys to enter command mode, then type:
acladd <joining_user_name>
{% endhighlight %}

## Screen Session Joining

As the joining user, run the following command to join a shared screen session:

{% highlight bash %}
$ screen -x <joining_user_name>/<screen_name>
# <screen_name> should be the name of the screen given by the initializing user
{% endhighlight %}

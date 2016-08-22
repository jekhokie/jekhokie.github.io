---
layout: post
title:  "Random Useful Commands"
date:   2013-01-03 08:14:08 -0400
categories: random linux ruby osx
---
This post is a bunch of random useful commands, data points, installation instructions, etc. that I've collected over the last several years. The data contained within this post are things that I found useful at the time, were difficult to find/fix, or are generally just useful sysadmin commands to keep around for historical lookup purposes.

## Ruby

### therubyracer dependencies

Specify the following in the gemfile, followed by bundle install:

{% highlight ruby %}
# in Gemfile
gem 'libv8'
gem 'therubyracer'
{% endhighlight %}

## Linux

### Missing C Compiler

In many cases, installing libraries and software requires compiling - if errors are thrown about
a missing compiler, usually this can be resolved via the following (assuming a CentOS-like system -
similar methodology can be followed for other types of operating systems, using the respective
package manager for the system).

{% highlight bash %}
$ yum install gcc gcc-c++
$ export CC=/usr/bin/gcc
# it is usually good to put the above in your .bash_profile file
{% endhighlight %}

At this point, you should be able to re-run your command and the compile commands should be able
to use the `CC` environment variable successfully.

### Soft and Hard ulimits

To list ulimit settings (global):

{% highlight bash %}
# list hard limits:
$ sudo ulimit -Hn
# list soft ulimits:
$ sudo ulimit -Sn
{% endhighlight %}

To set ulimits explicitly (i.e. for a user or group):

{% highlight bash %}
$ vim /etc/security/limits.d/<limit_name>.conf
# place the contents of the limit in this file
{% endhighlight %}

## OSX

### "No Devices Detected" for Wifi

Very infrequently, I lost my internet connection and (seemingly) my entire wireless network adapter.
Errors such as "No Devices Detected" that show up when attempting to connect to a wifi network were
showing up routinely. To address this, I followed [this]() article to reset the System Management
Controller.

1. Shut down the Mac.
2. Press the (left) Shift + Control + Option + Power Button keys at the same time.
..*. Wait a second or two.
3. Release all the keys at the same time.
4. Turn on the Mac using the normal power button.

This should reset the internal SMC and result in being able to connect to wifi networks.

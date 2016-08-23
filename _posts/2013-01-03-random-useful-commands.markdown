---
layout: post
title:  "Random Useful Commands"
date:   2013-01-03 08:14:08 -0400
categories: random linux ruby osx tcpdump java jnlp memory virtualization vmware
---
This post is a bunch of random useful commands, data points, installation instructions, etc. that I've
collected over the last several years. The data contained within this post are things that I found useful
at the time, were difficult to find/fix, or are generally just useful sysadmin commands to keep around
for historical lookup purposes.

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

### tcpdump

{% highlight bash %}
# monitor all requests except port 22
$ sudo tcpdump not port 22
{% endhighlight %}

### VirtualBox Command-Line

Command-line commands to interact with the VirtualBox application.

{% highlight bash %}
# list all VMs
$ VBoxManage list vms
# list all runing VMs
$ VBoxManage list runningvms
# power down (hard) a VM
$ VBoxManage controlvm <VM_ID> poweroff
# delete a VM
$ VBoxManage unregistervm <VM_ID> --delete
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

### Continuous Re-Prompt for Password

When continuously being prompted for a password for the Local/Login keychain, perform the
following to fix:

{% highlight bash %}
# remove the keychain cache
$ sudo rm -rf ~/Library/Keychains/<LONG_UUID>
# reboot
$ sudo reboot
{% endhighlight %}

### Error Attempting to Run JNLP

When attempting to run a JNLP file, if clicking the file opens the Apple Store or displays messages
such as "Bad Installation. No JRE found in configuration file", then run the following command to
fix the issue:

{% highlight bash %}
$ sudo /usr/libexec/PlistBuddy -c "Delete :JavaWebComponentVersionMinimum" /System/Library/CoreServices/CoreTypes.bundle/Contents/Resources/XProtect.meta.plist
{% endhighlight %}

### Unusually High Memory Consumption

To force a garbage collection on OSX if the memory seems to be running unnecessarily high:

{% highlight bash %}
$ sudo purge
{% endhighlight %}

### Re-Install VirtualBox with Extensions

To re-install a VirtualBox setup on OSX.

1. Uninstall any existing VirtualBox instances using the Uninstall script that comes with the .dmg.
2. Reboot the machine.
3. Run the installer for the new VirtualBox version.
4. Download the corresponding extension packs.
5. Install the extension packs from the command line:

{% highlight bash %}
$ VBoxManage extpack install Downloads/Oracle_VM_VirtualBox_Extension_Pack-4.2.10-84104.vbox-extpack
{% endhighlight %}

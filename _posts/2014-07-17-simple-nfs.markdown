---
layout: post
title:  "Simple NFS"
date:   2014-07-17 09:31:28 -0400
categories: linux nfs
---
Simple tutorial (very basic) on how to install and configure NFS for sharing a file system
between two hosts. This tutorial does not go into detail about the many capabilities of
NFS but, rather, is a proof of concept exercise.

### Process

To configure NFS for file system sharing between hosts, assume that we have two hosts
`HOSTA` and `HOSTB`. Note that this process was tested on a CentOS system, but can likely
be utilized on other operating systems with the correct substitution of commands.

On `HOSTA`:

{% highlight bash %}
$ sudo yum install nfs-utils nfs-utils-lib
# only required if you wish to ensure NFS starts on boot
$ sudo chkconfig nfs on
$ sudo service rpcbind start
$ sudo service nfs start
$ sudo mkdir /opt/shared_dir
$ sudo vim /etc/exports
# ensure the file contains the following line
#   /opt/shared_dir <HOSTB_IP>(rw,no_root_squash)
$ sudo exportfs -a
{% endhighlight %}

On `HOSTB`:

{% highlight bash %}
$ sudo yum install nfs-utils nfs-utils-lib
$ sudo mkdir /opt/shared_dir
$ sudo vim /etc/fstab
# ensure the file contains the following line
#   <HOSTA_IP>:/opt/shared_dir /opt/shared_dir nfs soft,intr,nosuid 0 0
$ sudo mount /opt/shared_dir
{% endhighlight %}

At this point, you should have a fully-functioning NFS share between the two hosts. To
test, create a file on one host, and inspect that the file is also visible on the second
host.

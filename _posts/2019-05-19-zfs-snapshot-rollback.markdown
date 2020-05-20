---
layout: post
title:  "Snapshot Rollbacks with ZFS"
date:   2019-05-19 13:47:01 -0400
categories: ubuntu linux zfs snapshot
logo: zfs.jpg
---
In creating a CI pipeline, often times the data layer/availability is the trickiest to handle regarding preparing
for tests. Seeding a database with the right data for starting tests and refreshing it when new/other tests are run
is cumbersome and often times requires customized scripts with long wait times for schema drop and data injection.
There are a few data virtualization technologies out there today (i.e. [Delphix](https://www.delphix.com/),  etc.).
If you're looking to just experiment with what a data virtualization technology could afford you, a good place to
start is to take your already existing customized scripts for seeding/re-injecting data and modifying them to use
the [Z file system](https://www.freebsd.org/doc/handbook/zfs.html) with its snapshot capability. This tutorial
attempts to explain some basic concepts behind ZFS using standard file create/rollback using snapshots, which could
further be expanded to database functionality.

### Technology Layout

This tutorial utilizes/makes the following assumptions about your compute infrastructure. The tutorial will be
basic and focused on an Ubuntu 17.04 VM running within VirtualBox with the following specifications:

- **Hypervisor Technology**: VirtualBox
- **Provisioner**: Vagrant
- **Number of VMs**: 1
- **Hostname**: zfs-test.localhost
- **Operating System**: Ubuntu 16.04
- **Arch**: 64-bit
- **CPUs**: 2
- **Mem**: 2GB
- **Disk**: 50GB

**NOTE**: All commands in this tutorial need to be run as the root user.

### Useful Terms

To better help with this tutorial, some initial definitions are specified:

**ZFS**: File system incorporating volume manager and advanced filesystem funtionality.
**Virtual Device**: Virtual disk on an operating system as though it were an externally-mounted device.
**Virtual Pool**: Collection of virtual devices.
**Dataset**: A file system on top of virtual pool devices.

### Setting Up ZFS

First, log into the Ubuntu VM and update your repositories, then install ZFS:

{% highlight bash %}
# install zfs
$ apt-get update
$ apt-get -y install zfs

# check zfs version
$ zfs --version
{% endhighlight %}

We'll now create a virtual device to experiment with ZFS, and add it to a new virtual pool:

{% highlight bash %}
$ truncate -s 500MB /tmp/disk1.img
$ zpool create -f zp1 /tmp/disk1.img
$ zpool status
# should display the new zpool details
{% endhighlight %}

Finally, let's create a file dataset (filesystem) within the pool:

{% highlight bash %}
$ zfs create -o mountpoint=/mnt/testfs zp1/testfs
$ zfs list -r zp1/testfs
# should show the newly-created dataset
{% endhighlight %}

### Snapshot and Rollback

Now that we have a ZFS setup configured, let's create a sample file with some contents. We will snapshot the file
system, then make some changes to the file, and roll it back:

{% highlight bash %}
# navigate to the file system
$ cd /mnt/testfs

# create a file with some content
$ echo "Original content" > somefile.txt

# snapshot the file system
$ zfs snapshot zp1/testfs@snapshot1

# list snapshots to show the snapshot taken
$ zfs list -t snapshot -r zp1/testfs

# make some modifications to the file and validate they are made
$ echo "changes" > somefile.txt
$ cat somefile.txt
# should output "changes"

# revert back to the original snapshot
$ zfs rollback zp1/testfs@snapshot1
{% endhighlight %}

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Using ZFS to Snapshot Your Database](http://labs.qandidate.com/blog/2014/08/25/using-zfs-to-snapshot-your-database/)
* [ZFS Features and Terminology](https://www.freebsd.org/doc/handbook/zfs-term.html)
* [Querying ZFS File System Information](https://docs.oracle.com/cd/E19253-01/819-5461/gazsu/index.html)
* [Rolling Back a ZFS Snapshot](https://docs.oracle.com/cd/E19253-01/819-5461/gbcxk/index.html)
* [Displaying and Accessing ZFS Snapshots](https://docs.oracle.com/cd/E36784_01/html/E36835/gbiqe.html)

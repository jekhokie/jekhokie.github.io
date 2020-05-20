---
layout: post
title:  "Adding a Drive to a Vagrant VM"
date:   2013-07-11 16:23:18 -0400
categories: linux vagrant vm virtualbox
logo: virtualbox.jpg
---
Often times I've created Virtual Machines using [Vagrant](https://www.vagrantup.com/), but
simply forgot to associate an additional disk/volume for storing information. This is a quick
process to attach a volume to an already-created VM that was created using Vagrant and lives
in the [VirtualBox](https://www.virtualbox.org/) hypervisor.

### Process

These steps provide one way to attach an external volume to an existing virtual machine
created using Vagrant and running inside VirtualBox, and format it for use. Note that these
instructions will likely work for non-Vagrant created VMs as well, assuming you switch the
technologies specified.

Stop the running VM using Vagrant commands:

{% highlight bash %}
$ vagrant halt <VM_NAME>
{% endhighlight %}

In VirtualBox, add a new disk image to the VM. Follow the instructions on the VirtualBox
site to perform this activity.

Start the VM using Vagrant commands:

{% highlight bash %}
$ vagrant up <VM_NAME>
{% endhighlight %}

Log in to the VM and look for the new disk image:

{% highlight bash %}
$ vagrant ssh <VM_NAME>
# once logged in...
$ sudo fdisk -l
# typically, the newly-attached drive would be something like /dev/sdb
{% endhighlight %}

Create the file system on the partition (assuming the drive is /dev/sdb). This command
initializes the drive with an EXT3 file system - swap out the file system type for your
desired configuration:

{% highlight bash %}
$ sudo mkfs -t ext3 /dev/sdb
{% endhighlight %}

Add the partition as a permanent mount point - note that this step mounts the drive
to the /opt partition:

{% highlight bash %}
$ sudo vim /etc/fstab
# add something like the following and save the file
#   /dev/sdb   /opt   ext3   defaults   0   0
{% endhighlight %}

Reboot the VM.

Once the VM has finished booting, inspect to ensure that the new drive is mounted
and accessible for use:

{% highlight bash %}
$ sudo df -h
# should show /dev/sdb mounted to /opt
{% endhighlight %}

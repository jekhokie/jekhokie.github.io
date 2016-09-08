---
layout: post
title:  "Adding Space to Existing LVM Setup"
date:   2015-01-08 07:12:31 -0400
categories: linux lvm disk
---
Useful commands/hints in extending an LVM partition to add disk space to an existing
virtual machine. This is especially useful if you do not wish to add an extra drive
or have software/data that is being written to an existing partition that is not easily
re-configurable.

## Useful Commands

Some useful commands related to this type of task:

{% highlight bash %}
# show logical volumes
$ sudo lvdisplay

# show volume groups
$ sudo vgdisplay

# show physical volumes
$ sudo pvdisplay

# show disks/layout
$ sudo fdisk -l

# format disk
$ sudo fdisk /dev/sdb
{% endhighlight %}

## Process

**Note**: The following process assumes the following:

* Newly added volume size is 12GB (found using "fdisk -l")
* Newly-added volume is mounted as "/dev/sdb" (found using "fdisk -l")
* Volume group is "centos" (found using "vgdisplay")
* Logical volume to extend is "/dev/centos/root" (found using lvdisplay)

The following are steps that are useful to follow if the above are true:

{% highlight bash %}
# create a physical volume out of the newly-attached volume
$ sudo pvcreate /dev/sdb

# add the physical volume to the volume group
$ sudo vgextend centos /dev/sdb

# add space from volume group to logical volume
$ sudo lvextend -L+11.9G /dev/centos/root

# make the space available/visible to the operating system
# Note: Next step depends on the partition type:
# CONDITION 1: DOS PARTITION
$ sudo yum install xfsprogs.x86_64
$ sudo xfs_growfs /dev/centos/root

# CONDITION 2: EXT PARTITION
$ sudo resize2fs /dev/centos/root
###

# reboot the virtual machine
$ sudo reboot

# once rebooted, view the new space on the mount - should show ~12GB of extra space
$ sudo df -h
{% endhighlight %}

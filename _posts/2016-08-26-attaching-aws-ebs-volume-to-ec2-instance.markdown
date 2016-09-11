---
layout: post
title:  "Attaching AWS EBS Volume to EC2 Instance"
date:   2016-08-26 19:28:21 -0400
categories: ubuntu aws ebs ec2 volume
---
Quick instructions on how to attach a secondary elastic block store GP2 AWS volume to an
existing Ubuntu AWS EC2 instance.

### Process

First, create a new volume. This is easily done via the AWS console using the following settings:

* Services -> EC2 -> Volumes
* Create Volume with settings:
  * Volume Type: General Purpose SSD (GP2)
  * Size (GiB): 100GB
  * Availability Zone: us-east-1a (zone that your EC2 instance is located in)
* Click "Create"

Once the volume is created (takes a couple minutes), click the check mark next to the volume.
Click the "Actions" drop-down and select "Attach Volume". When the "Attach Volume" dialog pops
up, enter the Instance ID of your EC2 instance (or alternatively start typing the name of your
instance and auto-fill should pre-populate with options). Once your EC2 instance is selected,
a default "Device" option is filled in (i.e. "/dev/sdf"), along with (possibly) a warning related
to the fact that newer Kernels may re-name the device to /dev/xvdf. Once the dialog is filled
out, click "Attach".

Now that the disk is associated with the VM, you need to SSH into your EC2 instance and
make the disk usable. To do this, perform the following steps:

{% highlight bash %}
# identify the volume first - this command will list all volumes, one of which (presumably
# /dev/xvdf) will complain about not having a valid partition table - this is the disk you
# want to target for the following commands
$ sudo fdisk -l

# make a file system on the disk - this example creates an ext4 type file system
$ sudo mkfs -t ext4 /dev/xvdf

# edit the fstab file to add the drive as an auto-mount on reboot
$ sudo vim /etc/fstab
# include the following line, which specifies the disk and the desired mount point
# in this case, the disk will be mounted to /var/lib/appstorage, which must already exist
#    /dev/xvdf  /var/lib/appstorage  auto  defaults,nobootwait 0  0

# manually mount the drive (avoid a reboot)
$ sudo mount -a

# if no errors appear, inspect the mount to ensure that it shows up the way you expect
# it to, including available space
$ df -h
# should show something along the following lines
#   /dev/xvdf        99G   60M   94G   1% /var/lib/appstorage
{% endhighlight %}

Note that depending on your settings, you may need to enable write access to the newly-
mounted disk/drive - this is entirely dependent on your setup/configuration.

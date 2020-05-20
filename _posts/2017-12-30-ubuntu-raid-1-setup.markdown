---
layout: post
title:  "Ubuntu RAID 1 Setup"
date:   2017-12-30 20:45:23 -0400
categories: ubuntu linux raid sysadmin
logo: ubuntu.jpg
---
How to set up RAID 1 (redundant copy) on an Ubuntu 17.04 instance.

### Background

This post is a refresher on how to configure a RAID 1 device on an Ubuntu 17.04 instance running in a
VirtualBox hypervisor to allow for ease of configuring additional physical devices for use. As a
reminder, RAID 1 is a redundant copy of all bytes on one drive to other drives in the array, meaning
the total usable space is the size of the first array in the disk as configured.

### Technology Ecosystem

There is a single host for this tutorial containing two additional physical drives for use:

**Ubuntu VM**

- **Hostname**: raidtest.localhost
- **OS**: Ubuntu 17.04
- **CPU**: 2
- **RAM**: 2048MB
- **Disk 1 (OS)**: 20GB
- **Disk 2**: 20GB
- **Disk 3**: 20GB
- **Network**: Private Network
- **IP**: 10.11.13.15

### Create and Attach Storage

As noted above, the Ubuntu 17.04 instance has one disk for the Ubuntu Operating System plus two
additional drives that will be used for the RAID configuration. Within VirtualBox, create two
VMDK hard disks and attach them to the VM prior to booting the VM. For the purposes of this
tutorial, it is acceptable to create each drive as 2GB in size, dynamically allocated.

These drives will serve as the drives for configuring the RAID array.

Once the drives are created/mounted, boot the VM.

### Create RAID 1 Array

First, let's install `mdadm`, the tool that is used for creating RAID arrays:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo apt-get install mdadm
{% endhighlight %}

Now that the VM is booted with the additional drives and the RAID tool is installed, let's inspect
to ensure the drives are present/available as we expect:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo lsblk
# should output something similar to the following:
#   NAME                        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
#   ...
#   sdb                           8:16   0    2G  0 disk
#   sdc                           8:32   0    2G  0 disk
#   ...
{% endhighlight %}

Note from the output that the additional drives we created/attached are identified as `sdb` and
`sdc`, each 2GB in size. Now that we know the device paths, let's create the RAID 1 array as
`/dev/md0`:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
# when prompted, type “Y” and press “ENTER”.
{% endhighlight %}

The array device should be created - let's inspect to ensure:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo cat /proc/mdstat
# should output something simliar to the following:
#   Personalities : [raid1]
#   md0 : active raid1 sdc[1] sdb[0]
#        2095104 blocks super 1.2 [2/2] [UU]
#   unused devices: <none>
{% endhighlight %}

Now let's create a filesystem on the RAID array and mount the filesystem to be usable under the
path `/mnt/md0`:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo mkfs.ext4 -F /dev/md0
$ sudo mkdir -p /mnt/md0
$ sudo mount /dev/md0 /mnt/md0
{% endhighlight %}

Next, validate that the space is available as expected:

{% highlight bash %}
# Host: raidtest.localhost
$ df -h -x devtmpfs -x tmpfs
# should output something similar to the following:
#   Filesystem                         Size  Used Avail Use% Mounted on
#   ...
#   /dev/md0                           2.0G  6.0M  1.9G   1% /mnt/md0
#   ...
{% endhighlight %}

You will note that the total available space is 2GB. As noted previously, RAID 1 is a replication
setup, meaning every bit written to one device will be replicated to all others.

Now that the array is created and mounted, let's ensure a scan reports it is identifiable:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo mdadm --detail --scan
# should output something similar to the following:
#   ARRAY /dev/md0 metadata=1.2 name=raidtest:0 UUID=4ea8351f:fe9bbe55:5dd04a78:588b93f3
{% endhighlight %}

The above indicates the array location and the unique identifier. The UUID is important and will
be referenced later in this document when describing how to recover a failed array device.

For more detailed information about the array specifically, perform the following:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo mdadm --detail /dev/md0
# should output something similar to the following:
#   /dev/md0:
#          Version : 1.2
#    Creation Time : Thu Dec 28 06:56:25 2017
#       Raid Level : raid1
#       Array Size : 2095104 (2046.00 MiB 2145.39 MB)
#    Used Dev Size : 2095104 (2046.00 MiB 2145.39 MB)
#     Raid Devices : 2
#    Total Devices : 2
#      Persistence : Superblock is persistent
#      Update Time : Thu Dec 28 07:04:09 2017
#            State : clean
#   Active Devices : 2
#  Working Devices : 2
#   Failed Devices : 0
#    Spare Devices : 0
#             Name : raidtest:0  (local to host raidtest)
#             UUID : 4ea8351f:fe9bbe55:5dd04a78:588b93f3
#           Events : 17
#
#      Number   Major   Minor   RaidDevice State
#         0       8       16        0      active sync   /dev/sdb
#         1       8       32        1      active sync   /dev/sdc
{% endhighlight %}

Note the details at the bottom of the output, indicating that both drives are present and
functioning in the array.

### Test the Array

Now that we have the array available, let's perform some tests to prove the value of a RAID 1 array.
First, let's write some data to the array and ensure that it persists:

{% highlight bash %}
# Host: raidtest.localhost
$ echo "TEST" | sudo tee -a /mnt/md0/testfile
$ cat /mnt/md0/testfile
# should output:
#   TEST
{% endhighlight %}

Now that we can write data to the array, let's test what would happen if one of the drives failed.
Since we are using RAID 1, we are insulating ourselves from a drive failure given that the premise
behind a RAID 1 array is replication for backup:

{% highlight bash %}
# Host: raidtest.localhost
# fail one of the drives:
$ sudo mdadm --manage --fail /dev/md0 /dev/sdc
# next, ensure the drive is failed:
$ sudo mdadm --detail /dev/md0
# should output something similar to the following:
#   ...
#   Number   Major   Minor   RaidDevice State
#         0       8       16        0      active sync   /dev/sdb
#         -       0        0        1      removed
#         1       8       32        -      faulty   /dev/sdc
#   ...
{% endhighlight %}

The above shows that the `/dev/sdc` drive has been put in a faulty state. Let's check that the file
still exists with the content originally entered:

{% highlight bash %}
# Host: raidtest.localhost
$ cat /mnt/md0/testfile
# should output:
#   TEST
{% endhighlight %}

Next, in order to simulate swapping of a physical drive, let's remove the failed drive:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo mdadm /dev/md0 -r /dev/sdc
# check that the drive is removed:
$ sudo mdadm --detail /dev/md0
# should output something similar to the following:
#   ...
#   Number   Major   Minor   RaidDevice State
#         0       8       16        0      active sync   /dev/sdb
#         -       0        0        1      removed
#   ...
{% endhighlight %}

To ensure integrity, check that the file still exists with the same contents:

{% highlight bash %}
# Host: raidtest.localhost
$ cat /mnt/md0/testfile
# should output:
#   TEST
{% endhighlight %}

Now that the faulty drive is removed, let's simulate re-installing a new drive:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo mdadm /dev/md0 -a /dev/sdc
# check that the new drive is added:
$ sudo mdadm --detail /dev/md0
# should output something similar to the following:
#   ...
#   Number   Major   Minor   RaidDevice State
#          0       8       16        0      active sync        /dev/sdb
#          2       8       32        1      spare rebuilding   /dev/sdc
#   ...
{% endhighlight %}

Note that the above indicates the `/dev/sdc` device is in a 'spare rebuilding' state. This indicates
that the RAID array is copying the file contents over to the new drive, bringing it in sync for the
array. Re-running the command some time later should result in the following:

{% highlight bash %}
# Host: raidtest.localhost
$ sudo mdadm --detail /dev/md0
# should output something similar to the following:
#   ...
#   Number   Major   Minor   RaidDevice State
#          0       8       16        0      active sync   /dev/sdb
#          2       8       32        1      active sync   /dev/sdc
#   ...
{% endhighlight %}

Note that now the drive is in sync and the array is functioning as expected. Let's check the file
contents in the array again just to ensure it is still present/in-tact:

{% highlight bash %}
# Host: raidtest.localhost
$ cat /mnt/md0/testfile
# should output:
#   TEST
{% endhighlight %}

### Ensure Array is Persistent

Now that we have the array, let's ensure that it will be available at all times (including boot time
in case of a restart):

{% highlight bash %}
# Host: raidtest.localhost
$ sudo mdadm --detail --scan
$ sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
$ sudo update-initramfs -u
$ echo ‘/dev/md0 /mnt/md0 ext4 defaults,nofail,discard 0 0’ | sudo tee -a /etc/fstab
{% endhighlight %}

The above ensures that any reboot will result in the RAID array being available.

### Simulate Recovering Drive Contents

Let's now assume that one of the drives has completely failed. Our intent now is to re-establish the
existing/surviving drive as a new array. In order to do this, we will need to establish a new/unique
UUID, mount point, etc.

{% highlight bash %}
# Host: raidtest.localhost
$ sudo mdadm --manage --fail /dev/md0 /dev/sdc
$ sudo mdadm /dev/md0 -r /dev/sdc
$ sudo mdadm --examine /dev/sdc
# grab the UUID from the output and modify a few bits to
# make it unique, using it in this next command
$ sudo mdadm --assemble --run /dev/md1 /dev/sdc --update=uuid --uuid=<UNIQUE_UUID>
$ sudo mkdir /mnt/md1
$ sudo mount /dev/md1 /mnt/md1
{% endhighlight %}

Now that the new drive is attached, let's once again test the file contents to ensure the file and
contents are still available:

{% highlight bash %}
# Host: raidtest.localhost
$ cat /mnt/md1/testfile
# should output:
#   TEST
{% endhighlight %}

As you'll note, the above mounts the surviving disk in a new array for extracting data and/or
creating a brand new array in the case of a recovery.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [How to Create RAID Arrays with mdadm on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-create-raid-arrays-with-mdadm-on-ubuntu-16-04#creating-a-raid-1-array)
* [How to Test a RAID 1 Array](http://www.linuceum.com/Server/srvRAIDTest.php)

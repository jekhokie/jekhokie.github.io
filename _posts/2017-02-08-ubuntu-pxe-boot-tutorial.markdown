---
layout: post
title:  "Ubuntu PXE Boot Tutorial"
date:   2017-02-08 19:15:29 -0400
categories: ubuntu linux pxe automation
logo: ubuntu.jpg
---
Tutorial explaining how to configure an Ubuntu 16.04 server as a
[Preboot Execution Environment (PXE)](https://en.wikipedia.org/wiki/Preboot_Execution_Environment)
server to support automated/unattended provisioning of additional unix-based hosts. Note that this
is generally an old implementation but useful to learn and understand infrastructure components.
This tutorial is an updated set of instructions from my archive of resources and details the
instructions for the latest version of Ubuntu - 16.04.

### Background

A somewhat older but still relevant and interesting concept is the method of PXE booting Unix-based
hosts. In a PXE boot environment, a single Unix-based system is configured to provide several services
and files/configurations to a network-attached resource in order for the resource to automatically
provision itself as a Unix-based host defined by the configurations provided.

The base services and configuration required for a PXE-boot environment are as follows:

- **DHCP Service**: A DHCP service is required to exist in order to auto-allocate an IP address as well
as tell the newly-created resource where to go in order to obtain its initialization RAM disk (initrd).
This file is then loaded into memory and defines/tells the resource what to do next (obtain the files
required to install and configure the operating system).
- **TFTP Service**: This service is used specifically to serve the initialization RAM disk file.
- **Web Service**: This is typically Apache or the like and is used to serve the Operating System
installation and configuration files. Although it is possible to tell the host being provisioned to
retrieve its OS files from an internet source, we are going to assume that provisioning is being done
on an internal network in order to increase the full scope of this tutorial.
- **OS-Specific Files**: The HTTP service listed above serves the operating system files from a
particular location and, as such, the files used to install the Operating System must be present and
available for the web service in a format similar to a disk image layout.
- **DNS Service**: The DNS service will be configured on the PXE server to provide a full name to the
resource(s) being provisioned.

### Technology Ecosystem

It would be most scalable to configure resources within a cloud environment such as Amazon Web
Services (AWS) or the like. However, in the interest of ensuring that network collisions are avoided
(i.e. making sure there are not multiple DHCP broadcasts and such) within the same network, we will
construct the resources for this tutorial using a locally-provisioned set of resources on a laptop
through Oracle's [VirtualBox](https://www.virtualbox.org/wiki/VirtualBox). The following Virtual
Machines will be created in this tutorial:

**PXE Boot Server**

- **OS**: Ubuntu 16.04
- **CPU**: 1
- **RAM**: 1024MB
- **Disk**: 50GB
- **Network**: 1 Adapter attached to "NAT Network"
- **IP**: 10.0.2.15

**Client Instance**

- **OS**: (to be provisioned through PXE server)
- **CPU**: 1
- **RAM**: 1024MB
- **Disk**: 30GB
- **Network**: 1 Adapter attached to "NAT Network"
- **IP**: 10.0.2.121

As can be seen above, each device is attached to "NAT Network". In order for communication to work,
a new NAT Network must be created via VirtualBox and the same NAT Network must be specified for
both hosts. To create the new NAT Network, perform the following in VirtualBox:

1. Navigate to the menu "VirtualBox" -> "Preferences".
2. Select the "Network" tab.
3. Under the "NAT Networks" option, select the "Add new NAT Network" icon (right side of list).
4. Select the newly-created network and click the "Preferences" icon (tool icon).
5. In the dialog that is presented, ensure "Enable Network" is checked, and provide a name and CIDR.
Also, ensure that "Supports DHCP" is checked.

### PXE Server

The first thing to do is to create a Virtual Machine to serve as the PXE boot server. Install the
Ubuntu 16.04 operating system on a new Virtual Machine in VirtualBox with the specifications from
the "Technology Ecosystem" section above. When prompted, specify the following for these relative
sections - all other items can be accepted as defaults:

- **Server Name**: pxeserver
- **User Account**: test
- **Password**: Password123!
- **Packages to Install**: Add "OpenSSH Server" to the selected list

As a setup step, SSH to the instance and perform a repository metadata update:

{% highlight bash %}
$ sudo apt-get update
{% endhighlight %}

#### DHCP Service

To install and configure the DHCP service, which is used for serving IP addresses to the hosts being
automatically provisioned and specifying where to obtain the initrd file:

{% highlight bash %}
$ sudo apt-get install isc-dhcp-server
$ sudo vim /etc/default/isc-dhcp-server
# update the "INTERFACES" declaration to reflect the name of your primary
# network adapter, which can be retrieved via the "ifconfig" command:
#   INTERFACES="enp0s3"

$ sudo vim /etc/dhcp/dhcpd.conf
# add the following to the bottom of the file, replacing "10.0.2.15" with
# the actual IP address of your PXE boot server - assumes your hosts will
# live on the 10.0.2.x network:
#   allow booting;
#   allow bootp;
#   option option-128 code 128 = string;
#   option option-129 code 129 = text;
#   subnet 10.0.2.0 netmask 255.255.255.0 {
#       option routers 10.0.2.1;
#       option broadcast-address 10.0.2.255;
#       option domain-name-servers 10.0.2.15;
#       option domain-name "test123.com";
#       next-server 10.0.2.15;
#       filename "pxelinux.0";
#   }
#
# a reserved IP address is useful for this tutorial given that we have a static
# DNS environment - in order to associate a specific IP with the client MAC,
# obtain the MAC address from the VM settings in VirtualBox under "Network" ->
# "Adapter 1" -> "Advanced" -> "MAC Address", and replace the IP address with
# the expected IP address for this tutorial:`
#   host pxeclient {
#       hardware ethernet 08:00:27:EC:8D:4A;
#       fixed-address 10.0.2.121;
#   }

$ sudo service isc-dhcp-server restart
{% endhighlight %}

#### TFTP Service

The TFTP service is managed by xinetd. To configure it, perform the following:

{% highlight bash %}
$ sudo apt-get install tftp tftpd xinetd
$ sudo vim /etc/xinetd.d/tftp
# add the following to the file:
#   service tftp
#   {
#       disable = no
#       port = 69
#       socket_type = dgram
#       wait = yes
#       protocol = udp
#       user = nobody
#       server = /usr/sbin/in.tftpd
#       server_args = -s /var/lib/tftpboot
#   }

$ sudo service xinetd restart
$ sudo netstat -lu
# should output contents that includes something similar to the following for tftp:
#   udp     0     0 *:tftp     *:*
{% endhighlight %}

Next, attach the Ubuntu 16.04 ISO to the Virtual Machine (can be found
[here](http://releases.ubuntu.com/16.04/)) using the VirtualBox settings for the instance. Once
the ISO has been attached, mount the drive and copy the initialization RAM disk and menu files into
their respective locations:

{% highlight bash %}
$ sudo mkdir /tmp/cdrom
$ sudo mount -o loop /dev/cdrom /tmp/cdrom
$ sudo mkdir /var/lib/tftpboot
$ sudo cp -r /tmp/cdrom/install/netboot/* /var/lib/tftpboot/
$ sudo chmod -R 777 /var/lib/tftpboot
$ sudo chown -R nobody /var/lib/tftpboot
{% endhighlight %}

Finally, configure the `pxelinux.cfg` file for boot options - note that the `append` statement below
contains many options (do not exclude any of them):

{% highlight bash %}
$ sudo vim /var/lib/tftpboot/pxelinux.cfg/default
# ensure the file contains the following, replacing "10.0.2.15" with
# the IP address of your specific PXE server:
#   default linux
#   prompt 0
#   timeout 1
#   label linux
#       kernel ubuntu-installer/amd64/linux
#       append auto=true priority=critical preseed/url=http://10.0.2.15/ubuntu/preseed.seed vga=normal preseed/interactive=false initrd=ubuntu-installer/amd64/initrd.gz -- quiet
{% endhighlight %}

If you wish to test the TFTP service functionality, you can perform the following (again, replace the
IP address 10.0.2.15 with your PXE server IP address):

{% highlight bash %}
$ tftp 10.0.2.15
tftp> get version.info
Received 60 bytes in 0.0 seconds
tftp> quit
{% endhighlight %}

#### Web Service (Apache)

To serve the Operating System files, we will install the Apache web server:

{% highlight bash %}
$ sudo apt-get install apache2
{% endhighlight %}

Next, we will copy the files required for the OS install to the respective directory (assumes the
steps to mount the Ubuntu ISO in the "TFTP Service" steps has been completed/the mount is still
present):

{% highlight bash %}
$ sudo mkdir /var/www/html/ubuntu
$ sudo cp -r /tmp/cdrom/* /var/www/html/ubuntu/
{% endhighlight %}

Now that the OS files are in place, we need to create a preseed file to tell the clients how to
configure their operating systems. You can either use a GUI for this (system-config-kickstart) or
start with a template from the documentation
[here](https://help.ubuntu.com/lts/installation-guide/armhf/apbs04.html). Either way, ensure
the contents of the preseed are placed in a file at the location `/var/www/html/ubuntu/preseed.seed`.
A sample `preseed.seed` file is shown below for reference:

```bash
# /var/www/html/ubuntu/preseed.seed

# Locale
d-i debian-installer/locale string en_US
d-i console-setup/ask_detect boolean false
d-i keyboard-configuration/xkb-keymap select us

# Clock/Time Zone
d-i clock-setup/utc boolean true
d-i time/zone string US/Eastern
d-i clock-setup/ntp boolean true

# Networking
d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string unassigned-hostname
d-i netcfg/get_domain string unassigned-domain
d-i netcfg/wireless_wep string

# Mirrors
d-i mirror/country string manual
d-i mirror/http/hostname string archive.ubuntu.com
d-i mirror/http/directory string /ubuntu
d-i mirror/http/proxy string

# Account
d-i passwd/user-fullname string
d-i passwd/username string test
d-i passwd/user-password password Password123!
d-i passwd/user-password-again password Password123!
d-i user-setup/encrypt-home boolean false

# Partitioning
d-i partman-auto/method string lvm
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
d-i partman-auto/choose-recipe select atomic
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true

# Package Selection
tasksel tasksel/first multiselect minimal
d-i pkgsel/include string openssh-server
d-i pkgsel/upgrade select none
popularity-contest popularity-contest/participate boolean false

# Finish
d-i finish-install/reboot_in_progress true
```

#### DNS Service

In order to ensure the client instances receive a Fully-Qualified Domain Name (FQDN) that is appropriate
to the domain we wish to configure, we will set up a DNS service. We will use BIND as the DNS service,
and set up a domain "test123.com":

{% highlight bash %}
$ sudo apt-get install bind9 dnsutils

$ sudo vim /etc/default/bind9
# ensure the OPTIONS parameter is specified as follows for IPv4 mode:
#   OPTIONS="-4 -u bind"
{% endhighlight %}

Next, we need to configure the zones:

{% highlight bash %}
$ sudo vim /etc/bind/named.conf.local
# ensure the following is present:
#   # forward
#   zone "test123.com" {
#       type master;
#       file "/etc/bind/zones/db.test123.com";
#   };
#
#   # reverse
#   zone "2.0.10.in-addr.arpa" {
#       type master;
#       file "/etc/bind/zones/db.10.0.2";
#   };

$ sudo mkdir /etc/bind/zones

$ sudo cp /etc/bind/db.local /etc/bind/zones/db.test123.com
$ sudo vim /etc/bind/zones/db.test123.com
# update:
# 1. all "localhost" references to reflect "ns1.test123.com".
# 2. "admin.localhost." to be "admin.test123.com.".
# 3. "Serial" value to be 1 more than the current (i.e. "2" becomes "3").
# 4. Delete all records after the SOA block.
# 5. Add the following records/lines to the end of the file (replace "10.0.2.15"
#    with the current IP address of your PXE boot server):
#                              IN    NS   ns1.test123.com.
#   ns1.test123.com.           IN    A    10.0.2.15
#   testclient.test123.com.    IN    A    10.0.2.121
# The file should now look something like this:
#   $TTL    604800
#   @       IN      SOA    ns1.test123.com. admin.test123.com. (
#                                3         ; Serial
#                           604800         ; Refresh
#                            86400         ; Retry
#                          2419200         ; Expire
#                           604800  )      ; Negative Cache TTL
#   ;
#                            IN      NS     ns1.test123.com.
#   ns1.test123.com.         IN      A      10.0.2.15
#   testclient.test123.com.  IN      A      10.0.2.121

$ sudo cp /etc/bind/db.127 /etc/bind/zones/db.10.0.2
$ sudo vim /etc/bind/zones/db.10.0.2
# update
# 1. all "localhost" references to reflect "ns1.test123.com".
# 2. "admin.localhost." to be "admin.test123.com.".
# 3. "Serial" value to be 1 more than the current (i.e. "1" becomes "2").
# 4. Delete all records after the SOA block.
# 5. Add the following records/lines to the end of the file (replace "10.0.2.15"
#    with the current IP address of your PXE boot server):
#   15     IN    PTR    ns1.test123.com.
#   121    IN    PTR    testclient.test123.com.
# The file should now look something like this:
#   $TTL    604800
#   @       IN      SOA    ns1.test123.com. admin.test123.com. (
#                                2         ; Serial
#                           604800         ; Refresh
#                            86400         ; Retry
#                          2419200         ; Expire
#                           604800  )      ; Negative Cache TTL
#   ;
#                            IN      NS     ns1.test123.com.
#   15                       IN      PTR    ns1.test123.com.
#   121                      IN      PTR    testclient.test123.com.
{% endhighlight %}

Once the configuration files have been created, check the syntax of the files, and if all is well,
restart the BIND service:

{% highlight bash %}
$ sudo named-checkconf
# no output should be presented - if any output exists, correct the
# issue prior to starting the named service

$ sudo named-checkzone test123.com /etc/bind/zones/db.test123.com
# should output something similar to the following:
#   zone test123.com/IN: loaded serial 3
#   OK

$ sudo named-checkzone 2.0.10.in-addr.arpa /etc/bind/zones/db.10.0.2
# should output something similar to the following:
#   zone 2.0.10.in-addr.arpa/IN loaded serial 2
#   OK

$ sudo service bind9 restart
{% endhighlight %}

Next, to configure the PXE server to utilize the new DNS service, perform the following:

{% highlight bash %}
$ sudo vim /etc/resolvconf/resolv.conf.d/head
# add the following:
#   search test123.com
#   nameserver 10.0.2.15

$ sudo resolvconf -u
{% endhighlight %}

Once the above has been completed, your PXE server should be configured and ready to resolve IP
addresses for the hosts defined. If you inspect the `/etc/resolv.conf` file, you will notice
that the IP address of the PXE server (now also a name server) is listed at the top of the
nameserver list of IP addresses.

Perform the following to ensure that the forward resolution is working as expected:

{% highlight bash %}
$ nslookup ns1.test123.com
# should output '10.0.2.15' for the IP address

$ nslookup testclient.test123.com
# should output '10.0.2.121' for the IP address
{% endhighlight %}

Next, confirm the reverse lookups:

{% highlight bash %}
$ nslookup 10.0.2.15
# should output 'ns1.test123.com' for the FQDN

$ nslookup 10.0.2.121
# should output 'testclient.test123.com' for the FQDN
{% endhighlight %}

If all commands above work as expected, your DNS environment is now ready for PXE booting.

### PXE Client

Now that the PXE server has been configured, we can create a client to PXE boot. In VirtualBox, create
a new Virtual Machine. Prior to launching the instance, ensure the resource has 1 network adapter
attached to "NAT". Also (*VERY IMPORTANT*) ensure that under the "System" -> "Boot Order" settings, the
boot order for selected devices is:

1. Hard Disk (checked)
2. Network (checked)

Failure to ensure the hard disk is specified first can result in the VM continuously rebooting and PXE
booting itself since it will always receive a DHCP Ack and continue down the PXE route even if it has
a valid Operating System on its hard drive (which it should after the first successful PXE boot
provisioning cycle).

Once the above are complete, start the instance.

Upon starting, several things will happen. The instance will broadcast a DHCP request for an IP address.
If the configurations for the network interface for this instance are correct, it should receive an ACK
from the DHCP server on your PXE boot server. Once an IP address is obtained, it will also make a
request to the TFTP server on the PXE boot server for the initialization RAM disk file, which will kick
off the process.

The `preseed.seed` file will be retrieved and, based on the specifications within, the instance will
start to auto-provision itself based on the descriptions within. Note that there are many other steps
that are occurring between, but for the purposes of this tutorial, those details are out of scope.

Watch the instance as it cycles through the various menu items on the screen and auto-completes the
various parts based on the definition in the seed file. Once complete, your VM will reboot and you can
then log into the instance using the "test" user with the password "Password123!" as defined in the
seed file - your PXE boot environment is now complete!

### Summary

This PXE boot server has been configured with the very basic settings for PXE booting Ubuntu instances.
However, there are likely more configuration items you should address, such as ensuring the DHCP
configurations are appropriate for your environment (instead of a dynamic range of IP addresses in a
single subnet), dynamic DNS to avoid having to map specific IP addresses to MAC addresses, etc. In
addition, certain security issues should be addressed, such as using a hashed version of the user
credentials in the seed file rather than clear-text (which was used in this tutorial to make it simpler
to recall the default password). Overall, however, this tutorial lays the ground work for a PXE boot
environment for auto-provisioning Ubuntu-based instances and can be expanded upon to handle more
complicated scenarios, such as multiple Operating System installations beyond Ubuntu-based systems, etc.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [PXEInstallServer](https://help.ubuntu.com/community/PXEInstallServer)
* [Configure PXE Server in Ubuntu 14.04](https://www.maketecheasier.com/configure-pxe-server-ubuntu/)
* [Preparing Files for TFTP Net Booting](https://www.debian.org/releases/jessie/amd64/ch04s05.html.en)
* [Contents of Preconfiguration File (for Xenial)](https://help.ubuntu.com/lts/installation-guide/armhf/apbs04.html)
* [Automatic Installations of Ubuntu with Preseeding](http://iomem.com/archives/12-Automatic-installations-of-Ubuntu-with-preseeding.html)
* [How to Configure Bind...](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-14-04)
* [Preboot Execution Environment](https://en.wikipedia.org/wiki/Preboot_Execution_Environment)

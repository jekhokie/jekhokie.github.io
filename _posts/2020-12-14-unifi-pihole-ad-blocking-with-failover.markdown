---
layout: post
title:  "Pi-hole Ad-Blocking with Failover on Unifi"
date:   2020-12-14 20:43:00 -0400
categories: unifi pihole networking
logo: pihole.jpg
---

There is a great project known as [Pi-hole](https://pi-hole.net/) that enables ad-blocking features (among other things) that can help both speed
up your browsing experience by blocking page call-outs to ad-based sites and protect browsing history by blocking user-tracking activities. This
article explains how to set up a Pi-hole active/failover pair on Raspberry Pi 3 B+ devices, and configure the devices to function as your home
network primary DNS for ad-blocking within a [Unifi Dream Machine Pro](https://store.ui.com/collections/unifi-network-routing-switching/products/udm-pro).
It will also explain the details behind using a service such as [keepalived](https://linux.die.net/man/8/keepalived) to establish a virtual IP that the
Pi-hole pair will share and manage if and when your primary Pi-hole instance becomes unavailable, and configuration of [gravity-sync](https://github.com/vmstan/gravity-sync)
to keep the secondary Pi-hole instance in sync with your primary Pi-hole configurations in case such a failover occurs so the secondary can simply pick
up where the primary left off, protecting your browsing experience on your home network. Note that this article specifically takes various works of art
from other fantastic community contributors (see Credits in article) and combines them into a specific/opinionated architectural solution - no ownership
of previous work is claimed or assumed.

## Requirements

This tutorial uses 2x Raspberry Pi 3 B+ devices as the primary and secondary Pi-hole instances, but the steps detailed here could be adapted to support
other device-specific implementations of Pi-hole primary/secondary assuming the Pi-hole and gravity-sync functionality support it (e.g. running multiple
docker containers, multiple VMs, separate hardware, etc.).

Additionally, this tutorial includes details about configuring the Pi-hole setup in your Unifi Dream Machine Pro configuration. However, up to this point
in the tutorial, everything is applicable for any network that has the ability to specify a DNS server endpoint and you do not actually need Unifi-specific
gear to reap the benefits of this architecture.

Specifically, this tutorial assumes the following architecture:

* Unifi Dream Machine Pro as core router in network with DHCP server issuing IPs and DNS endpoint IPs to clients.
* 2x Raspberry Pi 3 B+ devices installed with the Raspberry Pi operating system, hard-wired to the Dream Machine Pro or similar connected switch.
* Virtual IP `192.168.1.170` as the DNS IP to be used by the DHCP server issuing client IPs, which will fail over between Pi-hole devices as required.
* 2x static IP addresses `192.168.1.171` and `192.168.1.172` reserved and assigned to the Pi-hole instances for primary and failover, respectively.

It is also assumed that you know how to install the Raspberry Pi OS, enable SSH access to your Raspberry Pi instances, and understand the importance of
following certain provided security tips by the Raspberry Pi documentation (e.g. changing your default `pi` user password, etc.). Those items will not
be covered here.

## Prerequisites

There are several things you should consider doing and installing on your Raspberry Pi devices prior to proceeding. Run the following steps on *both* of
your primary/secondary Pi-hole devices.

First, upgrade all packages to bring your OS up to date with the latest/greatest:

```bash
$ sudo apt-get update
$ sudo apt-get upgrade
```

Next, install some prerequisite packages that gravity-sync depends on:

```bash
$ sudo apt-get install sqlite3 sudo git rsync ssh
```

Next, a little bit of IP management is in order. On whatever device is serving as your DHCP server (e.g. Dream Machine Pro), reserve the 3x IP addresses
indicated in the **Requirements** section above (or whatever IPs you wish to use, just ensure you replace the future content of the tutorial with your
respective IPs). Assign `192.168.1.171` to Raspberry Pi Pi-hole 1 (primary), and `192.168.1.172` to Raspberry Pi Pi-hole 2 (secondary). Once done, you
should likely restart your Raspberry Pi instances to both provide a clean boot with upgraded packages and receive the newly-allocated static IP addresses.

```bash
$ sudo reboot
```

## Installing and Configuring Pi-hole

At the risk of duplicating installation instructions captured by the official documentation [here](https://github.com/pi-hole/pi-hole/#one-step-automated-install),
the following command should be run to install Pi-hole on both Pi-hole devices (as root):

```bash
$ curl -sSL https://install.pi-hole.net | bash
```

Walk through the installation instructions, ensuring (most importantly) that the static IP address for each of the primary/secondary instances is correct
in the configuration when prompted. Once you have completed the setup on each Pi device, take note of the initial login password for the admin interface
and log into [http://192.168.1.171/admin](http://192.168.1.171/admin) and [http://192.168.1.172/admin](http://192.168.1.172/admin) web interfaces for
the primary and secondary Pi-hole instances, respectively, to ensure you can see the initial configuration and to change the login password to something
you're comfortable with.

You're free to now attempt using the primary Pi-hole and creating configurations in the web interface. However, ensure you only manipulate things in the
primary Pi-hole interface as any secondary instance configurations will be overwritten once we finally get gravity-sync installed and configured.

## Installing and Configuring keepalived

Next is our failover functionality for the Pi-hole instances. First, install `keepalived` and its dependencies on each of the Pi-hole instances, and enable
`keepalived` to start on boot of the device:

```bash
$ sudo apt-get -y install keepalived
$ sudo systemctl enable keepalived.service
```

You're likely going to want a static `/etc/hosts` file on each of the Pi-hole instances to avoid any DNS-related failures causing hiccups in your
failover setup. Edit the `/etc/hosts` file on each of the Pi-hole instances to include the 3x IP to host mappings for the Pi-Hole instances and
virtual IP address (should look similar to the following):

```bash
192.168.1.170   pihole.localdomain pihole
192.168.1.171   pihole1.localdomain pihole1
192.168.1.172   pihole2.localdomain pihole2
```

Each of the records should be resolvable on both Pi-hole instances at this point. Next, configure `keepalived` on each of the respective hosts by
editing the configuration file `/etc/keepalived/keepalived.conf`, replacing the password `<SUPERSECRETPASSWORD>` with a password that suits your needs:

```bash
# pihole1.localdomain
vrrp_instance ADBLOCK {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 255
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <SUPERSECRETPASSWORD>
    }
    virtual_ipaddress {
        192.168.1.170/24
    }
}
```

```bash
# pihole2.localdomain
vrrp_instance ADBLOCK {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 254
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <SUPERSECRETPASSWORD>
    }
    virtual_ipaddress {
        192.168.1.170/24
    }
}
```

Once the configuration file is in place, restart the `keepalived` service on each of the Pi-hole instances:

```bash
$ sudo systemctl restart keepalived.service
```

You can inspect the `/var/log/messages` log file on each instance, demonstrating which was elected as `MASTER` vs. which obtained `BACKUP` status, and
also have a look at the IP configurations on each instance to see that the `MASTER` should have the virtual IP allocated while the `BACKUP` should not:

```bash
# pihole1
$ sudo ip -brief addr show

# pihole2
$ sudo ip -brief addr show
```

Additionally, if you attempt to visit [http://192.168.1.170/admin/](http://192.168.1.170/admin/) you should receive a response from the `MASTER` instance,
or if your `curl` library on a device supports specifying `--dns-servers`, you can specify using `192.168.1.170` as the DNS server and you should be able
to see the request/response logs in `/var/log/pihole.log` on the `MASTER` instance.

### Testing Failover

Now that we have both Pi-hole instances configured with `keepalived`, let's simulate a failover/fail-back. On the `MASTER` Pi-hole instance, run the
following to turn off `keepalived`:

```bash
$ sudo systemctl stop keepalived.service
```

On the `BACKUP` Pi-hole instance in `/var/log/messages`, you should see something similar to `...Entering MASTER STATE` and any requests to the virtual
IP `192.168.1.170` should be responded to by `pihole2`. Now let's simulate a fail-back by re-starting the `keepalived` service on the original `MASTER` of
`pihole1`:

``` bash
$ sudo systemctl start keepalived
```

You should similarly see logs indicating that `pihole1` took back control of the virtual IP endpoint. Failover is working well!

## Gravity Sync

Having a primary and secondary Pi-hole instance is great, except currently (as of Pi-hole version `5.2.1`) there is no native mechanism to automatically sync
configurations between the Pi-hole instances, meaning any block/permit lists would need to be updated in both Pi-hole interfaces, which is cumbersome. Luckily,
a great project named [gravity-sync](https://github.com/vmstan/gravity-sync) has been developed and provides us the syncing capability we need. Although this
sync functionality does not address things such as DHCP, etc. - we only care about permit/block functionality in this tutorial, so this project will serve us
well.

First, install the `gravity-sync` components on the primary Pi-hole instance `pihole1`:

```bash
$ sudo su -
$ export GS_INSTALL=primary && curl -sSL https://gravity.vmstan.com | bash
```

Next, install the respective components on the secondary Pi-hole `pihole2`:

```bash
$ sudo su -
$ export GS_INSTALL=secondary && curl -sSL https://gravity.vmstan.com | bash
```

Ensure you answer the respective questions in the installer and set up the SSH key functionality you need in order to have gravity-sync successfully copy
configurations from the primary to the secondary. Once complete, on the secondary `pihole2`, you can do a check of what might need to be sync'd, perform a
one-time sync, and even schedule routine sync operations via cron. Note that the commands below assume the gravity-sync repository was cloned to
`~root/gravity-sync/` - it would be better to have this be more readily available as a callable script in `/usr/local/bin`, but that exercise will be
left up to the reader.

```bash
# on pihole2:

$ sudo su -
$ cd gravity-sync

# perform a diff check from the primary
$ ./gravity-sync.sh compare

# perform a one-time sync operation
$ ./gravity-sync.sh

# configure scheduled sync operations
$ ./gravity-sync.sh automate
# answer the questions asked
```

Scheduled sync operations are nice and in this tutorial, we're going to use 30 minute intervals for sync operations. In addition, following the sync configuration,
you can check the crontab for the root user (`crontab -l`) to see that the sync operation is in place.

You now have auto-syncing Pi-hole instances, which means you can edit all rules in your primary `pihole1` and only ever interact with a single administrative
interface vs. needing to keep things in sync by visiting and editing both web interfaces!

## Unifi Configuration for Pi-hole Setup

If you've made it this far, you should now have a 2-node Pi-hole failover cluster interacting with a single virtual IP address that can be used for any DNS
configuration of clients on your network. You can take this virtual IP (`192.168.1.170`) and put it into whatever configuration you need based on the vendors and
devices you own, but in our case, we will specifically point out that one way to use this functionality is to insert the virtual IP `192.168.1.170` into the
`Settings -> Networks -> LAN -> DHCP Name Server` option in a Unifi Dream Machine Pro controller, updating the `Auto` setting to instead be `Manual` and
specifying `192.168.1.170` as the `DNS server 1` parameter. Saving these settings will ensure that on a DHCP lease renew by a client, that client will then have
its DNS configuration (`/etc/resolv.conf` for \*nix-based devices, or similar) updated to point to your Pi-hole primary/secondary failover setup!

Happy ad-blocking!

## Further Improvements

There's a lot left to be desired in the above configuration - some topics that might be of use that the reader can implement:

1. Monitoring of `keepalived`: If you can configure email or other notification-based alerts (either natively in the `keepalived` configuration or external using
a monitoring agent such as [monit](https://mmonit.com/) or similar), you can be alerted to the fact that your primary Pi-hole has gone down/failover has ensued.
The risk in a nice failover configuration such as described above without having alerting is you may continue to surf the internet during a failover event and never
notice/be notified such that you would then be in a single point of failure if the primary Pi-hole is down.
2. Adjustment of gravity-sync Functionality: As mentioned previously, shuffling the `gravity-sync.sh` functionality into a more `PATH`-based implementation that
does not rely on a local git clone directory to function.

### Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

- [gravity-sync](https://github.com/vmstan/gravity-sync)
- [Clustered pihole](https://discourse.pi-hole.net/t/clustered-pihole-ive-done-it/12716)
- [keepalived man page](https://linux.die.net/man/8/keepalived)
- [Pi-hole](https://pi-hole.net/)
* [keepalived Basics](https://www.redhat.com/sysadmin/keepalived-basics)

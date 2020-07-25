---
layout: post
title:  "Vagrant Network - Connectivity & Routes"
date:   2020-07-24 22:38:00 -0400
categories: linux vagrant networking
logo: vagrant.jpg
---

It's so satisfying to spin up a Vagrant VM using VirtualBox, knowing that you now have a self-contained development environment...until
you figure out that you can't SSH to the instance or reach it via normal network methods. This tutorial is a very quick check/explanation
of the routing that is often needed to ensure your local OS can reach the VirtualBox VM as you expect.

### Network Configuration

Often, spinning up a Vagrant VM is quick and involves some kind of private, host-only networking to enable the instance to get an IP that
you expect. For example, the following line in a `Vagrantfile` is often all that you need to specify launching the VM with host-only
private networking using the host virtual bridge that Virtualbox creates automatically:

```
config.vm.network "private_network", ip: "10.11.12.13", netmask: "255.255.255.0"
```

However, if you can't ping the `10.11.12.13` IP address after launching the VM (even when you're sure the VM is up, available, and has
the IP bound to its interface - for example, you can use `vagrant ssh` just fine), then one of two (possibly more) options are likely an
issue):

1. You have allocated an IP in a range of IP addresses that already have a route defined on your local host instance to connect to other
networks with similar address ranges (e.g. VPN connectivity).
2. There is a firewall on your host instance blocking connectivity to the VM.
3. You don't yet have a route on your local host detailing that the IP range this VM resides in should be using the virtual bridge
created by the VirtualBox software.

#### Duplicate IP Range

If item 1 is the culprit (IP in a range of an existing network), you'll likely need to destroy your VM and re-create it in a network
CIDR block that is not used by other networks you intend to connect to using your host instance. If you don't, you risk having a duplicate
IP address issue between the VM and the other connected network, which will not result in good things.

#### Firewall

Checking your local firewall settings is always a good option. For Mac devices, inspect the networking settings for firewall rules, and
for Linux devices, check things like `firewalld` or `iptables` to ensure no rules are blocking connectivity to the VM.

#### Routes

To detect if routes are the issue (most likely the case), simply run a command to print the network routes that your host knows about
(this was run on a Mac device, but would very likely work on a Linux-based machine as well):

```bash
$ netstat -rn
```

If you don't see a route to the `10.11.12.0/24` CIDR range (for example, where this VM lives) tied to the virtual bridge that VirtualBox
creates (e.g. `vboxnet0`), your host instance is likely routing traffic through your default route, which is almost certainly not the
correct route to reach the VM. You will need to add an explicit route to the destination CIDR range via the VirtualBox virtual bridge.
Open the VirtualBox network settings to investigate the name of the virtual bridge (in this case, it is `vboxnet0`), and use this information
to create a route to the VM (in this case, residing on the `/24` network):

```bash
$ sudo route -n -v add -net 10.11.12 -interface vboxnet0
```

Re-run the `netstat -rn` command and you should see a route added via the `vboxnet0` interface, indicating that there is now a route
for the OS to use to direct traffic to the VM through the VirtualBox bounded bridge. You should see something similar to the following
after running `netstat -rn`:

```bash
Destination        Gateway            Flags        Refs      Use   Netif Expire
...
10.11.12/24        link#18            UCSc            0        0 vboxnet      !
...
```

If this shows up, you're likely in good shape and can ping/connect/SSH to the VirtualBox VM in this target network.

### Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

* [Vagrant Networking](https://www.vagrantup.com/intro/getting-started/networking)
* [Unable to connect to a vagrant private network from host](https://stackoverflow.com/questions/23497855/unable-to-connect-to-vagrant-private-network-from-host)

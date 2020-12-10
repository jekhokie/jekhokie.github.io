---
layout: post
title:  "iPerf3 on Unifi Dream Machine Pro"
date:   2020-12-08 11:55:00 -0400
categories: unifi networking iperf performance
logo: iperf.jpg
---

Setting up and testing your own home network can be both a lot of fun and one of the more frustrating things an engineer will do. Signal strength,
latency, channel selection, frequency selection, number of hops, and the list goes on and on. This post focuses specifically on a utility known
as [iPerf](https://iperf.fr), which can be used to peform checks of your network so that you can measure the impact of the changes you've made and
the overall performance of your network. Instead of attempting to explain all of the use cases of the iPerf utility, this post specifically focuses
on how to automatically run an iPerf server on a [Unifi Dream Machine Pro](https://store.ui.com/collections/unifi-network-routing-switching/products/udm-pro)
so that you can interact with it from various devices in your home network to measure and potentially rule out performance bottlenecks.

## Boot Functionality

There are a few steps needed in order to run the iPerf3 utility in your environment. A repository has been created which has Dream Machine utilities
and will serve as a starting point to get the iPerf container running on the Dream Machine Pro.

First, ensure SSH login is enabled for your devices. You can do this through the [Unifi Interface](https://unifi.ui.com/).

Next, follow [these instructions](https://github.com/boostchicken/udm-utilities/blob/master/on-boot-script/README.md#steps) to bootstrap the ability
to run scripts on boot of the Dream Machine Pro. After SSH-ing to your Dream Machine Pro as the root user, run the following commands to install the
boot functionality via a package, which will enable the functionality to persist between firmware upgrades since Unifi (at the time of this post) does not
wipe package-installed contents:

```bash
$ unifi-os shell
$ curl -L https://raw.githubusercontent.com/boostchicken/udm-utilities/master/on-boot-script/packages/udm-boot_1.0.2_all.deb -o udm-boot_1.0.2_all.deb
$ dpkg -i udm-boot_1.0.2_all.deb
$ exit
```

## Container Networking

In order to be able to run containers on the Dream Machine Pro, container networking needs to be set up. In order to ensure changes are applied on each
boot of the device (since again, some contents of the Dream Machine Pro are wiped/restored on reboot), create a boot script named
`/mnt/data/on_boot.d/10-cni-setup.sh` with the following contents, taken partially from
[this script](https://github.com/boostchicken/udm-utilities/blob/master/dns-common/on_boot.d/10-dns.sh):

```bash
#!/bin/sh

CNI_PATH=/mnt/data/podman/cni
if [ ! -f "$CNI_PATH"/macvlan ]; then
    mkdir -p $CNI_PATH
    curl -L https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-arm64-v0.8.6.tgz | tar -xz -C $CNI_PATH
fi

mkdir -p /opt/cni
rm -f /opt/cni/bin
ln -s $CNI_PATH /opt/cni/bin

for file in "$CNI_PATH"/*.conflist
do
    if [ -f "$file" ]; then
    ln -s "$file" "/etc/cni/net.d/$(basename "$file")"
fi
done
```

Once you've saved the file, make it executable by running `chmod +x /mnt/data/on_boot.d/10-cni-setup.sh` and run it one-time so you can set your
current environment up to test the iPerf3 container:

```bash
$ /mnt/data/on_boot.d/10-cni-setup.sh
```

Container networking should now be configured and ready so you can launch containers in your environment.

## iPerf3 Server Container

Once you have the container networking script written and run for your session, you can pull down a Docker image that contains an ARM64-compatible
version of iPerf3, referenced here as `<IPERF_CONTAINER>`. It's recommended you pin this to a version that you wish to use so you don't automatically
get "latest" and can store the image in the container cache on the Dream Machine Pro to help avoid malicious code injection by the provider when
the image is updated. The following command will both pull down the image (in this case, "latest" - again, not recommended) and start an iPerf server
in the foreground that you can use to test quickly - specifying `:latest` is for illustrative purposes only and should again be updated to reflect the
version/digest you wish to use:

```bash
$ podman run --name=iperf3-server -p 5201:5201 -p 5201:5201/udp <IPERF_CONTAINER>:latest -s
```

Once the container launches and iPerf shows as running in the foreground, launch your favorite iPerf client on a LAN/WLAN-connected device (something
such as the "iPerf3 Wifi Speed Test" app for iOS devices will work) and point it at your Dream Machine Pro IP, port 5201, to perform a test. You should
see output in the foreground container image and results on your client app corresponding to successful connectivity and testing.

Exit out of the running container (`CTRL-C`) and confirm that the iPerf3 image is still cached on the Dream Machine Pro by typing `podman images`.
Additionally, check that the server container you stopped is still registered with Podman by running `podman ps -a`, which should show a stopped
container with a name `iperf3-server`. This part is key for the following script to function (note - eventually it would be good if this script could
assume nothing about whether the container is available, created, running, stopped, etc. and simply handle those conditions on its own - for simplicity,
we'll leave that as an activity for the reader). Create the file `/mnt/data/on_boot.d/20-iperf3.sh` with the following contents:

```bash
#!/bin/sh

podman start iperf3-server
```

Once the file has been created, make the file executable via `chmod +x /mnt/data/on_boot.d/20-iperf3.sh` and reboot your Dream Machine Pro either by the
using the controller or via typing `reboot` in the terminal where you should still be connected via SSH.

When the Dream Machine Pro reboots and comes back online, you should be able to launch your client iPerf utility and test that connectivity and testing
still functions to the Dream Machine Pro - if so, congratulations, you now have iPerf3 running on your Dream Machine whenever the device boots up!

If you have issues connecting to the iPerf3 server on the Dream Machine Pro, SSH to the device and check podman status for containers to ensure the
container launched correctly (logs, etc.). There is likely something strange with the container configuration that is causing it to fail to launch.

## iPerf3 Packages for Server

If you wish to run an iPerf3 client or server manually on your Dream Machine Pro (or set up some other auto-launching mechanism different from the
description above), you can also do so fairly easily by installing the iPerf3 packages. Log into your Dream Machine Pro and install the respective
packages:

```bash
$ unifi-os shell
$ apt-get install iperf3
```

Following the above commands, you can launch an iPerf server or client functionality to test various endpoints in your network.

### Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

- [can't run interactive docker images...](https://www.reddit.com/r/Ubiquiti/comments/iz45ux/cant_run_interactive_docker_images_to_test_stuff/)
- [UDM Utilities](https://github.com/boostchicken/udm-utilities)
- [Unifi Boot Script Steps](https://github.com/boostchicken/udm-utilities/blob/master/on-boot-script/README.md#steps)
- [DNS Example Script](https://github.com/boostchicken/udm-utilities/blob/master/dns-common/on_boot.d/10-dns.sh)
- [WPA Supplicant Example Script](https://github.com/boostchicken/udm-utilities/blob/master/on-boot-script/examples/udm-files/on_boot.d/10-wpa_supplicant.sh)

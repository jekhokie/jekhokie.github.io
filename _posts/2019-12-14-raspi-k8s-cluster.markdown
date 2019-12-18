---
layout: post
title:  "Raspberry Pi k8s Cluster"
date:   2019-12-14 21:17:00 -0400
categories: raspberry-pi k8s linux
---
[Kubernetes](https://kubernetes.io/) (k8s) is a container orchestration platform that's transforming the way software services
are deployed and managed. For less than the cost of production-grade infrastructure, one can purchase multiple
[Raspberry Pi](https://www.raspberrypi.org/) boards, network them together, and create a fully functioning k8s cluster for
personal development and experimentation. This post details how to put together 3x Raspberry Pi 3 B+ hardware boards, link them
on a private network, and install a fully functioning k8s cluster for personal use and experimentation.

### Parts List

The following parts were used in this tutorial:

* [3x Raspberry Pi 3 B+ boards](https://www.amazon.com/ELEMENT-Element14-Raspberry-Pi-Motherboard/dp/B07BDR5PDW/ref=sr_1_3)
* [3x SanDisk Extreme 32GB MicroSD Cards](https://www.amazon.com/SanDisk-Extreme-microSDHC-UHS-3-SDSQXAF-032G-GN6MA/dp/B06XWMQ81P/ref=sr_1_2)
* [1x Netgear 8-port Gigabit Switch](https://www.amazon.com/NETGEAR-Gigabit-Lifetime-Protection-GS108Ev3/dp/B00M1C0186/ref=mp_s_a_1_3)
* [1x GeauxRobot Raspberry Pi 3 Model B 5-Layer Clear Case](https://www.amazon.com/GeauxRobot-Raspberry-Model-5-layer-Enclosure/dp/B01D90TX1O/ref=sr_1_8)
* [3x Cat6 Ethernet Cable](https://www.amazon.com/Cat-Ethernet-Cable-White-Pack/dp/B01IQWGI0O/ref=pd_sim_147_16)
* [1x Anker PowerPort 6](https://www.amazon.com/Anker-PowerPort-Charging-Premium-Motorola/dp/B00WI2DN4S/ref=pd_bxgy_147_img_2)
* MicroSD reader (or SD card reader with SD/MicroSD card adapter)

Note that the case is to make connections easier and cleaner but is not strictly required. Additionally, you might consider HDMI
cords and keyboard peripheral, but these instructions will not require UI-based changes (they will assume you are connecting to the
Raspberry Pi instances remotely via your laptop/personal device).

Finally, you can certainly expand your cluster to include additional Raspberry Pi nodes - this tutorial utilizes 3 total (1x master and
2x worker nodes), but you can expand the tutorial to use as many as you feel fit.

### Flashing the MicroSD Cards

You'll need to flash each of the MicroSD cards with an operating system for the Raspberry Pi instances. The OS used for this
tutorial is the [Hypriot OS](https://blog.hypriot.com/). The easiest way to flash the cards is to utilize the `flash` utility that
is available from the same provider.

First, we'll prepare a boot script that will configure the OS according to how you expect the system to operate/be configured. You'll
likely want to include the default wireless SSID and passphrase to enable your Raspberry Pi instances to get online and be reachable as
soon as they boot, so you'll insert your wireless network SSID and wireless passphrase in the `ssid` and `psk` parameters in the file
`configuration.yaml` below as indicated:

```yaml
#cloud-config

# Set your hostname here, the manage_etc_hosts will update the hosts file entries as well
hostname: black-pearl
manage_etc_hosts: true

# You could modify this for your own user information
users:
  - name: pirate
    gecos: "Hypriot Pirate"
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    groups: users,docker,video
    plain_text_passwd: hypriot
    lock_passwd: false
    ssh_pwauth: true
    chpasswd: { expire: false }

package_upgrade: false

# # WiFi connect to HotSpot
# # - use `wpa_passphrase SSID PASSWORD` to encrypt the psk
write_files:
  - content: |
      allow-hotplug wlan0
      iface wlan0 inet dhcp
      wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
      iface default inet dhcp
    path: /etc/network/interfaces.d/wlan0
  - content: |
      country=us
      ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
      update_config=1
      network={
      ssid="<WIRELESS_SSID>"
      psk="<WIRELESS_PASSPHRASE>"
      proto=RSN
      key_mgmt=WPA-PSK
      pairwise=CCMP
      auth_alg=OPEN
      }
    path: /etc/wpa_supplicant/wpa_supplicant.conf

# These commands will be ran once on first boot only
runcmd:
  # Pickup the hostname changes
  - 'systemctl restart avahi-daemon'

  # Activate WiFi interface
  - 'ifup wlan0'
```

Note that it is best not to use clear-text passphrases in the above, and if you'd like to instead create an encrypted psk,
you can use the `wpa_passphrase` command-line to do so. If you do, ensure that when you insert your encrypted passphrase, you
*do not* use double-quotes in the value (double-quotes are only required for when a clear-text passphrase is specified).

Next we'll download the Hypriot OS image file and flash utility:

1. Download the [Hypriot image](https://blog.hypriot.com/downloads/) - the version used in this tutorial is 1.11.4.
2. Download the [flash tool](https://github.com/hypriot/flash):
```bash
curl -LO https://github.com/hypriot/flash/releases/download/2.3.0/flash
chmod +x flash
```

Next, you'll want to flash each card in sequence. Insert each card into the USB reader for your device and run the following
command specifying the hostname as `node1`, `node2`, and `node3` where applicable for the SD card inserted. This parameter overrides
the `hostname` property in the configuration file, making it easier so you don't need to open/edit the file for each run:

```bash
./flash --hostname node1 --userdata wifi.yaml hypriotos-rpi-v1.11.4.img.zip
```

Once each of the cards is successfully flashed, you can continue assembling your cluster!

### Assembling

It's not the prettiest/cleanest layout, but this is an example of the assembly in case it helps:

[![k8s Pi Cluster][1]][1]

You'll now want to mount the Raspberry Pi boards in the case. Next, the assembly should be pretty straightforward, but in case
it helps, follow these steps:

1. Connect a Cat6 cable between each of the Raspberry Pi board ethernet adapters and the Netgear switch - the port used is not important.
2. Connect a micro USB cable between each of the Raspberry Pi board micro USB power plugs and the Anker 6-way USB power ports.
3. Plug the MicroSD card for each of the nodes into a Raspberry Pi instance.
4. Connect a Cat6 cable between your wireless router and the Netgear switch - the port used is not important.
5. Plug the power cable for the Anker 6-way USB power adapter and Netgear Switch into a wall power outlet.

At this point, your Raspberry Pi instances should begin to boot and configure. Note that it can take a few minutes for them to
stabilize as the first thing that occurs is an expansion of the SD card (to enable all space to be usable) and initial configuration
according to the YAML file you created.

### Communicating with Raspberry Pi Instances

After a few minutes, you should be able to see your Raspberry Pi nodes on your local wifi network. From a local device connected
to the same wireless network as the Raspberry Pi instances, run the command `sudo arp -a` - this should display a list of IP and
Mac addresses detected on your network. Within this list should be tags of `node1`, `node2`, and `node3`. If you don't see the names
of the nodes, you might need to go about this a different way (such as accessing your wireless router to view connected devices which
should contain the node names).

Once you identify the nodes, take note of the IP addresses for each. This should be an IP address assigned by your wireless router.
You might consider inserting these into your local `/etc/hosts` file to make it easier to reach the Raspberry Pi instances.

Using your favorite split terminal (tmux, etc.), SSH into each of the Raspberry Pi nodes using the default username/password of
`pirate`/`hypriot`. Immediately upon login, you should likely consider changing the default password for the `pirate` user. In addition
it's most efficient to insert your public SSH key into a file under the `pirate` user's home directory `~/.ssh/authorized_keys` so you
can SSH to each host using the `pirate` user without a passphrase (only using your SSH keys).

*Note*: Under certain circumstances, the default route to the internet on the Pi instances via the wireless ethernet device may be lost, causing
you to lose internet connectivity. This most often occurrs when performing a major upgrade of the operating system, etc. Under these circumstances,
you can run the following command, which will restore internet connectivity via the wireless adapter (replacing the `via` IP gateway with the
actual gateway of your wireless network): `sudo ip route add default via 192.168.86.1 dev wlan0`

### Master Network Configuration

Setting up the k8s master node is the first step. Create a file `/etc/network/interfaces.d/eth0` with the following contents (we'll be
following the subnet recommendation from the "Terraform: Up and Running" book as referenced in the credits section at the end of this
blog):

```bash
allow-hotplug eth0
iface eth0 inet static
    address 10.0.0.2
    netmask 255.255.255.0
    broadcast 10.0.0.255
```

Once the above file is created, reboot the k8s master node so it can claim the IP address. Once booted, SSH back into the device and run
the `ifconfig` command to ensure the `eth0` device has the IP address `10.0.0.2`.

### DHCP Server/Worker Node Network Configuration

Next we'll need to install a DHCP server to ensure the remaining `node2` and `node3` nodes (and any additional you may wish to add) can
automatically obtain IP addresses in the same subnet:

```bash
sudo apt-get -y install isc-dhcp-server
```

Edit the file `/etc/dhcp/dhcpd.conf` to include the following directive. Replace the `hardware` values for each of the nodes with the
respective `ether` (MAC address) that shows up for the `eth0` network device on each of the nodes when running the `ifconfig` command:

```bash
option domain-name "cluster.home";
option domain-name-servers 8.8.8.8, 8.8.4.4;

default-lease-time 600;
max-lease-time 7200;
ddns-update-style none;

authoritative;

subnet 10.0.0.0 netmask 255.255.255.0 {
    option broadcast-address 10.0.0.255;

    group {
        host node1 {
            hardware ethernet b8:27:eb:d9:e4:38;
            fixed-address 10.0.0.2;
        }

        host node2 {
            hardware ethernet b8:27:eb:7f:7b:bb;
            fixed-address 10.0.0.3;
        }

        host node3 {
            hardware ethernet b8:27:eb:29:4e:03;
            fixed-address 10.0.0.4;
        }
    }
}
```

You'll also need to update the file `/etc/default/isc-dhcp-server` to ensure the following property is present and specified:

```bash
INTERFACESv4="eth0"
# you can leave the following property blank as we'll not be using IPV6
INTERFACESv6=""
```

Then, restart the DHCP server for the configuration to take effect: `sudo systemctl restart isc-dhcp-server`.

Once the above is complete (and the DHCP server restarts gracefully), issue a reboot command on each of your `node2` and `node3` instances.
When they have finished rebooting, SSH back into the devices and ensure each has their `eth0` network adapter specified with a `10.0.0.3` and
`10.0.0.4` IP address, respectively, while the `wlan0` (wireless) adapter remains specified as an IP on your wireless network, ensuring they
can continue to access the public internet.

### Firewall and Traffic Forwarding

On each node, data exchange between the `wlan0` and `eth0` devices is needed and is configured through IPTables routing rules and
`sysctl` settings. Configure IP forwarding by editing `/etc/sysctl.conf`:

```bash
# edit this property to be un-commented and set to a value of "1"
net.ipv4.ip_forward=1
```

Next, add the following statements to the file `/etc/rc.local` to ensure the IPTables rules are correctly specified:

```bash
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
```

When done, reboot your Raspberry Pi instance for the settings to take effect.

### Kubernetes Software Installation

On each of the nodes, add the encryption key for the k8s packages, add the k8s repository, and install the packages required to run k8s:

```bash
sudo su -
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" \
        >> /etc/apt/sources.list.d/kubernetes.list
apt-get -y update
apt-get -y upgrade
apt-get -y install kubelet kubeadm kubectl kubernetes-cni
```

The required software is now installed on each of the nodes.

### Configuring the k8s Master

Let's now configure the master instance - run the following commands on the `node1` k8s master instance::

```bash
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 \
     --apiserver-advertise-address 10.0.0.2 \
     --apiserver-cert-extra-sans kubernetes.cluster.home
```

The above command will take several minutes to download container images, configure network components, etc. Once complete, there are 2 important
items the logs will print that need to be taken care of in the following sections.

#### Setting Up `kubectl` Command

In order to interact with the k8s cluster from any node, you'll need to copy the `admin.conf` over to the host you're using to access the cluster.
Assuming you're going to interact with the cluster from the k8s master instance (`node1`), run the following commands to set up configuration for
your user to successfully interact with the cluster (obviously this can be performed wherever you wish to interact with the cluster from):

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Joining Worker Nodes

Now that the k8s master is configured you need to configure the worker nodes to connect to the master and be usable in the cluster. Pay attention
to the command output when running the master install and capture the command to join a worker node to the master instance/cluster. It should
look something like the following:

```bash
sudo kubeadm join 10.0.0.2:6443 --token <TOKEN> \
     --discovery-token-ca-cert-hash <TOKEN_HASH>
```

Run the above on each of the `node2` and `node3` worker nodes.

#### Showing Node Status

Run the command `kubectl get nodes` to list the nodes in the cluster. You should see something similar to the following, with the `NotReady` showing
for all nodes due to the fact that we have not yet installed cluster (Pod-to-Pod) networking:

```bash
NAME    STATUS      ROLES    AGE   VERSION
node1   NotReady    master   5m   v1.17.0
node2   NotReady    <none>   4m   v1.17.0
node3   NotReady    <none>   3m   v1.17.0
```

### Cluster (Pod-to-Pod) Networking

We will be using [Flannel](https://github.com/coreos/flannel) for our Cluster networking, which enables Pod-to-Pod communication. The following steps
will deploy several containers that will initialize and configure the network fabric to enable pods to communicate. First, download a default flannel
deployment configuration:

```bash
curl https://rawgit.com/coreos/flannel/master/Documentation/kube-flannel.yml \
     > kube-flannel.yaml
```

Next, we'll need to update the configuration to use the `host-gw` configuration instead of `vxlan`, and specify using `arm` instead of `arm64`. Edit
the `kube-flannel.yaml` file and replace all instances of `vxlan` with `host-gw`, as well as replace any instance of `arm64` with `arm`. Once complete,
apply the flannel configuration:

```bash
kubectl apply -f kube-flannel.yaml
```

At this point, several containers should be deployed and start to configure the network fabric. You can re-run `kubectl get nodes` over the next several
minutes and should start to see each node change state to `Ready`, indicating the network configuration is complete on that node.

### Dashboard

Some users prefer a UI to investigate issues and see the state and configuration of their cluster. We'll deploy the default k8s dashboard following the
default instructions [here](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/#deploying-the-dashboard-ui). First, we'll deploy
the dashboard:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta6/aio/deploy/recommended.yaml
```

#### Dashboard User and Binding

In order to access the dashboard, we'll need to create a user and bind the user to enable access. The steps are to create a service account and then
bind the service account to enable access. Create a file `dashboard-adminuser.yaml` with the following contents:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

Apply the configuration to create the service account:

```bash
kubectl apply -f dashboard-adminuser.yaml
```

Next, create a binding configuration file named `adminuser-binding.yaml` with the following contents:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Apply the adminuser binding:

```bash
kubectl apply -f adminuser-binding.yaml
```

#### Dashboard User Token

In order to access the dashboard, a token is required. Run the following command, and copy the `token` value to be used in order to access
the dashboard:

```bash
kubectl -n kubernetes-dashboard describe secret \
    $(kubectl -n kubernetes-dashboard get secret \
                 | grep admin-user \
                 | awk '{print $1}')
```

#### Accessing the Dashboard

Now that we have a user and associated token, we can access the dashboard. From your local device, port forward to the k8s master node `node1`:

```bash
ssh -L8001:localhost:8001 pirate@node1
```

Once on the k8s master node, run the following command to start the dashboard:

```bash
kubectl proxy
```

You should see a message indicating the dashboard is being served from `127.0.0.1:8001`. From your local device (not the k8s master, but the
device you port forwarded *from*), access the dashboard via the following URL in a browser:

`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/overview?namespace=default`

Use the `Token` option to log in, and paste the token you captured from the "Dashboard User Token" section above. You should now see the
dashboard and all configurations/resources in your cluster!

### Connecting from Local Device

There is a caveat if you wish to connect to the cluster from a local device (laptop or otherwise). Due to the fact that the `eth0` interface
is hosting the cluster IP for the node subnet, if your local device is not on the same subnet/network (`10.x`), you will likely need to SSH tunnel
and port forward the interface for `kubectl` to operate. First, copy the `/home/pirate/.kube/config` file to your local machine `~/.kube/config` file.
Then, port forward SSH-tunnel into the k8s master `node1` instance for the `kubectl` interface:

```bash
ssh -L6443:localhost:6443 pirate@node1
```

Finally, open a new window and run the following command to see the node status for your cluster:

```bash
kubectl get nodes --insecure-skip-tls-verify=true
```

If all goes well, you should see your cluster node status displayed!

**Note**: If you don't want to have to include the `--insecure-skip-tls-verify=true` switch for each command and you aren't worried about man-in-the-middle
attacks (or don't want to mess with certs), you can update the cluster config to remove cert checking/TLS verification entirely by using the following
command: `kubectl config set-cluster kubernetes --insecure-skip-tls-verify=true`. Note that it is much preferred that you resolve the certificate validation
issues than use this method, but for quick starts, this may suffice for your needs.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Kubernetes - Up and Running (Book)](https://www.amazon.com/Kubernetes-Running-Dive-Future-Infrastructure/dp/1492046531/ref=sr_1_3)
* [Kubernetes](https://kubernetes.io/)
* [Raspberry Pi](https://www.raspberrypi.org/)
* [Docker Running on Raspberry Pi](https://blog.hypriot.com/getting-started-with-docker-and-mac-on-the-raspberry-pi/)
* [Hypriot](https://blog.hypriot.com/)
* [Hypriot Flash Tool](https://github.com/hypriot/flash)
* [Flannel](https://github.com/coreos/flannel)
* [k8s Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/#deploying-the-dashboard-ui)
* [Dashboard User](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)

[1]: /assets/images/2019-12-14-raspi-k8s-cluster-as-built-image.jpg

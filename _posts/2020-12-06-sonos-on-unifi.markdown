---
layout: post
title:  "Sonos on Unifi Network Gear"
date:   2020-12-06 19:42:00 -0400
categories: unifi sonos networking
logo: sonos.jpg
---

[Unifi](https://unifi-network.ui.com/) networking gear is currently some of the best Prosumer and SMB network gear around. However, if you own any Sonos
equipment in your home, you'll potentially have trouble setting up your system in a way that both isolates the Sonos equipment the way you want it to and
allows for continued control/communication with it through the Sonos app on a different network within your home. There are many posts detailing various
configuration settings, some of which I've found work while others do not, and this post attempts to detail the architecture and corresponding configuration
settings that work for the setup I currently have.

## Background/Layout

When installing the Unifi gear, I wanted to have my Sonos equipment off on its own VLAN for various reasons. However, doing so meant that many configuration
parameters needed to be tweaked, the outcome of which was a lot of trial/error to get the system working the way I wanted it (Sonos equipment on its own
WLAN SSID and VLAN, separate from the rest of my mobile devices in the home, and having the SSID hidden/not advertised). This is a sumary of the changes
that I made which ultimately resulted in a working ecosystem.

Note that the description of how to navigate to the various sections is using the "legacy" interface - if you're using the new UI, simply do a search for
the setting to find where in the interface the setting is located.

### Separate VLAN

In the Unifi controller, navigate to `Settings -> Networks -> Create New Network` and specify the following:

* **Name**: Sonos
* **Type**: Corporate
* **Interface**: LAN
* **VLAN**: 12
* **Gateway IP/Subnet**: 192.168.86.0/24 (click auto-fill DHCP)
* **IGMP Snooping**: (checked)
* **DHCP Range**: (taken from above Gateway IP/Subnet)

### Separate WLAN/SSID

In the unifi controller, navigate to `Settings -> Wireless Networks -> Create New Wireless Network` and specify the following:

* **Name**: sonos
* **Enabled**: (checked)
* **Security**: WPA Personal
* **Security Key**: (use some super secret password)
* **Network**: Sonos
* **WiFi Band**: 2.4GHz (do NOT use 5GHz)
* **Hide SSID**: (checked)
* **WPA Mode**: WPA2 Only - AES/CCMP Only
* **Multicast Enhancement**: (checked)

### STP and Switch Priority

For each switch connected to your core router/switch (in this case the core was a Dream Machine Pro), RSTP needs to be switched to STP in order for the Sonos
equipment to work correctly. This setting can be found in `Devices -> (SWITCH) -> Config -> Services -> Spanning Tree` for each `(SWITCH)` you own, selecting
the `STP` radio button and applying the changes.

In addition, for each level of switch you have in your network (distance from core), configure the following "Priority" under `Devices -> (SWITCH) -> Services ->
Priority`:

* **1st-Level**: 4096
* **2nd-Level**: 8192
* **3rd-Level**: 16384
*(...and so on and so forth)*

### MDNS

In the unifi controller, navigate to `Settings -> Services -> MDNS` and ensure `Enable Multicast DNS` is checked.

### Access Points

For each access point, ensure that `Enable Meshing` is selected under the `Config -> Radios` setting. Although this setting should only impact mesh devices
connected to the access points, it was specifically shown to break visibility to Sonos equipment when deactivated, so it must be enabled for the Sonos equipment
to show up in your Sonos controller.

### Move Sonos to New SSID

In the Sonos controller, if you are switching networks of your devices, you'll need to go through a sequence of connecting one of the devices to
the LAN via a network cable to move everything over if you have more than one device. Once done, make sure you go into the Sonos controller settings
and "Forget" the existing SSID/network so the devices don't attempt to re-connect to the existing network.

Part of the trouble in moving the Sonos devices was the fact that they had held on to the old SSID WPA information and re-distributed it even though
the existing SSID information was deleted from the Sonos controller. Ensure you give enough time to elapse after having selected to delete the SSID
from the Sonos controller to prevent re-propagation of the information.

## Result

After the above changes, having an iPhone connected to the primary SSID (non-Sonos) and all Sonos equipment running on its own `sonos` SSID worked perfectly
where the iPhone was able to see and control the Sonos equipment. This gives excellent flexibility and isolation of the Sonos equipment to operate in its
own VLAN and can offer enhanced control for the future.

One thing to note is it's likely a good idea to have the Sonos equipment bound to a specific WiFi channel to ensure it's both a dedicated space and isolated
to avoid chatter and interference from/to other devices on the network. This exercise is left for the reader to implement.

### Credit

There are many posts and detailed documentation on the Unifi site that detail the various configuration settings needing to be tweaked in order for
the above setup to work. In this particular case, you can search many message boards, formal documentation, etc., but as I did not keep a list of links
(apologies if you feel you were the source that contributed to this outcome) much of the credit goes to those Network warriors that helped test and
troubleshoot many of these configurations. A couple articles that were recalled as important to starting this journey:

- [Unifi Best Practices for...](https://help.ui.com/hc/en-us/articles/360001004034-UniFi-Best-Practices-for-Managing-Chromecast-Google-Home-on-UniFi-Network)
- [Reset your Sonos Product](https://support.sonos.com/s/article/1096?language=en_US)

---
layout: post
title:  "Tuning Unifi Wireless"
date:   2020-12-07 21:37:00 -0400
categories: unifi wireless networking
logo: unifi.jpg
---

This post details how to tune your [Unifi](https://unifi-network.ui.com/) networking gear wireless settings for enhanced throughput and speed. Simply
plugging the equipment in and using it already provides a great experience, but often, tuning will significantly enhance things such as wireless connectivity
improvements, reduced interference, increased throughput, increased speed, etc. This is an opinionated post based on the environmental conditions of one home
and preferences around configuration settings that resulted in improvement from ~250Mbps rates with standard configurations up to ~470Mbps sustained on iPhone
XR devices connected to home Wireless Access Points [Unifi In-Wall 802.11ac Wave 2 Wi-Fi Access Points](https://inwall-hd.ui.com/).

## Approach

There's no one approach to tuning your network for your home or work environment and it requires patience and persistence, with many points of measurement.
Find a good application for your mobile/wireless device (e.g. for iPhone, something akin to WiFi SweetSpots will work) and measure the rates you get at various
points in your living space in order to both place your wireless access points correctly and get a feel for how much coverage you need. The following are a few
details of things to consider and improve, and which settings worked for this opinionated and specific environment.

For additional performance and architecture layouts, refer to the previous articles on this blog that walk through how to set up a Sonos system and also enable
testing of home networking using iPerf on a Dream Machine Pro.

As a note (and mentioned in the article heading), after the following changes were applied, performance of an iPhone XR on the wireless network went from a rate
of ~250Mbps sustained up to roughly ~470Mbps sustained, which was a great improvement for the home network. While the changes below are likely not all that you
can make (and some may even not work as well in your environment), it's a reference in case it helps.

## Wireless SSIDs for Frequencies

In the home configuration, having multiple SSIDs for dedicated frequencies is desirable given the band steering functionality of many devices is great, but is
not an adopted standard by devices. In this case, creating an SSID broadcasting 2.4GHz and a separate SSID for 5GHz provides this flexibility, and if you connect
your mobile devices to the respective SSID you wish to use, it should start to "prefer" the more performant based on whatever learning and rating characteristics 
it uses for the various networks it knows about.

## Wireless Channel

Use the Unifi In-Wall HD access points to perform scans (`Devices -> (HD AP) -> Tools -> RF Environment -> Scan`) to measure channel and frequency interference
for the specific access point. Then, pick a channel with the lowest usage/interference and set your access point to use that specific channel (`Devices -> (HD AP) ->
Config -> Radios`) as opposed to allowing auto-allocation. Do this for both the 2.4GHz and 5GHz radios, for each access point.

## Power and Channel Width

As 2.4GHz implicitly has longer range but 5GHz is preferred for performance, adjust the power settings and channel widths (depending on how many APs you have, of
course). For an environment with 6x APs, all access points were configured with 2.4GHz on "Medium" broadcast power/VHT20 channel width, and the 5GHz radios
were configured with "High" broadcast power/VHT80 channel width. This allows the best chance for devices that support it to connect to the more performant
5GHz frequency.

### Credit

There are many posts and detailed documentation on the Unifi site that detail the various configuration settings needing to be tweaked in order for
the above enhancements to work. In this particular case, you can search many message boards, formal documentation, etc., but as I did not keep a list of links
(apologies if you feel you were the source that contributed to this outcome) much of the credit goes to those Network warriors that helped test and
troubleshoot many of these configurations.

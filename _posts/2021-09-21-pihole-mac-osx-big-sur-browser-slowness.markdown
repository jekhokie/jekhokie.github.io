---
layout: post
title:  "Pi-hole and Browser Slowness on Mac OSX Big Sur"
date:   2021-09-21 19:23:00 -0400
categories: pihole
logo: pihole-osx-big-sur.jpg
---

[Pi-hole](https://pi-hole.net/) is a fantastic ad-blocker software that can be run from a Raspberry Pi device. However, in a more recent
version of Mac OSX (Big Sur to be specific), it was noticed that any/all browsers were exhibiting significant slowness. When digging in,
this slowness was actually due to a way the OS was dealing with ad-blocking that changed from prior versions. This blog post details how to
correct the issue (hint: small change to the Pi-hole ad-blocking response when intercepting DNS requests for ads).

## Background

When upgrading Mac OSX to Big Sur, it was noticed that all browsers tested (Firefox, Chrome, Safari) would exhibit SIGNIFICANT latency in
loading almost any website. In digging into the network traces for web page requests, it appeared that there was a significant delay in responses
for sites that appeared to be "blocked" by the Pi-hole ad-blocking software. Not obvious at first, but after trying to simply disable Pi-hole
ad-blocking for 30 seconds, any site visited loaded speedily (with ads), proving that the Pi-hole software was in fact causing the slowness.

## Fix

The issue, as it turns out, had to do with the way Pi-hole was responding for ads-based sites. There are several modes of responses that can
be sent by the Pi-hole when an ad-based domain is requested. As it turns out, the default mode (unspecified IP blocking response in the version
being used) resulted in long resolution/timeout times by the browsers in Big Sur. The fix was to update the `BLOCKINGMODE` configuration in
the Pi-hole configuration settings to use `NXDOMAIN`, which results in an immediate action by the browser upon receipt. To update to using this
mode specifically, the following process can be followed on your Pi-hole device/OS:

```bash
# edit the FTL configuration file
$ sudo vim /etc/pihole/pihole-FTL.conf

# add to bottom of file or replace existing
# property with value:
#   BLOCKINGMODE=NXDOMAIN

# restart the FTL service
sudo service pihole-FTL restart
```

Once the above has been performed, re-try your browser and you should notice an immediate performance change to your previous experience!

### Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

- [NXDOMAIN and Null Blocking with FTLDNS](https://pi-hole.net/2018/05/18/nxdomain-and-null-blocking-with-ftldns/#page-content)
- [Firefox and Chrome Painfully Slow in Big Sur](https://www.reddit.com/r/MacOS/comments/k0abtt/firefox_and_chrome_painfully_slow_in_big_sur/)


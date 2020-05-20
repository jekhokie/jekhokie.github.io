---
layout: post
title:  "SSH Tunnel"
date:   2013-07-11 16:23:18 -0400
categories: linux ssh
logo: terminal.jpg
---
Often it is useful to have a "jump host" between zones on your network to help with segregating
various security domains (i.e. PCI). However, this can be cumbersome for system administrators
to access the hosts using the jump host due to the multiple SSH commands needed to be run. This
is a simple SSH tunnel implementation to avoid the unnecessary "2-hop" jump host.

### Process

When using a jump host, it is necessary to SSH twice (once to the jump host, and the second time
to the destination host you wish to access) to reach your destination. The following steps set up
an SSH tunnel through the jump host so that connections can be made in one single SSH command using
the existing tunnel connection.

First, set up the tunnel to the destination host (through the jump host) that you wish to access:

{% highlight bash %}
$ ssh -L 2222:<DESTINATION_HOST>:22 <USER>@<JUMP_HOST>
{% endhighlight %}

Now that the port forwarding has been established, you can simply use SSH pointed at localhost
port 2222 and you will arrive at <DESTINATION_HOST> over port 22 (assuming SSH is listening on
port 22 on the destination host):

{% highlight bash %}
$ ssh localhost -p 2222
{% endhighlight %}

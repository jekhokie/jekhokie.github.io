---
layout: post
title:  "syslog-ng Tutorial"
date:   2016-12-20 21:14:01 -0400
categories: syslog syslog-ng logging linux security
---
Tutorial on how to install and configure [syslog-ng](https://syslog-ng.org/) for system logging,
including some more advanced configurations such as configuring a central server and SSL for
security/encryption.

### Background

[syslog-ng](https://syslog-ng.org/) is an open-source log management solution providing enhanced
capabilities for collecting, parsing, classifying, and correlating logs across endpoints.

This article will focus on installing syslog-ng on a Linux-based system (specifically, Ubuntu 16.04).
There are packages available to install and configure syslog-ng on various operating systems (including
Windows-based systems), but those are out of scope for this tutorial.

This tutorial will be focused on defining steps required to accomplish the following (in order). The
content within this post is not necessarily that interesting itself, but will serve as a larger building
block for a data processing/streaming system:

1. Install, configure, and test syslog-ng on a local operating system.
2. Re-configure the syslog-ng service to forward logs to a remote syslog-ng server (aggregation).
3. Update the remote syslog-ng instance for securing connections.

Each step in the process will be slightly more complicated than the previous. By the end of this
tutorial, you should have a fairly sophisticated and secure syslog-ng technology framework that will
allow you to send logs to a remote instance, including from network devices, other operating systems,
etc.

### Underlying Compute Technology/Ecosystem

This tutorial utilizes/makes the following assumptions about your log infrastructure. Although the
instructions can be extended to accommodate cloud-based and/or other infrastructure environments quite
easily, all details are explained with respect to local development for the purposes of simplicity:

- **Hypervisor Technology**: VirtualBox
- **Provisioner**: Vagrant
- **Number of VMs**: 2
- **Operating System**: Ubuntu 16.04
- **Arch**: 64-bit
- **CPUs**: 1
- **Mem**: 2GB
- **Disk**: 20GB

This tutorial is built with 2 virtual machines - one of the resources will be the syslog-ng client
while the other will be a "remote" syslog-ng instance for log forwarding. The following hostnames
and corresponding IP addresses will be used for the 2 virtual machine instances:

- **node1.localhost**: 10.11.13.15
- **node2.localhost**: 10.11.13.16

The respective versions of software used in this tutorial are as follows. Other versions **may** work
using these instructions but, as usual, your mileage may vary:

- **syslog-ng**: 3.5.6-2.1

Finally, the code blocks included in this tutorial will list, as a comment, the node(s) that the commands
following need to be run on. For instance, if required on both nodes, the code will include a comment like
so:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
{% endhighlight %}

If the code block only applies to one of the nodes, it will list the specific node it applies to like so:

{% highlight bash %}
# Node: node2.localhost
{% endhighlight %}

All commands assume that your user is `vagrant` and the user's corresponding home directory is `/home/vagrant`
for the purposes of running sudo commands.

### End State Architecture

At the end of this tutorial, the following end-state architecture will be realized (where "Server" is the
remote instance of syslog-ng where the Client will forward logs to):

{% highlight bash %}
|---------------------|               |---------------------|
|       Client        |               |       Server        |
|   node1.localhost   |               |   node2.localhost   |
|    (10.11.13.15)    |               |    (10.11.13.16)    |
|                     |     Log       |                     |
|     |-----------|   |  Forwarding   |     |-----------|   |
|     | syslog-ng |------------------------>| syslog-ng |   |
|     |-----------|   |     (SSL)     |     |-----------|   |
|                     |               |                     |
|---------------------|               |---------------------|
{% endhighlight %}

### Installing syslog-ng

**Note**: All commands in this section are only applicable to both nodes - the installation is fairly
straightforward and can be done in parallel on both nodes.

First, ensure the operating system repository cache is updated:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo apt-get update
{% endhighlight %}

Now, install the syslog-ng packages required:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo apt-get install syslog-ng
{% endhighlight %}

Once the package has been installed, it is automatically started as well. To validate, you can tail
the `/var/log/syslog` log file and execute the following command in a new window:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ logger "Testing this out"
{% endhighlight %}

If all goes well, you should see "Testing this out" show up in the `/var/log/syslog` log file that is
being tailed in the other window. If so, you're all set to continue!

### Configure syslog-ng for Local Logging

**Note**: The steps in this section are applicable only to "node1.localhost".

Now that you have a fully functioning syslog-ng service running, let's configure it for something more
custom/useful than the `/var/log/syslog` file. For instance, imagine you have a custom application that
logs to the file `/var/log/myapp.log`. Let's configure syslog-ng to read this file and store the results
in another log file named `/var/log/myapp_mirror.log`. To be clear, we would likely never do this, but
it is a stepping stone to our soon-to-be remote push solution:

{% highlight bash %}
# Node: node1.localhost
$ sudo vim /etc/syslog-ng/conf.d/myapp.conf
# ensure the file contains the following:
#   source myapp_log {
#     file("/var/log/myapp.log");
#   };
#   destination myapp_mirror_log {
#     file("/var/log/myapp_mirror.log");
#   };
#   log {
#     source(myapp_log);
#     destination(myapp_mirror_log);
#   };

$ sudo service syslog-ng restart
{% endhighlight %}

Once the service completes restarting, you should be able to write lines (logs) to the file
`/var/log/myapp.log` and see the resulting logs also show up in the `/var/log/myapp_mirror.log` file:

{% highlight bash %}
# Node: node1.localhost
$ echo "Testing this log capability" | sudo tee -a /var/log/myapp.log

# resulting logs should also show up in the parallel file, including a date/timestamp:
$ sudo cat /var/log/myapp_mirror.log
#   Dec 20 21:35:39 node1 Testing this log capability
{% endhighlight %}

If this works as expected, you have successfully configured your "myapp.log" mirroring to the local
disk!

### Configure syslog-ng for Server Mode

Now that we have a fully-functional syslog-ng instance logging/mirroring logs locally, we will configure
and connect a syslog server so that logs are no longer mirrored to the local disk but, rather, forwarded
to a remote location.

Assuming you have followed the steps in the previous section, you should now have syslog-ng installed on
the server host. From here, we define a configuration file to gather log messages from incoming
connections and log to a local file. We are going to us TCP for the connection protocol due to the next
section/steps involving configuring SSL for encrypted communication:

{% highlight bash %}
# Node: node2.localhost
$ sudo vim /etc/syslog-ng/conf.d/myapp.conf
# add the following:
#   source myapp_network {
#     tcp(ip(0.0.0.0) port(601));
#   };
#   destination myapp_local {
#     file("/var/log/myapp.log");
#   };
#   log {
#     source(myapp_network);
#     destination(myapp_local);
#   };

$ sudo service syslog-ng restart
{% endhighlight %}

Once the service restarts, it is ready to accept incoming messages from client syslog-ng instances.
Let's now re-configure the client on "node1.localhost" to send messages to the syslog-ng server
instead of mirroring the logs locally:

{% highlight bash %}
# Node: node2.localhost
$ sudo vim /etc/syslog-ng/conf.d/myapp.conf
# update the "destination" resource to read as follows, replacing with your syslog-ng server IP address:
#   destination myapp_remote {
#     network("10.11.13.16" port(601) transport("tcp"));
#   };
# update the "log" resource to read as follows:
#   log {
#     source(myapp_local);
#     destination(myapp_remote);
#   };

$ sudo service syslog-ng restart
{% endhighlight %}

At this point, you can now send some data to the `myapp.log` file on "node1.localhost" and you should
then see the resulting log entry show up on the syslog-ng server (remote):

{% highlight bash %}
# Node: node1.localhost
$ echo "Testing this remote log capability" | sudo tee -a /var/log/myapp.log

# Node: node2.localhost
# resulting logs should show up on the syslog-ng server:
$ sudo cat /var/log/myapp.log
#   Dec 20 21:58:42 node1 Testing this remote log capability
{% endhighlight %}

If the above proves successful, you now have a working central logging server architecture!

### Update Architecture for Security (SSL)

Now that we have a client/server architecture configured, we can update the ecosystem to include some
security components, including encryption and the ability to restrict client forwarding of logs unless
the client is valid (prevents unauthorized logging by unknown/random clients). The way in which this
happens is via a Certificate Authority which issues certificates to clients that should be
authorized to log to the endpoint.

#### Certificate Authority

First, we need to set up a Certificate Authority - this should be done as the root user on the central
(remote) syslog-ng server:

{% highlight bash %}
# Node: node2.localhost
# note - all steps performed as 'root' user
$ sudo su -

# set up the directory structure
$ mkdir /root/CA
$ cd /root/CA
$ mkdir certs crl newcerts private
$ chmod 700 private
$ touch index.txt
$ echo 1000 > serial

# create the configuration file from the default template - there are many other options you can
# configure by default to help with default certificate attributes, but they are left out of this
# tutorial
$ cp /etc/ssl/openssl.cnf openssl.cnf
$ vim openssl.cnf
# ensure that the 'dir' directive under the [ CA_default ] resource is specified as a dot '.'
# indicating the current directory:
#   [ CA_default ]
#   ...
#   dir = .
#   ...

# create the certificate for the CA
$ openssl req -new -x509 -keyout private/cakey.pem -out cacert.pem -days 365 -config openssl.cnf
# answer the questions, and ensure you keep track of the password - for this tutorial, we will
# use 'root1234' as the password (horrible password, but simple for the tutorial). In addition,
# for the FQDN/Common Name, we will use "CA" - note that this MUST be unique.

# validation
$ openssl x509 -noout -text -in cacert.pem
# should output details about the certificate...
{% endhighlight %}

#### syslog-ng Server Certificate

Now that the Certificate Authority certificate has been created, we will create a certificate signed
by the CA for the syslog-ng server itself:

{% highlight bash %}
# Node: node2.localhost
# note - all steps performed as 'root' user
$ sudo su -

# create the certificate
$ cd /root/CA
$ openssl req -nodes -new -x509 -keyout private/serverkey.pem -out certs/serverreq.pem -days 365 -config openssl.cnf
# answer the questions, and for the FQDN/Common Name, use "10.11.13.16" or whatever the IP address
# is of your server host

# create a signing request
$ openssl x509 -x509toreq -in certs/serverreq.pem -signkey private/serverkey.pem -out tmp.pem

# sign the certificate
$ openssl ca -config openssl.cnf -policy policy_anything -out certs/servercert.pem -infiles tmp.pem
# enter the CA passphrase when prompted - again, for this tutorial, 'root1234', and ensure that you
# opt for yes ('y') when asked to sign and commit the newly-signed certificate

# validation
$ openssl x509 -noout -text -in certs/servercert.pem
# should output details about the certificate...
$ openssl verify -CAfile cacert.pem certs/servercert.pem
# should output that the chain of trust is valid:
#   certs/servercert.pem: OK

# clean-up
$ rm tmp.pem
{% endhighlight %}

#### syslog-ng Server Certificate Configuration

Once the CA has been established and a server certificate generated/signed by the CA, we can move
items into place and configure syslog-ng to utilize the certificates for encryption. To configure
syslog-ng to utilize the certificates for encrypted communication, perform the following:

{% highlight bash %}
# Node: node2.localhost
# put the certs and keys in place
$ cd /etc/syslog-ng/
$ sudo mkdir cert.d ca.d
$ sudo cp /root/CA/certs/servercert.pem /root/CA/private/serverkey.pem cert.d/
$ sudo cp /root/CA/cacert.pem ca.d/

# generate and symlink the appropriate hash
$ openssl x509 -noout -hash -in ca.d/cacert.pem
# copy the resulting hash, and substitute it for the <HASH> variable in the following
# command - note that the hash MUST have a suffix of '.0'
$ sudo ln -s cacert.pem ca.d/<HASH>.0
{% endhighlight %}

Now that the certificate and key files are in place, we can update the syslog-ng configuration for
the log acceptance:

{% highlight bash %}
# node2.localhost
$ sudo vim /etc/syslog-ng/conf.d/myapp.conf
# update the 'source' resource as follows:
#   source myapp_network {
#     tcp(ip(10.11.13.16)
#         port(6514)
#         tls(key_file("/etc/syslog-ng/cert.d/serverkey.pem")
#             cert_file("/etc/syslog-ng/cert.d/servercert.pem")
#             ca_dir("/etc/syslog-ng/ca.d"))
#         );
#   };

$ sudo service syslog-ng restart
{% endhighlight %}

Once restarted, syslog-ng should be listening on and accepting connections over the encrypted
channel via port 6514 - our configuration of the server is complete at this point and we can move
on to the client configuration.

#### syslog-ng Client Certificate

Now that the Certificate Authority and the server certificate have been configured and put in place,
we can move to generating the client syslog-ng certificate. First, we will need to generate and sign
a certificate for the client on the server instance (where the CA resides):

{% highlight bash %}
# Node: node2.localhost
# note - all steps performed as 'root' user
$ sudo su -

# create the certificate
$ cd CA/
$ openssl req -nodes -new -x509 -keyout private/clientkey.pem -out certs/clientreq.pem -days 365 -config openssl.cnf
# answer the questions, and for the FQDN/Common Name, use "10.11.13.15" or whatever the IP address
# is of your client host

# create the signing request
$ openssl x509 -x509toreq -in certs/clientreq.pem -signkey private/clientkey.pem -out tmp.pem

# sign the certificate
$ openssl ca -config openssl.cnf -policy policy_anything -out certs/clientcert.pem -infiles tmp.pem
# enter the CA passphrase when prompted - again, for this tutorial, 'root1234', and ensure that you
# opt for yes ('y') when asked to sign and commit the newly-signed certificate

# validation
$ openssl x509 -noout -text -in certs/clientcert.pem
# should output details about the certificate...
$ openssl verify -CAfile cacert.pem certs/clientcert.pem
# should output that the chain of trust is valid:
#   certs/servercert.pem: OK

# clean-up
$ rm tmp.pem
{% endhighlight %}

Once the `clientkey.pem` and `clientcert.pem` files have been generated and signed, copy (scp) them
over to the client node "node1.localhost". We will now set up the client host directory structure and
configuration to utilize the certificates for encrypted communication with the server (replace
\<CERT_PATH\> with the location where the key and certificate reside/were copied from the server):

{% highlight bash %}
# Node: node1.localhost
# put the certs and keys in place
$ cd /etc/syslog-ng/
$ sudo mkdir cert.d ca.d
$ sudo cp <CERT_PATH>/clientcert.pem <CERT_PATH>/clientkey.pem cert.d/
$ sudo cp <CERT_PATH>/cacert.pem ca.d/

# generate and symlink the appropriate hash
$ openssl x509 -noout -hash -in ca.d/cacert.pem
# copy the resulting hash, and substitute it for the <HASH> variable in the following
# command - note that the hash MUST have a suffix of '.0'
$ sudo ln -s cacert.pem ca.d/<HASH>.0
{% endhighlight %}

Now that the certificate and key files are in place, we can update the syslog-ng configuration for
the log forwarding to utilize the encryption components:

{% highlight bash %}
# Node: node1.localhost
$ sudo vim /etc/syslog-ng/conf.d/myapp.conf
# update the 'destination' resource as follows:
#   destination myapp_remote {
#     tcp("10.11.13.16"
#         port(6514)
#         tls(key_file("/etc/syslog-ng/cert.d/clientkey.pem")
#             cert_file("/etc/syslog-ng/cert.d/clientcert.pem")
#             ca_dir("/etc/syslog-ng/ca.d"))
#     );
#   };

$ sudo service syslog-ng restart
{% endhighlight %}

**NOTE**: If you do not have an adequate DNS environment set up, you will likely receive errors
such as `...certificate verify failed...` when testing this configuration. It is possible to avoid
certificate validation, but for security purposes, likely not sufficient. To get around this in the
test environment, simply update the `/etc/hosts` file on the client ("node1.localhost") to include
the IP address and hostname of the server, like so:

{% highlight bash %}
# Node: node1.localhost
$ sudo vim /etc/hosts
# ensure the file contains a line as follows:
#   10.11.13.16 node2.localhost node2
{% endhighlight %}

At this point, you should be able to test the log communication with the remote server:

{% highlight bash %}
# Node: node1.localhost
$ echo "Testing this remote log encrypted capability" | sudo tee -a /var/log/myapp.log

# Node: node2.localhost
# resulting logs should show up on the syslog-ng server:
$ sudo cat /var/log/myapp.log
#   Dec 20 22:15:11 node1 Testing this remote log encrypted capability
{% endhighlight %}

If the above functions as expected, your encrypted communication/forwarding is now complete!

#### Troubleshooting

In some cases, the hostname/identifier for the server or client may not be as expected. You can
inspect the `/var/log/syslog` file on the client host to investigate errors that may correspond to
client validation or hostname validation failures. In these cases, you will need to ensure that the
CA certificate is correctly configured on the client host and/or the Common Name of the certificate
matches what the actual reported hostname of the machine is, respectively.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [The syslog-ng Open Source Edition 3.8 Administrator Guide](https://www.balabit.com/sites/default/files/documents/syslog-ng-ose-latest-guides/en/syslog-ng-ose-guide-admin/html/index.html)
* [Secure logging using TLS](https://www.balabit.com/documents/syslog-ng-ose-latest-guides/en/syslog-ng-ose-guide-admin/html/concepts-tls.html)
* [OpenSSL Certificate Authority](https://jamielinux.com/docs/openssl-certificate-authority/)
* [TLS encryption and mutual authentication using syslog-ng Open Source Edition](https://www.linux.com/blog/tls-encryption-and-mutual-authentication-using-syslog-ng-open-source-edition)

---
layout: post
title:  "Installing SmokePing"
date:   2016-11-12 13:21:56 -0400
categories: ubuntu smokeping monitoring
---
Tutorial and steps related to the installation, configuration, and use of the
[SmokePing](http://oss.oetiker.ch/smokeping/index.en.html) software product for monitoring latency
within network environment.

### Background

The ability to monitor your network environments can be handled via a number of various tools and
services, not the least of which are SaaS products provided through external vendors. If you are the
"do it yourself" kind of engineer, exploring a tool such as
[SmokePing](http://oss.oetiker.ch/smokeping/index.en.html) is an easy and affordable (free) way to
get some very quick latency information exposed to your customers.

This post will explore the installation, configuration, and (basic) usage of the SmokePing product.
It is intended to be a primer on the subject, not a full/in-depth tutorial and, as such, some aspects
of the "productization" of the service will not be included.

### Underlying Compute Technology/Ecosystem

This tutorial utilizes/makes the following assumptions about your compute infrastructure. Although the
instructions can be extended to accommodate cloud-based and/or other infrastructure environments quite
easily, all details are explained with respect to local development for the purposes of simplicity:

- **Hypervisor Technology**: VirtualBox
- **Provisioner**: Vagrant
- **Number of VMs**: 2
- **Operating System**: Ubuntu 16.04
- **Arch**: 64-bit
- **CPUs**: 2
- **Mem**: 4GB
- **Disk**: 50GB

This tutorial is built with 2 virtual machines serving as the compute resources - one of the resources
will be the Master/UI instance while the other will be a dedicated poller/Slave instance. The following
hostnames and corresponding IP addresses will be used for the 2 virtual machine instances:

- **node1.localhost**: 10.11.13.15
- **node2.localhost**: 10.11.13.16

The following software versions are used as part of this tutorial - again, versions can/may be changed,
but there is no guarantee that using different versions will result in a working/functional technology
stack. The versions listed are such that they represent a fully-functional and working technology
ecosystem:

- **RRDTool**: 1.6.0 (dependency)
- **Apache2**: 2.4.18-2 (Master - UI web server)
- **SmokePing**: 2.6.11 (internal version 2.006011)

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

At the end of this tutorial, the following end-state architecture will be realized:

{% highlight bash %}
|--------------------|               |--------------------|
|       Master       |               |        Slave       |
|   node1.localhost  |               |   node2.localhost  |
|    (10.11.13.15)   |               |    (10.11.13.16)   |
|                    |               |                    |
|     |----------|   |               |     |----------|   |
|     |  Master  |   |               |     |  Slave   |   |
|     |          |   |     Ping      |     |          |   |
|     |          |   |<------------------->| |------| |   |
|     |          |   |   Ping Data   |     | |Poller| |   |
|     |          |<------------------------| |------| |   |
|     |          |   |               |     |----------|   |
|     |          |   |    Ping       |                    |
|     | |------| |<----------------->|                    |
|     | |Poller| |   |   Ping Data   |--------------------|
|     | |------| |<---------|
|     |          |   |      |
|     |          |<---------|
|     |----------|   |
|                    |
|--------------------|
{% endhighlight %}

### Prerequisites/Dependencies

SmokePing requires several components to be installed prior to installing the software on your
host system. Most Ubuntu-based distributions come with Perl pre-installed - if this is not the
case, install Perl for your system using the native package manager. In addition, a C-compiler
is required in order to install the versions of software specific to this tutorial. Note that
we will install some prerequisites for both nodes up front in preparation for the Slave node
configuration towards the end of the tutorial, while other packages are strictly installed on
the Master due to the capabilities desired (i.e. sendmail for Email handling):

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo apt-get update
$ sudo apt-get -y install perl \
                          gcc \
                          libpango1.0-dev \
                          libxml2-dev \
                          fping

# Node: node1.localhost
$ sudo apt-get -y install sendmail
{% endhighlight %}

Once Perl is installed, you will need RRDTool - to install the version used in this tutorial,
perform the following steps. Note that the actual build and installation of the rrdtool software
is only done on `node1.localhost` where the actual graphs are generated:

{% highlight bash %}
# Node: node1.localhost
# download, extract, and build the rrdtool binary from source
$ wget http://oss.oetiker.ch/rrdtool/pub/rrdtool-1.6.0.tar.gz
$ tar xzf rrdtool-1.6.0.tar.gz
$ cd rrdtool-1.6.0/
$ sudo ./configure
$ sudo make
$ sudo make install
$ sudo ln -s /opt/rrdtool-1.6.0/bin/rrdtool /usr/bin/rrdtool
{% endhighlight %}

Following the installation, running the following command should result in the output of the
rrdtool version installed:

{% highlight bash %}
# Node: node1.localhost
$ rrdtool --version
# should output:
#   RRDtool 1.6.0 ...
{% endhighlight %}

This SmokePing installation is going to use the Apache web server with CGI support to display its web
pages - install the Apache software/supporting module and create the SmokePing directory for serving
the application via Apache:

{% highlight bash %}
# Node: node1.localhost
$ sudo apt-get -y install apache2 \
                          libapache2-mod-fcgid
$ sudo mkdir /var/www/smokeping
{% endhighlight %}

Finally, we need to configure some defaults related to resolving the hostnames between the node1.localhost
and node2.localhost nodes. Update the `/etc/hosts` file on each node to provide name resolution:

{% highlight bash %}
# Node: node1.localhost
$ echo "10.11.13.16 node2.localhost node2" | sudo tee -a /etc/hosts

# Node: node2.localhost
$ echo "10.11.13.15 node1.localhost node1" | sudo tee -a /etc/hosts
{% endhighlight %}

### Installation

Once the installation prerequisites are in place, you can download and install the SmokePing product.
First, download, extract, and compile the SmokePing package for the version you wish to install to
the Master and Slave nodes:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ wget http://oss.oetiker.ch/smokeping/pub/smokeping-2.6.11.tar.gz
$ tar xzf smokeping-2.6.11.tar.gz
$ cd smokeping-2.6.11/
{% endhighlight %}

To ensure that the appropriate Perl modules are in place, run the setup script to install any that
may be missing, along with the `apt-get install` command for the librrds-perl package. Although the
RRD files and corresponding graphs are only generated on the Master instance, the build of the
SmokePing software will ultimately fail if the `librrds-perl` package is not installed, so we will
install this on both instances:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo ./setup/build-perl-modules.sh /opt/smokeping/thirdparty
# should see a bit of output, possibly with an ending like so:
#   ...
#   13 distributions installed

# install the rrds package for Perl
$ sudo apt-get -y install librrds-perl
{% endhighlight %}

Once the package has been extracted and dependencies for the Perl environment are in place, proceed
to building and installing the software:

{% highlight bash %}
# Node: node1.localhost
$ sudo ./configure --prefix=/opt/smokeping
# if successful, you should see something like the following
#   ...
#   Continue installation with
#     /usr/bin/make install

$ sudo /usr/bin/make install
{% endhighlight %}

If the above commands were successful, you should be able to run the following and receive output
corresponding to the version of SmokePing installed:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo /opt/smokeping/bin/smokeping --version
# should output a version like so:
#   2.006011
{% endhighlight %}

### Configuration

Now that the SmokePing application code is in place, we can continue to configure the application
for use. In this particular installation, the configuration file was created with a trailing `.dist`
name. This will cause some issues with the way the scripts are expecting the configuration file to
exist and, as such, we will create a symlink to the configuration file without the `.dist` trailing
name to ensure everything works as expected:

{% highlight bash %}
# Node: node1.localhost
$ sudo ln -s /opt/smokeping/etc/config.dist /opt/smokeping/etc/config
{% endhighlight %}

Edit the /opt/smokeping/etc/config file for your environment. There are a lot of configuration options
available and the format is quite complicated - we will focus on a few paramaters to start with:

{% highlight bash %}
# Node: node1.localhost
$ sudo vim /opt/smokeping/etc/config
# some useful parameters to set:
#   owner = <YOUR_NAME>
#   contact = <YOUR_EMAIL>
#   mailhost = <YOUR_SMTP_SERVER>
#   sendmail = /usr/sbin/sendmail
#   ...
#   *** Alerts ***
#   to = <YOUR_EMAIL>
#   from = alert@smokeping.com
#   ...
#   + FPing
#   binary = /usr/bin/fping
#   ...
#   title = SmokePing Tutorial
#   remark = This is a SmokePing tutorial
{% endhighlight %}

In addition to the above configuration changes, re-open/edit the configuration file to replace
the section `++ James` with the Master/localhost configuration information so that data can be
visualized for the Master instance:

{% highlight bash %}
# Node: node1.localhost
$ sudo vim /opt/smokeping/etc/config
# replace '++ James' with '++ Master' below:
#   ++ Master
#   menu = node1.localhost
#   title = node1.localhost
#   alerts = someloss
#   host = node1.localhost

# create a new record below the Master configuration listed above:
#   ++ Slave
#   menu = node2.localhost
#   title = node2.localhost
#   alerts = someloss
#   host = node2.localhost
{% endhighlight %}

Before starting the application, several directories must exist in order for the out of box
configuration to function as expected, and certain permissions must be in place for the out
of box security to work:

{% highlight bash %}
# Node: node1.localhost, node2.localhost
$ sudo mkdir /opt/smokeping/{data,cache,var}
$ sudo chmod 640 /opt/smokeping/etc/smokeping_secrets.dist

# Node: node1.localhost
$ sudo chown -R www-data /opt/smokeping/cache
$ sudo ln -s /opt/smokeping/cache /var/www/smokeping/cache
{% endhighlight %}

The Apache server needs to be configured to serve the web interface for the SmokePing application -
to configure this, copy the files from the SmokePing `htdocs` folder into the respective Apache web
folder, and edit the Apache default virtual host file:

{% highlight bash %}
# Node: node1.localhost
$ sudo cp -r /opt/smokeping/htdocs/* /var/www/smokeping/
$ sudo vim /etc/apache2/sites-available/000-default.conf
# update the following directive in the file:
#   DocumentRoot /var/www/smokeping
#   <Directory /var/www/smokeping>
#       DirectoryIndex smokeping.fcgi.dist
#       Options +ExecCGI
#   </Directory>
{% endhighlight %}

Once the configuration file is in place, enable the FastCGI and SUExec modules and restart the
Apache web server:

{% highlight bash %}
# Node: node1.localhost
$ sudo a2enmod fcgid suexec
$ sudo service apache2 restart
{% endhighlight %}

Now that the web server is up and running, visit the web page located at http://10.11.13.15/ and
you should see the SmokePing web interface presented to you. There is likely no data due to not yet
having run any pollers to collect information.

We need to start the SmokePing daemon in order to start generating some data and graphs. Start by
launching the application in the foreground ("debug" mode) and inspecting for errors:

{% highlight bash %}
# Node: node1.localhost
$ sudo ./bin/smokeping --config=/opt/smokeping/etc/config.dist \
                       --debug
{% endhighlight %}

After some time, the output will generate some results that ultimately result in some graphs being
generated. Navigate back to your browser and reload the webpage - in the left menu navigation,
select the "Targets -> node1.localhost" options to view the generated graphs. If no data appears but
no errors are seen in the command-line debug session, leave the agent running in the foreground for
a few minutes until the page appears with data.

If all is successful, you can terminate the foreground agent job and launch the smokeping agent in
daemon mode to have it run continuously in the background on your Master node like so:

{% highlight bash %}
# Node: node1.localhost
$ sudo ./bin/smokeping --config=/opt/smokeping/etc/config.dist \
                       --logfile=/var/log/smokeping.log
{% endhighlight %}

Refresh your web page and you should see updates to the graphs for the Master node (node1.localhost).
The graph will appear towards the top of the page. You can also select the "Test -> node2.localhost"
node to view the graph data for the Slave node as well (being pinged from the Master).

### Slave Poller

Now let's add a secondary poller on the Slave "node2.localhost". This poller will check both the
Master and the Slave and will demonstrate distributing pollers on separate machines that report
back to the Master instance. Note that in order for the following configurations/setup to work, the
result of the command `hostname` on the `node2.localhost` node should report `node2`.

First, we need to configure the Master instance to accept the Slave communication as well as tell the
Slave which nodes to poll - update the configuration sections as follows on the Master:

{% highlight bash %}
# Node: node1.localhost
$ sudo vim /opt/smokeping/etc/config.dist
# create a new record below the *** Slaves *** section::
#   *** Slaves ***
#   ...
#   +node2
#   display_name = node2.localhost
#   color = 00ff00

# update the Master and Slave configuration sections:
#   ++ Master
#   ...
#   slaves = node2
#   ...
#   ++ Slave
#   ...
#   slaves = node2
#   ...
{% endhighlight %}

Before reloading the configuration, we need to update the secret for the Slave to be able to
communicate with the Master, and ensure the Apache process can read it:

{% highlight bash %}
# Node: node1.localhost
$ sudo vim /opt/smokeping/etc/smokeping_secrets.dist
# ensure contains at least 1 line like so:
#   node2=supersecretpassword
$ sudo chown www-data /opt/smokeping/etc/smokeping_secrets.dist
{% endhighlight %}

Next, reload the configuration on the Master node for the new Slave information to take effect,
and ensure that the Apache process is also restarted for this to work:

{% highlight bash %}
# Node: node1.localhost
$ sudo ./bin/smokeping --reload
$ sudo service apache2 restart
{% endhighlight %}

Once you've updated the configuration on the Master node, we need to set up the Slave instance.
Note that if you try to view the web interface at this point you will likely see various errors
corresponding to missing 'slave1' data/graphs - this is because the Slave has not yet been
configured and communicated with the Master for the first time.

If you ran the commands to install the SmokePing on the Slave instance in parallel as the instructions
at the beginning of this tutorial stated, you can simply proceed with the next steps (if you did
not follow the instructions and do not yet have the SmokePing application installed, go back and
repeat the relative steps for the Slave instance above).

First, there are a few Perl dependencies we need to install by hand since we did not install Apache
and/or the relative dependencies associated with the Master instance:

{% highlight bash %}
# Node: node2.localhost
$ sudo cpan CGI \
            Config::Grammar \
            Digest::HMAC_MD5 \
            LWP::UserAgent
{% endhighlight %}

Once complete, create a secret file on the Slave so that when the process starts it knows how to
communicate with the Master instance, and secure the file:

{% highlight bash %}
# Node: node2.localhost
$ echo "supersecretpassword" | sudo tee /opt/smokeping/etc/secret.txt
$ sudo chmod 600 /opt/smokeping/etc/secret.txt
{% endhighlight %}

Now you are ready to launch the poller:

{% highlight bash %}
# Node: node2.localhost
$ sudo /opt/smokeping/bin/smokeping --master-url=http://10.11.13.15/ \
                                    --cache-dir=/opt/smokeping/cache \
                                    --shared-secret=/opt/smokeping/etc/secret.txt \
                                    --logfile=/var/log/smokeping.log
# should produce output like the following:
#   Sent data to Server and got new config in response.
#   Note: logging to syslog as local0/info.
#   Daemonizing /opt/smokeping/bin/smokeping ...
{% endhighlight %}

If you received output as indicated, your Poller/Slave instance has successfully established
connectivity and is communicating with the Master. If you received some other kind of output,
inspect the log output to see what may have gone wrong.

Navigate back to the SmokePing web interface and refresh/reload the web page. You should now see
stacked graphs showing poller results for the Master node poller along with a secondary graph
indicating the results from the node2.localhost (Slave) poller. If there is no data within the
secondary graph, give the poller a few minutes to finish its first round of polling and sending
the results to the Master for rendering.

There are many more additional capabilities that are possible with this tool - this primer is
stricly a starting point. Visit the project page for more information about the tool and its
operation.

### Disclaimers

As stated in prior tutorials, this tutorial is in no way intended to be a 'production-ready'
setup. There are various tuning and configuration parameters, security settings, and overall
architecture/directory structure changes that would need to be made in order to make this
tutorial match a production-ready setup of the SmokePing application. It is intended to be a
primer of installing and configuring the application for basic use and, as such, the knowledge
contained within can be leveraged to further develop a more production-ready solution.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [SmokePing Install](http://oss.oetiker.ch/smokeping/doc/smokeping_install.en.html)
* [SmokePing Master/Slave](http://oss.oetiker.ch/smokeping/doc/smokeping_master_slave.en.html)

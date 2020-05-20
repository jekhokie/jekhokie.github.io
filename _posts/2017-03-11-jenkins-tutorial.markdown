---
layout: post
title:  "Jenkins Tutorial"
date:   2017-03-11 10:45:19 -0400
categories: ubuntu linux jenkins ci devops
logo: jenkins.jpg
---
It has been some time since I've actually installed and used the [Jenkins](https://jenkins.io/)
software for continuous integration within a CI/CD pipeline. This tutorial is to get re-acquainted
with the service and offer some insight into installing, configuring, and setting up a basic Ruby
project within Jenkins.

### Technology Ecosystem

In order to establish the basic pipeline for Jenkins, we will utilize git as our source code
repository. VirtualBox/Vagrant is used to set up a virtual machine that is capable of hosting
the Jenkins software:

**Jenkins VM**

- **OS**: Ubuntu 16.04
- **CPU**: 2
- **RAM**: 2048MB
- **Disk**: 50GB
- **Network**: Private Network
- **IP**: 10.11.13.15

In addition, we will assume the following software versions:

- **Jenkins**: 2.7.4
- **Ruby**: 2.3.1
- **RVM**: 1.29.1

#### WARNING

This tutorial sets up the **minimum** items required to install, configure, and use the Jenkins
software. It does not include details related to scalability, high-availability, distributed job
nodes, web server configuration, or the like. If you wish to set up Jenkins for a production
environment, you will absolutely want to consider these missing points. In addition, we make
assumptions to simplify this tutorial, such as installing Ruby and RVM on the Jenkins instance
itself, hosting the git repository locally, etc. which again, should likely not be performed for
production-load systems.

### Installing Dependencies

First we will install some required dependencies:

{% highlight bash %}
$ sudo apt-get install python-software-properties python-pycurl daemon
$ sudo apt-add-repository ppa:openjdk-r/ppa
# press <Enter> when prompted
$ sudo apt-get update
$ sudo apt-get install openjdk-7-jdk

# validate the Java JDK installation
$ java -version
# output should be similar to the following:
#   java version "1.7.0_95"
#   OpenJDK Runtime Environment (IcedTea 2.6.4) (7u95-2.6.4-3)
#   OpenJDK 64-Bit Server VM (build 24.95-b01, mixed mode)
{% endhighlight %}

### Installing Jenkins

Once the dependencies are in place, we can download and install Jenkins. The package required for this
tutorial can be downloaded [here](https://pkg.jenkins.io/debian-stable/binary/jenkins_2.7.4_all.deb).
Once you've downloaded the software, run the following commands to install/start the Jenkins server:

{% highlight bash %}
$ sudo dpkg -i jenkins_2.7.4_all.deb
{% endhighlight %}

Once the package installs, the service is auto-started. Visit the following URL to access the Jenkins
interface, which should inform you that you must unlock the Jenkins interface via a password written
to the log files for Jenkins:

[http://10.11.13.15:8080/](http://10.11.13.15:8080/)

If you see the modal stating the "unlock" comment, back in the terminal for the Jenkins VM, get the
password from the file `/var/lib/jenkins/secrets/initialAdminPassword`, paste it into the prompt, and
click "Continue" in the browser to start configuring Jenkins. Note that you will likely need to access
the file using `sudo` as the permissions for the file are locked down.

Your Jenkins instance is now installed and ready to be configured.

### Configuring Jenkins

Now that the base Jenkins software has been installed, you will need to configure it for your needs.
The first modal presented following the password unlock is a prompt to select whether you wish to have
'default' plugins installed, or manually select which plugins you wish to install. For this tutorial,
select "Install suggested plugins" (we will re-visit the plugin selection process shortly).

Once you elect to have Jenkins install suggested plugins, you will be presented with an installation
progress screen - wait for the plugin installation to complete (could take a couple/few minutes). You
will then be prompted to create an Admin user - enter the information required and make sure you save
the credentials in a safe location. For our purposes, we will use the username 'admin' and password
'admin' to make it easy to remember.

Once you are logged into the Jenkins instance, you will want to install the Ruby/RVM plugins for this
tutorial as we will be testing the installation using a simple Ruby project. Select "Manage Jenkins"
from the left-hand menu, and then select "Manage Plugins" from the resulting menu - this will bring
you to the plugins configuration screen. Click the "Available" tab, and search for/select the plugin
named "Rvm" - then click "Install without restart". You will be brought to the plugins page and see
the Rvm plugin selected being installed. Once you see "Success" for the plugin installation, your
Jenkins software and plugins are installed, configured, and ready for use.

### Test Job

Now that we have a fully-functional Jenkins instance, let's create a test job to ensure that it works
as we expect. Since this job is going to be a Ruby job using RVM, we first need to install the RVM
software/environment for the Jenkins user so that it can perform our builds:

{% highlight bash %}
# switch to the 'jenkins' user:
$ sudo su - jenkins

# note that the next commands are correct for the time of this tutorial - please
# visit https://rvm.io when performing these two steps for the latest key/install
$ gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
$ curl -sSL https://get.rvm.io | bash -s stable
$ source /var/lib/jenkins/.rvm/scripts/rvm
$ rvm install ruby-2.3.1
# note that the install of the base ruby package may result in RVM attempting to
# update system dependencies - if so, it's best to ensure that you update the
# dependencies as the root user and NOT grant the Jenkins user sudo permission to
# to so

# return to the 'vagrant' user for all following commands if you have not yet done so:
$ exit
{% endhighlight %}

Next, we'll want to create a git repository somewhere that Jenkins can access it. For simplicity and
isolation, we'll create a git repository directly on the local VM that the Jenkins service is located.
Perform the following on the Jenkins VM instance (can be done as the `vagrant` user, which is why
there are no `sudo` prefixes in the below commands):

{% highlight bash %}
# create and initialize the repository
$ mkdir /home/vagrant/test-repo
$ cd /home/vagrant/test-repo
$ git init

# create the Ruby sample application:
$ rvm --ruby-version use 2.3.1@test-repo --create
$ vim Gemfile
# ensure contains the following:
#   source "https://rubygems.org"
#   gem "rspec"
$ mkdir lib
$ vim lib/hello_world.rb
# ensure contains the following:
#   class HelloWorld
#     def self.say_it
#       "Hello World"
#     end
#   end
$ mkdir spec
$ vim spec/hello_world_spec.rb
# ensure contains the following:
#   require 'hello_world'
#   describe HelloWorld do
#     describe ".say_it" do
#       it "returns 'Hello World'" do
#         expect(HelloWorld.say_it).to eq('Hello World')
#       end
#     end
#   end

# commit the sample Ruby application
$ git add .
$ git commit -m "Initial Commit"
{% endhighlight %}

Once you've created the repository, navigate back to the Jenkins interface in your browser. Select
"New Item" from the menu on the left-side of the screen. When prompted, select "Freestyle project"
and name it "test-job", then select "OK". You will then be prompted with a screen to configure the job.

For the job configuration, configure the respective sections as follows:

- **Source Code Management**: Select the "Git" radio button and enter `file:///home/vagrant/test-repo`
for the "Repository URL" field.
  - Note that depending on your desired build settings, you may want to ensure that all branches are
automatically built on detected changes, but for this tutorial, we will simply focus on the master
branch (default values specified by Jenkins in this configuration section - no change required).
- **Build Environment**: Ensure that the option "Run the build in a RVM-managed environment" is
selected, and accept the default "." option for the "Implementation" parameter.
- **Build**: Select the drop-down named "Add build step" and select "Execute shell". In the "Command"
parameter text box, specify the following:
{% highlight bash %}
#!/bin/bash
gem install bundler
bundle install
rspec spec
{% endhighlight %}

Once you have configured the above, click "Save" to create/save the job and you will be brought to
the job screen. In the left menu, select "Build Now" to trigger a built - after waiting a few seconds,
you should see a build number under "Build History" show up. Click on the build number under "Build
History" - if all went well, the build should have succeeded. You can click on the "Console Output"
option in the left menu of the build screen to inspect what occurred during the build, which should
show the installation of bundler and corresponding gems, followed by RSpec running the tests we
specified in the sample Ruby application, resulting in "1 example, 0 failures" -> "SUCCESS" at the
end of the output.

### Summary

This tutorial demonstrated the basic functionality of Jenkins using a sample Ruby application and
RSpec test. At this point, you should understand the basics of setting up an almost vanilla instance
of Jenkins and configuring a job. You can move on to more useful configurations such as periodic
builds, building of branches within a git repository, automated deployment triggers, notifications
via chat/email/other, etc.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Jenkins Documentation](https://jenkins.io/doc/)
* [How to Install Jenkins Automation Server with Apache on Ubuntu 16.04](https://www.howtoforge.com/tutorial/how-to-install-jenkins-with-apache-on-ubuntu-16-04/)
* [Using a local Git repository in Jenkins](https://minuteware.net/2014-02-09-using-a-local-git-repository-in-jenkins.html)

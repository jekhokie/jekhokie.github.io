---
layout: post
title:  "Ansible Tutorial"
date:   2017-03-12 16:08:51 -0400
categories: ubuntu linux ansible devops
---
Keeping with the "things I haven't used in a while" theme, this tutorial is a re-visit of the
[Ansible](https://www.ansible.com/) automation platform. It will cover the basics of how to install
and configure Ansible on an Ubuntu 16.04 instance as well as use it to provision a Jenkins
master instance based on [this previous post]({% post_url 2017-03-11-jenkins-tutorial %}).

### Background

There are a few major automation frameworks at play these days - specifically (and most major),
[Puppet](https://puppet.com/), [Chef](https://www.chef.io/), and [Ansible](https://www.ansible.com/).
Each of these technologies touts feature sets that vary from simple SSH-based provisioning to full
life-cycle automation and compliance enforcement. As Ansible has been top of mind and mentioned by
at least a few individuals I've interacted with over the last few months, this post will focus on
the installation, configuration, and use of the Ansible automation framework. Specifically, at the
end of this post, you will have a fully configured Ansible framework and playbook capable of installing
and configuring a Jenkins instance based on [this previous post]({% post_url 2017-03-11-jenkins-tutorial %}).

### Technology Ecosystem

There will be 2 hosts for this tutorial, each provisioned through Vagrant. The first host will serve
as the Ansible Controller Machine, and the second host will serve as the target on which we wish to
install the Jenkins continuous integration and build server. Specifically, the following setup will
be realized:

**Ansible VM**

- **Hostname**: ansible.localhost
- **OS**: Ubuntu 16.04
- **CPU**: 2
- **RAM**: 2048MB
- **Disk**: 50GB
- **Network**: Private Network
- **IP**: 10.11.13.15

**Jenkins VM**

- **Hostname**: jenkins.localhost
- **OS**: Ubuntu 16.04
- **CPU**: 2
- **RAM**: 2048MB
- **Disk**: 50GB
- **Network**: Private Network
- **IP**: 10.11.13.16

In addition, we will assume the following software versions:

- **Ansible**: 2.2.1.0
- **Jenkins**: 2.7.4

### Notes

The code blocks included in this tutorial will list, as a comment, the node(s) that the commands
following need to be run on. For instance, if required on all nodes, the code will include a comment
like so:

{% highlight bash %}
# Node: ansible.localhost, jenkins.localhost
{% endhighlight %}

If the code block only applies to one of the nodes, it will list the specific node it applies to
like so:

{% highlight bash %}
# Node: ansible.localhost
{% endhighlight %}

All commands assume that your user is `vagrant` and the user's corresponding home directory is
`/home/vagrant` for the purposes of running sudo commands.

### Install Ansible

Perform the following on the Ansible virtual machine to install the Ansible software:

{% highlight bash %}
# Host: ansible.localhost
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible

# check the Ansible installation:
$ ansible --version
# should output something similar to the following:
#   ansible 2.2.1.0
#     config file = /etc/ansible/ansible.cfg
#     configured module search path = Default w/o overrides
{% endhighlight %}

### Configure Ansible User and SSH Key

Once the Ansible software has been installed, create an `ansible` user and group on both the Ansible
VM and Jenkins VM. Note that this is helpful when things are being managed by multiple users -
**however**, if you wish to use a service account in managing your infrastructure, ensure you have
a good audit trail that can trace actions back to a physical human being to ensure you maintain
auditability of actions taken on your infrastructure. In addition, the user ID in the following
commands are just examples and you can utilize whatever mechanism and actual value you wish for
assigning IDs to each item:

{% highlight bash %}
# Host: ansible.localhost, jenkins.localhost
$ sudo useradd -m -d /home/ansible -G sudo -s /bin/bash -u 501 -U ansible

# verify the user was created successfully
$ sudo id -a ansible
# should output the following:
#   uid=501(ansible) gid=1000(ansible) groups=1000(ansible),27(sudo)
{% endhighlight %}

In order for Ansible to accurately manage most aspects of a system, it needs to be able to execute
password-less sudo on the target instance. Therefore, we will add an entry to the `/etc/sudoers` file
on the `jenkins.localhost` VM that will allow passwordless sudo for the Ansible user:

{% highlight bash %}
# Host: jenkins.localhost
$ sudo visudo
# at the very end of the file, insert the following, then save and exit
#   ansible    ALL=(ALL)    NOPASSWD: ALL
{% endhighlight %}

Next, we need to set up the SSH keys for the user. We will create the SSH key on the Ansible host and
copy it to the `authorized_keys` file of the `ansible` user on the Jenkins host:

{% highlight bash %}
# Host: ansible.localhost
$ sudo su - ansible
$ ssh-keygen -t rsa -b 2048
# accept default for location, and press <Enter> for no passphrase

# Host: jenkins.localhost
$ sudo su - ansible
$ mkdir .ssh
$ vim .ssh/authorized_keys
# add the contents of the /home/ansible/.ssh/id_rsa.pub file on
# the ansible.localhost host to this file, then write and quit
{% endhighlight %}

To test that the SSH key setup worked as expected, attempt to SSH from the `ansible.localhost` host
to the `jenkins.localhost` host and run a command using `sudo`. If you are able to SSH and run `sudo`
without any password prompt, your configuration is now complete:

{% highlight bash %}
# Host: ansible.localhost
$ sudo su - ansible
$ ssh 10.11.13.16
# you should now be on the "jenkins.localhost" host as the ansible user
$ sudo echo "This worked!"
# you should see "This worked!" echoed to the terminal without a password prompt
{% endhighlight %}

### Configure Inventory and Ping

Now that the SSH key and sudo setup are complete, we can start configuring Ansible to understand
our current environment. First, edit the host file for Ansible to make it aware of the Jenkins
host:

{% highlight bash %}
# Host: ansible.localhost
$ sudo vim /etc/ansible/hosts
# add the IP of the Jenkins host to the end of the file,
# then write and quit the file:
#   10.11.13.16
{% endhighlight %}

Once the host file contains the Jenkins host IP, run the ansible `ping` command as the `ansible`
user to check that the Jenkins host is reachable by Ansible:

{% highlight bash %}
# Host: ansible.localhost
$ sudo su - ansible
$ ansible all -m ping --sudo
# should output something similar to the following, indicating success:
#   10.11.13.18 | SUCCESS => {
#       "changed": false,
#       "ping": "pong"
#   }
{% endhighlight %}

If the above output returns, your Ansible configuration between the hosts is now complete and we
can start building the playbook for the Jenkins installation.

### Jenkins Playbook

Ansible utilizes what are known as playbooks to specify commands to run to provision hosts. The
playbooks are written in YAML syntax. We will be creating a playbook that corresponds to the steps
required to install the Jenkins service. We will utilize the directory layout specified
[here](http://docs.ansible.com/ansible/playbooks_best_practices.html#directory-layout) and since
we only have 1 host, we will simply include the playbook for that host in the master `site.yml`
playbook - create a `site.yml` file in the `ansible` user's home directory and ensure it contains
the following:

{% raw %}
```
---
- hosts: all
  become: true
  vars:
    dependencies: [ 'python-software-properties', 'python-pycurl', 'daemon' ]
    jenkins_deb: jenkins_2.7.4_all.deb
  tasks:
  - name: Update Apt Cache
    apt:
      update_cache: true
  - name: Install Dependencies
    apt:name={{ item }} state=latest
    with_items: "{{ dependencies }}"
  - name: Add OpenJDK Repository
    apt_repository:
      repo: 'ppa:openjdk-r/ppa'
  - name: Install OpenJDK
    apt: name=openjdk-7-jdk state=latest
  - name: Download Jenkins Package
    get_url:
      url: https://pkg.jenkins.io/debian-stable/binary/{{ jenkins_deb }}
      dest: /tmp/{{ jenkins_deb }}
      mode: 770
  - name: Install Jenkins
    apt: deb="/tmp/{{ jekins_deb }}"
```
{% endraw %}

One item to note about the above code - this code is very elementary and does not consider situations
such as whether software is already installed, a file is present prior to utilizing it, etc. The point
is, there are many optimizations that can be made to the code (as well as defensive mechanisms), but
for simplicity sake, these items are included in this tutorial. A brief explanation of what the above
`site.yml` master playbook accomplishes in the respective order:

- Target all hosts (since we only have 1).
- Run commands as root (using `sudo`).
- Update the Apt cache (just in case brand new VM is out of date).
- Install required dependencies and OpenJDK.
- Download and install the Jenkins package.

Now that the playbook is defined, we can run it:

{% highlight bash %}
# Host: ansible.localhost
$ sudo su - ansible
$ ansible-playbook site.yml
# at this point, ansible will kick off the playbook on the remote
# Jenkins host - if all goes well, you should see success logs
# scroll by and an overall success for the installation
{% endhighlight %}

If all went well, you can now visit the Jenkins UI via visiting the following URL in your browser:

[http://10.11.13.16:8080](http://10.11.13.16:8080)

If the Jenkins UI pops up, you have successfully completed the Jenkins installation using Ansible!

### Summary

There are many other useful things associated with Ansible that were not even considered in this post.
I encourage you to look at the Ansible documentation as it is quite complete and understandable, and
details the true capabilities of the Ansible automation framework. In addition, having a pattern and
standards associated with how to construct Ansible playbooks, share code, audit actions, etc. is
essential to ensuring the integrity of your resources and cleanliness of your Ansible code. Organization
of your code, grouping of your hosts, and least-privileged access are just some components that will
be most critical to mass adoption of the Ansible framework while being able to maintain the integrity
of your environments.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Ansible Documentation](http://docs.ansible.com/ansible/index.html)

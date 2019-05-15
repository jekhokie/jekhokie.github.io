---
layout: post
title:  "CI/CD Pipeline for Infrastructure VMs"
date:   2019-05-14 18:15:02 -0400
categories: ci cd jenkins virtualbox vm infrastructure docker ansible
---
In a traditional IT/Operations department, there is often slow feedback in delivering updated VM templates that
are hardened and tuned (with possibly platform components installed), so by the time someone discovers there is a
zero-day vulnerability or memory leak in a platform software component, there is a day(s) (sometimes weeks) long
process to roll out new infrastructure or patch existing. This post details a sample CI/CD pipeline and process that
enables security, infrastructure and platform teams to manage the templates used for software deployments in a
software-like delivery method (infrastructure as code, CI process with smoke testing and fast feedback, etc). The
tutorial attempts to establish a governance and feedback process to arrive at AWS AMI-like functionality where an
automated pipeline, along with Ansible scripts and other code, produces new VM templates that can be consumed by
an organization, providing fast feedback to infrastructure, security, and platform teams if and when a change ends
up breaking downstream VM template components.

### Disclaimer and Project Materials

As a note, this setup is intended to prove functionality and is more of a proof of concept - the development and
architecture is intended to enable local functionality as much as possible. If desirable, the expectation is this
architecture would be adapted and designed for production workloads and not run like the proof of concept.

There is a sample repository that contains the end-state code that can be used for this tutorial if desired.
The project can be found [here](https://github.com/jekhokie/infra-ci-pipeline).

### Proposed Pipeline and Functionality

This tutorial assumes a standard Jenkins pipeline that consists of the following stages:

```bash
       1                  2                  3                  4
 |-----------|      |-----------|      |----------|      |------------|
 |  Vanilla  |      | Security  |      |  Kernel  |      | Middleware |
 | Operating | ---> | Hardening | ---> |  Tuning  | ---> |   Install  |
 |  System   |      |           |      |          |      |            |
 |-----------|      |-----------|      |----------|      |------------|
                                         /     \               |   \
                                        /       \              |     \
                                       V         V             V       V
                                |----------| |----------| |----------| |------------|
                                | Hardened | | Hardened | | Hardened | |  Hardened  |
                           5    |  CentOS  | | Windows  | | Linux    | |  Windows   |
                                | Template | | Template | | Tomcat   | | SQL Server |
                                |          | |          | | Template | |  Template  |
                                |----------| |----------| |----------| |------------|
```

As you can see above, the idea is to have a Jenkins pipeline with various build stages, each representing the various
teams/skills involved with releasing a VM template that can be consumed which is either a hardened/tuned operating system
(base starting point) or inclusive of some platform and middleware technology for management and patching of that
technology in a fast-feedback fashion. The various stages are explained below (proposed):

1. Starting with a vanilla operating system (i.e. CentOS, Windows, etc), create a base template (or download one).
2. Apply any and all security hardening scripts against the base OS templates to harden them.
3. Apply any kernel tuning against the security-hardened templates.
4. Install and configure any/all Middleware and Platform software.
5. At points 3 and 4, you end up with VM templates produced in 2 categories - first, VMs that are base operating systems
hardened, and second (stage 4) VMs that include hardened operating systems plus platform/middleware software installed
and configured.

Each of the steps is optional/specific to your needs and you may or may not elect to produce VM templates at the various
stages (depending on whether they are useful to your consumers). For instance - if your consumers always manage their
own installs of platform or middleware software, you may elect to only produce security-hardened operating systems, etc.

### Required Software and Architecture

For the purposes of this tutorial, we will set up, configure, and realize the above pipeline architecture using the
following software stack:

* [Docker for Mac](https://docs.docker.com/v17.12/docker-for-mac/install/): Software to deploy and manage Docker containers on a MacBook.
* [Jenkins](https://jenkins.io/): Traditional CI pipeline, deployed via Docker container locally on the MacBook laptop.
* [VirtualBox](https://www.virtualbox.org/): Virtualization technology to create and save VMs, installed on the MacBook laptop.
* [Git](https://git-scm.com/): Source control used for the sample project which will realize the Jenkins pipeline, installed on the MacBook laptop.
* [GitHub](https://github.com/): Remote source control repository storage.

The following architecture will be realized by the end of this tutorial:

```bash
 |------------------------------------------------------------|  |-----------------------|
 |                      MacBook Pro                           |  |      github.com       |
 | |-------------------------------------|  |---------------| |  |                       |
 | |             VirtualBox              |  | Docker Engine | |  | |-------------------| |
 | |                                     |  |               | |  | | infra-ci-pipeline | |
 | |  |-----------|    |------------|    |  | |-----------| | |  | |  git repository   | |
 | |  | Base OS   |-|  | Middleware |-|  |  | |  Jenkins  | | |  | |-------------------| |
 | |  | Templates | |  | Templates  | |  |  | |  Docker   | | |  |                       |
 | |  |-----------| |  |------------| |  |  | | Container | | |  |-----------------------|
 | |    |-----------|    |------------|  |  | |-----------| | |
 | |                                     |  |               | |
 | |-------------------------------------|  |---------------| |
 |                                                            |
 |          /Users/<MYUSER>: User (your) home directory       |
 |                                                            |
 |------------------------------------------------------------|
```

In the above architecture, the MacBook laptop is running VirtualBox for the VM template creation/management, a Docker engine
with Jenkins running within, and has a local user "\<MYUSER\>" which is your user account with a home directory `/Users/<MYUSER>`
for running the build agent. Jenkins will be reaching out to GitHub to access the sample repository used for the CI builds.

**WARNING**: It is likely that VirtualBox will get installed under your user account and, as such, Virtual Machines and their
respective information will be stored/located under your home directory. You could create a separate user ('jenkins', or similar)
on the MacBook to have the Jenkins master log in as for the node, but this would require re-organizing the VirtualBox storage
locations and is likely a bit more effort than it's worth for this configuration (more specifically, the local service account
would not be able to access the base VM created or manipulate VMs without having access to your local home directory where the
VM files are located unless permissions are adjusted).

### Install Dependent Software

On the MacBook, install the dependent git and virtualbox software required:

```bash
$ brew install git
$ brew cask install virtualbox
```

The instructions for installing Docker for Mac are pretty clear and we won't repeat them here. Instead, follow
[these instructions](https://docs.docker.com/v17.12/docker-for-mac/install/).

### Deploy and Configure Jenkins

**Note**: You may need to modify the "Sharing" settings under the MacBook "System Preferences" to enable remote login of your account
if it's not already possible to SSH to your local MacBook using your local account.

You can now deploy Jenkins in a container using a vendor-provided distribution. Run the following command to download the blueocean
Docker container image and run it as a container:

```bash
$ docker run -p 8080:8080 jenkinsci/blueocean
```

On startup, you should see an admin key scroll by in your terminal - copy this. Launch a browser and navigate to
[http://localhost:8080](http://localhost:8080) and you should be prompted to insert the Jenkins admin key to activate the Jenkins
instance. Paste the key captured in the logs scrolling by and progress to the next step, where you should select to install the
"Standard" plugins. After a few minutes, you should be greeted with the Jenkins home page.

### Configure MacBook as Jenkins Node

You should now be logged into the Jenkins interface. Navigate to "Manage Jenkins" -> "Manage Nodes", and select "New Node". Fill out
the following properties:

* **Node Name**: MacBook Pro
* **Permanent Agent**: Checked

Click "OK", and in the next form, fill out the following fields, using the hostname and Jenkins user password you created previously:

* **Remote root directory**: /Users/<MYUSER>
* **Host**: <MACBOOK_HOSTNAME>
* **Credentials**: Select "Add" -> "Jenkins", and enter your username and password for logging into the MacBook Pro.
Once done/saved, select the credentials created from the drop-down (should be something like "<USERNAME>/******").
* **Host Key Verification Strategy**: Non verifying Verification Strategy

Then click "Save". In a minute or two, you should see the "MacBook Pro" node show as active (no red color/X symbol). Once this shows
active, click the gear icon next to the "master" node, set "# of executors" to "0" to remove any execution on the master node, and
click "Save". You're now set up with your MacBook Pro as your primary build agent (all jobs should route through the MacBook).

### Phase 1 - Sample Jenkinsfile (Simple Jobs)

The first project component we are going to create and use for testing is a sample Jenkinsfile that contains only blank build stages,
with a "Test" stage that captures the hostname of the build node. Create a directory named `infra-ci-pipeline` and a `Jenkinsfile`
within:

```bash
$ mkdir infra-ci-pipeline
$ cd infra-ci-pipeline
$ vim Jenkinsfile
# ensure contains the following:
#   pipeline {
#       agent any
#   
#       stages {
#           stage('Security Hardening') {
#               steps {
#                   echo 'Hardening...'
#               }
#           }
#           stage('Kernel Tuning') {
#               steps {
#                   sh 'hostname -f'
#               }
#           }
#           stage('Middleware Install') {
#               steps {
#                   echo 'Installing Middleware...'
#               }
#           }
#       }
#   }
```

Then, initialize the directory as a git repository and add/commit the file to the repository. Then push the repository
contents to the remote GitHub repository to ready it for the Jenkins job (see GitHub instructions for doing this if you are unsure how):

```bash
$ git init .
$ git add Jenkinsfile
$ git commit -m "Initial Commit"
```

You're now ready to configure a Jenkins job to use this repository.

### Set up Initial Jenkins Job

We will now create the Jenkins job using the repository in GitHub. Launch the Jenkins interface in your browser if it's not
still open, and log in. Select "New Item". Name the project "infra-ci-pipeline" and select "Multibranch Pipeline" as the project option,
then select "OK".

In the next form, enter the following details:

* **Display Name**: infra-ci-pipeline
* **Branch Sources -> Add source**: Select "GitHub", then enter the following details:
  * **Credentials**: - none -
  * **Owner**: \<your GitHub user handle\>
  * **Repository**: infra-ci-pipeline
* **Scan Multibranch Pipeline Triggers**: Select "Periodically if not otherwise run" and for "Interval", select "1 minute"

Leave all other options as defaults and select "Save". You will then see some log output indicating the repository is being cloned for
initial creation. You may end up hitting a quota on the number of requests that can be made to GitHub in a specific timeframe and be put
into a queue/wait state - let the job finish (it will take several minutes waiting) before proceeding.

**Note**: If you want to eliminate the wait/queue time, you could configure the job to have credentials of your GitHub user account so the job
makes authenticated requests rather than unauthenticated (which have an upper limit).

Once the initial sync completes, you can see the first build status by selecting the "Up" option in the left navigation, then clicking on
"infra-ci-pipeline" -> "master" and selecting the first build job. The job should succeed/show all green, indicating that all stages were
completed successfully. If the build failed, select the build number and then select "Console Output" to troubleshoot what happened. It is
possible that the build failed due to the VirtualBox components not yet being configured.

### Base VM Template in VirtualBox

Now that we have a full-functioning CI pipeline, we can start to build actual VM template functionality. We will be downloading a base CentOS
image and using it for our base template. First, download the VDI for the CentOS image from [here](https://sourceforge.net/projects/osboxes/files/v/vb/10-C-nt/7/7-18.10/181064.7z/download)
and unpack the VDI from the 7Zip file. Next, we will create a Virtual Machine using the VDI from the command line on the MacBook (assumes
the VDI is in the current directory where the following commands are being run):

**Note**: The network adapter specified in the host-only network configuration "vboxnet0" below assumes there is a vboxnet0 virtual network device
(typically default for VirtualBox). If this is not available, you may need to create it prior to running the command above.

```bash
$ vboxmanage createvm --name CentOS7 --ostype "Linux_64" --register
$ vboxmanage modifyvm CentOS7 --memory 2048 --cpus 2 --ioapic on --vram 16
$ vboxmanage storagectl CentOS7 --name "SATA Controller" --add sata --controller IntelAhci
$ vboxmanage storageattach CentOS7 --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium ./CentOS\ 7-18.10\ \(64bit\).vdi
$ vboxmanage storagectl CentOS7 --name "IDE Controller" --add ide
$ vboxmanage storageattach CentOS7 --storagectl "IDE Controller" --port 0 --device 1 --type dvddrive --medium emptydrive
$ vboxmanage modifyvm CentOS7-harden-16 --nic1 hostonly  --hostonlyadapter1 vboxnet0
$ vboxmanage showvminfo CentOS7
```

If all goes well, you should see information about the CentOS 7 VM created (and can view it in the VirtualBox interface if you prefer a GUI).

#### Installing Guest Additions/Extensions

You will likely need to launch the VM in GUI mode and log in/install the VirtualBox extensions. The default box for the tutorial used from
OSBoxes has a login user with "osboxes" - the default password for this is "osboxes.org". Log into the VM.

Open a terminal and install the dependent packages:

```bash
$ sudo yum -y install gcc make perl kernel kernel-devel kernel-devel-3.10.0-957.el7.x86_64.rpm
```

Once complete, from the VirtualBox menu (outside the VM) select "Devices" -> "Insert Guest Additions CD Image...", which will mount the DVD to
the optical drive and will present a CD icon on the desktop. Within a minute or two, you will be prompted to run the auto-installer - select
"Run" and then enter the "osboxes.org" password. Wait a few minutes and the job will complete.

At this point, you have the guest additions for VirtualBox installed and enabling you to query the VM for its IP address. In a terminal on your
MacBook, run the following command to obtain the IP address of the running VM:

```bash
$ vboxmanage guestproperty get CentOS7 "/VirtualBox/GuestInfo/Net/0/V4/IP" | awk -F' ' '{print $2}'
```

You should see the IP address of the running VM.

#### Passwordless Sudo for Service Account

Ideally you would want a separate service account using SSH keys to access and modify the VM in the various stages of the pipeline. To simplify
the pipeline and reduce scope, we will simply use the "osboxes" user with a password to log in, and ensure the user has the ability to perform
passwordless sudo to further simplify the pipeline. Log into the VM again using the osboxes user and perform the following:

```bash
$ sudo su -
$ visudo
# ensure this lines exists somewhere in this file:
#   osboxes ALL=(ALL) NOPASSWD: ALL
```

At this point, the user can escalate their privileges using `sudo su -` without a password.

### Update Jenkinsfile for VirtualBox Base Image

Let's now update our Jenkinsfile to include the first stage in our CI process - hardening. We will add an informational stage to our pipeline
as the first step to print out the base image details and then add creation of a VM from the base template for the purposes of hardening.
Update the Jenkinsfile to read as follows:

```python
pipeline {
    agent any

    stages {
        stage('Base Image/Build') {
            steps {
                echo 'Base image is...'
                sh 'vboxmanage showvminfo CentOS7'
                echo "Build ID: ${env.BUILD_ID}"
            }
        }

        stage('Security Hardening') {
            steps {
                echo 'Hardening...'
                echo 'Cloning base VM template to new VM...'
                sh 'vboxmanage 
            }
        }

        stage('Kernel Tuning') {
            steps {
                sh 'hostname -f'
            }
        }

        stage('Middleware Install') {
            steps {
                echo 'Installing Middleware...'
            }
        }
    }
}
```

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [VirtualBox 6.0 on Fedora 29/28, CentOS/RHEL 7.5/6.10](https://www.if-not-true-then-false.com/2010/install-virtualbox-with-yum-on-fedora-centos-red-hat-rhel/)
* [Managing VirtualBox Virtual Machines through Command Line (with VBoxManage)](https://www.systemcodegeeks.com/virtualization/virtualbox/managing-virtualbox-virtual-machiness-command-line-vboxmanage/)
* [OSBoxes - CentOS](https://www.osboxes.org/centos/)
* [Create Virtual Machine using VBoxManage](https://networking.ringofsaturn.com/Unix/Create_Virtual_Machine_VBoxManage.php')
* [Blue Ocean Jenkins CI Docker Image](https://hub.docker.com/r/jenkinsci/blueocean/)
* [How to Install VirtualBox on macOS using Homebrew](https://www.code2bits.com/how-to-install-virtualbox-on-macos-using-homebrew/)
* [Create VirtualBox VM from the Command Line](https://www.perkin.org.uk/posts/create-virtualbox-vm-from-the-command-line.html)

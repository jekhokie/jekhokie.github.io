---
layout: post
title:  "AWS Active Directory Domain Setup"
date:   2017-02-06 23:22:16 -0400
categories: aws active-directory domain windows
---
Details on setting up an Active Directory domain in an Amazon Web Services Virtual Private Cloud.
Includes domain-joining a Windows-based virtual machine to the domain as well.

### Background

Although not that complex and likely very familiar to most system administrators, setting up an Active
Directory domain is one of the first steps a company takes to establish their technology ecosystem. This
tutorial will focus on configuring a Windows Server 2012 R2 instance in an AWS Virtual Private Cloud
environment to serve as an Active Directory Domain Controller (DC) with minimal configuration or
customization. The installation will include instructions on how to additionally provision the instance
as a Domain Name Server (DNS) to ensure ease of domain joining by hosts wishing to join the domain.

Following the configuration of the Domain Controller, we will also focus on domain joining a separate
Windows instance to the domain and inspect the auto-creation of the "Computer" object within the domain
itself.

As a note, the trend for many companies is to move to Azure AD, which is a Microsoft-based cloud
solution for Active Directory built within the Microsoft Azure cloud-based compute environments. This
tutorial focuses on the middle-ground approach to Active Directory, which is to say it is not "on-prem"
(built using hardware within the company walls) but, rather, cloud-based within AWS, where you still
build the AD environment yourself but the hardware on which it runs is entirely the responsibility of
Amazon Web Services.

#### WARNING

This tutorial is intended to be a VERY vanilla approach to setting up a new domain - it is absolutely
*not* intended to serve as instructions for doing so in a production environment. Although the steps
within this tutorial are a good starting point for learning how to start with Active Directory, there
are almost certainly far more complicated configuration steps required to get a fully-functioning
Active Directory environment for your needs. In addition, this setup will specifically exclude the
Dynamic Host Configuration Protocol (DHCP) component that is possibly also required for your internal
network to function as expected if you do not utilize router/switch DHCP capabilities.

Note that this tutorial does result in resources being built within AWS that cost money - please ensure
that you inspect and are OK with the rates for each resource prior to building any of the resources
defined. Wherever possible, the tutorial attempts to minimize cost while ensuring that speed and
processing power are still appropriate for the software being installed and configured.

### Technology Ecosystem

The following specifications are used in this tutorial for building the new Active Directory domain:

**AWS**
- **Region**: us-east (N. Virginia)

**Active Directory Domain Controller**

- **OS**: Windows Server 2012 R2 Base
- **AWS Instance Type**: t2.medium

**Test Domain-Join Host**

- **OS**: Windows Server 2012 R2 Base
- **AWS Instance Type**: t2.small

### Prerequisites

This tutorial assumes that the new domain is created within its own Virtual Private Cloud (VPC)
within AWS. This will eliminate potential conflict of domain management within existing/already-created
domains. As such, there are several steps that need to be completed to create and configure the new VPC
prior to getting started with the tutorial. Most of the configuration can be handled by the AWS wizard
for creating new VPCs, which is what we will use to create our new cloud environment:

1. Log into your AWS account.
2. Navigate to the "VPC" service.
3. Click "Start VPC Wizard".
  - Select the "VPC with a single Public Subnet" option.
4. Specify the following for the options requested:
  - **IPv4 CIDR block**: 10.90.0.0/16
  - **IPv6 CIDR block**: No IPv6 CIDR block
  - **VPC name**: test-vpc
  - **Public subnet's IPv4 CIDR**: 10.90.0.0/24
  - **Availability Zone**: No Preference
  - **Subnet name**: Test Public Subnet
  - **Enable DNS hostnames**: Yes
  - **Hardware tenancy**: Default
  - Click "Create VPC"

Once you click "Create VPC", AWS will start to provision your VPC for you. This includes the
provisioning of now only the VPC itself, but the default subnets, security groups, etc. required for
your VPC to function as expected.

As a note, it is best to provide a Name for the VPC, as the AWS Wizard does not do so. Navigate back
to the VPC service in AWS and select "Your VPCs" in the left navigation pane. Then, for the VPC listed
that has the IPv4 CIDR of "10.90.0.0/16", double click in the blank "Name" column and specify the name
as "Test VPC", then click the check mark. This will update the name of the VPC for future reference.

Finally, in order for you to access resources within your VPC, you will need to create a Security Group
to explicitly allow access. When you create resources such as a Windows OS EC2 instance within the VPC,
the EC2 provisioner will ask and auto-create a Security Group for you which will open access for Remote
Desktop connections to the Windows-based instance. However, for file sharing and other functionality that
does not use the same Remote Desktop port and protocol, you will need to explicitly grant access to the
instance in your environment. If you are managing this device from a single laptop/computer (your
computer) only, then you can grant access to all ports and protocols for your specific device IP address.
However, it is almost always certainly preferred that you grant access to a range of IP addresses in
your own network so that anyone with an IP address in the range of IPs can access the instance. We will
configure a Security Group in the following steps assuming the latter, and assuming that the devices that
need access to the EC2 resources reside on a network defined as 10.0.0.0/8.

1. Log into your AWS account.
2. Navigate to the "VPC" service.
3. Click the "Security Groups" option in the left navigation pane.
4. Click the "Create Security Group" button.
5. Specify the following for the options requested:
  - **Name tag**: test-security-group
  - **Group name**: test-security-group (auto-populated from "Name" tag)
  - **Description**: Allow all from 10.0.0.0/8
  - **VPC**: (ensure the "Test VPC" resource is selected)
  - Click "Yes, Create"

Once the Security Group has been created, you will need to specify the Inbound Rules for it.

1. Select the security group from the list.
2. In the details pane, click the "Inbound Rules" tab and click "Edit".
3. For the new rule, specify the following:
  - **Type**: All Traffic
  - **Protocol**: ALL
  - **Port Range**: ALL
  - **Source**: Specify the CIDR of the network on which the devices you wish to use to manage
this EC2 instance reside - for example, "10.0.0.0/8" if the devices sit in a 10.x network.
  - Click "Save"

You have now completed the security group configuration to access the EC2 instance we will be creating.

### Domain Controller

Now that we have the VPC constructed and the Security Group available to grant access, we can provision
and configure the Active Directory Domain Controller (DC).

#### Provision and Configure EC2 Instance

1. Log into your AWS account.
2. Navigate to the "EC2" service.
3. Click the "Launch Instance" button.
4. Select the 64-bit "Windows Server 2012 R2 Base".
5. Select the "t2.medium" Instance Type and click "Next: Configure Instance Details".
6. Specify the following:
  - **Number of instances**: 1
  - **Purchasing Option**: (leave this un-checked unless you understand "Spot" instances)
  - **Network**: (select the "Test VPC" you created in the Prerequisites)
  - **Subnet**: Test Public subnet
  - **Auto-assign Public IP**: Enable
  - **Domain join directory**: None
  - **IAM role**: None
  - **Shutdown behavior**: Stop
  - **Enable termination protection**: (check this if you wish to ensure the instance is not terminated by accident)
  - **Monitoring**: (check this if you intend to provide longer-term support of the instance)
  - **Tenancy**: Shared - Run a shared hardware instance
  - **Network Interfaces**: (leave as default-specified options)
  - Click "Next: Add Storage"
7. On the "Storage Option" screen, leave the default 30GB General Purpose SSD (GP2) option, and ensure that
the "Delete on Termination" option is checked. Then, click "Next: Add Tags".
8. On the "Add Tags" screen, add any key/value pairs you wish to associate with the instance. Then,
click "Next: Configure Security Group".
9. On the "Configure Security Group" screen, there are defaults to create a new security group and
auto-grant RDP access over TCP port 3389 for any host. Leave the default, and click "Review and
Launch".
10. On the "Review Instance Launch" screen, review the configurations and select "Launch". Once you
click the Launch button, you will be prompted to select or create a key pair for accessing the
instance. Ensure that you either create a new key pair and save the credentials, or select an
existing key pair that you already have the credentials for, as this is the only way you will be
able to access the instance from this point forward.

Now that you have launched the instance, it will take several minutes to be able to access it.
Windows-based instances typically take some time to provision and configure in order to access them
via RDP. While the instance is provisioning, we can take the opportunity to assign the Security Group
we created earlier to specifically grant access to the instance. Select the instance in the EC2
instances list, and perform the following:

1. Click the "Actions" drop-down, and select "Networking" -> "Change Security Groups".
2. Leave a check mark next to the default security group, and place an additional check mark next to
the security group with description "Allow all from 10.0.0.0/8".
3. Click "Assign Security Groups".

Next, you will need to wait for the instance to finish provisioning. Once the instance is ready, you
can configure the instance as a Domain Controller (DC).

#### Configure as Domain Controller and DNS

First, RDP into the EC2 instance created. This can be done using several types of software - if you
are on a Mac-based device, using something such as "Microsoft Remote Desktop" should suffice. The
key pair you selected when launching the instance is what you will use to obtain the "Administrator"
password via the AWS EC2 service. Next, perform the following steps on the EC2 instance to configure
it as a Domain Controller/set up a new domain:

1. Select the "Server Manager" option from the task bar.
2. When the "Server Manager" window opens/loads, click the "Manage" menu option and select "Add Roles
and Features".
3. In the Add Roles and Features Wizard, perform the following:
  - **Before you Begin**: Click "Next"
  - **Installation Type**: Select "Role-based or feature-based installation" and click "Next"
  - **Server Selection**: Accept defaults and click "Next"
  - **Server Roles**: Select "Active Directory Domain Services" and "DNS Server" "and click "Next" - note
that when you select the options, a window will likely pop up prompting you to confirm and/or specify
validation issues - you can simply either select "Add Features" or "Continue" depending on the options
available in each window
  - **Features**: Accept defaults and click "Next"
  - **AD DS**: Accept defaults and click "Next"
  - **DNS Server**: Accept defaults and click "Next"
  - **Confirmation**: Click "Install" - you will next be shown the "Results" screen with the progress of
the installation.
  - **Results**: Wait for the installation to complete - once complete, within the results window itself,
you will see a link/option to "Promote this server to a domain controller" - click it.

Once you've clicked the "Promote this server to a domain controller" option, you will be presented with
another Wizard:

1. Perform the following:
  - **Deployment Configuration**: Select "Add a new Forest", and specify a Root domain name - for our
purposes, we will use "test123.com" - then click "Next"
  - **Domain Controller Options**: Leave the defaults for functional levels and capabilities, and specify
a Directory Service Restore Mode password - for our purposes, we will use "Password123!"
  - **DNS Options**: Leave defaults, and ignore any warnings - click "Next"
  - **Additional Options**: The "NETBIOS domain name" should pre-populate with "TEST123" - if not, specify
"TEST123" and click "Next"
  - **Paths**: Leave defaults, and click "Next"
  - **Review Options**: Review the configuration specified, and click "Next"
  - **Prerequisites Check**: A prerequisites check will be performed, which may take several minutes - once
complete, ignore warnings and click "Install"
  - **Installation**: Installation may take several minutes - wait for the install to complete, and it will
automatically bring you to the "Results" window.
  - **Results**: Review the results and click "Close" - the EC2 instance will then reboot

Once the EC2 instance has rebooted, re-establish the RDP session and investigate the newly-created Active
Directory domain:

1. Click the "Start" icon and then select "Administrative Tools".
2. Double click the "Active Directory Users and Computers" option.

At this point, the Active Directory Users and Computers manager tool will open in a new window. Click the
drop-down icon next to the "test123.com" domain to expand the resources within, which should show folders
for items such as Users, Computers, Domain Controllers, etc. Your new domain is now configured and ready
for use. In order to join hosts manually, we will need either a newly-created user that has the join
privileges, or to use the Administrator account - we will use the latter, so we'll need to change the
Administrator password so we know what it is:

1. Click on the "Users" folder in the Active Directory Users and Computers manager.
2. Right-click on the "Administrator" account in the right-hand pane and select "Reset Password...".
3. Enter a new password for the Administrator account - we will use "Password123!" (note that this is
obviously not desirable due to it being simple and exactly the same as the recovery password, but for
ease of use, this will suffice).
4. Click "OK" to save the newly-specified password.

You now have the Administrator account set up for use in domain joining the client instance.

### Client Instance and Domain Joining

Ideally, when hosts join a particular network, they are automatically joined to the respective domain.
However, full automation of this process is outside the scope of this article and, as such, we will be
manually instantiating and configuring an EC2 instance to join the newly-created domain.

1. Log into your AWS account.
2. Navigate to the "EC2" service.
3. Click the "Launch Instance" button.
4. Select the 64-bit "Windows Server 2012 R2 Base".
5. Select the "t2.small" Instance Type and click "Next: Configure Instance Details".
6. Specify the following:
  - **Number of instances**: 1
  - **Purchasing Option**: (leave this un-checked unless you understand "Spot" instances)
  - **Network**: (select the "Test VPC" you created in the Prerequisites)
  - **Subnet**: Test Public subnet
  - **Auto-assign Public IP**: Enable
  - **Domain join directory**: None
  - **IAM role**: None
  - **Shutdown behavior**: Stop
  - **Enable termination protection**: (check this if you wish to ensure the instance is not terminated by accident)
  - **Monitoring**: (check this if you intend to provide longer-term support of the instance)
  - **Tenancy**: Shared - Run a shared hardware instance
  - **Network Interfaces**: (leave as default-specified options)
  - Click "Next: Add Storage"
7. On the "Storage Option" screen, leave the default 30GB General Purpose SSD (GP2) option, and ensure that
the "Delete on Termination" option is checked. Then, click "Next: Add Tags".
8. On the "Add Tags" screen, add any key/value pairs you wish to associate with the instance. Then,
click "Next: Configure Security Group".
9. On the "Configure Security Group" screen, there are defaults to create a new security group and
auto-grant RDP access over TCP port 3389 for any host. Leave the default, and click "Review and
Launch".
10. On the "Review Instance Launch" screen, review the configurations and select "Launch". Once you
click the Launch button, you will be prompted to select or create a key pair for accessing the
instance. Ensure that you either create a new key pair and save the credentials, or select an
existing key pair that you already have the credentials for, as this is the only way you will be
able to access the instance from this point forward.

As before when we created the Active Directory EC2 instance, it will take some time for the newly-
created EC2 instance to be reachable. Take this time to specify the additional Security Group for
the instance in the EC2 management console:

1. Click the "Actions" drop-down, and select "Networking" -> "Change Security Groups".
2. Leave a check mark next to the default security group, and place an additional check mark next to
the security group with description "Allow all from 10.0.0.0/8".
3. Click "Assign Security Groups".

Once the instance is reachable, we can join the instance to the new "test123.com" domain. To do so,
we first need to configure the instance to utilize the Active Directory instance's DNS services:

1. Establish RDP session with the new client EC2 instance.
2. Open a PowerShell window and execute the `Get-DnsClientServerAddresses` command to obtain the DNS
server currently in use. Capture/copy the IP address next to the "Ethernet" option displayed (in the
case of this test environment, should be something like "10.90.0.2").
3. Select the "Server Manager" option from the task bar.
4. Select the "Local Server" option in the left navigation panel.
5. In the "Properties" sub-panel, click on the link next to the "Ethernet" property (will be listed as
something such as "IPv4 address assigned by DHCP, IPv6 enabled").
6. In the "Network Connections" window that opens, double click the "Ethernet" icon.
7. In the "Ethernet Status" window that opens, click the "Properties" button.
8. In the "Ethernet Properties" window that opens, select the "Internet Protocol Version 4 (TCP/IPv4)"
item in the list, and click "Properties".
9. Under the "General" tab in the window that opens, select the "Use the following DNS server addresses"
radio button and specify the *Private* IP address of the Active Directory EC2 instance in the"Preferred
DNS Server" property, and the original DNS server IP captured in step 2 for the "Alternate DNS server"
property.
10. Click "OK".
11. Close all other windows.

Once the DNS configurations are in place, we can then join the device to the newly-created domain:

1. Establish RDP session with the new client EC2 instance.
2. Select the "Server Manager" option from the task bar.
3. Select the "Local Server" option in the left navigation panel.
4. In the "Properties" sub-panel, click on the "Computer Name" link to open properties for the computer.
5. In the "System Properties" window, click the "Change..." button next to the text that
reads "To rename this computer or change its domain or workgroup, click "Change".
6. In the "Computer Name/Domain Changes" window under the "Member of" section, select the "Domain" radio
button and specify "test123.com", then click "OK".
7. When prompted, enter the Administrator credentials we set up on the Domain Controller - in our case,
the username will be "Administrator" and the password will be "Password123!".

At this point, you should be prompted with a dialog that states something along the lines of "Welcome to
the test123.com domain", followed by a prompt to restart the instance. Click "OK" and reboot your instance.

To confirm everything is working as expected, re-establish an RDP session with the Active Directory
controller instance and launch the "Active Directory Users and Computers" manager (Start Menu ->
Administrative Tools -> Active Directory Users and Computers). When the manager launches, click on the
"Computers" folder - you should then see the EC2 instance that you previously domain-joined, indicating
success.

### Conclusion

As previously stated, this tutorial does not attempt to create a fully functional Active Directory
ecosystem for "production" use. There are *many* facets of this setup that are undesirable in a real-
life environment, but the tutorial does attempt to illustrate some of the most basic components of
setting up an Active Directory environment for a new domain.

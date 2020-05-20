---
layout: post
title:  "AWS Elastic Beanstalk Patterns"
date:   2016-09-29 19:13:51 -0400
categories: ubuntu aws elastic-beanstalk
logo: aws-elasticbeanstalk.jpg
---
Details related to the setup and usage patterns of the [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/)
service as it pertains to development patterns. Explains some useful details related to developing an
application that can be managed and deployed through AWS Elastic Beanstalk, as well as details related
to supporting such an application across a development team (configurations, etc).

### Background

The AWS Elastic Beanstalk service is a quick way to deploy web applications for almost any language/
framework available in a scalable infrastructure. The service provides web servers that are best
suited for the language and framework being developed (i.e. Apache, Nginx, Passenger, IIS) and
it automatically handles capacity provisioning, load balancing, auto-scaling, health monitoring, and
more. Cost of an Elastic Beanstalk application is only associated with the cost of the resources
used within the application (i.e. EC2 instances, Elastic Load Balancer, etc).

Note that these instructions/this tutorial is geared towards an Ubuntu system (steps have been
specifically tested on a Ubuntu 16.04 system).

***Warning***: The steps in this post/tutorial will likely result in incurred costs associated with
using AWS resources.

### Prerequisites/Installation

The EB CLI is installable via the Python pip tool. To install pip:

{% highlight bash %}
$ sudo apt-get install python-pip
{% endhighlight %}

Once pip is installed, use it to install the EB CLI. Note the command does not use `sudo` and
specifies the --user flag - this is intentional and installs the command line tools in a subdirectory
of the current account's home directory (so as to not modify/impact system-level libraries):

{% highlight bash %}
$ pip install --upgrade --user awsebcli
{% endhighlight %}

Now that the `eb` command line tool has been installed, add the executable directory to your
path by using the `export` command (for your current session) and placing the `export` command in
your home `.bash_profile` (preferred) to ensure it is always available for new sessions as well. Note
again that these commands are to be run as the current user (so as to not affect the `root` user
or other system aspects):

{% highlight bash %}
# add the path for your current session
$ export PATH=~/.local/bin:$PATH

$ vim ~/.bash_profile
# add the export command for future sessions
#   export PATH=~/.local/bin:$PATH
{% endhighlight %}

Now that the `eb` command line is available, ensure it functions as expected by running a version
check of the tool:

{% highlight bash %}
$ eb --version
# EB CLI 3.7.8 (Python 2.7.6)
{% endhighlight %}

If the above output is received without errors, you are ready to get starting with using the `eb`
command line tool!

### Interacting with the `eb` CLI

The following steps are useful for getting started and interacting with the `eb` CLI.

#### Initializing the Environment

Once you have the `eb` command line tool installed, you will want to initialize your environment.
To do so, run the `eb init` command and provide several data points to the tool.

Note - if you do not have the AWS CLI tools installed/have never interacted with AWS from the command
line, you will likely receive a message such as the following:

{% highlight bash %}
# You have not yet set up your credentials or your credentials are incorrect
# You must provide your credentials.
{% endhighlight %}

If so, follow the questions according to the credentials associated with your account, and a new
directory will be created in your home directory `.aws` which will contain the credentials that are
required to access your AWS account.

Now, continue with the `eb init` command:

{% highlight bash %}
$ eb init
# Answer several questions to initialize your environment configurations:
#   "Select a default region" - use whichever region suits your purposes
#   "Select an application to use" - usually you will select "[ Create new Application ]"
#                                    if this is your first time
#   "Enter Application Name" - give your application a new name - something with an environment
#                              such as 'my-app-dev' is likely appropriate
#   "Select a Platform" - if one of the out of box platforms applies, choose that - this helps
#                         EB configure some sane defaults for your environment
#   "Do you want to set up SSH for your instances?" - usually specifying 'y' is a good idea
#   "Select a keypair" - if you specified SSH access, choose an AWS keypair to use for SSH access
{% endhighlight %}

Once you run through the above questions, you will likely see no output and be returned to
the command prompt. Don't be afraid - this is likely OK as the `eb init` command has set up
your specified configurations and stored them. You can check your specified options in the
`.elasticbeanstalk` directory under your home directory:

{% highlight bash %}
$ cat ~/.elasticbeanstalk/config.yml
# should output all settings you specified in the 'eb init' command
{% endhighlight %}

Your environment configurations are now complete!

#### Creating the Environment in AWS

Once you have configured your `eb` command-line tool for interaction with AWS, you can create
the application you specified via the following:

{% highlight bash %}
$ eb create --tags "customtag1"="tagvalue1","customtag2"="tagvalue2"
# the create command will take a couple of minutes to run/complete

# note that if you wish to create your Elastic Beanstalk application with a database, you can
# append the following switches to the 'eb create' command above:
#   --database \
#   --database.engine postgres \
#   --database.username <DB_USER> \
#   --database.password <DB_PASS> \
#   --database.instance db.t2.small

# additionally, if you have a saved configuration (from a previous environment) that you wish
# to use, you can append the '--cfg' parameter with the name of the saved configuration to use
# to pre-populate the environment settings as well
{% endhighlight %}

#### Configuring Your Environment

Once the application/environment has been created in AWS, you can navigate to the AWS console
(one method to do this) and add/edit the configuration variables for your application. This is
likely necessary for any application that has environment-specific configuration variables.
When developing an application for the AWS Elastic Beanstalk environment, depending on the
programming language/framework chosen, you can typically access the configuration variables
via the following within your code (this example is for a NodeJS application):

{% highlight bash %}
# <VAR_NAME> is the configuration variable set in the Elastic Beanstalk configurations
#   process.env.<VAR_NAME>
{% endhighlight %}

In addition, if you specified a database for the Elastic Beanstalk application using the
various `--database` switches, you can access the database endpoint/user/etc. in the same
fashion as the `process.env` solution above (again, for NodeJS applications):

{% highlight bash %}
# the following environment variables are available for configuring database (RDS) interaction
#   process.env.RDS_HOSTNAME
#   process.env.RDS_USERNAME
#   process.env.RDS_PASSWORD
#   process.env.RDS_PORT
{% endhighlight %}

#### Deploying Code and Testing

Once your Elastic Beanstalk application has been configured and is ready to receive code for
your application, you can deploy the code from your project directory via the following command:

{% highlight bash %}
$ eb deploy
{% endhighlight %}

The deployment will likely take a few minutes (especially if your project is large in size as all
project files must be copied into the AWS cloud for deployment). Following completion, you can
access the application via a browser by accessing the URL identified in the Elastic Beanstalk
application through the AWS Console, or via the following command if you have GUI capability
available to your terminal:

{% highlight bash %}
$ eb open
{% endhighlight %}

#### Tearing Down (Destroying) an Environment

If the Elastic Beanstalk application/environment you are using is no longer needed, you can tell AWS
to relinquish the resources (tear down/destroy the Elastic Beanstalk application) to ensure that
you are no longer charged for resources associated with the EB application:

{% highlight bash %}
$ eb terminate
{% endhighlight %}

### Elastic Beanstalk Extensions

There are many ways to inject infrastructure and code-related configuration settings, and some of them
vary completely based on the programming language/framework you use. One method is to place configuration
files into a directory named `.ebextensions` within the project folder itself. Files within this
directory are named with the `.config` extension, and Elastic Beanstalk will run them in sequential order
based on naming - therefore, it is useful to name the files using a numbering scheme such as the following:

* `00-cfg-changes.config`
* `01-env-settings.config`
* etc.

Either way, these files are likely good to commit to your source repository. It allows other developers
to deploy the same application in a consistent fashion and provides a reference for the project.

There are many various configuration settings that you can use - a few of the useful ones will be
mentioned in this post.

#### Infrastructure-Related Extensions

Elastic Beanstalk provides many various infrastructure-related settings that can be accessed through
the various infrastructure related to an EB application (RDS, load balancer, EC2 instances, etc). Some
of the configurations are very likely useful to configure and have static(ish) in your project. To
accomplish this, create a `.config` file in your `.ebextensions/` folder that contains the following
(as an example - there are far more options than the following, but the below are consistently used
for applications):

{% highlight bash %}
#  option_settings:
#      aws:autoscaling:launchconfiguration:
#          InstanceType: t2.micro
#          SecurityGroups: sg-aaaaaaaa
#      aws:ec2:vpc:
#          VPCId: vpc-bbbbbbbb
#          Subnets: 'subnet-cccccccc,subnet-dddddddd'
#          ELBSubnets: 'subnet-eeeeeeee,subnet-ffffffff'
#          AssociatePublicIpAddress: true
#      aws:autoscaling:asg:
#          MinSize: 1
#          MaxSize: 2
#      aws:elb:listener:443:
#          ListenerProtocol: HTTPS
#          SSLCertificateId: arn:aws:iam::999999999999:server-certificate/example_cert
#          InstancePort: 80
#          InstanceProtocol: HTTP
{% endhighlight %}

Explanations of the various configurations can be found within the AWS documentation itself. As
a summary, the above:

* Specifies a launch configuration for when the EB application is instantiated
  * EC2 Instance types to use
  * Security groups to apply to each EC2 instance
* Specifies the VPC to use, along with associated subnets, and specifies to assign a public IP address
* Specifies the minimum and maximum number of EC2 instances within the load balanced pool
* Specifies that a HTTPS listener should be instantiated using a specific SSL certificate

#### Custom Configuration Variables

In addition to infrastructure-related settings, you can also specify configuration settings (custom
configuration variables) to seed your application with when the EB application is created in AWS. It
is unlikely that you will wish to store sensitive information for environment-specific settings within
this file as it will be committed to the SCM tool, but it is a good method to ensure that all required
configuration variables are pre-seeded into the application with default values to remind anyone
deploying a new instance of the application that they are the minimum settings required to configure
the application. Here is an example of such a file:

{% highlight bash %}
#   option_settings:
#     aws:elasticbeanstalk:application:environment:
#         CUSTOM_VAR_1: 'placeholder'
#         CUSTOM_VAR_2: 'placeholder'
#         SECRET_PASSWORD: 'placeholder'
{% endhighlight %}

As a note, many configuration variables may likely be real values in the configuration above if they
are not sensitive in nature, reducing the burden of the deployer to have to update the `placeholder`
values.

As stated earlier, these configuration settings are injected into the application for reference. This
allows the developer to "expect" that the variables will simply exist at launch time/run time and
access them (for instance, using the NodeJS method) via `process.env.<VAR_NAME>` within the code
itself.

**Note**: AWS EB environment settings typicaly make it more difficult to develop and run an application
local to your developer machine - to help, use various libraries available across different programming
languages known as 'dotenv'. These libraries allow the developer to create a `.env` file (or the like) in
the project directory with the same configuration settings as the EB environment settings and the code
can access them in the same way it would access the AWS EB environment settings (for instance, in NodeJS,
via using the `process.env` environment variable).

#### Environment Manifest

In addition to the above infrastructure and configuration related methods of configuring an Elastic
Beanstalk application, an environment manifest can be used (`env.yaml`). This method also supports
the organization of groups of Elastic Beanstalk applications (multiple applications under a single
project/repository) with overrides, as well as items such as tags. For details on how to do this, see
[this](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environment-cfg-manifest.html) article.

### Additional Useful Commands

Some additional useful commands are listed below, with their intent:

{% highlight bash %}
# note: you can have several environments at a time (i.e. 'prd', 'dev') and switch between them

# to list available environments:
$ eb list

# to switch to/use a specific environment
$ eb use my-app-dev

# health information about your environment
$ eb health

# to view configuration information about an environment once it has been created in AWS
$ eb config

# retrieve log files for local viewing
$ eb logs --all

# deploy staged changes within a git repository (not yet committed changes) - useful for testing
$ eb deploy --staged
{% endhighlight %}

### Credit

Contributions to some of the above were gleaned from:

* [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/)
* [AWS EB Installation](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html)
* [AWS EB RDS with NodeJS](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_nodejs.rds.html#create_deploy_nodejs.rds.newDB)
* [AWS EB Environment Manifest](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environment-cfg-manifest.html)
* [AWS EB Environment Management](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environment-mgmt-compose.html)

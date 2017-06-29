---
layout: post
title:  "AWS Lambda - Part 1: Using Serverless Framework"
date:   2017-06-23 19:04:21 -0400
categories: nodejs aws lambda serverless nodejs
---
Tutorial to document the steps to create an [AWS Lambda](https://aws.amazon.com/lambda/) function
using the [Serverless Framework](https://github.com/serverless/serverless) for NodeJS using the
basic "hello world" project.

### Background

This is a very basic tutorial on how to use the [Serverless Framework](https://github.com/serverless/serverless)
to create and deploy [AWS Lambda](https://aws.amazon.com/lambda/) functionality. The patterns used
in this tutorial are not to say this is the best practice, but rather, to explore ways in which
you can quickly and easily create NodeJS functionality and deploy to AWS Lambda with minimal effort
and reasonable defaults.

Note that all instructions within are assumed for an Ubuntu-16.04 installation. While the commands
may also work on various other operating systems of the Unix type, your mileage may vary.

In addition, you will need an AWS account to deploy the functionality into - you can, however, perform
all of the steps up to the "AWS Deploy" section to get a feel for the framework and run the functionality
locally prior to setting up the AWS components if you wish.

### NodeJS Setup

To make life easier, we will use [nvm](https://github.com/creationix/nvm) for NodeJS versions. First,
install the capability (replace the version in the path with whatever version you wish to use/are specified
in the installation documentation):

{% highlight bash %}
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.2/install.sh | bash

# to start using nvm immediately, perform the following (or re-login/re-establish your session):
$ export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# verify
$ nvm --version
#  0.33.2
{% endhighlight %}

Now that we have nvm installed, we need to install a NodeJS version:

{% highlight bash %}
$ nvm install 6
# will perform NodeJS 6.x installation

# verify
$ node --version
#  v6.10.3
{% endhighlight %}

### Project Setup

Now that we have our environment configured for NodeJS, we can create our base project structure:

{% highlight bash %}
# install the serverless framework
$ npm install serverless -g

# create a serverless function
$ serverless create --name sls-example \
                    --template aws-nodejs \
                    --path ./sls-example
$ cd sls-example/

# initialize the project for npm
$ npm init
# answer questions appropriately
{% endhighlight %}

At this point you should have a few default files in the directory:

* `.gitignore`: Tells git which files to ignore during commits
* `handler.js`: Main entry point of the Lambda function
* `serverless.yml`: Configuration for creating the service in AWS
* `package.json`: Standard npm dependency and project file

For convenience in project setup, create a new file named `.nvmrc` in the root directory with a
single line that reads `v6.10.3`. This will ensure that any person wishing to use this project can
rely on NVM to manage the NodeJS version appropriate for the example application.

Then, you can instruct NVM to utilize the correct version moving forward, and install any initial
dependencies:

{% highlight bash %}
$ nvm use
$ npm install
{% endhighlight %}

### Testing Default Installation

Without any further changes, you can invoke the functionality locally to mimic what the AWS Lambda
functionality will invoke. While a disclaimer about this is necessary (given that the environment *may*
differ and it is likely best to do pre-production testing against an actual AWS Lambda setup), it
allows for rapid local development and testing.

To test, simply run the following command:

{% highlight bash %}
$ sls invoke local -f hello
{% endhighlight %}

If all went successfully, you should receive an output to the console similar to the following:

{% highlight json %}
{
    "statusCode": 200,
    "body": "{\"message\":\"Go Serverless v1.0! Your function executed successfully!\",\"input\":\"\"}"
}
{% endhighlight %}

You now have a fully functioning Serverless environment that can be run/used locally!

### AWS Deploy

Now that you can run things locally, it's time to deploy your application to AWS. The `serverless.yml` file
manages the configurations for how the application is deployed in conjunction with the AWS environment variables
present in your environment for the AWS account you are deploying to.

This tutorial utilizes static AWS credentials through the AWS IAM service. It is desirable to instead utilize
the Secure Token Service (STS) capability to generate short-lived tokens over persistent/long-living tokens,
which may be discussed in future tutorials, but we will defer to persistent IAM security credentials for now.

First, log into your AWS account and navigate to the IAM service and create a security credential for yourself.
Then, for simplicity, we will use the `aws` command to configure your profile on the local VM.

{% highlight bash %}
# install AWS command line interface
$ sudo apt-get install awscli

# configure the initial security credentials for your account
$ aws configure
# answer each question providing the information from the security credentials
# created through the IAM service for your user
#   AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
#   AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
#   Default region name [None]: [PRESS ENTER]
#   Default output format [None]: [PRESS ENTER]
{% endhighlight %}

In most cases, the `AWS_PROFILE` environment variable must be set in order for libraries to automatically detect
the default profile credentials you wish to use (in the case where there are more than one). For consistency, we
will set this variable explicitly to demonstrate how you can change the profile in the case where you have
different secret credentials for different functionalities:

{% highlight bash %}
$ export AWS_PROFILE=default
{% endhighlight %}

In addition, you will need to ensure that the user you generated credentials for has the respective permissions
to be able to deploy AWS Lambda functionality. Obviously fine-grained permissions are desirable, but again, for
simplicity, grant the user (through the AWS IAM interface) the Policy "AWSLambdaFullAccess" in order to test
this functionality. Later, you will want to restrict the capabilities further to only the fine-grained access the
user requires.

The environment should now be set up for deploying the functionality - let's deploy the Lambda functionality to
your AWS account:

{% highlight bash %}
# WARNING: This command will generate resources that may result in costs to your account
$ sls deploy -v
{% endhighlight %}

After several minutes following multiple verbose output lines, you should see that your Lambda functionality has
been successfully created/deployed and ready for test!

Execute the following to test the function:

{% highlight bash %}
$ sls invoke -f hello
{% endhighlight %}

You should again receive output similar to the following:

{% highlight json %}
{
    "statusCode": 200,
    "body": "{\"message\":\"Go Serverless v1.0! Your function executed successfully!\",\"input\":\"\"}"
}
{% endhighlight %}

If something goes wrong/you do not get the output you expect, you can stream the logs for the function to inspect
what might be going on. Open a new tab/window and invoke the following from the project directory:

{% highlight bash %}
$ sls logs -f hello -t
{% endhighlight %}

Finally, if you wish to remove the resources created by the Serverless framework for your project, simply run the
following command and all AWS resources allocated will be destroyed:

{% highlight bash %}
$ sls remove -v
{% endhighlight %}

### Next Steps

If you're interested in continuing to explore, check out [this next post]({% post_url 2017-06-28-aws-lambda-part-2-securing-data-using-kms %})
that details how to secure data in an AWS Lambda project using the AWS Key Management System.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [nvm](https://github.com/creationix/nvm)
* [NodeJS Serverless](https://github.com/serverless/serverless)
* [AWS Lambda](https://aws.amazon.com/lambda/)
* [AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
* [Building a REST API in Node.js...](https://serverless.com/blog/node-rest-api-with-serverless-lambda-and-dynamodb/)

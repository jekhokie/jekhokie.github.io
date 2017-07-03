---
layout: post
title:  "AWS Lambda - Part 3: Writing Data to S3"
date:   2017-06-29 19:31:31 -0400
categories: nodejs aws lambda serverless s3
---
Tutorial that expands on [this previous post]({% post_url 2017-06-28-aws-lambda-part-2-securing-data-using-kms %})
demonstrating how to take data in to an AWS Lambda function and write the data in a consistent file-naming format
to [AWS Simple Storage Service (S3)](https://aws.amazon.com/s3/), demonstrating somewhat of an "archiving"
functionality.

### Background

This post is an expansion of the [previous AWS Lambda Post]({% post_url 2017-06-28-aws-lambda-part-2-securing-data-using-kms %})
describing how to secure sensitive information in your AWS Lambda project/environment. This post expands
further and demonstrates how an AWS Lambda function, when sent data along with a request, can write the
data to an S3 bucket using a naming convention in line with the date/time the request was received,
demonstrating somewhat of an archiving functionality. Although there are ingest services in AWS that can
do this more natively using AWS services, this is a good proof of concept to demonstrate how to link an
AWS Lambda function to S3.

As previously, all instructions within are assumed for an Ubuntu-16.04 installation. While the commands
may also work on various other operating systems of the Unix type, your mileage may vary.

Like the previous tutorial, you will also need an AWS account to deploy the functionality into and test
using AWS S3 (and the previous KMS functionality). As a note, S3 is *very* inexpensive for file storage.

### Prerequisites

As before, this posts assume you follow the [previous post]({% post_url 2017-06-28-aws-lambda-part-2-securing-data-using-kms %})
steps for setting up the environment and having a basic code base. Please follow the previous post in
its entirety before proceeding with this post.

### Exploring S3

Fist and foremost, we'll detail some commands that you can use to explore your S3 environment. Assuming
you already have the AWS CLI configured (per the Prerequisites section above), you can run S3 commands
to explore how to interact with the AWS S3 service:

{% highlight bash %}
# list buckets in S3 (no output, no buckets created yet)
$ aws s3 ls

# make a bucket in S3 - note that you may run into a naming conflict
# given that bucket names MUST be unique within a system - if so, simply
# choose a different unique name that will work
$ aws s3 mb s3://test-bucket
# should output:
#   make_bucket: test-bucket

# create a local file and copy the contents to the bucket
$ echo "This is a test" > test-file.txt
$ aws s3 cp test-file.txt s3://test-bucket
# should output:
#   upload: ./test-file.txt to s3://test-bucket/test-file.txt

# list the buckets
$ aws s3 ls
# should output:
#   2017-06-29 19:35:34 test-bucket

# list the bucket contents
$ aws s3 ls s3://test-bucket
# should output:
#   2017-06-29 19:35:45 15 test-file.txt

# remove the test bucket from S3
$ aws s3 rb s3://test-bucket --force
# should output:
#   delete: s3://test-bucket/test-file.txt
#   remove_bucket: test-bucket
{% endhighlight %}

### Coding in Serverless

Now that we have some experience interacting with AWS S3 via the command line, let's code our
application to also interact with the service. We are going to update the code base so that
whenever the Lambda function receives a request, if the request contains data, we record the
data to a file in an S3 bucket, with a filename corresponding to the date/time the request was
made.

We are going to assume that the bucket we wish to use already exists, so let's go ahead and
create one from the command line like we did before:

{% highlight bash %}
$ aws s3 mb s3://hello-bucket
# should output:
#   make_bucket: hello-bucket

# double-check that the bucket exists
$ aws s3 ls
# output should contain:
#   2017-06-29 19:44:18 hello-bucket
{% endhighlight %}

Next, we need to set up permissions for the Lambda function to be able to write to the S3 bucket,
specify a custom parameter containing the bucket name to reduce code duplication, and add a handler
that will ultimately write the data to S3. Note that the ARN for the S3 bucket is constructed using
standard notation and the custom bucket name (which is one of the reasons why bucket names must be
unique):

{% highlight bash %}
$ vim serverless.yml
# ensure the following lines exist in the respective areas:
#   ...
#   custom:
#     bucketName: hello-bucket
#   ...
#   provider:
#     iamRoleStatements:
#       ...
#       - Effect: Allow
#         Action:
#           - s3:PutObject
#         Resource: "arn:aws:s3:::${self:custom.bucketName}/*"
#   ...
#
# also, update the 'functions' for the hello function to include an
# environment variable decarling the bucket name:
#
#   ...
#   functions:
#     hello:
#       ...
#       environment:
#         S3BUCKET: ${self:custom.bucketName}
{% endhighlight %}

We'll install a library that will make it easier to format dates and times:

{% highlight bash %}
$ npm install --save moment
{% endhighlight %}

Require the library at the top of your `handler.js` file:

{% highlight JavaScript %}
const moment = require('moment');
{% endhighlight %}

We'll now add some code that will check for incoming data and, if present, write the data to
the S3 bucket using a file name that includes the date and time the data was received. Add the
following function declaration somewhere in the `handler.js` file:

{% highlight JavaScript %}
function writeDataToS3(data) {
  return new Promise((r, x) => {
      if (typeof(data) === 'undefined' || data == '') {
        x("No data provided");
      } else {
        const filename = moment().format("YYYY-MM-DD-hhmmss") + '.txt',
              s3 = new Aws.S3();

        s3.putObject({ Bucket: process.env.S3BUCKET, Key: filename, Body: data }, function(err, data) {
          if (err) x(err);
          else r(filename);
        });
      }
  });
}
{% endhighlight %}

Then, update the `hello` function to reflect the following Promise chain and functionality:

{% highlight JavaScript %}
module.exports.hello = (event, context, callback) => {
  getDataKey()
    .then((dataKey) => { return getDecryptedFile(dataKey); })
    .then((data) => {
      return writeDataToS3(event);
    })
    .then((filename) => {
      const response = {
        statusCode: 200,
        body: JSON.stringify({
          message: 'Data stored to S3 location: ' + "s3://" + process.env.S3BUCKET + "/" + filename,
          input: event,
        }),
      };

      callback(null, response);
    })
    .catch(callback);
};
{% endhighlight %}

The Lambda function now inspects the incoming data element (`event`) to ensure it has data (if not,
it throws an exception given that the intent of this exercise is to store data in S3). Once it verifies
data exists, it writes the data to a file named with the date/time that the event occurred and returns
the data location in S3 to the requestor. Let's run the Lambda function locally again and inspect the
output:

{% highlight bash %}
$ sls invoke local -f hello --data "This is some test data"
# should output something similar to the following:
#   {
#       "statusCode": 200,
#       "body": "{\"message\":\"Data stored to S3 location: s3://hello-bucket/2017-06-29-204551.txt\",\"input\":\"This is some test data\"}"
#   }
{% endhighlight %}

It appears the data has been successfully stored - let's check the S3 bucket via the command line
tools to ensure we can see the file created:

{% highlight bash %}
$ aws s3 ls s3://hello-bucket
# should output something similar to the following:
#   2017-06-29 20:45:53          22 2017-06-29-204551.txt
{% endhighlight %}

Now, let's run a final test to make sure the function fails as expected when no data is provided:

{% highlight bash %}
$ sls invoke local -f hello
# should output something similar to the following:
#   {
#       "errorMessage": "No data provided"
#   }
{% endhighlight %}

It looks like things are working as expected - let's deploy the function to AWS and test
it/verify that it works and stores the files as expected:

{% highlight bash %}
$ sls deploy -v
# wait for the function to deploy

$ sls invoke -f hello --data "This is some other test data"
# should output a message indicating the S3 storage location
{% endhighlight %}

Everything is working great! Let's perform some clean-up to ensure we aren't billed for the usage
moving forward:

{% highlight bash %}
$ sls remove -v
# wait for the resources to be removed

$ aws s3 rb s3://hello-bucket --force
# wait for the bucket to be removed
{% endhighlight %}

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [AWS S3](https://aws.amazon.com/s3/)
* [Using S3 Commands](http://docs.aws.amazon.com/cli/latest/userguide/using-s3-commands.html)
* [Serverless S3 Example](https://github.com/serverless/examples/tree/master/aws-node-fetch-file-and-store-in-s3)

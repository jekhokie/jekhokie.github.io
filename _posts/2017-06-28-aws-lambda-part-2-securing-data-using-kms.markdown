---
layout: post
title:  "AWS Lambda - Part 2: Securing Data Using KMS"
date:   2017-06-28 17:12:43 -0400
categories: nodejs aws lambda serverless kms
logo: aws-kms.jpg
---
Expands on the [first post]({% post_url 2017-06-23-aws-lambda-part-1-using-serverless-framework %}) demonstrating
basic AWS Lambda functionality using NodeJS Serverless by adding security via the
[AWS Key Management System](https://aws.amazon.com/kms/) to protect sensitive information for deployed
Lambda functions.

### Background

This post is an expansion of the [previous AWS Lambda Post]({% post_url 2017-06-23-aws-lambda-part-1-using-serverless-framework %})
describing the basic AWS Lambda setup using the NodeJS Serverless framework. This post expands on the
previous functionality by providing a way to secure sensitive information and patterns for ensuring
that the sensitive information can live alongside the code base in a secure fashion.

As previously, all instructions within are assumed for an Ubuntu-16.04 installation. While the commands
may also work on various other operating systems of the Unix type, your mileage may vary.

In addition, you will need an AWS account to deploy the functionality into and test using AWS KMS. Unlike
the previous tutorial, you cannot run the project locally without at least having access to an AWS
account due to the requirement of generating keys through the AWS Key Management Service.

### Prerequisites

This posts assume you follow the [previous post]({% post_url 2017-06-23-aws-lambda-part-1-using-serverless-framework %})
steps for setting up the environment and having a basic code base. Please follow the previous post in
its entirety before proceeding with this post.

#### AWS KMS Key Generation

To encrypt and decrypt data, we will want to generate a KMS Data key. The process
(outlined [here](http://docs.aws.amazon.com/cli/latest/reference/kms/generate-data-key.html)) involves first
creating a Customer Master Key.

**WARNING**: All commands from this point forth, when specifying a region, MUST match the region specified
in the `serverless.yml` file, otherwise the Lambda function in AWS will not be able to access the keys
generated. To force the region, add the `region: us-east-2` attribute in the `serverless.yml` file under
the `provider` section.

{% highlight bash %}
# generates the customer master key in us-east-2, for example
$ aws kms create-key --description "Test Key" --region us-east-2
{% endhighlight %}

Take note of the output - specifically, the "ARN" parameter. Copy this value as we will need to reference it
in several places. Next, we can generate the data key which will be encrypted by the customer master key by
specifying the customer master key ARN `[CMK_ARN]`:

{% highlight bash %}
# generates the data key in us-east-2, for example
$ aws kms generate-data-key --key-id [CMK_ARN] --key-spec AES_256 --region us-east-2
# capture the CiphertextBlob (base64-encrypted form) of the Data Key, and take
# note of the "plaintext" value from the output
{% endhighlight %}

Finally, we'll ensure that the base64-encrypted data key can be decrypted with the customer master key and
have it match the "plaintext" value in the output from the above `generate-data-key` command by decoding and
copying the `CiphertextBlob` value from the `generate-data-key` command above to a file and decrypting
the contents:

{% highlight bash %}
# decode the base64-encoded CiphertextBlob data and store in a file
$ echo [CiphertextBlob] | base64 --decode > dk.enc

# using the customer master key, decrypt the data key into its base64-encoded value
$ aws kms decrypt --ciphertext-blob fileb://dk.enc \
                  --output text \
                  --query Plaintext \
                  --region us-east-2
# ensure that the output (base64-format) matches the "plaintext" value from the
# generate-data-key command that was run in the previous step - if so, you're
# good to go!
{% endhighlight %}

Once you have the key generated, you'll need to update the `serverless.yml` file to account for permission for
the Lambda function to access the key and decrypt the secret configuration file we will be creating shortly:

{% highlight bash %}
$ vim serverless.yml
# ensure the following is present under 'provider', replacing '[KEY_ARN]' with
# the actual ARN of the KMS Customer Master Key (CMK) created earlier:
#   provider:
#     iamRoleStatements:
#       - Effect: "Allow"
#         Action:
#           - 'kms:Decrypt'
#         Resource: [KEY_ARN]
{% endhighlight %}

##### Bonus: Encrypt/Decrypt with Customer Master Key

Although definitely not recommended, it is interesting to explore how the Customer Master Key (CMK) can be used
to encrypt and decrypt data as well. Again, this is not recommended as best-practice, but here are some steps to
test this functionality:

{% highlight bash %}
# store a string "This is a test" in a file in plain text
$ echo "This is a test" > output.plain

# encrypt the file and store the encrypted text in a file "output.encrypted"
$ aws kms encrypt --key-id [KEY_ARN] \
                  --plaintext fileb://output.plain \
                  --output text \
                  --query CiphertextBlob \
                  --region us-east-2 | base64 --decode > output.encrypted

# decrypt the file using the same key, ensuring "This is a test" is what you
# receive in return
$ aws kms decrypt --ciphertext-blob fileb://output.encrypted \
                  --output text \
                  --query Plaintext \
                  --region us-east-2 | base64 --decode
# should output the following:
#   This is a test
{% endhighlight %}

#### Creation of Encrypted Secrets

Now that we have a way to encrypt data, we can create and encrypt the configuration file for the project. The
process by which Amazon recommends this is:

1. Decrypt the Data Key using the Customer Master Key.
2. Use the decrypted data key to encrypt the data.
3. Throw away the decrypted data key.

First, we'll add the plain-text configuration file that stores the encrypted/base64-encoded data key to our
project. We also need to ensure that the plain-text file is ignored for both git and the serverless framework
upon deployment of the function to AWS Lambda and create the initial sensitive configuration file locally:

{% highlight bash %}
$ vim config.json
# ensure contains the following, where [KMS_DATA_KEY] is the base-64 encoded encrypted data key (original
# "CiphertextBlob" parameter returned when generating the key):
#   {
#     "kmsDataKey": "[KMS_DATA_KEY]"
#   }

# ignore our sensitive plain-text file just in case
$ echo "hello.json" >> .gitignore
$ vim serverless.yml
# ensure there is an 'exclude' section below the 'package' declaration like so:
#   package:
#     exclude:
#       - hello.json

# initial plain-text configuration file (sensitive)
$ vim hello.json
# ensure contains the following:
#   {
#     "test_data": "This is sensitive data"
#   }
{% endhighlight %}

Now that we have our sensitive file, let's encrypt it with the Data Key. Recall earlier when we generated the
Data Key we captured the "Plaintext" value from it. First, save the plaintext to a file (note that it is
CRITICAL that you delete this later). Then, use the plain-text key to encrypt the secret/sensitive file, and
check that the decryption works as expected. This is a very undesirable way to handle this as it would be better
to decrypt the key on the fly and use it to encrypt rather than saving it, but for simplicity in commands, we
will save it to a file:

{% highlight bash %}
# save the plain-text version of the key to a file
$ echo [DATA_KEY_PLAINTEXT] > key

# encrypt the sensitive file and base64-encode - note that CTR is only available as of
# openssl version 1.0.1+ also note that openssl salts by default - NodeJS does not,
# so avoid any kind of salting
$ openssl enc -aes-256-ctr -nosalt -a -in hello.json -out hello.json.enc -pass file:./key

# test that the file can also be decrypted with the same key
$ openssl enc -d -a -aes-256-ctr -nosalt -in hello.json.enc -pass file:./key
# since no '-out' parameter was specified, the following should be printed to the screen:
#   {
#     "test_data": "This is sensitive data"
#   }
{% endhighlight %}

#### Code to Decrypt Secrets

We're now ready to write some code that will decrypt and parse the sensitive configuration file using the key
system we've set up. First, we'll need some supportive libraries:

{% highlight bash %}
$ npm install --save aws-sdk crypto
{% endhighlight %}

In the `handler.js` within the `hello` function, add code right above the response
variable declaration that will perform the decryption of the configuration file, grab the `test_data` value
from the configuration, and return it as part of the response.

**WARNING**: There are a lot of issues with this code including, but not limited to, embedded configurations,
code duplication, terrible organization, etc. - however, the code is presented this way to ensure a
consolidated view into the functionality.

{% highlight JavaScript %}
'use strict';

const Aws = require('aws-sdk'),
      config = require('./config.json'),
      crypto = require('crypto'),
      fs = require('fs');

function getDecryptedFile(dataKey) {
  return new Promise((r, x) => {
    fs.readFile('./hello.json.enc', 'utf8', (err, data) => {
      if (err) x(err);
      else r(data);
    })
  })
  .then((data) => {
    const cipher = crypto.createDecipher('aes-256-ctr', dataKey),
          decryptedValue = (cipher.update(data, 'base64', 'utf8') + cipher.final('utf8'));

    return decryptedValue;
  });
}

function getDataKey() {
  return new Promise((r, x) => {
    const kms = new Aws.KMS({ apiVersion: '2014-11-01', region: 'us-east-2' });

    kms.decrypt({ CiphertextBlob: Buffer.from(config.kmsDataKey, 'base64') }, (err, data) => {
      if (err) x(err);
      else r(data.Plaintext.toString('base64'));
    });
  });
}

module.exports.hello = (event, context, callback) => {
  getDataKey()
    .then((dataKey) => { return getDecryptedFile(dataKey); })
    .then((data) => {
      const response = {
        statusCode: 200,
        body: JSON.stringify({
          message: 'Value of secret: "' + JSON.parse(data).test_data + '"',
          input: event,
        }),
      };

      callback(null, response);
    })
    .catch(callback);
};
{% endhighlight %}

To test the functionality, first, run it locally to ensure no syntax/other errors are present:

{% highlight bash %}
$ sls invoke local -f hello
# should output the following:
#   {
#     "statusCode": 200,
#     "body": "{\"message\":\"Value of secret: \\\"This is some sensitive data\\\"\",\"input\":\"\"}"
#   }
{% endhighlight %}

If the response comes back successfully, you're now ready to ship things off to AWS for full production-based
testing. First, delete the sensitive file for consistency in testing, and then deploy and test the code:

{% highlight bash %}
# deploy the code
$ sls deploy -v

# test the deployed code
$ sls invoke -f hello
# if all is successful, the following message will output from the function in AWS Lambda:
#   {
#     "statusCode": 200,
#     "body": "{\"message\":\"Value of secret: \\\"This is sensitive data\\\"\",\"input\":\"\"}"
#   }

# terminate the stack
$ sls remove -v
{% endhighlight %}

At this point, you now have a fully functioning AWS Lambda function that can store and handle sensitive
information using keys through AWS KMS for security! It's likely unreasonable to expect storing the key,
encrypting using the key file, removing the key, etc. in each pattern of development when new sensitive
variables are needed, so take a few minutes to write some helper functions to perform this automatically
for you to make your development patterns easier.

### Next Steps

Take the next step and check out [this next post]({% post_url 2017-06-29-aws-lambda-part-3-writing-to-s3 %})
that details how to store data that is sent to an AWS Lambda function to AWS S3.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [AWS KMS JavaScript SDK](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/KMS.html)
* [AWS KMS - Creating Keys](http://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html)
* [Encrypt and decrypt content with NodeJS](http://lollyrock.com/articles/nodejs-encryption/)

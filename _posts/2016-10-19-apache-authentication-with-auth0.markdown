---
layout: post
title:  "Apache Authentication with Auth0"
date:   2016-10-19 21:22:31 -0400
categories: ubuntu apache auth0 authentication openid-connect
---
[Auth0](https://auth0.com/) is a fantastic cloud-hosted authentication mechanism providing many different
authentication connection possibilities (social networks, email, ADFS, etc). The SDKs and libraries for
many various languages help the adoption of utilizing this service - however, when it is not possible or
cost prohibitive to modify an existing application to uitlize it, it is helpful to know how to utilize
something like an [Apache Web](https://httpd.apache.org/) proxy to handle the Auth0 interaction on behalf
of the application.

### Background

Upon exploring how to integrate [Auth0](https://auth0.com/) into almost everything, I quickly discovered that
not all software supports native multi-provider auth solutions or SAML/OpenID Connect implementations. Also,
adding such cability (contributing to the source code) would take quite a bit of time that I simply don't have
at the moment.

As somewhat of a workaround (and somewhat of a general good pattern for learning how to front applications
with an auth proxy), an [Apache Web](https://httpd.apache.org/) server can be used as a proxy in front of the
application to handle the Auth0 interaction. Apache has been around for quite some time and there have been
many, MANY plugins developed for interaction with various endpoints. One such plugin that can be used is the
OpenID-Connect plugin, which can interact with Auth0 via the OpenID Connect protocol.

This tutorial attempts to explain how one might set up an Apache proxy in front of a web application in order
to support authentication through Auth0. The tutorial itself will focus on installing and configuring Apache
to serve a default web page (index.html) with authentication using Auth0, but serves as a base/starting
configuration for extending the server as a true proxy instance. Shortly, I should have another post available
which details the proxy functionality.

***Note***: This tutorial assumes an Ubuntu 16.04 operating system is in use.

### Apache - Initial Installation and Configuration

Prior to configuring Apache for OpenID Connect functionality, the software must be installed and configured,
along with (in this case, self-signed) SSL certificates to protect the users' credentials.

#### Software Installation

The following steps will install the latest version of Apache2 on an Ubuntu-based system:

{% highlight bash %}
$ sudo apt-get update
$ sudo apt-get install apache2
{% endhighlight %}

At this point, the Apache server has been installed with a default configuration and started. You can visit
http://\<IP_OR_HOSTNAME\>/ to validate that your Apache instance is running.

#### Self-Signed SSL Certificates

When users visit the desired endpoint that is fronted by the Apache proxy, they will need to enter credentials
to authenticate. This means it is certainly a good idea to encrypt the communication in order to ensure that
the users' credentials are protected (not sent in the clear). For simplicity, we will use self-signed
certificates, but it is a much better idea to use a valid/verified certificate for non-development environments
(such as those issued by certificate authorities such as [GlobalSign](https://www.globalsign.com/en/).

First, generate a self-signed certificate using the following commands (this will use the RSA encryption algorithm
with a validity period of 365 days/1 year for the certificate):

{% highlight bash %}
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout testproxy.key -out testproxy.crt
# the command will result in a series of questions to answer, after which a .crt and .key file will be generated

# Apache expects a pem file for the certificate, so let's convert the crt, and remove the crt once complete
$ openssl x509 -in testproxy.crt -out testproxy.pem -outform PEM
$ rm testproxy.crt
{% endhighlight %}

Once you have generated a certificate and key pair, create a directory in the Apache directory structure to
store the files in. This directory should also have appropriate permissions applied to prevent unauthorized
access to the files:

{% highlight bash %}
# move and lock down cert
$ sudo mv testproxy.pem /etc/ssl/certs/testproxy.pem
$ sudo chmod 644 /etc/ssl/certs/testproxy.pem

# move and lock down key
$ sudo mv testproxy.key /etc/ssl/private/testproxy.key
$ sudo chmod 640 /etc/ssl/private/testproxy.key
{% endhighlight %}

Now that the SSL files are in place, update the configuration file corresponding to the SSL listener. By default,
at the time of this post, the configuration resides in the sites-available/default-ssl.conf file:

{% highlight bash %}
$ sudo vim /etc/apache2/sites-available/default-ssl.conf
# ensure the following directives are in this file, within the <VirtualHost> tags:
#   SSLEngine on
#   SSLCertificateFile /etc/ssl/certs/testproxy.pem
#   SSLCertificateKeyFile /etc/ssl/private/testproxy.key
{% endhighlight %}

Once the SSL configuration file is updated, you can enable the site based on the configuration specified. Note that
you must also enable the SSL module for this to be successful:

{% highlight bash %}
$ sudo a2enmod ssl
$ sudo a2ensite default-ssl
$ sudo service apache2 reload
{% endhighlight %}

Once the above has been completed and the Apache server has been reloaded, you can visit the hosted site via
the URL https://\<IP_OR_HOSTNAME\>/. Note that your browser will likely warn you that the certificate presented
by the server is not verified and/or does not match the hostname of the instance (something along the lines of
'Your connection is not private') - this is expected since we are utilizing self-signed certificates.

For extra security, it is best to disable the non-HTTPS endpoint that is installed/configured by default in
the Apache2 installation, and restart the Apache server to ensure no connections can occur over non-encrypted
ports:

{% highlight bash %}
# disable port 80
$ sudo vim /etc/apache2/ports.conf
# remove the 'Listen 80' directive in the file

# disable the default site listening on port 80
$ sudo a2dissite 000-default
$ sudo service apache2 reload
{% endhighlight %}

### Auth0 Module

Now that you have a fully-functioning Apache instance running over SSL using self-signed certificates, we can
start to integrate and configure the Auth0 OpenID Connect integration.

#### Auth0 Application

First, log into your Auth0 account and create a new application. Note that the application should have at least
one connector configured to provide the authentication (backend). In addition, you must provide an allowed
callback URL corresponding to the location of your Apache instance - something like https://\<IP_OR_HOSTNAME\>/index.html
should suffice.

***NOTE***: A very important step that you must account for is updating the JsonWebToken Signature Algorithm for the
application you just created. In the Auth0 interface under "Advanced Settings" for the client application, select the
"OAuth" tab, and ensure that "JsonWebToken Signature Algorithm" specified is RS256. Without this, you will likely end
up seeing an error message in your Apache logs when attempting to sign in corresponding to the following:

{% highlight bash %}
...
oidc_handle_authorization_response: could not parse or verify the id_token contents, return HTTP_UNAUTHORIZED,
...
{% endhighlight %}

Once you have created the application, obtain the clientId and clientSecret string values (corresponding to the
soon-to-be-shown Apache configuration directives of 'OIDCClientID' and 'OIDCCLientSecret', respectively). Keep these
two values in a safe place.

#### OpenID Connect Authentication Module

Apache needs the Authentication module in order to understand how to communicate with OpenID Connect. Obtain the
package corresponding to your operating system - at the time of this post, the code below demonstrates the version
utilized for the Ubuntu operating system on which the Apache proxy is installed:

{% highlight bash %}
$ wget http://ftp.us.debian.org/debian/pool/main/liba/libapache2-mod-auth-openidc/libapache2-mod-auth-openidc_1.6.0-1_amd64.deb
$ sudo apt-get install libjansson4
$ sudo dpkg -i libapache2-mod-auth-openidc_1.6.0-1_amd64.deb
{% endhighlight %}

Once the module is installed, you need to enable it - do not worry about restarting/reloading Apache after enabling
the module since we will need to do so after updating the configuration files anyways:

{% highlight bash %}
$ sudo a2enmod auth_openidc
{% endhighlight %}

#### Apache OpenID Connect Configuration

Now that the OpenID Connect module has been installed and enabled for Apache, we need to update Apache to tell
the server to use the module and secure parts of the site. Edit the SSL configuration file to configure and
secure a part of the site using Auth0 - note that most of these configuration values will "just work":

{% highlight bash %}
$ sudo vim /etc/apache2/sites-available/default-ssl.conf
# edit the section within the <VirtualHost> tag to ensure it includes the following:
#   OIDCProviderIssuer https://<YOUR_ACCT>.auth0.com
#   OIDCProviderAuthorizationEndpoint https://<YOUR_ACCT>.auth0.com/authorize
#   OIDCProviderTokenEndpoint https://<YOUR_ACCT>.auth0.com/oauth/token
#   OIDCProviderTokenEndpointAuth client_secret_post
#   OIDCProviderUserInfoEndpoint https://<YOUR_ACCT>.auth0.com/userinfo

#   OIDCClientID <CLIENT_ID>
#   OIDCClientSecret <CLIENT_SECRET>
#   OIDCProviderJwksUri https://<YOUR_ACCT>.auth0.com/.well-known/jwks.json

#   OIDCScope "openid name email"
#   OIDCRedirectURI https://<YOUR_APACHE_SERVER>/index.html
#   OIDCCryptoPassphrase "somethingsupersecret"
#   OIDCCookiePath /

#   <Location />
#       AuthType openid-connect
#       Require valid-user
#       LogLevel debug
#   </Location>
{% endhighlight %}

***WARNING***: The `OIDCRedirectURI` parameter defines the callback URL that Auth0 will return data to
once the user authenticates. This URL must be included as one of the allowed callback URLs in the
application within Auth0 - if not, after successfully authenticating, you will end up seeing a "Callback
URL mismatch." error in your browser.

To test the proxy authentication, restart the Apache server:

{% highlight bash %}
$ sudo service apache2 restart
{% endhighlight %}

Once the server is restarted, visit the URL https://\<IP_OR_HOSTNAME\>/. You should be prompted to authenticate
through Auth0 using whichever connector is specified for the application (i.e. ADFS, Facebook, etc). If all goes
well, you will be passed through to the index.html page and see the default/out of box Apache index page.

At this point, you have a fully-functional Apache proxy providing Auth0 authentication!

### Credit

Contributions to some of the above were gleaned from:

* [Auth0 for Legacy Apps](https://auth0.com/blog/sso-for-legacy-apps-with-auth0-openid-connect-and-apache/)
* [Auth0 Apache](https://auth0.com/docs/quickstart/webapp/apache)
* [Self-Signed SSL](https://www.sslshopper.com/article-how-to-create-and-install-an-apache-self-signed-certificate.html)
* [mod_auth_openidc](https://github.com/pingidentity/mod_auth_openidc)

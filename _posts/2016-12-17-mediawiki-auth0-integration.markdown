---
layout: post
title:  "MediaWiki Auth0 Integration"
date:   2016-12-17 19:15:01 -0400
categories: mediawiki auth0 authentication
logo: mediawiki.jpg
---
Tutorial on how to install and configure [MediaWiki](https://www.mediawiki.org/wiki/MediaWiki) to
enable [Auth0](https://auth0.com/) as the Authentication provider.

### Background

[MediaWiki](https://www.mediawiki.org/wiki/MediaWiki) is an open-source wiki platform hosted by many
organizations and individuals. It is written in PHP, utilizes a database back-end for content storage,
and provides numerous capabilities via extensions that can be developed and/or downloaded and installed
from other contributors. 

This article will focus on installing a base MediaWiki instance. The version used is a bit dated
based on the fact that the extension for OpenID authentication for the newest version of the wiki is
incompatible with the auth handlers within the technology. The intent is to provide a proof of concept
and demonstration of integrating Auth0 as the authentication and identity broker for the wiki itself.

### Underlying Compute Technology/Ecosystem

This tutorial makes the following assumptions about your compute infrastructure. Although the
instructions can be extended to accomodate cloud-based and/or other infrastructure environments quite
easily, all details are explained with respect to local development for the purposes of simplicity:

- **Hypervisor Technology**: VirtualBox
- **Provisioner**: Vagrant
- **Number of VMs**: 1
- **Operating System**: Ubuntu 16.04
- **Arch**: 64-bit
- **CPUs**: 1
- **Mem**: 4GB
- **Disk**: 50GB

This tutorial is built with 1 virtual machine to serve as the compute, web, and database resource. This
is obviously *NOT* recommended for production, but for simplicity, suffices for this tutorial.

The following software versions are used as part of this tutorial - again, versions may be changed, but
there is no guarantee that using different versions will result in a functional technology stack. The
versions listed are such that they represent a fully-functional and working technology ecosystem. As
mentioned previously, these versions are older than the latest provided, but are the latest identified
working with the OpenID Connect extension available:

- **MediaWiki**: 1.23.15
- **Apache2**: 2.4.18
- **MySQL**: 5.7.16-0
- **PHP**: 5.6.29-1

### Prerequisites/Dependencies

Prior to installing any dependencies, ensure the package management references are up to date:

{% highlight bash %}
$ sudo apt-get update
{% endhighlight %}

This legacy version of MediaWiki requires a PHP version earlier than 7 - since PHP 7 is typically the
version natively available on Ubuntu 16-based systems using the apt package manager, we will need to
add a package repository for an earlier version using a slightly different repository:

{% highlight bash %}
$ sudo add-apt-repository ppa:ondrej/php
$ sudo apt-get update
{% endhighlight %}

Once the required repository for the correct PHP version is in place, the dependencies can then be
installed via the ```apt``` package manager:

{% highlight bash %}
$ sudo apt-get install apache2 \
                       php5.6 \
                       php5.6-mysql \
                       mysql-server \
                       libapache2-mod-php5.6 \
                       php5.6-xml \
                       php5.6-mbstring
# you will likely be prompted for a MySQL root password - provide one, and remember it
{% endhighlight %}

### Installation

Once the installation prerequisites are in place, you can download and install the MediaWiki product.
First, download and extract the MediaWiki package for the version you wish to install to the
```/var/lib/``` directory:

{% highlight bash %}
$ wget https://releases.wikimedia.org/mediawiki/1.23/mediawiki-1.23.15.tar.gz
$ tar -xzf mediawiki-1.23.15.tar.gz
$ sudo mv mediawiki-1.23.15/ /var/lib/mediawiki
{% endhighlight %}

Once you have placed the MediaWiki files into the ```/var/lib/mediawiki``` directory, it's time to
configure Apache to serve the Wiki content. First, add a symlink to the root directory for the
Apache installation - note that this will specifically add a "mediawiki" namespace to the path of
the resulting URL to access the wiki on this host:

{% highlight bash %}
$ sudo ln -s /var/lib/mediawiki /var/www/html/mediawiki
{% endhighlight %}

One last step is required prior to proceeding as the order in which dependencies were installed
prior to the Apache installation/start may differ - restart the Apache server to ensure the correct
libraries are loaded in the running process:

{% highlight bash %}
$ sudo service apache2 restart
{% endhighlight %}

### Configuration

Once the product has been installed, it's time to utilize the configuration wizard to set the wiki
up with some default settings. Access the MediaWiki interface via the following URL, where
```<WEB_HOST>``` is either the IP address or FQDN of the node that you have the Apache server/wiki
product installed on:

```http://<WEB_HOST>/mediawiki```

If successful, you should see an introduction page with the "MediaWiki" title and the version of
wiki installed, along with a "set up the wiki" link to get started. If you received a connection
error or timeout, ensure that your Apache server is in fact running on the host machine and bound
to an interface that is accessible (*not* the loopback/127.0.0.1 adapter).

Next, click the "set up the wiki" link, and follow these steps, substituting any values for your own
environment/preferences:

- Language:
  - Your language: English (Default)
  - Wiki language: English (Defualt)
  - Click "Continue"
- Welcome to MediawWiki!
  - Read, then click "Continue"
- Connect to database
  - Database type: MySQL (Default)
  - Database host: localhost (Default)
  - Identify this wiki
    - Database name: my_wiki (Default)
    - Database table prefix: (Leave Blank/Default)
  - User account for installation
    - Database username: root (NOT a good idea, but suffices for this effort - in prod, ensure a
      separate user is defined)
    - Database password: root (change this based on your specific password during installation of
      mysql-server)
  - Click "Continue"
- Database settings:
  - Database account for web access
    - Use the same account for installation (check this checkbox)
  - Storage engine: InnoDB (Default)
  - Database character set: Binary (Default)
  - Click "Continue"
- Name
  - Name of wiki: Auth0 Integration Test
  - Project namespace: Same as the wiki name (Default)
  - Administrator account
    - Your username: test_user
    - Password: pass1234
    - Password again: pass1234
    - Email address: your_email@your_domain.com
  - Ensure the "I'm bored already, just install the wiki." radio-button is selected.
  - Click "Continue"
- Install
  - Click "Continue" to start the installation.

Once you've completed the above steps, you will be presented with a page denoting the progress and
success of each step in the provisioning process. Click "Continue" and you will automatically have
your browser download a file named ```LocalSettings.php```.

Take this ```LocalSettings.php``` file and place it in the directory on the host machine under
```/var/lib/mediawiki/```. Once done, open a new browser tab and navigate back to the wiki base URL:

```http://<WEB_HOST>/mediawiki```

If all goes well, you should see the landing page of your newly-configured wiki! You can review the
versions of software installed by searching for "special:version" within your new wiki.

### Auth0 Integration

Now that we have a functioning MediaWiki instance, we can update the Apache instance to provide
[Auth0](https://auth0.com/) integration for authentication, including the ability for multi-factor
auth. Note that we have installed the wiki using a non-HTTPs endpoint - this is **HIGHLY**
frowned-upon, especially as it pertains to performing any kind of auth endpoint that deals with
credentials and session keys. We are going to leave it up to the reader to determine how best to
SSL-enable their site and proceed with this gap in the tutorial.

**Note:** If you happen to run into any issues configuring Auth0 as the auth provider for MediaWiki,
you can reference [this post]({% post_url 2016-10-19-apache-authentication-with-auth0 %}) for help
as the integration is directly related to that post given that the MediaWiki installation and setup
in this tutorial utilizes an Apache web server.

#### Auth0 Application Creation

To integrate Auth0, we first need to set up an application within the Auth0 account. This assumes
you already have an account - typically you can sign up for free trials. The important points to
configure once an application has been set up are as follows:

- **Name**: my-mediawiki-auth0-test
- **Client Type**: Native
- **Allowed Callback URLs**: http://<WEB_HOST>/mediawiki/index.php
- **Advanced -> OAuth -> JsonWebToken Signature Algorithm**: RS256

Once you have set up the application, there are many other data fields required to copy and paste
into the upcoming Apache configuration - keep this Auth0 application window open to allow for quick
copy/paste capability into your Apache configuration.

#### Apache Configuration

We need to install and enable the OpenID Connect authentication module for Apache, which will
handle the interaction with Auth0. Do not worry about restarting the Apache service at this time as
it will need to be restarted following some other configuration changes.

{% highlight bash %}
$ sudo apt-get install libjansson4 \
                       libcurl3
$ wget http://ftp.us.debian.org/debian/pool/main/liba/libapache2-mod-auth-openidc/libapache2-mod-auth-openidc_1.6.0-1_amd64.deb
$ sudo dpkg -i libapache2-mod-auth-openidc_1.6.0-1_amd64.deb
$ sudo a2enmod auth_openidc
{% endhighlight %}

Next we move over to the Apache configuration file. Edit the file to provide the following
configuration directives within the ```<VirtualHost *:80>``` resource definition. An explanation
of the variables are as follows:

- **\<WEB_HOST\>**: The IP or FQDN of the host where the MediaWiki software is installed.
- **\<YOUR_ACCT\>**: Your Auth0 account name.
- **\<CLIENT_ID\>**: Taken from the "Client ID" attribute of your application in Auth0.
- **\<CLIENT_SECRET\>**: Taken from the "Client Secret" attribute of your application in Auth0.

{% highlight bash %}
$ sudo vim /etc/apache2/sites-available/000-default.conf
# add the following lines to the beginning of the <VirtualHost *:80> directive:
#   OIDCProviderIssuer https://<YOUR_ACCT>.auth0.com
#   OIDCProviderAuthorizationEndpoint https://<YOUR_ACCT>.auth0.com/authorize
#   OIDCProviderTokenEndpoint https://<YOUR_ACCT>.auth0.com/oauth/token
#   OIDCProviderTokenEndpointAuth client_secret_post
#   OIDCProviderUserInfoEndpoint https://<YOUR_ACCT>.auth0.com/userinfo
#
#   OIDCClientID <CLIENT_ID>
#   OIDCClientSecret <CLIENT_SECRET>
#   OIDCProviderJwksUri https://<YOUR_ACCT>.auth0.com/.well-known/jwks.json
#
#   OIDCScope "openid name email"
#   OIDCRemoteUserClaim "email"
#   OIDCRedirectURI http://<WEB_HOST>/mediawiki/index.php
#   OIDCCryptoPassphrase "somethingsupersecret"
#   OIDCCookiePath /
#
#   <Location />
#       AuthType openid-connect
#       Require valid-user
#       LogLevel debug
#   </Location>
{% endhighlight %}

Note that in this case, the ```LogLevel debug``` directive is optional but may help with debugging
auth issues if you experience any.

Now that the Apache configuration has been updated, we can restart the Apache service:

{% highlight bash %}
$ sudo service apache2 restart
{% endhighlight %}

You can test that Apache directs you to the Auth0 lock widget whenever you attempt to access the
wiki site via visiting the MediaWiki URL again:

```http://<WEB_HOST>/mediawiki/index.php```

If you were presented with a lock widget for Auth0, congratulations, you are one step closer!

#### MediaWiki Configuration

Now that Apache is handling the front-end auth for the MediaWiki software, we need to configure
MediaWiki to accept and understand how to grant access to valid incoming users. This is handled via
the "Auth_Remoteuser" extension that can be found [here](https://www.mediawiki.org/wiki/Extension:Auth_remoteuser).

First, download and un-pack the extension into the MediaWiki ```extensions``` directory - note
that the ```wget``` URL below may change - the point is, you should download the extension and
specify version "1.23" of MediaWiki via the following URL: 
[Extension Download](https://www.mediawiki.org/wiki/Special:ExtensionDistributor/Auth_remoteuser)

{% highlight bash %}
$ wget https://extdist.wmflabs.org/dist/extensions/Auth_remoteuser-REL1_23-90d3808.tar.gz
$ sudo tar -xzf Auth_remoteuser-REL1_23-90d3808.tar.gz -C /var/lib/mediawiki/extensions/
{% endhighlight %}

Once the extension is in place, it's time to configure MediaWiki to start using it. Edit the
```LocalSettings.php``` file and add the following to the very end of the file:

{% highlight bash %}
$ sudo vim /var/lib/mediawiki/LocalSettings.php
# add the following to the very end of the file:
#   require_once "$IP/extensions/Auth_remoteuser/Auth_remoteuser.php";
#   $wgAuth = new Auth_remoteuser();
#   // Don't let anonymous people do things...
#   $wgGroupPermissions['*']['createaccount']   = false;
#   $wgGroupPermissions['*']['read']            = false;
#   $wgGroupPermissions['*']['edit']            = false;
{% endhighlight %}

Once all is completed, re-visit your wiki page. You should be redirected to the Auth0 lock widget.
Authenticate with Auth0 using whatever backend-connector is configured. If successful, you will be
redirected back to your wiki and automatically logged in with your email address as your login ID.

### Conclusion

This tutorial is a primer on how to integrate MediaWiki with Auth0 as an authentication solution.
There are quite a few gaps in production-hardening this solution (logout does not work as expected,
client login does not obtain all data fields, solution is not SSL-enabled), but suffices for a
'getting started' guide on how to integrate MediaWiki with Auth0. There are also other security
aspects to the configuration of the wiki that should be configured (such as disallowing unauthenticated
access/permissions to do certain things) in addition to firewall-based security enforcements put
in place, but again, this is a primer/tutorial on how to integrate Auth0 with MediaWiki using the
Apache web server the wiki runs on.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Running MediaWiki on Debian or Ubuntu](https://www.mediawiki.org/wiki/Manual:Running_MediaWiki_on_Debian_or_Ubuntu)
* [How to setup/install PHP 5.6 on Ubuntu 14.04 LTS](https://www.dev-metal.com/install-setup-php-5-6-ubuntu-14-04-lts/)
* [Auth_Remoteuser MediaWiki Extension](https://www.mediawiki.org/wiki/Extension:Auth_remoteuser)

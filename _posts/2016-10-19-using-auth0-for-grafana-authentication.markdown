---
layout: post
title:  "Using Auth0 for Grafana Authentication"
date:   2016-10-19 22:11:31 -0400
categories: ubuntu grafana apache auth0 authentication openid-connect
---
Expanding on a [previous post]({% post_url 2016-10-19-apache-authentication-with-auth0 %}) related
to installing and configuring Apache2 for authentication through Auth0, this post expands the scope
to include integrating one such Auth0 proxy with the [Grafana](http://grafana.org/) application.
This allows users to configure authentication for Grafana through Auth0 using the
[AuthProxy](https://blog.raintank.io/authproxy-howto-use-external-authentication-handlers-with-grafana/)
functionality of the Grafana software and the OpenID Connect module in Apache.

### Prerequisites

This post assumes that you have already successfully installed and configured an Apache2 instance
hooked up to Auth0 per the [previous post]({% post_url 2016-10-19-apache-authentication-with-auth0 %}).

### Installing Grafana

Assuming you are going to install the Grafana software/service on the same instance that the Apache2
server is running on (simplest to start with), the following steps will suffice for installation of a
Grafana 3.1.1 instance on an Ubuntu operating system:

{% highlight bash %}
$ sudo wget https://grafanarel.s3.amazonaws.com/builds/grafana_3.1.1-1470047149_amd64.deb
$ sudo apt-get install -y adduser libfontconfig
$ sudo dpkg -i grafana_3.1.1-1470047149_amd64.deb
{% endhighlight %}

It is never recommended to use a Sqlite3 database for production software - however, for the purposes
of this demo, we will utilize the Sqlite3 database for simplicity purposes. To do so, we first need
to install the sqlite3 client (will be useful later in this tutorial to perform some validation
queries):

{% highlight bash %}
$ sudo apt-get install sqlite3
{% endhighlight %}

Once installed, configure the Grafana server to use a Sqlite3 database for storing its persistent data:

{% highlight bash %}
$ sudo vim /etc/grafana/grafana.ini
# search through the file for the [database] section - once there, ensure the following lines are un-commented:
#   ...
#   type = sqlite3
#   ...
#   path = grafana.db
#   ...

# start the Grafana process, which will listen on port 3000 by default
$ sudo /etc/init.d/grafana-server start
{% endhighlight %}

If you have changed nothing else within the configuration file, at the time of this post, Grafana will
instantiate a Sqlite3 database as `/var/lib/grafana/grafana.db`. You can verify that this database
both exists and can be interacted with by using the `sqlite3` command-line command:

{% highlight bash %}
$ sqlite3 /var/lib/grafana/grafana.db

# attempt to view the users in the database:
sqlite> select * from user;
# should output the admin user - something like:
#   1|0|admin|admin@localhost||...

# exit sqlite
sqlite> .quit
{% endhighlight %}

Now that the database has been verified, you can visit the Grafana application in your browser via
the URL http://\<IP_OR_HOSTNAME\>:3000/

### Configure Apache2 as Reverse Proxy

Configuring Apache2 as a reverse proxy in this case allows incoming requests to be accepted by the
Apache instance, which will handle authentication via Auth0 and forward valid/authenticated requests
to the back-end Grafana instance. First, we will enable some Apache modules to help us handle proxying
and rewrite/assign operations:

{% highlight bash %}
$ sudo a2enmod proxy
$ sudo a2enmod proxy_http
$ sudo a2enmod rewrite
$ sudo a2enmod headers
{% endhighlight %}

Once you have enabled the modules, update the default-ssl.conf file (assuming you have followed
[this previous post] ({% post_url 2016-10-19-apache-authentication-with-auth0 %}) and still have the
default-ssl.conf configurations in place). First, we will configure Apache as a standard/non-auth
reverse proxy sitting in front of Grafana. This will ensure your proxy is first configured and
functioning prior to re-integrating with Auth0:

{% highlight bash %}
$ sudo vim /etc/apache2/sites-available/default-ssl.conf
# update the <VirtualHost> definition - remove the entire <Location /example/> definition and add the following:
#   # previously-defined Auth0 configurations here
#   ...
#
#   RequestHeader unset Authorization
#
#   ProxyRequests Off
#   ProxyPass / http://localhost:3000/
#   ProxyPassReverse / http://localhost:3000/
{% endhighlight %}

The above configuration will ensure that any incoming requests to the root of the web server endpoint
will be forwarded to 'localhost' port 3000 (the default port that Grafana should be currently
listening on). Restart the Apache instance for the above configurations to take effect:

{% highlight bash %}
$ sudo service apache2 restart
{% endhighlight %}

Vising the URL https://\<IP_OR_HOSTNAME\>/ in a browser should now redirect you to the path '/grafana'
and you should be presented with a Grafana log in page. If this is the case, your reverse proxy is now
configured and we can move on to the Auth0 integration.

### Integrate Apache2 Reverse Proxy with Auth0

Now that we have a fully-functioning reverse proxy Apache instance, we are going to update the
configuration to add Auth0 integration for authentication. Edit the default-ssl.conf file and
adjust/add the Auth0 directives to enforce authentication requirements through our reverse proxy:

{% highlight bash %}
$ sudo vim /etc/apache2/sites-available/default-ssl.conf
# update the existing Auth0 lines for the following parameters:
#   ...
#   OIDCScope email
#   ...
#   OIDCRedirectURI https://10.11.13.15/grafana
#   ...
#   OIDCCookiePath /
#
# add the following below the Auth0 configuration parameters
#   ...
#   <Proxy *>
#       # note these next 3 lines are essentially the same as what was in the previous <Location /example/> resource
#       AuthType openid-connect
#       Require valid-user
#       LogLevel debug
#
#       RewriteEngine On
#       RewriteRule .* - [E=PROXY_USER:%{LA-U:REMOTE_USER},NS]
#       RequestHeader set X-WEBAUTH-USER "%{PROXY_USER}e"
#   </Proxy>
{% endhighlight %}

As explained in [this fantastic post](https://blog.raintank.io/authproxy-howto-use-external-authentication-handlers-with-grafana/)
and re-summarized here, Apache will accept incoming connections, perform authentication using the
openid-connect module (assuming, again, that you still have all previously-defined Auth0 configurations
within the configuration file), and if authenticated, pass the user along to the Grafana back-end.
There is some magic rewrite logic to retrieve the resulting user information from the Auth0 authenticated
response and set the `X-WEBAUTH-USER` header for Grafana to ID the user and store them in the database.
We will be configuring Grafana to accept this shortly.

Additionally, you will need to log into your Auth0 account and update the client application to
ensure that the "Allowed Callback URLs" is updated to be https://\<IP_OR_HOSTNAME\>/grafana.

Once you have updated the configurations above, restart the Apache instance:

{% highlight bash %}
$ sudo service apache2 restart
{% endhighlight %}

Now, visiting the same URL (https://\<IP_OR_HOSTNAME\>/) should present you with an Auth0 login
prompt. Successfully logging in via Auth0 connectors should then bring you back to the Grafana
login page.

### Grafana Accept Auth0 as Authentication Mechanism

Now that we have updated Apache to authenticate users through Auth0 and forward their requests
through to the Grafana login page, we need to update Grafana to ensure that it accepts the
Auth0 authentication parameters passed to it as a valid login for the incoming users - it would
be terrible if we required users to authenticate through Auth0 and then AGAIN force them to
log into Grafana as a separate user.

In order to successfully configure Grafana, there are a few configuration directives that are
required - these can be updated in the grafana.ini file:

{% highlight bash %}
$ sudo vim /etc/grafana/grafana.ini
# ensure the following directives are set:
#
# under the [users] section:
#   allow_sign_up = false
#   auto_assign_org = true
#   auto_assign_org_role = Editor
#
# under the [auth.proxy] section:
#   enabled = true
#   header_name = X-WEBAUTH-USER
#   header_property = email
#   auto_sign_up = true
{% endhighlight %}

The above will disable any signup via users and auto-create them once they have authenticated
through the Auth0 proxy. The auth.proxy module in Grafana will take the `X-WEBAUTH-USER` header
(recall we used mod_rewrite and mod_headers to set this in the Apache configurations) and
create a user in the database using the user's email address.

Restart the Grafana instance for the configurations to take effect:

{% highlight bash %}
$ sudo /etc/init.d/grafana-server restart

# restart Apache to ensure that you are re-prompted for a new session auth
$ sudo service apache2 restart
{% endhighlight %}

Once Grafana has restarted, you can re-visit the URL https://\<IP_OR_HOSTNAME\>/ in your browser.
At this point, you should receive an Auth0 prompt - successfully authenticating results in your
being logged directly into the Grafana application with an 'Editor' role.

If you wish to validate that an account has been created for you, you can log back into the
Grafana sqlite database and view the user account that was created for you:

{% highlight bash %}
$ sqlite3 /var/lib/grafana/grafana.db

# view the users
sqlite> select * from user;
# you should now see your new user account that you successfully logged in with using Auth0

# exit sqlite
sqlite> .quit
{% endhighlight %}

### Summary

If you've made it this far you have successfully configured Grafana to accept authentication
via an Apache reverse proxy endpoint connected through Auth0 - congratulations! This tutorial
was very specific to an Apache/Auth0/Grafana architecture - however, the Apache reverse proxy
authentication via Auth0 can be utilize as a building block for almost any back-end service
that can accept header parameters and auto-provision users.

A couple things to note:

1. If the user logs out of the Grafana application, their Auth0 token and session will likely
still be valid. In this case, they will end up seeing the Grafana specific log in page, which
is undesirable. There are probably some ways to help this (i.e. by invalidating the user's
session and/or sending them to a logout URL of some type), but that exercise is left up to the
reader of this post to figure out.
2. The proxying capability of the Apache instance based on the above configurations is such that
proxying is performed over non-SSL connection endpoints to Grafana. When dealing with user
information, it is sometimes best to encrypt the entire connection path. However, since the Apache
instance is the front proxy through which the encrypted connection terminates, so long as the
Grafana instance is behind a firewall of some kind (i.e. the network communication between the
Apache instance and the Grafana instance is not accessible), it is likely acceptable to keep this
setup for some implementations. This tutorial was not intended to promote thorough security
implementations for this setup.

### Credit

Contributions to some of the above were gleaned from:

* [External Auth with Grafana](https://blog.raintank.io/authproxy-howto-use-external-authentication-handlers-with-grafana/)

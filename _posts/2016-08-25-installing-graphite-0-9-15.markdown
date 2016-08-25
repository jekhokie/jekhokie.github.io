---
layout: post
title:  "Installing Graphite - Version 0.9.15"
date:   2016-08-25 09:03:14 -0400
categories: ubuntu graphite carbon whisper
---
(**Version 0.9.15**) Tutorial related to the installation and configuration of
[Graphite](http://graphiteapp.org/) on an Ubuntu 16.04 virtual machine. Note that these instructions
follow the "Install using pip" process from the documentation in an attempt to avoid dependency issues
among Python packages (although as seen in the instructions, some versions are specifically required
in order for the setup to function on the Ubuntu system).

### Dependencies

Install the required dependencies:

{% highlight bash %}
$ sudo apt-get update
$ sudo apt-get install -y python-pip \
                          python-dev \
                          apache2 \
                          libcairo2-dev \
                          libapache2-mod-wsgi \
                          memcached \
                          libffi-dev \
                          python-cffi \
                          libfontconfig1-dev \
                          libfreetype6-dev
$ sudo -E pip install "cffi>=1.7.0"
$ sudo -E pip install "django==1.6"
$ sudo -E pip install "django-tagging<0.4"
$ sudo -E pip install pytz enum34
$ sudo -E pip install fontconfig
$ sudo -E pip install python-memcached
$ sudo -E pip install cairocffi
{% endhighlight %}

### Process

#### Installation/Configuration

Install the respective components using pip and the version(s) desired. Note that these instructions
function for the 0.9.15 release and are not guaranteed to function with later releases/versions:

{% highlight bash %}
# whisper - places files in /usr/local/bin, /usr/local/lib
$ sudo -E pip install whisper==0.9.15

# carbon - places files in /opt/graphite
$ sudo -E pip install carbon==0.9.15

# graphite-web - places files in /opt/graphite
$ sudo -E pip install graphite-web==0.9.15

# set up environment settings
$ export PATH=/opt/graphite/bin:$PATH
{% endhighlight %}

Create the Graphite group and user:

{% highlight bash %}
sudo -E groupadd _graphite
sudo -E useradd -c "User for graphite" -g _graphite -s /dev/null _graphite
{% endhighlight %}

Create the configuration (default) files:

{% highlight bash %}
# carbon config
$ cd /opt/graphite/conf
$ sudo cp carbon.conf.example carbon.conf
# update the following configuration parameter in the carbon.conf file
# ensures it does not run as superuser, and increases the max updates/sec
#   USER = _graphite
#   MAX_UPDATES_PER_SECOND = 800

# whisper config
$ cd /opt/graphite/conf
$ sudo cp storage-schemas.conf.example storage-schemas.conf
# update the storage-schemas.conf file to ensure the following are included
#   ...
#   [default_10s1d_60s30d_15m455d]
#   pattern = .*
#   retentions = 10s:1d,60s:30d,15m:455d
#   xFilesFactor = 0.0
#   ...

# graphite web config
$ cd /opt/graphite/conf
$ sudo cp graphite.wsgi.example graphite.wsgi
$ cd /opt/graphite/webapp/graphite
$ sudo cp local_settings.py.example local_settings.py
# update the local_settings.py file to uncomment and update the following, respective to your environment
# these settings will ensure that the date/time is set correctly, memcache is known and a secret is defined
#   SECRET_KEY = 'super_secret_key'
#   TIME_ZONE = 'America/New_York'
#   MEMCACHE_HOSTS = ['127.0.0.1:11211']
{% endhighlight %}

Perform the database migration for graphite-web:

{% highlight bash %}
$ sudo -E PYTHONPATH=/opt/graphite/webapp django-admin.py syncdb --settings=graphite.settings --noinput
{% endhighlight %}

Configure permissions for the various users:

{% highlight bash %}
$ sudo -E chown www-data:www-data /opt/graphite/storage/graphite.db
$ sudo -E mkdir -p /opt/graphite/storage/log/carbon-cache
$ sudo -E mkdir -p /opt/graphite/storage/log/carbon-relay
$ sudo -E mkdir -p /opt/graphite/storage/log/carbon-aggregator
$ sudo -E chown -R _graphite:_graphite /opt/graphite/storage/log
$ sudo -E chmod 775 /opt/graphite/storage
$ sudo -E chown -R _graphite /opt/graphite/storage/whisper
$ sudo -E chown www-data:_graphite /opt/graphite/storage
$ sudo -E chown www-data:_graphite /opt/graphite/conf/graphite.wsgi
$ sudo -E chown -R www-data /opt/graphite/storage/log/webapp
{% endhighlight %}

Configure the Apache instance for the Graphite Web application:

{% highlight bash %}
$ sudo cp /opt/graphite/examples/example-graphite-vhost.conf /etc/apache2/sites-available/graphite-webapp.conf
# edit the file to ensure the following are adjusted - most other defaults are sufficient
# ...
#   WSGISocketPrefix /var/run/apache2/wsgi
# ...
#   <Location "/content/">
#     SetHandler None
#     Require all granted
#   </Location>
#
#   ...
#
#   <Directory /opt/graphite/conf/>
#     Require all granted
#   </Directory>
# ...
{% endhighlight %}

Disable the default site, enable the Graphite Web application, and start Apache to serve the application:

{% highlight bash %}
# disable the default site
$ sudo -E a2dissite 000-default

# enable the graphite-web application
$ sudo -E a2ensite graphite-webapp

# restart apache to start with the new configurations
$ sudo -E service apache2 restart
{% endhighlight %}

Start the Carbon Cache daemon and ensure it is running:

{% highlight bash %}
$ sudo -E /opt/graphite/bin/carbon-cache.py start
# to check the status of the carbon-cache daemon:
$ sudo -E /opt/graphite/bin/carbon-cache.py status
#   carbon-cache (instance a) is running with pid 15812
{% endhighlight %}

#### Validation

Ensure the carbon process is running/listening on correct ports:

{% highlight bash %}
$ cat /opt/graphite/storage/carbon-cache-a.pid
# 15182
$ pgrep -fa carbon
# 15812 /usr/bin/python /opt/graphite/bin/carbon-cache.py start
$ netstat -vant | grep LISTEN | grep '[27]00'
# tcp        0      0 0.0.0.0:2003            0.0.0.0:*               LISTEN
# tcp        0      0 0.0.0.0:2004            0.0.0.0:*               LISTEN
# tcp        0      0 0.0.0.0:7002            0.0.0.0:*               LISTEN
{% endhighlight %}

Ensure that data is being collected for the carbon host:

{% highlight bash %}
$ /usr/local/bin/whisper-fetch.py /opt/graphite/storage/whisper/carbon/agents/devbox-1/cpuUsage.swp
# should output a list of metrics like so:
#   1471861740       0.012192
#   1471861800       0.011875
#   1471861860       0.013435
{% endhighlight %}

Send some new data and validate the data for the carbon cache:

{% highlight bash %}
# send a metric "test.metric.foo" with value 2 at current time to local port 2003
$ echo "test.something.foo 2 `date +%s`" | nc localhost 2003
# check to ensure that the metric shows up
$ /usr/local/bin/whisper-fetch.py /opt/graphite/storage/whisper/test/something/foo.wsp
# should output a bunch of metrics for time series, including value sent above
{% endhighlight %}

### Credit

Contributions to some of the above were gleaned from:

* [GraphiteApp](http://graphite.readthedocs.io/en/0.9.15/)

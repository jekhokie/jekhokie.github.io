---
layout: post
title:  "Installing Graphite - Bleeding Edge (0.10)"
date:   2016-08-22 04:36:04 -0400
categories: ubuntu graphite carbon whisper
---
**Version 0.10** Tutorial related to the installation and configuration of [Graphite](http://graphiteapp.org/) on
an Ubuntu 16.04 virtual machine. Note that these instructions follow the "Install from Source" process
from the documentation in order to facilitate the highest level of configurability, and assume the
"bleeding-edge" version at the time of this post (version 0.10, which is still currently in development).

### Warning

Note that the resulting Graphite setup from these instructions is **NOT** for production use. These
instructions are for setting up an initial test/dev setup of Graphite to understand how the application
functions.

### Dependencies

Install the required dependencies:

{% highlight bash %}
$ sudo apt-get install -y libcairo2-dev \
                          libffi-dev \
                          pkg-config \
                          python-dev \
                          python-pip \
                          python-virtualenv \
                          fontconfig \
                          apache2 \
                          libapache2-mod-wsgi \
                          memcached \
                          librrd-dev \
                          libldap2-dev \
                          libsasl2-dev \
                          git-core \
                          gcc
{% endhighlight %}

### Process

#### Installation/Configuration

Create the Python virtual environment and set up the path settings:

{% highlight bash %}
$ sudo virtualenv /opt/graphite
$ export PATH=/opt/graphite/bin:$PATH
{% endhighlight %}

Clone and build/install the required components:

{% highlight bash %}
# whisper storage
$ cd /usr/local/src
$ sudo git clone https://github.com/graphite-project/whisper.git
$ cd whisper
$ sudo -E python setup.py install

# carbon cache
$ cd /usr/local/src
$ sudo git clone https://github.com/graphite-project/carbon.git
$ cd carbon
$ sudo -E pip install -r requirements.txt
$ sudo -E python setup.py install

# graphite-web - likely see warnings here, can safely ignore
$ cd /usr/local/src
$ sudo git clone https://github.com/graphite-project/graphite-web.git
$ cd graphite-web
$ sudo -E pip install -r requirements.txt
$ sudo -E python setup.py install
{% endhighlight %}

Create the configuration (default) files:

{% highlight bash %}
$ cd /opt/graphite/conf

# carbon
$ sudo cp carbon.conf.example carbon.conf
# update the following configuration parameter in the carbon.conf file to ensure not running as superuser
#   USER = carbonusr

# whisper
$ sudo cp storage-schemas.conf.example storage-schemas.conf
# update the storage-schemas.conf file to update the default section with the following, which allows 1 second resolution
#   ...
#   retentions = 1:30d     # 1 second resolution for 30 days
#   ...

# graphite webapp
$ sudo cp graphite.wsgi.example graphite.wsgi
$ cd /opt/graphite/webapp/graphite
$ sudo cp local_settings.py.example local_settings.py
# update the local_settings.py file to uncomment and update the following, respective to your environment. These
# settings will ensure that the date/time is set correctly and that 1 second resolution is possible
#   SECRET_KEY = 'super_secret_key'
#   TIME_ZONE = 'America/New_York'
#   MEMCACHE_HOSTS = ['127.0.0.1:11211']
#   MEMCACHE_DURATION = 1
{% endhighlight %}

Create the carbon group and user for the carbon service:

{% highlight bash %}
$ sudo groupadd carbon
$ sudo useradd -c "User for the carbon service" -g carbon -s /dev/null carbonusr
{% endhighlight %}

Create the database (replace 'migrate' with 'syncdb' for older versions of Django):

{% highlight bash %}
$ sudo -E PYTHONPATH=/opt/graphite/webapp django-admin.py migrate --settings=graphite.settings --run-syncdb
{% endhighlight %}

Adjust permissions for databases, files, binaries and directories for the Carbon and Apache users:

{% highlight bash %}
$ sudo chown www-data:www-data /opt/graphite/storage/graphite.db
$ sudo mkdir -p /opt/graphite/storage/log/carbon-cache
$ sudo mkdir -p /opt/graphite/storage/log/carbon-relay
$ sudo mkdir -p /opt/graphite/storage/log/carbon-aggregator
$ sudo chown -R carbonusr:carbon /opt/graphite/storage/log
$ sudo chmod 775 /opt/graphite/storage
$ sudo chown -R carbonusr /opt/graphite/storage/whisper
$ sudo chown www-data:carbon /opt/graphite/storage
$ sudo chown -R www-data /opt/graphite/storage/log/webapp
{% endhighlight %}

Start the carbon cache daemon, and ensure it is running:

{% highlight bash %}
$ sudo -E /opt/graphite/bin/carbon-cache.py start
# to check the status of the carbon-cache daemon:
$ sudo -E /opt/graphite/bin/carbon-cache.py status
#   carbon-cache (instance a) is running with pid 15812
{% endhighlight %}

Create the Apache virtual host file, enable the site, and start Apache:

{% highlight bash %}
# disable the default site
$ sudo a2dissite 000-default
# create the graphite-webapp virtual host file
$ sudo vim /etc/apache2/sites-available/graphite-webapp.conf
# ensure the file has following contents:
#    <VirtualHost *:80>
#        WSGIScriptAlias / /opt/graphite/conf/graphite.wsgi
#
#        <Directory /opt/graphite/conf>
#            Require all granted
#        </Directory>
#    </VirtualHost>
$ sudo a2ensite graphite-webapp
$ sudo service apache2 start
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

* [GraphiteApp](http://graphite.readthedocs.io/en/latest/)
* [Monitoring with Graphite (book)](http://shop.oreilly.com/product/0636920035794.do)

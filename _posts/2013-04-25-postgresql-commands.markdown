---
layout: post
title:  "PostgreSQL Commands and Sockets"
date:   2013-04-25 09:43:21 -0400
categories: postgres sql
logo: postgresql.jpg
---
Collection of commands and information related to interaction with a PostgreSQL database. These are
likely rudimentary but are useful for needing to remember them after not having worked with the
database for some time.

### Commands

Some useful commands for interacting with the PostgreSQL database.

{% highlight sql %}
# list databases
$ \l
# connect to database
$ \c db_name '<db_name>'
# show database tables
$ \d
# show users and details
$ \du
# quit
$ \q
# drop an existing database
$ drop database <db_name>;
# create a database
$ create database <db_name>;
# give user superuser access
$ alter user <user_name> with superuser;
# create a role
$ create role <role_name>;
# modify role to add role
$ alter role <role_name> with <role>;
{% endhighlight %}

### Initialization

When installing PostgreSQL, first initialize the database:

{% highlight bash %}
# PostgreSQL 8
$ initdb

# PostgreSQL 9
$ pg_ctl initdb
{% endhighlight %}

Default socket locations are usually not sufficient - edit /data/pgdata/<data_dir\>/postgresql.conf:

{% highlight bash %}
# PostgreSQL 8
unix_socket_directory = '/tmp'

# PostgreSQL 9
unix_socket_directory = '/data/sockets'
{% endhighlight %}

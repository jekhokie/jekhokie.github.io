---
layout: post
title:  "MySQL Snapshots Using ZFS"
date:   2019-05-19 14:41:00 -0400
categories: ubuntu linux zfs snapshot mysql
logo: mysql.jpg
---
This post is an expansion on [this previous post]({% post_url 2019-05-19-zfs-snapshot-rollback %}) detailing basic usage of the
[Z File System](https://www.freebsd.org/doc/handbook/zfs.html). This post expands on the Z File System example and provides
snapshot and rollback capability for a [MySQL Database](https://www.mysql.com/). By the end of this post, you will have a
fully installed/functioning MySQL database with snapshot/rollback capability like a time machine for your data. This functionality
can further expand your CI use cases by providing a refresh capability that is much faster than continuously seeding a database
with data on each test run.

### Technology Layout

This tutorial expands on [this previous post]({% post_url 2019-05-19-zfs-snapshot-rollback %}) and assumes you still have your
Ubuntu-based VM available with ZFS installed. If this is not the case, please visit the previous post and follow the instructions
prior to proceeding.

**NOTE**: All commands in this tutorial need to be run as the root user.

### Create ZFS Dataset for MySQL

MySQL uses the `/var/lib/mysql` directory as its default data directory. This is the directory that will need to be configured
as a ZFS dataset in order to provide the snapshot/refresh capability. Note that this is **NOT** a good method for any production
use cases as the virtual device residing in the `/tmp` directory is an unstable location, but serves well for this tutorial:

{% highlight bash %}
$ truncate -s 1GB /tmp/mysql.img
$ zpool create -f mysql /tmp/mysql.img
$ zfs create -o mountpoint=/var/lib/mysql mysql/datafiles
{% endhighlight %}

### Install MySQL

Now that we have a dataset that we can use for the MySQL installation, let's got ahead and install MySQL:

{% highlight bash %}
$ apt-get -y install mysql-server
{% endhighlight %}

The MySQL server should be installed and running - you can check using the command `service mysql status`.

### Create Initial Database and Snapshot

Let's go ahead and create a database, some tables, and some initial data that we can use as our starting point to snapshot. Log
into the MySQL instance and run the following commands (for reference, you can log into the MySQL instance using the command
`mysql -u root -p`):

{% highlight sql %}
mysql> CREATE DATABASE testdb;
mysql> USE testdb;
mysql> CREATE TABLE test_table (`test_col` CHAR(128));
mysql> INSERT INTO test_table (test_col) VALUES ('initial data');
mysql> SELECT * FROM test_table;
{% endhighlight %}

You now have a database named "testdb" with a single table named "test_table" containing 1 row with the text "initial data" in
the "test_col" column. You can verify this by performing a `SELECT * FROM test_table;` while in the database.

We can now snapshot the database to provide a rollback capability (these commands should be run from the Ubuntu Linux command line):

{% highlight bash %}
$ zfs snapshot mysql/datafiles@snapshot1
$ zfs list -t snapshot -r mysql/datafiles
# should show the "mysql/datafiles" snapshot just created
{% endhighlight %}

We now have a rollback point - let's modify the data in the table and add a few rows of data in the MySQL instance:

{% highlight sql %}
mysql> USE testdb;
mysql> UPDATE test_table SET test_col = 'updated data' WHERE test_col = 'initial data';
mysql> INSERT INTO test_table (test_col) VALUES ('some extra data');
{% endhighlight %}

If you query the `test_table` table, you should see 2 rows - the first with value "updated data" and the second with value "some extra
data". We now have a modified data set in the database. Let's execute the rollback procedure for the database to revert to our initial
snapshot (again, from the Ubuntu Linux command line). This process requires shutting down the MySQL instance prior to performing the
rollback so the database instance does not have data switched out from under it:

{% highlight bash %}
$ service mysql stop
$ zfs rollback mysql/datafiles@snapshot1
$ service mysql start
{% endhighlight %}

Log back into the database - if you query the data in the `test_table` table, you should see a single record with value 'initial data'.
If so, you've successfully performed a snapshot rollback of your database!

### Moving Ahead

The example above lays the grounwork for what could be much more advanced capability. For example, you could write a script that performed
"bookmarking" of your database based on the date/time of the snapshot request, and then use these bookmarks for testing or a CI pipeline.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [MySQL - Initializing the Data Directory](https://dev.mysql.com/doc/mysql-installation-excerpt/5.7/en/data-directory-initialization.html)
* [Using ZFS to Snapshot Your Database](http://labs.qandidate.com/blog/2014/08/25/using-zfs-to-snapshot-your-database/)

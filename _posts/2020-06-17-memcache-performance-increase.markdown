---
layout: post
title:  "Memcached for Performance Increases"
date:   2020-06-17 21:42:00 -0400
categories: memcache docker flask mysql
logo: memcached.jpg
---

[Memcached](https://memcached.org/) is an open source distributed memory caching technology used by many applications to speed up response
time of common data retrievals and reduce overall impact on back-end systems for high volumes of requests. This tutorial walks through using
[Docker](https://www.docker.com/) to set up a small ecosystem of a Python Flask application that communicates with a MySQL back-end for query
results to be displayed to a user, and how adding a distributed memory cache can increase the performance/response time of requests for data
that is requested frequently or requires a large amount of computational time.

### Prerequisites

This tutorial primarily relies on the Docker engine and a MySQL client library. All other aspects will be explained/referenced for how to
get the ecosystem up and running, including simulating delay and testing for performance measurement.

Before proceeding, ensure you have a Docker engine running on the device you wish to use for this tutorial, as well as a mysql client
executable for communicating with the MySQL database.

### Concept

This tutorial will primarily be composed of an application/web server (Python Flask) with a database back-end (MySQL), and we will introduce
a distributed memory cache (Memcached) as a means to demonstrate the power of having such a technology at your disposal.

### Network Setup

We'll first create a network to enable easy container communication in an isolated fashion:

```bash
$ docker network create testnetwork
```

### MySQL Setup

First, we'll deploy a MySQL instance using Docker, and create a stored procedure to simulate what could be a large query that takes several
seconds to respond. Since ingesting and preparing a large data set that could introduce delay in response time is time intensive, we will
instead construct a stored procedure with an artificial delay which simulates query time.

#### MySQL Deploy

Let's first deploy a MySQL instance using Docker. We'll do so with a very straightforward command, also providing a default root password
for use. Obviously this is not good practice (weak password, using root account, etc.) but will suffice for the tutorial:

```bash
# pull down the mysql image
$ docker pull mysql/mysql-server:latest

# run an instance of mysql with a default root password
$ docker run --name mysql \
             -e MYSQL_ROOT_PASSWORD=root \
             -e MYSQL_ROOT_HOST=% \
             -p 3306:3306 \
             --net testnetwork \
             -d mysql/mysql-server:latest

# inspect that the container is running
$ docker ps
```

Now that the image is running, connect to the database using the root password `root` provided in the above command. Note that it may take
a couple minutes for the database to initialize (you may get an error if you try to connect too soon):

```bash
$ mysql --host 127.0.0.1 --port 3306 -u root -p
```

#### Data and Stored Procedure

Now that we're logged into the MySQL instance, let's create a database, table, and record within the table that we can query:

```bash
mysql> CREATE DATABASE test;
mysql> USE test;
mysql> CREATE TABLE test (id INT(6) UNSIGNED AUTO_INCREMENT PRIMARY KEY, some_value VARCHAR(30) NOT NULL);
mysql> INSERT INTO test (some_value) VALUES ('this is a test');
```

We'll now create our example stored procedure. This procedure will again simulate a long-running query by adding a sleep/delay prior to
returning the record we created in the table. Again, this assumes you're still logged in to the MySQL instance and using the `test`
database:

```bash
mysql> DELIMITER //
mysql> CREATE PROCEDURE RunTest()
BEGIN
  DO SLEEP(3);
  SELECT some_value FROM test WHERE id = 1;
END
//
mysql> DELIMITER ;
```

Next, we'll test the query:

```bash
mysql> CALL RunTest();
```

If all goes well, you should see nothing for approximately 3 seconds, and then the value we inserted into the table should return as
the query response. This indicates our stored procedure is working as we expected. You can now exit the mysql instance by typing `exit`.

### Flask Application Setup

Now that we have our database ready, let's focus on the Flask application. First, create a directory for the app:

```bash
$ mkdir flask_app
$ cd flask_app/
```

Next, create a `requirements.txt` file with the following:

```python
flask
mysql-connector-python
```

The `app.py` file which contains the Flask application logic should look like the below contents.

**Note**: Fair warning this code is brittle and unsafe - passwords in application code, lack of error handling, the list goes
on and on. The point here is that we're taking the happy path in order to focus on the material in this post - fixing the
inefficiencies and insecurities of this code is an exercise left up to the reader.

```python
from flask import Flask
import mysql.connector as mysql
app = Flask(__name__)

@app.route('/')
def index():
    # connect to our test database
    db = mysql.connect(
        database="test",
        host="mysql",
        user="root",
        password="root",
        auth_plugin="mysql_native_password"
    )

    # call the stored proc - there will only ever be one result
    # in our test case
    cursor = db.cursor()
    cursor.callproc("RunTest")
    for result in cursor.stored_results():
        r = result.fetchall()[0][0]
    cursor.close()
    db.close()

    return 'Value: {}\n'.format(r)

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

Note that the Flask application is quite simple - it takes a request at the root URI, attempts to query the database for the record we
created (using the stored procedure), and then returns a message with the value of the record.

Next, create a `Dockerfile` with the following contents, which defines how the Flask application should be packaged and run:

```docker
FROM ubuntu:16.04

RUN apt-get update -y && \
    apt-get install -y python-pip python-dev

COPY ./requirements.txt /app/requirements.txt

WORKDIR /app

RUN pip install -r requirements.txt

COPY . /app

ENTRYPOINT [ "python" ]

CMD [ "app.py" ]
```

Finally, build and run the image, exposing the ports that Flask uses by default:

```bash
$ docker build -t flask-app:latest .
$ docker run --name flask-app \
             -p 5000:5000 \
             --net testnetwork \
             -d flask-app:latest
```

We can test the Flask application to see that we get a response within ~3 seconds (again, no caching is in place yet, so each
request is a separate query to our MySQL database stored procedure, which simulates a long response time due to large data queries):

```bash
$ curl http://localhost:5000
# after ~3 seconds, you should see the DB value returned
```

### Memcache Optimization

Now that we know we can call our web application and it interacts with the database appropriately, let's see if we can improve the
efficiency/speed of responses using something like a distributed memory cache such as Memcached. Once again, we have a Docker image
that we can easily use for this task, and we'll walk through adjusting resources in our Flask app to utilize the cache to improve our
overall performance.

#### Create Memcache Store

First thing's first - let's deploy a memcached instance using Docker:

```bash
# pull down the image
$ docker pull memcached:latest

# deploy the memcached instance
$ docker run --name memcache \
             -p 11211:11211 \
             --net testnetwork \
             -d memcached:latest \
             memcached -m 128
```

Next, we'll test that we can reach and interact with the memcache instance by capturing some stats about the instance:

```bash
$ echo stats | nc localhost 11211
# you should see a bunch of memcached stats
```

If you receive a response of statistics about the memcache instance, your cache instance is ready for use - let's update the application
to use the cache!

#### Update App to Use Cache

Let's update our Flask application to utilize the cache. Add the library `pymemcache` to the `requirements.txt` file, and then Edit the
`app.py` file to look like the following:

```python
from flask import Flask
import mysql.connector as mysql
from pymemcache.client.hash import HashClient
app = Flask(__name__)

# set up the memcache interaction using
# consistent hashing
client = HashClient([('memcache', 11211)])

@app.route('/')
def index():
    # attempt to get the value from the cache
    r = client.get('x')

    # cache miss
    if r is None:
        # connect to our test database
        db = mysql.connect(
            database="test",
            host="mysql",
            user="root",
            password="root",
            auth_plugin="mysql_native_password"
        )

        # call the stored proc - there will only ever be one result
        # in our test case
        cursor = db.cursor()
        cursor.callproc("RunTest")
        for result in cursor.stored_results():
            r = result.fetchall()[0][0]
        cursor.close()
        db.close()

        # seed the cache for the next call
        client.set('x', r)

    return 'Value: {}\n'.format(r)

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

Note that the above code also suffers many of the inefficiencies of the previous code in addition to slightly un-safe cache handling,
race conditions, per-request DB connections, etc. Again, this post is focused on demonstrating the concept of utilizing a cache for
performance and therefore leaves much to be desired on other topics around expanding its use.

Once you have the above, re-build and deploy a new Docker image via the following:

```bash
$ docker ps
# capture the ID of the flask app container
$ docker stop <FLASK_APP_ID>
$ docker rm <FLASK_APP_ID>

# build the new docker image
$ docker build -t flask-app:latest .

# deploy the new image
$ docker run --name flask-app \
             -p 5000:5000 \
             --net testnetwork \
             -d flask-app:latest
```

If all goes well, you should be able to interact with your Flask application and see results. On the very first query prior to any other
requests, you should get a delay of roughly 3 seconds prior to receiving a response. However, subsequent requests should return VERY fast
as the Flask application should be able to pull the resulting value from the memcache cache:

```bash
# will be slow - ~3 seconds for a response
$ curl http://localhost:5000

# this (and subsequent calls) should be very fast
$ curl http://localhost:5000
```

As you can see, the initial call (warming the cache) is slow as the application server has a cache miss with memcached and needs to query the
database, resulting in the full ~3 seconds for response time. However, the second time around (and subsequent occurrences) are much faster
due to the data already being in cache and not needing to utilize the database to retrieve the information.

### Conclusion

There is still a lot of content to discuss around consistent hashing, race condition protection, cache expiry, etc. This post is simply an
introduction to caches (specifically, memcached), and given the final experiment, you can see how the performance of your application would be
impacted if you had very large data sets or a very large number of queries against a specific piece of data for high volume requests. Feel free
to use this test layout to run benchmarking with higher volumes, staggered requests, play with expiry times of the cache data, expand the
number of cache servers, etc. It can only help improve your understanding of what benefits a distributed cache system can provide!

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [MySQL Docker Image](https://hub.docker.com/_/mysql)
* [Set Delay for MySQL Trigger Operation](https://www.tutorialspoint.com/how-to-set-delay-for-mysql-trigger-procedure-execution)
* [MySQL Create Procedure](https://www.mysqltutorial.org/getting-started-with-mysql-stored-procedures.aspx/)
* [Flask Quickstart](https://flask.palletsprojects.com/en/1.1.x/quickstart/)
* [Getting Started with MySQL in Python](https://www.datacamp.com/community/tutorials/mysql-python#IMySQL)
* [Memcached Docker Image](https://hub.docker.com/_/memcached)
* [MySQL Connector/Python Developer Guide](https://dev.mysql.com/doc/connector-python/en/connector-python-introduction.html)

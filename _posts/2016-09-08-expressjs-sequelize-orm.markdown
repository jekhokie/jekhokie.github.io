---
layout: post
title:  "ExpressJS with Sequelize as ORM"
date:   2016-09-08 23:25:31 -0400
categories: nodejs expressjs sequelize orm database
logo: expressjs.jpg
---
The NodeJS [ExpressJS](http://expressjs.com/) framework is fantastic for creating performant
web applications. This post details how to integrate the
[Sequelize](http://docs.sequelizejs.com/en/latest/) NodeJS package with ExpressJS to achieve
ORM capability.

### Versions

At thhe time of this post, the following relevant versions of packages were used:

* co (4.6.0)
* express (4.13.4)
* pg (6.1.0)
* pg-hstore (2.3.2)
* sequelize (3.24.1)
* umzug (1.11.0)

### Assumptions

This tutorial assumes that you already have an ExpressJS application created and you are working
within the directory of the project. The tutorial also assumes you are using something similar
to [dotenv](https://www.npmjs.com/package/dotenv) for tracking configuration parameters, or at least
provide the parameters as environment variables when running the application.

### Process

First, install the Sequelize package and required dependencies:

{% highlight bash %}
$ npm install --save sequelize co pg umzug
{% endhighlight %}

* `sequelize` is package which will provide the ORM capability.
* `co` is used for serializing the database creation/migration prior to launch of the ExpressJS listener.
Given JavaScript's asynchronous nature and callbacks, it's best to serialize this initial launch process.
* `pg` is used for creating the database itself if it has not yet been created (`sequelize` does not
currently provide this capability).
* `umzug` provides the ability to run migrations within the code itself (rather than by using the
sequelize command-line binary). It natively creates a database table to keep track of which migrations
have been run for Sequelize.

Next, update your `bin/www` file to create two functions prior to binding the listener. Note that the `co`
package expects a [Promise](https://www.promisejs.org/) resolve() for each step in the serial sequence:

{% highlight js %}
...

// create a user, database, and grant privileges to the user on the database
var pg = require('pg');
function createDatabase(resolve, reject) {
    // set up connection string for creating the database, using any AWS RDS env vars first
    var dbUserName = (process.env.RDS_USERNAME || process.env.DATABASE_USERNAME);
    var dbUserPass = (process.env.RDS_PASSWORD || process.env.DATABASE_PASSWORD);
    var conStringPrefix = "pg://" +
                          dbUserName +
                          ":" +
                          dbUserPass +
                          "@" +
                          (process.env.RDS_HOSTNAME || process.env.DATABASE_HOST) +
                          ":" +
                          (process.env.RDS_PORT || process.env.DATABASE_PORT) +
                          "/";
    var conStringInit = conStringPrefix + 'postgres';
    var newDBName = (process.env.RDS_DB_NAME || process.env.DATABASE_NAME);
    var conStringNewDB = conStringPrefix + newDBName;

    // create the user, database, and grant privileges
    // note that this is not very defensive, but serves its purpose for the tutorial
    pg.connect(conStringInit, function(err, client, done) {
        client.query('CREATE USER ' + dbUserName + ' WITH LOGIN PASSWORD \'' + dbUserPass + '\'', function(err) {
            client.query('CREATE DATABASE ' + newDBName, function(err) {
                client.query('GRANT ALL PRIVILEGES ON DATABASE ' + newDBName + ' TO ' + dbUserName, function(err) {
                    client.end();
                    resolve();
                });
            });
        });
    });
};

// run any database migrations that have not yet been run using Umzug linked to Sequelize
// note - require the sequelize initialized via the models/index.js file via the tutorial
var models = require('../models');
var Umzug = require('umzug');

function migrateDatabase(resolve, reject) {
    var umzug = new Umzug({
        storage: 'sequelize',
        storageOptions: {
            sequelize: models.sequelize
        },
        migrations: {
            params: [models.sequelize.getQueryInterface(), models.sequelize.constructor]
        }
    });

    // determine any pending migrations, and feed those into the executor
    // again, more defensive programming required, but good enough for tutorial
    umzug.pending().then(function(migrations) {
        umzug.execute({
            migrations: migrations.map(function(migration) { return migration['file']; }),
            method: 'up'
        }).then(function(migrations) {
            resolve();
        });
    });
};

...
{% endhighlight %}

Once the above functions have been created in the file, wrap the listen invocation in the same
`bin/www` file to ensure that the database is created and migrations are run prior to the application
launching and binding to the interface:

{% highlight js %}
...

// set up database, then launch listener
// more defensive coding required - again, fine for tutorial
var co = require('co');
co(function*() {
    yield new Promise(createDatabase);
    yield new Promise(migrateDatabase);
}).then(function() {
    server.listen(port);

    ...
});

...
{% endhighlight %}

The application will then be available for use following the `server.listen(port);` line of code.

### Extra - User Tracking

Assuming that you already have a migration for adding user information which includes email address
and number of logins as `logins`, the following code snippett is a nice way to create a user upon
first login, and increment the total number of logins each time they log in (if they already exist):

{% highlight js %}
// create the user if not exist with default values, otherwise, if already exists...
// increment counter for total number of logins for user
models.User.findOrCreate({
    where: { email: req.user.email },
        defaults: {
            name: req.user.name,
            logins: 1
        }
    })
    .spread(function(user, created) {
        if (typeof(created) !== 'undefined' && created == false) {
            user.increment('logins', {by: 1});
        }
    });
{% endhighlight %}

### Summary

With the above setup, each time the application launches, it will run through the following process
(in sequence) prior to the application being available. Although the startup time may be a bit longer,
it is actually a quite nice automated way to ensure that your application deployments always include
the latest up to date migrations:

* Create the database user (ignore error if exists)
* Create the database (ignore error if exists)
* Grant the database user privileges to the database (ignore error if exists)
* Run any migration in the `migrations/` folder that have no yet been run
* Launch the listener for the application, making it available

### Credit

Contributions to some of the above were gleaned from:

* [sequelize Documentation](http://docs.sequelizejs.com/en/v3/)
* [co Documentation](https://github.com/tj/co)
* [umzug Documentation](https://github.com/sequelize/umzug)
* [pg Documentation](https://www.npmjs.com/package/pg)
* [Express Database Integration](https://expressjs.com/en/guide/database-integration.html)

---
layout: post
title:  "Sensu and RabbitMQ Tuning"
#date:   2016-09-20 20:42:21 -0400
categories: sensu rabbitmq tuning
---
Overview of notional (not necessarily tested) best practices as they pertain to tuning the
[Sensu](https://sensuapp.org/) and [RabbitMQ](https://www.rabbitmq.com/) products for production
use.

### Background

Sensu monitoring utilizes RabbitMQ as its transport mechanism (one of a couple different possible
transports available). In order to effectively operate both Sensu and the RabbitMQ services, there
are several recommended 'best practices' as stated by the documentation. This post is an attempt
to collect the best practices into a consolidated post for reference.

TODO: FILLMEIN

### Sensu Check Counts

One very useful item to monitor is the total number of checks that the Sensu setup is running. This
metric can be tracked/forwarded to a Graphite/other Time-Series Database to evaluate the consumption
of your Sensu stack and add to the metrics associated with watching performance of your stack.

The idea is simple - using the Results API, you can obtain a list of all checks for a Sensu cluster
instance via the following Curl request (assuming your Sensu API instance is listening on the default
port of 4567):

{% highlight bash %}
$ curl -s https://<SENSU_HOSTNAME_OR_IP>:4567/results | jq .
{% endhighlight %}

The JSON data returned is an array of objects, each object being a check result. Finding the length of
the array will give you the total number of checks for the Sensu cluster:

{% highlight bash %}
$ curl -s https://<SENSU_HOSTNAME_OR_IP>:4567/results | jq '. | length'
{% endhighlight %}

### Credit

Many of the best practices above were gleaned from the following:

* [RabbitMQ Production Checklist](https://www.rabbitmq.com/production-checklist.html)
* [Sensu-Recommended RabbitMQ Config](https://sensuapp.org/docs/0.24/reference/rabbitmq.html)
* [Sensu Scaling](https://sensuapp.org/docs/0.24/reference/server.html#scaling-sensu)
* [Sensu Results API](https://sensuapp.org/docs/0.24/api/results-api.html)

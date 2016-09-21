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

### Credit

Many of the best practices above were gleaned from the following:

* [RabbitMQ Production Checklist](https://www.rabbitmq.com/production-checklist.html)
* [Sensu-Recommended RabbitMQ Config](https://sensuapp.org/docs/0.24/reference/rabbitmq.html)
* [Sensu Scaling](https://sensuapp.org/docs/0.24/reference/server.html#scaling-sensu)

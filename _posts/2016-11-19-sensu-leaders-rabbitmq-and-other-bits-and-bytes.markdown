---
layout: post
title:  "Sensu - Leaders, RabbitMQ, and Other Bits and Bytes"
date:   2016-11-19 13:53:41 -0400
categories: sensu rabbitmq transport architecture messaging leader-election
---
Overview and detailed explanation of the [Sensu](https://sensuapp.org/) interaction/utilization of
[RabbitMQ](https://www.rabbitmq.com/) as a transport, including details related to messages, queues,
channels, exchanges, and various other AMQP 0-9-1 topics. Also includes some various data points
related to the operation and construction of the Sensu Server and Client agents based on a brief code
review.

### Background

In exploring the Sensu monitoring system interaction with the RabbitMQ message queue technology, it
became clear very early on that there does not seem to be any good resources that explain the specifics
behind how Sensu utilizes its transport (in this case, the suggested technology of RabbitMQ) to interact
with clients and generally supply monitoring information. The documentation explaining the utilization
of a transport is vague in nature, which is good for someone trying to set up the ecosystem, but not as
helpful for someone who wishes to understand the technology stack and specifics behind what is going on
in order to help troubleshoot deep-rooted issues.

This post attempts to explain how the messaging architecture functions, what types of messages are sent
in order to make the Sensu monitoring stack "just work", and various other topics related to the
transport technology. It focuses on the recommended technology for the transport of RabbitMQ - note that
the information and descriptions contained within this post are specific to RabbitMQ and utilizing a
different transport technology (i.e. Redis) will likely require a separate exploration of how the
interaction occurs.

In addition, through exploring how the messaging interaction occurs, several other discoveries were made
based on the code review and testing that explain other important aspects of how Sensu operates (such as
Sensu Server leader election). These items are less detailed but will be included as well for completeness.

At the time of this post, the following technology versions were used and explored, making this
information specifically accurate for these versions of software. The data within this post may be
accurate for previous/future versions, but should not be expected to:

* **Sensu (server, client, API)**: 0.26.5
* **RabbitMQ**: 3.6.5

### RabbitMQ - Primer

RabbitMQ is a messaging broker that implements the
[AMQP 0-9-1 protocol](https://www.rabbitmq.com/amqp-0-9-1-reference.html). It provides clustering and
mirroring capabilities to make it a fairly robust solution. However, the concepts related to the
technology itself are quite numerous and warrant some explanation as they relate to how Sensu integrates
so that the rest of the subject areas below make sense. This is absolutley by no means intended to be a
full detailed explanation but, rather, a very basic primer in terminology and concepts that will be
helpful in the below Sensu categories.

***Note***: The following section is both a summary of
[this](https://www.rabbitmq.com/tutorials/amqp-concepts.html) reference as well as demonstrated testing
on my behalf.

RabbitMQ is a message **broker** - this means it is a technology that lives between **publishers** and
**consumers**.  Publishers send messages to the broker, and the broker routes messages to consumers.
The distribution of messages can be performed either via a deliver/push method (broker pushes to
consumers) or a fetch/pull method (consumer fetches from the broker).

Messages are published to **exchanges** within the broker. Exchanges then move the messages to
**queues** using rules defined as **bindings**. There are many other various attributes that can be
defined to impact/effect publishing, routing, or forwarding, but we will consider those out of scope for
this exercise.

An exchange can have various defined routing algorithms. As it relates to Sensu, two primary routing
types are utilized (in the simplest of cases) - **direct** and **fanout**.

* **Direct**: In direct routing, the messages are routed to the queues based on whether the routing key
of the message matches the routing key of the queue within the exchange. If so, the message is forwarded.
* **Fanout**: In fanout routing, messages coming into the exchange are routed to all queues bound to it,
regardless of routing key.

Sensu utilizes direct routing for the typical queues (keepalives and results), and fanout routing for
subscription-based check publications (advertise to all subscribed). There are more complicated routing
setups established for items such as roundrobin subscriptions, which use direct routing in a specific
fashion.

There are many ways the various topics discussed above can be linked together, but the most common way
that Sensu utilizes this is via the following:

1. Establish an exchange for X, Y, Z data.
2. Bind (set up routing rules) for the various exchanges to the various queues that need to receive
the data, utilizing either a fanout routing methodology or a direct methodology, depending on the data
and type of engagement.

Another term that will be seen via the management interface and in general is the concept of a **channel**.
Channels are a construct that defines how a client connects to the broker. Channels typically exist one
per process/thread from a consumer, and are the way in which RabbitMQ deals with communication between
clients (producers and/or consumers) and the respective data transacting the system (via exchanges, queues,
etc).

The above should be enough information to understand the following sections. One last item should be
mentioned in the clustering of RabbitMQ as it relates to mirroring (Sensu expects the keepalives and
results queues to be fully mirrored). Mirroring simply creates copies of the data across however many
nodes are defined for the mirroring. However, in RabbitMQ, this does not mean that the data itself can
be manipulated on any host - it is important to remember that RabbitMQ enforces that the data
manipulation can only be performed on the master instance itself, meaning if you are using a load
balancer in front of your RabbitMQ cluster for infrastructure handling with round-robin routing (most
popular), it's likely that a significant portion of the connections make it to one of the slave/mirror
nodes for the data, at which point the particular slave node redirects the request to the master for
actual access/manipulation of the data. This is significant overhead and should be considered in the
architecture of the cluster engagement with various technologies.

#### Sensu Server - Setup/Transport Communication

When starting a Sensu server instance, the bootstrap process includes the following interaction with
the transport (RabbitMQ in this case):

- Subscribe to the "keepalives" queue, which is used for determining whether a host has gone offline
or is unreachable/Sensu service is down.
- Subscribe to the results queue, which is used for check results sent from the client (and in some
cases, the Sensu servers internally as well).
- Sets up leader election through Redis (more on this in the "Sensu Server - Leader Election" section
below).
- Initializes various other components such as logging, daemon process handling, etc.

#### Sensu Client - Setup/Transport Communication

Client startup and bootstrap is slightly different than the Sensu server:

- Publish an initial keepalive, which also serves as the client registration, to the keepalives queue
in the transport.
- Set up periodic publication of keepalive data to the keepalives queue every 20 seconds.
- Set up subscriptions for configuration-defined subscriptions (i.e. "linux", etc), which triggers
the creation of an exchange with the subscription name (if not exist) followed by binding to the
queue associated with the host connection to the transport.
- Set up subscriptions for "client:<HOSTNAME>" - which creates an exchange with this name (if not
exist) followed by binding to the queue associated with the host connection to the transport. This
allows for generating ad-hoc check requests via the /request API.
- Set up periodic standalone checks based on the check descriptions, which publish to the results
queue.

#### Transport Example - Specific Client

As a way to make the previous content more tangible, an example is in order. Consider two compute
nodes, "node1.localhost" and "node2.localhost", each of which is running a Sensu client service, and
"node1.localhost" is also hosting the Sensu server service. Also suppose that there is a subscription
check named "dev" that is available to the clients, both of which subscribe to the check. From the
transport perspective, the following channels/exchanges/queues are established for communication
purposes between the clients and the server:

- Server node "node1.localhost" establishes a channel with RabbitMQ with either the IP address or
hostname of the server node.
- Server node "node1.localhost" subscribes to the "keepalives" and "results" queues via exchanges
named "keepalives" and "results", respectively.
- Server node publishes checks to the "dev" queue for subscriptions to the check via the "dev"
exchange.
- Client nodes "node1.localhost" and "node2.localhost" each establish a channel with RabbitMQ,
named with either the hostname or IP address of the client node.
- Client nodes "node1.localhost" and "node2.localhost" establish exchanges with the name(s)
"client:node1.localhost" and "client:node2.localhost", respectively.
- Client nodes "node1.localhost" and "node2.localhost" establish queues with the name(s)
"node1.localhost-0.26.5-<EPOCH>" and "node2.localhost-0.26.5-<EPOCH>" respectively, where
0.26.5 is the version of Sensu client they are running and EPOCHTIME is the Epoch time when the
queue is created and subscribed to.
- Client nodes "node1.localhost" and "node2.localhost" bind the queues "node1.localhost-0.26.5-<EPOCH>"
and "node2.localhost-0.26.5-<EPOCHTIME>" to exchanges "client:node1.localhost" and
"client:node2.localhost" respectively. These exchange bindings are only really used for ad-hoc
triggers of checks for specific clients via the "/results" endpoint of the Sensu API service.
- Client nodes "node1.localhost" and "node2.localhost" bind the queues "node1.localhost-0.26.5-<EPOCH>"
and "node2.localhost-0.26.5-<EPOCHTIME>" respectively to the "dev" subscription exchange for delivery
of subscribed check requests by the server.
- Client nodes "node1.localhost" and "node2.localhost" publish keepalives and results to the
"keepalives" and "results" queues, respectively.

A graphical representation of the above exchange/queue interaction is detailed below:

```
                     |-----------------------------|--------------------------------------|
                     |         EXCHANGES           |               QUEUES                 |
                     |-----------------------------|--------------------------------------|
                     | |------------------------|      |--------------------------------| | sub |--------------------------|
                     | | client:node1.localhost | ---> | node1.localhost-0.26.5-<EPOCH> |------>|       sensu-client       |
                     | |                        |      |                                | |     |     (node1.localhost)    |
                     | |------------------------|    ->|--------------------------------| |     |--------------------------|
                     |                             /                                      |       |                        |
|--------------| pub | |------------------------| /                                       |       |                        |
| sensu-server | ----->|         dev            |                                         |       |                        |
|--------------|     | |------------------------| \                                       |       |                        |
  ^     ^            |                             \                                      |       |                        |
  |     |            | |------------------------|   -> |--------------------------------| | sub   |  |-------------------| |
  |     |            | | client:node2.localhost |      | node2.localhost-0.26.5-<EPOCH> |---------|->|    sensu-client   | |
  |     |            | |                        | ---> |                                | |       |  | (node2.localhost) | |
  |     |            | |------------------------|      |--------------------------------| |       |  |-------------------| |
  |     |            |                                                                    |       |    |              |    |
  |     |       sub  | |------------------------|      |--------------------------------| | pub   |    |              |    |
  |     |--------------|       keepalives       | ---> |           keepalives           |<--------|    |              |    |
  |                  | |                        |      |                                |<-------------|              |    |
  |                  | |------------------------|      |--------------------------------| | pub                       |    |
  |                  |                                                                    |                           |    |
  |             sub  | |------------------------|      |--------------------------------| | pub                       |    |
  |--------------------|        results         | ---> |            results             |<----------------------------|    |
                     | |                        |      |                                |<---------------------------------|
                     | |------------------------|      |--------------------------------| | pub
                     |-----------------------------|--------------------------------------|
```

### Sensu Server - Leader vs Worker

When instantiating 1 or more Sensu server instances, a leader is established to handle specific
responsibilities associated with the Sensu infrastructure. The leader elected is the only Sensu
server instances that will perform the following tasks, which are specifically focused around
stale monitoring and updating of clients and checks:

- Publish check requests to the transport for all configured subscription checks.
- Monitor client registry (Redis) and create events based on stale keepalives if clients have
not checked in within the specified timeout period.
- Monitor check results and create check TTLs to determine stale check results.
- Monitoring check aggregates and pruning stale aggregate results.

If you've used Sensu before, you might be asking "well what do the other host(s) in the cluster do
if the leader does all of the above". The "clustering" of Sensu provides for horizontal scaling of
event processing. The workers handle keepalive results (subscribe to the keepalives) and check
results (subscribe to the results queue). This can generally be a resource-intensive task based on
the types of handlers, mutators, and other constructs available in Sensu that are configured for
the server(s) and checks.

In addition, the workers routinely check on the health of the Sensu leader - in the event that the
leader is no longer available or able to perform its duties, one of the workers will attempt to
take over and assume responsibilities of the leader. This process is explained in further detail
in the below section.

#### Leader Election

Sensu utilizes automatic leader election to determine the instance that will be responsible for the
previously-mentioned leader tasks. Leader election is done at the very begining of the process start
as well as routinely thereafter, and deals with 2 specific keys within the Redis data store:

- **lock:leader**: value is a "time since epoch" timestamp.
- **leader**: value is a UUID generated by the current leader.

When a Sensu server is first started, it attempts to set (if not set) the "lock:leader" key in Redis.
If the key is already set, the worker will then check the "lock:leader" timestamp to see if it is
greater than 30 seconds old. If the value is recent (less than 30 seconds old), the Sensu server will
become a worker as the leader is already elected and functioning. If the value is greater than 30
seconds old, the Sensu server assumes that the leader is no longer available/can no longer assume
its duties, and attempts to set the lock timestamp in Redis. Likewise, the leader will continuously
and at more frequent intervals (10 seconds) update the timestamp within the "lock:leader" key in order
to retain leader responsibilities if it is healthy.

If a previous leader was up and functioning, but lost connectivity or some other adverse condition
occurred, another Sensu worker will likely take over as the leader following a 30 second time lapse.
However, if/when the previous leader comes back online, you could end up with multiple leader
instances because both the new and the previous leader think they are still the leaders. This is where
the "leader" key in the data store comes in to play. As part of the functionality to update the leader
lock (every 10 seconds for the leader), the leader will first check if the "leader" value in the data
store matches its own expected leader UUID. If not, it is assumed that some other Sensu worker has
already taken over, and as a result, the current leader will "resign" and become a regular Sensu
worker node.

### Credit

Much of the information obtained above was obtained via testing/exploring the data packets being sent
between Sensu Server and Client agents and the RabbitMQ transport. In addition, the following
resources were helpful in determining how the code functioned as well as various AMQP 0-9-1 protocol
specifics:

* [AMQP 0-9-1 Concepts](https://www.rabbitmq.com/tutorials/amqp-concepts.html)
* [AMQP 0-9-1 Protocol](https://www.rabbitmq.com/amqp-0-9-1-reference.html)
* [Sensu Source Code](https://github.com/sensu/sensu)
* [Sensu Transport Source Code](https://github.com/sensu/sensu-transport)
* [Sensu Settings Source Code](https://github.com/sensu/sensu-settings/)
* [Sensu Client Subscription Configuration](https://sensuapp.org/docs/latest/reference/clients.html#client-subscription-configuration)

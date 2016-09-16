---
layout: post
title:  "Carbon Cache - Code Review"
date:   2016-09-15 21:11:21 -0400
categories: ubuntu graphite carbon-cache code-review
---
Some notes related to my browsing of the Graphite
[Carbon Cache Python code](https://github.com/graphite-project/carbon) in an attempt to better
understand the operation of the cache such that better decisions can be made related to tuning
the product.

### Background

In my various attempts and iterations of tuning/testing the Graphite stack, it became apparent to
me based on lack of information spread across the web that the only way I was really going to
understand the Graphite stack was via reading the actual code associated with the product. Lack
of understanding related to the operation of a product is one of my pet peeves when it comes to
dedicating time and effort into implementation, and even moreso when it comes to operationalizing
a product that has both on-call and peoples' personal lives associated with it.

### Digging In - The Code

This review is not intended to be an overall picture of the code base by ANY stretch of the
imagination. In fact, I specifically gloss over the mass of it based on my targeted attempt to
understand some items critically related to the operation and tuning of the product itself. The
focus is specifically around the metrics associated with the operation of the tool, throttling,
and various parameters defined in the configuration as well as where they are utilized.

#### `service.py`

Starting out, I investigated the `service.py` file. This is the entry point for the setup of many
various services and threads associated with the Carbon instance. The writer is instantiated in
this file (the thread associated with persisting the metrics received to disk via the Whisper
database), and the `metricsReceived` and `metricsGenerated` metrics are bound within this file.

#### `protocols.py`

At a high level, the `protocols.py` file contains much of the receiver code itself. It has both
the `MetricsReceiver` and `LineReceiver` definitions. The `metricsReceiver` implementation is
the base code which is extended by the `lineReceiver` implementation, which also restricts (throws
an error) when input received is greater than 400 chars by default (which is why in some cases,
when data is not filtered prior to being sent to the Carbon Cache instances, you may see various
'invalid line' errors in the logs - this is the code that will typically detect and report this
issue).

Also within the `protocols.py` file is the `CacheManagementHandler`, which deals with
queries against the Cache instance itself (i.e. from Graphite-Web or other query sources that
may query the Cache instance directly). This code base is also responsible for incrementing the
`cacheQueries` and `cacheBulkQueries` metrics reported by the Cache (each time such an event
is performed against the Cache).

#### `writer.py`

The next file in the sequence was `writer.py`. This file contains the code that is responsible for
writing (persisting) the metrics within the Cache to the disk via the Whisper file system. The
method by which the metrics are written to disk is via a bucket of available write operations. If
the user has specified `MAX_UPDATES_PER_SECOND` as a non-'inf' value in the `carbon.conf`, a bucket
(token bucket of available updates) is created for the update operations. Likewise, if the user has
specified `MAX_CREATES_PER_MINUTE` as a non-inf value in `carbon.conf`, a bucket is also created with
tokens for available write operations.

First, creates - if a bucket is created due to a non-'inf' value in the `carbon.conf` for the
`MAX_CREATES_PER_MINUTE` configuration, the token bucket created is essentially the throttle that
the Carbon instance uses to ensure it respects the maximum defined in the configuration. Each time
a write operation is attempted, it first checks to ensure there is an available token in the create
bucket. If so, it deducts the token from the bucket and performs the create, as well as updates the
`creates` metric associated with the Cache instance. If there are no tokens available, the write
operation is aborted and the metric will essentially have to wait until the next go-around to be
written to disk. which causes the overall memory footprint of the Cache instance to increase.

Following the create operation, there is a check for update operations (metrics that already have
a database file on disk and incoming data points need to be written to the whisper file). There is
a slight difference in the implementation of the update operation. In this update condition, the
token bucket is queried for available tokens to update. However, if there are no tokens available,
the code actually blocks until there is an available token (whereas the create operation would have
simply passed over the metric write/create and attempted again during the next pass of the code).
Once a token is available for the update operation, it attempts the update. If an error occurs, the
`errors` metric for the Cache instance is increased. If successful, two additional metrics are updated.
The first is the `committedPoints` metric, which is incremented with the total data points within
the updated metric. The second is the `updateTimes` metric, which is appended with the total time
it took to update the existing metric.

#### `util.py`

The `util.py` file contains a few different things, but the interesting code worth mentioning is
the code that implements the token buckets as previously mentioned in the `writer.py` file. This
code implementation is concerned with throttling the update and create operations based on the
configuration file directives for `MAX_UPDATES_PER_SECOND` and `MAX_CREATES_PER_MINUTE`. Reading
through the code itself will prove interesting, but in summary, the `TokenBucket` implementation
is the throttling mechanism configured by the `carbon.conf` file. As mentioned previously, if the
call is specified as blocking, the bucket code will block until the requested tokens are available,
which ends up synchronizing the calling code base.

#### `instrumentation.py`

As mentioned in the previous sections, many events are fired/metrics stored about the Carbon Cache
instance itself. The events themselves are handled by the code contained within the `instrumentation.py`
file. This code base handles incrementing, appending, and calculating metrics associated with the
oepration of the Cache instance, and also records some information about the CPU and Memory
consumption of the running process. Injecting its own instance metrics, the code is careful to not
artificially inflate the metrics with its own metrics.

#### `events.py`

The `events.py` file contains the actual events associated with the Carbon Cache instance.  There are
more than several events within this file, but a few worth explaining here. The code within this file
actually handles binding the events to handler functionality.

1. `cache.overflow`: This event fires when the instance cache size exceeds the `MAX_CACHE_SIZE`
configuration in the `carbon.conf`. The event is a couter that is incremented when an overflow occurs.
2. `cacheTooFull`: Boolean corresponding to when the cache is too full to accept metrics. The
boolean itself helps direct throttling/dropping metrics on the floor so that the Carbon instance does
not die as a result of an out of memory error.
3. `PauseReceivingMetrics` and `ResumeReceivingMetrics`: These events are generated as a result of the
cache being too full/having available space, respectively. They are utilized when the `USE_FLOW_CONTROL`
configuration in the `carbon.conf` file is set to True, and again, directly affect the ability of the
instance to ingest net metric data.

---
layout: post
title:  "Graphite Metrics (Somewhat) Explained"
date:   2016-08-24 09:34:04 -0400
categories: graphite carbon whisper metrics
---
In my quest to figure out how to tune Graphite, it became very quickly apparent that a good source
explaining what each Carbon Cache metric stands for in the cache reporting was lacking. I found
[this](https://answers.launchpad.net/graphite/+question/145032) particular source, which I'm detailing
below for reference:

### Explanation (Somewhat)

Quoted from referenced article under "Contributions":

> ...
>
> cache.queries - the number of queries made against "the cache".
>
> cache.queues - the number of queues in the cache, which logically corresponds to the number of distinct metrics that have datapoints waiting to be written.
>
> cache.size - the sum total of the sizes of all the queues (the number of datapoints in "the cache").
>
> metricsReceived - the number of (metric, datapoint) pairs received by carbon.
>
> cpuUsage - carbon's own measurement of its user + system cpu time.
>
> creates - the number of new metrics (new wsp files) created each minute, this is typically 0.
>
> errors - a quantitative measurement of bad joo-joo.
>
> updateOperations - as the writer thread iterates all the queues in the cache, it takes a queue and writes all of its datapoints to a wsp file in a single update operation. This measures the number of update operations occurring each minute. Note that some updates may be a single datapoint while others may involve many datapoints, depending on how much data is in the queues.
>
> pointsPerUpdate - the average number of datapoints written in each update during the minute.
>
> avgUpdateTime - the average time each update operation takes. In my youthful stupidity I chose to measure this in seconds, thus the values are typically extremely small... Likely to change to microseconds in the future.
>
> committedPoints - the total number of datapoints written each minute. Generally this should be equal to updateOperations times pointsPerUpdate.
>
> ...

### Credit

Contributions to some of the above were gleaned from:

* [LaunchPad - Graphite](https://answers.launchpad.net/graphite/+question/145032)

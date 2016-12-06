---
layout: post
title:  "Technical Approach"
date:   2016-12-05 18:16:01 -0400
categories: tech mentor career
---
General technical approach that I take to prototype and experiment with new technologies. The steps
detailed in this post are iterative and specific to the way that I am able to best analyze and assess
the functionality of a new software tool, and have been applied to hardware-based solutions as well.

### Background

As a means to explain my approach to problem-solving, I will explain how I dealt with the deep-dive into
the Graphite stack and the posts that resulted. The following posts are all within scope of this
explanation of how I approach problem-solving (in sequential order of approach/timeframe written):

- [Installing Graphite - Bleeding Edge (0.10)]({% post_url 2016-08-22-installing-graphite-bleeding-edge %})
- [Add Carbon Relay to Graphite]({% post_url 2016-08-23-add-carbon-relay-to-graphite %})
- [Graphite Metrics (Somewhat) Explained]({% post_url 2016-08-24-graphite-metrics-somewhat-explained %})
- [Installing Graphite - Version 0.9.15]({% post_url 2016-08-25-installing-graphite-0-9-15 %})
- [Tuning Graphite - Version 0.9.15]({% post_url 2016-08-25-tuning-graphite %})
- [Tuning Graphite - Part 2 (More Carbon)]({% post_url 2016-09-10-tuning-graphite-part-2 %})
- [Carbon Cache - Code Analysis]({% post_url 2016-09-15-carbon-cache-code-analysis %})
- [Tuning Graphite - Part 3 (C-Relay)]({% post_url 2016-09-15-tuning-graphite-part-3 %})

### Problem Statement and/or Intent

When starting out exploration of a technology, I first start with a defined intent. When it comes to
exploring new technology that I am not familiar with, this can be quite difficult and the problem
statement/intent will most likely change or morph over time, but in general, it is still good to keep
an idea in mind about what you wish to prove/disprove/learn.

In the case of the Graphite stack, the problem was lack of understanding about the technology as a whole
and its corresponding performance tuning. My intent was to understand the technology and its performance
characteristics such that I could make calculated adjustments to change the operating characteristics of
the technology stack. This is a mouthful - however, it served as my "epic" (for those of you familiar with
"Agile" methodologies). As I am very much an iterative engineer both in practice and in mindset, I knew
this would be a good starting point because it kept me focused on the end goal of my explorations while
allowing me to traverse various rabbit holes until I was able to achieve the goal.

The intent statement served as the end-goal that I would keep my sights on. The following sections of this
post detail the steps and iterative methods I utilized to achieve the intent statement.

### Step 1 - Install, Configure, Test with No Tuning

Whenever exploring a brand new technology stack that I have no or very little knowledge about, I always
start with the basics - find a quick start guide and high-level architecture diagram and get to work.

#### Documentation

The very first step I always take to learn new technology is to read the documentation. Many software
implementations (especially in the open-source community) can have very spotty documentation, so this step
is actually quite good for gauging how mature and established the software is in the community. If
documentation is lacking (formal documentation) it is sometimes helpful to perform this step regardless
as you can discover various use-cases and current customer perspectives that can help formulate an opinion
of the technology before you even get started. However, just remember from my previous posts - data is
king, so do not let that opinion based on other peoples' experiences define the technology for you - use
it as a data point that you prove/disprove based on your own understanding and configuration of the
technology.

#### Install Software

Next, install and configure the software - from "scratch", most of the time without any "helpers". By
helpers I actually mean operating-system package management systems and various other things that
make software more easily consumable (in other words, I build from source if possible). This may
sound insane to most people, but I've on more than one occasion discovered some deep dark secrets of
software via this method that I would have otherwise glossed over had I utilized a packaging system
or similar (think "database table default naming convention not documented anywhere").

So, I proceed - I downloaded the packages for the Graphite stack (latest version) and got to work. Now,
I did use the Python package management system to install dependencies but still avoided using native
operating system package management systems (in this case, Ubuntu "apt"). Additional dependencies
required by the operating system, however, I did use the native package management system for as this
was not an exercise in understanding the dependencies but, rather, the software itself.

#### Configure Software

Most software will include a "quick start" guide that gets you up and running with a minimal set of
configuration options. This is fine for most instances, and is typically where I start my exploration
to limit the amount of knowledge required for a first-pass "it's working" setup.

Minimal configuration for a first pass in the Graphite ecosystem included just a few tweaks to get the
configuration file in line with something that works. Doing just this, I opted for minimal configuration
changes based on the quick start and shelved the configuration file exploration for later.

#### Testing and Success Proof

As with any scientific method, there needs to be a way to prove (or disprove) your hypothesis. Once I
had a baseline fully-functional Graphite stack up and running, testing was in order. Some simple
testing to ensure services are running is definitely key, as is finding where the logs are located to
help troubleshoot issues when (not if) they are encountered. Testing the ingest abilities and actually
visualizing the results was priority in accomplishing prior to proceeding.

One thing is for certain - you can't declare success without data. In the interest of visualization,
I explored what was available to me for the Graphite stack in the form of operational metrics. As I
started digging into the metrics, it became apparent that there weren't very good sources of
definitions for what each metric stood for in the reporting pipeline. Unfortunately, this resulted
in one of my major discoveries - Graphite is very, VERY poorly documented from the perspective of
wanting to truly understand the technology stack. Later in this post you will see where I stopped my
activities and dug directly into the code base of the software itself to better understand the
transactional model of the software itself - at this point, I simply focused my code searching efforts
on metrics specifically, and was able to come up with a cohesive list of metric names and corresponding
explanations for each, thus allowing me to set up some metrics tracking for future activities.

### Step 2 - Expand Architecture and Consider "The Things"

Most software installs as a standalone process for testing and prototyping/development activities. This
is great for learning a new technology but typically falls apart whenever concepts like "scale" or
"high availability" come into play. Once I get something working I typically like to explore some of
the extra components that make it both scalable and highly-available (if any). In general, there are
various ways you can identify and troubleshoot bottlenecks that may exist in either the hardware,
software, network, or various other components, but I typically limit the initial exploration to the
hardware and software directly. In order to prove either, however, you need a good load testing
capability.

#### Consider the Hardware

When running a technology stack, I start with what seems like it may be the smallest footprint required
to get reasonable output from the software itself. If there is documentation indicating metrics, then
you are typically at an advantage, but most software (especially open source) leaves it up to the
consumer to figure this out for themselves/their own workload.

For the Graphite implementation, I picked a reasonably-provisioned compute resource for the Graphite
node. From there, I load tested and pushed the instance to its limit. This is another important point -
identifying the failure scenario for the software itself can be difficult. In this particular case, I
knew that near-real-time metrics were the objective and any deviation from that concept was considered
an anomaly/failure. With this, I continuously scaled the load-testing script to push the system harder
at each iteration, making sure to collect metrics along the way. When you find your breaking point, it's
important to hold onto the metrics to ensure that you have a baseline for your current setup.

#### Failure Identification

Once you've pushed your single (or minimal, whichever is applicable) software instance to its limit,
identifying bottlenecks can be somewhat easy if you've collected the "right" metrics. However, in
many cases, this requires quite a bit of skill and experience to pinpoint issues that may not be so
obvious, and it is generally good to both run multiple tests with the same hardware and also run a
similar test with the exact same input profile with different hardware. Identifying easy things such
as "memory filled up quickly" or "CPU was overloaded" will help in identifying how to adjust your
hardware such that you can make a somewhat educated guess in the progressive direction.

The trick for me is making sure that I have multiple types of hardware mixed with multiple runs of the
same benchmarking profile on each hardware. This has become immensely easier given cloud-based
technologies such as AWS and Azure, and can be fairly inexpensive if you understand the load you are
generating and the price profile for the load. The point is, without data points, graphs, spreadsheets,
or various other items of quantitative data, it is very difficult to know whether you are identifying
and resolving the correct bottlenecks or are just swinging a stick and hoping to hit something. Often
times, if you do not fully understand the software you are operating with based on lacking documentation,
the operational characteristics of the software can become very obvious simply by varying the resources
available to the compute resource on which the software runs. If, however, you do somewhat understand
the operational characteristics of the software, by all means utilize that knowledge to make hardware-
based adjustments more appropriately.

The below sections are not listed or intended to be ordered - in other words, benchmarking and testing
requires iterations of each of the below sections, and it is by no means a perfect science.

##### Hardware-Based vs. Software-Based Failures

Without somewhat of a systems or software background, the overall analysis of performance tuning can
be daunting to some. If you are ever in a position/opportunity to perform this type of analysis for
a new technology stack, take this amazing opportunity to "learn up" and sharpen your skills in areas
you may not be comfortable with. One of the best ways to become a better and more well-rounded
engineer is to step outside your comfort zone, and such an opportunity as this is the perfect time to
do so!

#### Consider the Configuration

Tuning a technology stack can be difficult, especially if documentation is poor. Read through the
various included configuration file samples to try and understand the operational configurations
associated with the software. Doing so can help lead to an "aha" moment that documentation never
provided, such as thread tuning, memory tuning, data tuning, etc.

In Graphite, I was able to identify and adjust some of the data ingest parameters, which changed
the operational characteristics of the software quite visibly. Things such as "how many updates per
minute maximum" or "how many creates per minute maximum" completely adjust the operating profile of
a system that is at its limit. However, be careful with tuning and make sure you understand the intent.
If you are not completely aware of or cannot infer the operational changes associated with updating
tuning parameters, read through the documentation and/or (even better) identify in the code base what
is actually happening with that value to better understand how to treat the value in your own
operational environment.

#### Consider the Software

Another component to consider is the operation of the software itself. In the case of Graphite, I
noticed when I did my testing that a baseline-approach (little tuning to the software) resulted in
almost complete starvation of one single CPU on the compute resource. Reading through the Graphite
documentation and code base a bit it became clear and made sense why this was happening - Graphite
utilizes (out of box) a sorting algorithm prior to laying down metrics data to the Whisper file
system in order to optimize disk I/O. This sort algorithm can be very expensive, and as Graphite
is written in Python, its implementation is not capable of taking advantage of multiple CPUs. The
sort operations were very expensive for high volumes of fragmented time-series data, and as I was
utilizing a solid-state drive for storage, disk I/O was never a visible issue in my tests at the
time.

Identifying the bottleneck in this case pointed to an easy question - could I simply increase the
CPU speed to gain more efficiency. The answer, unfortunately, was "not without significant cost"
within the ecosystem I was operating. Therefore, I started looking at the software and discovered
that Graphite was written to allow a "Relay" to funnel data to multiple Carbon Cache processing
resources. Thinking through this, an architecture of this setup would allow me to double the number
of sorting processes which should theoretically halve the compute resource consumption.

In this example, I was able to identify a particular software/architecture-based adjustment that
should bring efficiencies into the processing pipeline. Once identifying this, since I already
have a test framework in place for load-testing/benchmarking, making a small configuration change
is shortly thereafter followed by re-testing and collecting metrics for each test.

However, when making architecture changes, understand the implication and operational characteristic
updates. In this case, I needed to ensure I could not only track metrics for the Carbon Cache
instances but also the relay itself. Without being able to collect relay information, I would be
blind to and possibly completely miss the fact that although the multiple Carbon Cache instances
were in fact able to better utilize the multiple CPUs on the compute resource, the efficiency of
the overall instance was still quite low due to the Relay being the bottleneck at that point (I had
traded one software-based bottleneck for another).

#### Consider Alternatives

Sometimes, the software is only as good as the algorithm it implements, the runtime libraries it
includes, the interpreter it utilizes, or any number of other environmental factors that you or
the author may have no control over. Aside from testing (quite a bit of it in order to identify
this as the potential issue or bottleneck, and requires quite a bit of expertise in the respective
area), searching the web for information about the technology stack can prove to be quite fruitful.

In the case of Graphite, I ran up against a hard point that I simply could not breach with respect
to the Relay instance. Thinking through this, I iterated back to hardware-based and moved the Relay
onto its own instance to reduce the memory footprint and see if efficiencies could be gained. This
did alleviate *some* of the issues, but in general, I observed relatively the same bottleneck in the
Relay as I had with the single Carbon Cache instance - no good, entirely processor-based.

At this point, I started scouring the internet a bit for some quick wins related to the Relay. As
expected, this is a common issue and is directly related to the slow operational footprint of the
Python programming language (slow relative to fast processing requirements). Alternatives included
Relay software that had been developed in C and Go programming languages. After picking up and
replacing my current Python-based Relay with a C implementation, I immediately saw a magnitude of
improvement in the processing pipeline to the point that, in my testing, I could continuously add
Carbon Cache instances and the Relay would far exceed the processing capabilities of its downstream
Cache instances (I tuned it to the point that was sufficient for my use case and then stopped).

### Conclusion - Go Forward

In general, the "approach" summarized above seems very ad-hoc - and this is by design. In my
experience, approaching different problems with the same approach has never quite yielded the same
success across the board. Maybe it's my approach that is incorrect, or maybe it's just the nature
and complexity of the solutions I've attempted to come up with, but in general, the very iterative
and somewhat chaotic-seeming approach outlined above (in the sense of "bouncing around" between
various things to explore) has always worked for me and has proven to be quite fast in arriving
at conclusions. I encourage you to identify your own method for solving complicated problems,
taking with you some of the pieces (or all) from the above details. When in doubt, iterate, as
you never know what you'll find that you may have missed on your first-pass at a problem.

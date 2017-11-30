---
layout: post
title: Monitoring TaskCluster infrastructure
summary:
tags: [taskcluster]
---

Responding to incidents for TaskCluster has been a bit of good logging, helpful
people, and black magic.  As the platform matures, the monitoring must also grow
with it.

Throughout the year, the TaskCluster team has been integrating more logging and metrics into the platform
by aggregating logs into Papertrail and storing metrics in InfluxDB. This data has proven to be invaluable and has allowed us to
make informed choices as to capacity and the health of TaskCluster.  However, as great as
these metrics and logs are, one large drawback is that they mainly have been only useful when someone
is actively looking at them. Logging alerts are also only as good as the messages
that are logged and do not detect the absence of a message (at least when using a
service like Papertrail).

Recognizing the gap in our operations processes around TaskCluster, the team brainstormed
some ideas while at the Berlin work week and decided on a phased approach over the
next few quarters.  Our primary goal is to increase the stability of the platform
and also be the first responders to issues rather than our end-users.

So, where we're at:

* Metrics from TaskCluster Platform as well as some TaskCluster Services are recorded
in InfluxDB and graphed on a [Grafana](http://grafana.taskcluster.net/) instance.
* Logging from various services is redirected to Papertrail
* Alerts are configured within Papertrail for certain scenarios
* High level [status page](http://status.taskcluster.net/) for TaskCluster Heroku services reported

In the coming quarters, we will be looking at adding additional operational monitoring
in a few phases.

#### Phase 1 (Q4/2015) - Metrics Alerting

We have done a lot of work to get useful metrics into InfluxDB and it has proven
to be valuable with capacity planning and platform troubleshooting.  However,
these metrics do not help us detect issues before they happen unless we have some
form of monitoring in place to focus our attention.

Some of the things to watch out for are:

* Decision tasks not running
* Pending backlogs growing at an unusual rate
* Services not reporting aliveness checks
* API call response times/statuses

To monitor these queries and get alerted, a combination of services will be used. First,
a service will be implemented that will query Influx and send pulse messages when abnormalities
are detected.  Once the pulse messages are being published, a service can receive those
and act upon them.  In this phase that will be handled by a bot that can post informational
messages within a channel.

#### Phase 2 (Q4/2015) - Services Monitoring

TaskCluster currently has a [status page](http://status.taskcluster.net/) that is
useful for getting a high level overview of the health of various platform services
we deploy in Heroku.  Unfortunately this does not get a clear picture of where a problem
might reside.

In Q4 work will be done to enhance this status page to include an overall TaskCluster
status based on some heuristics of all TaskCluster services, as well as services they depend on.

Not only will this give a clearer picture of the overall health, but also will allow
one to see the individual services that might be degrading TaskCluster as a whole.

The hopes of making these changes will be to inform people to issues with various services
as well as empower them to know where they should direct efforts to resolve it.

#### Phase 3 (TBD) - Log Alerting

This phase is an extension to things we have already been doing with Papertrail.  Some of our
components follow the convention of prefixing log statements with "[alert-operator]" for events
that are exceptional and should be reported.  In this phase the way that we log these should make
use of a standard logging library used across all components. The events that we find useful
should be configured and documented for those to discover the various events we are concerned about.

Also in this phase alternative logging vendors should be evaluated.  One of the downsides of Papertrail
currently is that it cannot parse the actual messages we are sending and allow us to alert
based on information within the message (such as alert when a value logged is > N).

#### Phase 4 (TBD) - Task Alerting

Our tasks can get very complex with the series of operations that they performed.  Within tasks,
outside of just building a product, the task is responsible for pulling down source code, configuring the environment,
and updating various other services.  Sometimes it can be useful for these tasks to alert based on some situations.  One
case that comes to mind is when there are complications communicating with different VCS systems that could alert
us of a larger problem that is growing.  Enabling tasks to provide a stream of this information that is aggregating somewhere and
alerts based on that information can be very informative and powerful.

#### Phase 5 (TBD) - Self healing TaskCluster

There are times where components of TaskCluster need to be cycled, destroyed, or scaled based.  An interesting
idea is for TaskCluster to be able to detect these situations, or some kind of system monitoring TaskCluster,
and respond to these changes.  This work still needs some thought and requirements drawn up, but the idea could
solve some of the headaches that result from Heroku apps that do not auto restart, or apps that need to be scaled temporarily
based on demand.

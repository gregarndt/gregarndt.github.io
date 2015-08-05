---
layout: post
title: Taskcluster Overview
summary: Taskcluster is a distributed task platform used to power builds/tests within Mozilla.
categories: taskcluster
---

[Taskcluster](http://docs.taskcluster.net/) is a distributed task platform that was originally created to support
the build and test infrastructure for Firefox OS.  We've come a long way since the original
incarnation and have many more hurdles to overcome, but Taskcluster has been running production
Firefox OS jobs since the beginner of the year and there are plans to port Firefox desktop builds over soon
thanks to the awesome work of the Mozilla release engineering team.

In future posts I will go over the specifics of the different pieces of Taskcluster, but for now,
a brief overview will suffice.

At its core, Taskcluster as a whole is unaware of the tasks that are running, and just provides a means
of scheduling jobs and having workers run them.  These tasks are either submitted to a
[scheduler](http://docs.taskcluster.net/scheduler/) as a task graph, or as individual task submitted
directly to the [queue](http://docs.taskcluster.net/queue).

The scheduler is responsible for scheduling tasks within a task graph.  It will manage the task
dependencies and ensure that tasks are only scheduled when the tasks they depend on are completed
successfully.  These tasks are created within the queue as if submitted the individual task
independently.  The scheduler is solely responsible for managing tasks within a task graph.

The queue is the hub of creating and managing the task life cycle.  This includes defining,
scheduling, resolving, as well as creating artifacts.  When the queue is contacted by clients
and tasks are interacted upon, messages are posted to [Pulse](https://pulse.mozilla.org/), which is
an AMQP exchange.  This allows downstream consumers to be notified of task state and perform additional
actions depending on how the task changed.

Once scheduled, these tasks are placed within queues within Azure that workers poll frequently.
When a pending task is available for a worker, the worker will call the queue to claim the task and begin work.  After
all work has been completed (either successfully or not), the worker will resolve the task within the queue.

Are these workers just machines listening for work all the time?

No! That's the beautify of it.  We heavily use AWS EC2 spot instances for working on tasks.  These workers
are created by the Taskcluster [provisioner](http://docs.taskcluster.net/aws-provisioner/) depending
on the amount of worker pending and the settings for different worker pools (we call them worker types).

In short, if there is work, workers will be provisioned and provisioned workers that do not have any
more work will apply some logic for when to shutdown to minimize the cost.  Taskcluster tries to minimize
the idleness of workers, unlike machines that are always on regardless if they have work or not.

This is the 10,000 ft overview, but should give you an idea of the basics of Taskcluster.  In future posts
I'll go over the components in more detail.  Feel free to read the docs or come chat with us in #taskcluster
on the Mozilla IRC network.

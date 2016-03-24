---
layout: post
title:  Birth of a new worker
summary: A new cross platform TaskCluster worker is under development allowing multiple engine and plugin support.
categories: taskcluster
---

One of the first projects I worked on when I joined Mozilla
was to learn Node.js and help with a crazy thing called docker-worker.  This worker
has become the de facto platform for Linux tasks for over a year now. It has come
a long way since we first began working on it, but over time it has shown its
age and as we bring on other workers into the TaskCluster ecosystem the time has come
to reevaluate our direction for the workers.

In this past year docker-worker went into production and a Windows based worker was
being developed.  Hard work is happening from multiple teams to start converting our
existing Buildbot related Windows jobs to TaskCluster using this Windows based worker.

While work continued on both the Linux docker and Windows based workers, it became clear that they started to
fall out of parity with each other and made it confusing to those wanting to use either. Supporting
two distinct workers was going to become a challenge as well.

Both of these workers will continue to run tasks for the foreseeable future but work has begun
on a new worker, taskcluster-worker.  This new worker will adhere to a set of goals
based on the knowledge we now have of running production workers across platforms and involve
the entire team throughout the process.

### Cross-platform

taskcluster-worker must have the ability to be run on multiple operating systems
and should use a language which makes this possible and easier.  The team has already worked
on other services within TaskCluster that use Go, including the generic Windows worker,
so moving towards Go felt like a natural choice.

### Shared Functionality Abstraction

One of the things that's clear when writing workers is that there is a considerable amount
of shared functionality (host management, task claim/resolution, configuration management)
between all of them. This functionality will be incorporated into the worker in an
engine/plugin agnostic way so that engine/plugin writers can focus on that particular
piece rather than reimplementing the same functionality each time for a unique worker
experience.

### Multiple Engine Support

Engines should be swappable within the worker and loaded based on configuration settings.  These engines
will define the configuration and task payload data that is necessary to run a targeted task.

In future posts we'll go into how to write an engine, but for now understand that engines are given a contract
with the worker that they must adhere to.  This [interface](https://github.com/taskcluster/taskcluster-worker/blob/ea81ad1e3f3de1c876fc3b1ebeda298a60dc7af0/engines/engine.go#L35)
is documented in the engines package of the worker and defines the methods that all engines
must either implement or raise an error explaining that the feature is not supported.

These engines will provide methods that are used to build a sandboxed task execution environment (think docker container or isolated process),
execute this sandbox, process results, and allow plugins to hook into this environment and manipulate it.

Engines could be as simple as taking a string in the payload and writing "Hello, {string}" to the task log, or
spinning up a virtual machine environment to run the task in.  It's entirely up to the engine on what inputs
it accepts and how a task is executed.

### Plugin Architecture

While the worker will support only one running engine at a time, it will support 0 or more plugins per task execution.  These plugins can
be written independently of the worker and incorporated in at build time.

At each stage of the task life cycle, a [method](https://github.com/taskcluster/taskcluster-worker/blob/ea81ad1e3f3de1c876fc3b1ebeda298a60dc7af0/plugins/plugin.go#L72)
will be called on every loaded task plugin and allow the plugin to provide additional services and hook into the task
execution environment.

Some examples of plugins would be a stats collector parsing task logs, an artifact handler that
will parse the task payload and upload each specified artifact upon task completion, or provide
https access to the log of the running task (live logging).

These plugins can be written in a general way to work across engines allowing them to be reused
for all engines without duplicating functionality and logic.

### What's Next?
While it's sad to see docker-worker
deprecated in the future, it has served its purpose and I'm excited to see this new direction
for taskcluster-worker.  Active development will continue this year on Linux, Windows, and OS X
so that we have multiple platform support while migrating tasks.

Take a look at the [taskcluster-worker repo](https://github.com/taskcluster/taskcluster-worker) and the growing
documentation on [godoc](https://godoc.org/github.com/taskcluster/taskcluster-worker).

Come by and say hi in #taskcluster on the Mozilla IRC network.


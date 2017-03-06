---
layout: post
title: February Highlights
summary: February might be shorter than other months, but it wasn't short on new things landing.
categories: taskcluster
---

February might be shorter than other months, but it wasn't short on new things landing.

#### Periodic Tasks

One of the benefits of a task execution platform is the ability to schedule tasks to run
outside of the typical commit/build/test workflow.  This can be done for rebuilding toolchains
frequently to receive package updates or ship a product on a predefined interval.

TaskCluster's [hook service](https://tools.taskcluster.net/hooks/) allows a developer to define
a task and a schedule to run it on.  However, this configuration is stored within the hooks manager,
requires the developer to have the appropriate permissions within TaskCluster, and can create a secondary
workflow for defining tasks to run on a repository.

Today, similar to how repositories could define a .taskcluster.yml as an entry point into scheduling tasks on push, support for a [.cron.yml](https://dxr.mozilla.org/mozilla-central/source/.cron.yml)
file has now been added to our Gecko branches that allows one to define tasks to run as well as a schedule.

While this work is very Gecko specific, and makes use of existing taskgraph generation logic,
the concept can very much be reapplied to a repository hooked into TaskCluster, which leads me to the next
amazing announcement.

You can read more about it [here](https://gecko.readthedocs.io/en/latest/taskcluster/taskcluster/cron.html).

#### Github Integration Improvements

For some time now, github repositories under the mozilla org could define a [.taskcluster.yml](https://github.com/taskcluster/taskcluster-github/blob/master/.taskcluster.yml)
file in the root of the repository to schedule tasks for pushes and pull requests.  Our integration service
listens for events under the mozilla org using a webhook and then schedule the appropriate tasks based on that file.

We are fortunate, and thrilled!, to be able to participate in the [Outreachy](https://www.gnome.org/outreachy/) program and during this session, [Irene](https://github.com/owlishDeveloper) worked hard
with the team to deliver some awesome improvements to our github integration as well as supporting the new first class github support for [integrations](https://developer.github.com/early-access/integrations/).

To read more about this, including checking out the new quickstart tool, head over to Dustin's [blog](http://code.v.igoro.us/posts/2017/02/taskcluster-github-improvements.html)

#### Fast-azure blob storage

We also have [Elena](https://github.com/elenasolomon) participating in the current Outreachy session
improving our [fast-azure-storage](https://github.com/taskcluster/fast-azure-storage) library and implementing it into some of our services.  With this new version, the library supports
the full blob storage API, allows specifying a schema for a blog entity, and adds the ability to use Shared Access Secrets for authentication.

I'm happy to report that Elena has shipped version 1.0!  In addition to shipping the library, Elena has integrated
a new [API endpoint](https://github.com/taskcluster/taskcluster-auth/pull/94) into taskcluster-auth that allows a shared access signature to
be returned for access to a blob storage container.

#### Action Tasks

Once tasks are scheduled for a push, a developer might find them in the position of wanting to reschedule that same task, maybe even many times.  This is simple
using Treeherder's retrigger functionality, but it does not give the flexibility of specifying alternate parameters for the task and also relies on task graph
duplication logic found in another integration service.

Rather than specifying actions within Treeherder or other tools that live outside of the in-tree task scheduling logic, a specification has been
developed that describes how decision tasks can define actions during taskgraph generation, which tools, such as Treeherder, can use to assist a developer in performing actions
on that task.

You can read more about it [here](http://gecko.readthedocs.io/en/latest/taskcluster/taskcluster/in-tree-actions.html) and check out the example [hello-world.py](https://dxr.mozilla.org/mozilla-central/source/taskcluster/actions/hello-action.py) action task.

#### Neutrino

Beginning last year, we have been making incremental improvements to our front-end properties, such as our tools and documentation sites.  While working on that
it was clear that we needed better build tools and [Neo](https://blog.eliperelman.com/neo-8bf3d7325f7#.t7y6mp4dc) was born.  This got us far, but we wanted to make
use of webpack and because TaskCluster's core platform is built upon Node.js, we would like a way to build our applications with the same toolchain as our front-end.

From Neo came [Neutrino](https://hacks.mozilla.org/2017/02/using-neutrino-for-modern-javascript-development/), a tool that brings the best parts of modern
JavaScript toolchains, including webpack, into the hands of the developer to easily build web and Node.js applications using shareable presets.

Check out the project on [github](https://github.com/mozilla-neutrino/neutrino-dev) and help us make it even better!

<hr>
As always you can find us in #taskcluster on irc.mozilla.org, or through one of our many projects on [github](https://www.github.com/taskcluster).

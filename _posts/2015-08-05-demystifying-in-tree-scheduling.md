---
layout: post
title:  Demystifying in-tree TaskCluster scheduling
summary:
tags: [taskcluster]
---

Since earlier this year Firefox OS tasks have been defined
[in-tree](https://dxr.mozilla.org/mozilla-central/source/testing/taskcluster/tasks)
and scheduled within TaskCluster.  Things are in progress for porting Android and Firefox Desktop
builds as well.

There are a few interactions that need to take place when scheduling tasks and reporting
them to treeherder.  These interactions typically are handled by an integration component
named [mozilla-taskcluster](https://github.com/taskcluster/mozilla-taskcluster).

#### mozilla-taskcluster

tl;dr mozilla-taskcluster makes sure those nicely colored letters appear for
each taskcluster task scheduled for each push on treeherder.

mozilla-taskcluster monitors the push log every few seconds for changes for a given set
of gecko repositories and will create a task graph when new pushes are detected.  The [initial
task](https://dxr.mozilla.org/mozilla-central/source/testing/taskcluster/tasks/decision/branch.yml)
within this graph is typically referred to as the decision task. Its
responsibility is to decide what tasks should be added to the task graph for a given
branch/project/repository (the names are used interchangeably in many places) using some
in-tree logic.

mozilla-taskcluster is responsible for creating the resultset within Treeherder,
creating the task graph with decision task, and also responsible for posting job collections
to Treeherder when tasks complete.

#### mach taskcluster-graph

The heart of deciding what tasks will be included in the graph for a push is the ['mach taskcluster-graph'](https://dxr.mozilla.org/mozilla-central/source/testing/taskcluster/mach_commands.py#206)
target.  This target when called will read [in-tree branch specific configurations](https://dxr.mozilla.org/mozilla-central/source/testing/taskcluster/tasks/branches)
and determine what task definition files to parse and compose into a json blob that will be used
to extend the taskcluster graph.

The decision for what jobs to include is based on if it was a Try push, or a push to any other
branch.  For Try pushes, the commit message will be [parsed](https://dxr.mozilla.org/mozilla-central/rev/5cf4d2f7f2f2b3df2f1edd31b8bdce7882f3875c/testing/taskcluster/taskcluster_graph/commit_parser.py#202)
and used for determining which tasks to run.

It's worth noting that this target only prints out json.  It's the responsibility of
the consumer of this to extend the task graph or use it to create an entirely new graph.

In TaskCluster, once the json is created, the worker used to complete this task has features
in place to automatically extend the original task graph with the contents of this
json blob as long as the original task graph has the [scopes](http://docs.taskcluster.net/presentations/scopes/#)
encompassing all scopes used within those additional tasks.

#### In-tree branch configurations (job_flags.yml)

The in-tree scheduling for a given branch is specified in a job\_flags.yml located at
`<gecko>/testing/taskcluster/tasks/branches/<branch>/job_flags.yml`. This is what
the mach target will use for determining what should be scheduled (along with some
logic within the mach target itself).

These configurations are composed of keys that define the build/tests that are
enabled for that given branch as well as their relationships.

Taking a look at a snippet of sample branch config, you can see that there are some familiar
keys under builds and tests.  These might remind you of [try](https://wiki.mozilla.org/Build:TryChooser)
flags...and that's because they are!  But you might ask yourself why we are using try
flags for a branch that is not Try.  Simple, it's a (kind of) well understood syntax for specifying
builds and tests that should be run, so we treat every branch configuration the same and
reuse Try flags within the configurations.  Commit messages for Try pushes are parsed
by `mach taskcluster-graph`, and all other branches are defaulted to using the try message `try: -b do -p all -u all`.

After parsing either the try commit message, or the default 'all' message, all other logic is the same
for composing the task graph json.

Example configuration:

```yaml
---
builds:
  linux64_gecko:
    platforms:
      - b2g
    types:
      opt:
        task: tasks/builds/b2g_desktop_opt.yml
      debug:
        task: tasks/builds/b2g_desktop_debug.yml
  linux64-mulet:
    platforms:
      - Mulet Linux
    types:
      opt:
        task: tasks/builds/mulet_linux.yml
tests:
  gaia-build:
    allowed_build_tasks:
      tasks/builds/b2g_desktop_opt.yml:
        task: tasks/tests/b2g_build_test.yml
      tasks/builds/mulet_linux.yml:
        task: tasks/tests/mulet_build_test.yml
```

The flags under builds and tests can be specified individually in a try commit message,
such as:

```
try -b o -p linux64_gecko -u gaia-build
```
or included if 'all' is used. Tasks that
are included in 'all' are specified in the [base\_job\_flags.yml](https://dxr.mozilla.org/mozilla-central/source/testing/taskcluster/tasks/branches/base_job_flags.yml)
file.

####Builds

######builds.platforms

This value is for when restricting test suites to a given platform.  For example,
this will cause the gaia-build tests to only run for the Mulet Linux build, and not b2g desktop:

```
try: -b do -p all -u gaia-build[Mulet Linux]
```

######builds.\<build flag\>.types

All builds have at least an 'opt' build as that is what will be used by default. `opt.task`
defines where to find the task definition for that particular build. For 'try' this
will used for `try: -b [d|o|do]`. All other branches will use '-b do'.

####Tests

Tests are broken up into their 'try' flags and define not only the task definition to use
(tests.\<flag\>.allowed\_build\_tasks.\<build task file\>.task) but also what builds
that test flag applies to (tests.\<flag\>.allowed\_build\_tasks.\<build task file\>)

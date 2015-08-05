---
layout: post
title:  How can I add my cool new task to TaskCluster?
summary: Build and test tasks can be added in-tree to be run per push.
categories: taskcluster
---

Since the beginning of the year we have been rolling out tasks running in TaskCluster
that build and test Firefox OS related components, such as b2g desktop, emulators, and
device images.  The tasks are defined [in tree](https://dxr.mozilla.org/mozilla-central/source/testing/taskcluster/tasks/)
and run on each push.

That's great for the tasks that already are defined in there, but what about getting
other things build and tested within TaskCluster?  It's one of the more common questions
we receive in #taskcluster (come visit us by the way!) and so hopefully this will
remove some of the mystery around defining in tree tasks.  These tasks can either be run
when passing in the right flags in a [Try](https://wiki.mozilla.org/ReleaseEngineering/TryServer)
push or when pushes happen on one of the branches.

Diving in, let's first look at how to define a task.  All tasks reside under
<gecko>/testing/taskcluster/tasks/.  In the current implementation we treat things
as either a build or a test.  Builds can be defined and have zero or more tests depend
on the build artifact for testing.

The easiest task to create is a build because it requires no other dependencies to run.  Within
<gecko>/testing/taskcluster/tasks/builds/ we will create a yaml file such as the following and save
it as test_build.yml.

#### Mach taskcluster-graph
The mach target [taskcluster-graph](https://dxr.mozilla.org/mozilla-central/source/testing/taskcluster/mach_commands.py#206)
is the orchestrator of much of the in-tree logic involved with kicking off a TaskCluster
graph

```yaml
---
taskId: {{build_slugid}}

task:
  created: "{{now}}"
  deadline: '{{#from_now}}24 hours{{/from_now}}'
  metadata:
    name: 'Test Task'
    source: 'http://test.task.com'
    owner: 'mozilla-taskcluster-maintenance@mozilla.com'
    description: 'Test Build'

  tags:
    createdForUser: {{owner}}

  workerType: b2gbuild-desktop
  schedulerId: task-graph-scheduler
  provisionerId: 'aws-provisioner-v1'

  payload:
    image: 'ubuntu:14.04'
    maxRunTime: 120

    command:
      - /bin/bash
      - -c
      - >
        mkdir -p /home/worker/artifacts &&
        cd /home/worker/artifacts &&
        touch target.zip &&
        touch test_packages.json &&
        ls -lah .

    artifacts:
      'public/build':
        type: directory
        path: '/home/worker/artifacts/'
        expires: '{{#from_now}}1 year{{/from_now}}'

  extra:
    locations:
      build: 'public/build/target.zip'
      test_packages: 'public/build/test_packages.json'
    treeherder:
      symbol: 'TEST-BUILD'
      groupSymbol: "?"
      groupName: Submitted by taskcluster
      machine:
        platform: b2g-linux64
$```


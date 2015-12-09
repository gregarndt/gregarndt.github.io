---
layout: post
title:  Feature Announcement: Build Images on Push
summary: Building in-tree images on push can happen automatically.
categories: taskcluster
---

When looking at a [task definition](https://queue.taskcluster.net/v1/task/AWx91r7eTKyY2YS0PofZ3Q)
it's not shocking to find some similarities between the definition and some
of the inputs for starting a container with the docker remote api.

At the core of our [docker based worker](http://www.github.com/taskcluster/docker-worker) is
a wrapper around the docker engine that allows tasks to be executed in
an isolated reproducible environment.  Tasks just need to specify an image to use,
the command to run, and any possible environment variables and the worker will execute
those in much the same way as running `docker run [...]`.

TaskCluster has benefited by the hard work the Docker team has put towards their
products, especially the docker registry which hosts most of our images that we
use in production.  In the past 30 days 1.4 million tasks have been completed
in this environment which has resulted in 2.8 million containers being started and over
270,000 images pulled from the registry.

Of the images that run within TaskCluster, most are defined
[in our gecko repository](http://hg.mozilla.org/mozilla-central/file/tip/testing/docker).

Changing these images requires editing in-tree image configurations, bumping version
numbers in a specific file, running a convenience wrapper around `docker build`,
and pushing to the docker registry.  These steps must be completed before pushing to any branch that triggers
tasks that use those images.  On more than one occasion image tags have either been overwritten
resulting in images being used that were not intended, or images not being pushed prior
to pushing commits that required those versions of the image, resulting in a failed
task.

For me, one of the larger disruptive disadvantages of this workflow is that it
requires pushing the images to a registry.  Anyone that has pushed an image knows that
even on a decent upstream connection, this takes time.  Even more painful when you
want to test these images on Try and must push multiple revisions.

Looking at the current state of handling images, it was clear that the workflow needed
to become simpler.  At the same time we must also try to tame the problem of the elphant

When reviewing our current state of handling docker images, it became clear, there were two problems
we needed to solve.  Simplifying the workflow of editing and push images and not
relying on an external service for production jobs.

### Hosting internal docker registry

The most immediate idea that came to mind when trying to solve the external resource problem
was to host our own docker registry which we could control.  However, this did not solve
the problem of making the docker image workflow easier, the problem just shifted from
a registry hosted by docker to one that we could control.  Also, it made a huge assumption
that we would be able to ensure reliable uptime and authentication management of this service.

The TaskCluster team does have a philosophy of offloading systems management to services
that designed for that, so we can get back to solving problems that interest us more.  Moving to
our own hosted registry did not seem to fit with this model nor solve our initial issues with
the workflow.

### Task Artifacts

For those that have not looked much at TaskCluster tasks, all tasks have the ability
to upload public or private files (artifacts) that can be referenced after the task has completed.

These files are uploaded to s3 and access control is provided by our TaskCluster auth and
api endpoints.  Using a system like this has been battle tested within production tasks as
our build tasks produce test packages and build artifacts that are then used by downstream tasks.

### Docker Image Tarball

Docker has the ability to save docker images and metadata as tarballs, which can
then be transported to other hosts and loaded.  The one downside to moving images
around like this is that it does not take advantage of the layer caching system
of docker.  Using this within AWS in CI is an acceptable trade-off for the simplicity
that this solution provides.


### Task Image Artifacts

Marrying these two solutions seemed like a great fit for solving some of our problems.
Storing the image tarballs within s3 as a task artifact is a proven concept, and
relying on s3 as an external resource has been an acceptable practice for production
tasks.

Images can now be built using the `dind` feature of our docker based workers by enabling the flag
`task.payload.feature.dind`.  When this feature is enabled and used by an image that
contains the docker client, images can be built and saved without needing a docker
daemon within the task container itself.  These images can then be put wherever
one would like, including saving it as an artifact.

TODO: Add task example for building ubuntu that has some kind of package available

Using this image for another task requires specifying the task ID of the image along
with the artifact path to find the image.

Here is one such example:

```js
payload: {
    image: {
        type: 'task-image',
        taskId: '1234',
        path: 'public/image.tar'
    }
}
```

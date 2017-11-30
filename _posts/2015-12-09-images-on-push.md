---
layout: post
title:  Build Images on Push
summary: Building in-tree images on push can happen automatically.
categories: taskcluster
tags: [taskcluster]
---

When looking at a [docker task definition](https://queue.taskcluster.net/v1/task/AWx91r7eTKyY2YS0PofZ3Q)
it's not shocking to find some similarities between the definition and some
of the inputs for starting a container with the docker remote api.

At the core of our [docker based worker](http://www.github.com/taskcluster/docker-worker) is
a wrapper around the docker engine that allows tasks to be executed in
an isolated reproducible environment.  Tasks just need to specify an image to use,
the command to run, and any possible environment variables.  The worker will execute
those in much the same way as running `docker run [...]`.

TaskCluster has benefited by the hard work the Docker team has put towards their
products, especially the docker registry which hosts most of our images that we
use in production.  In the past 30 days 1.4 million tasks have been completed
in this environment which has resulted in over 2.8 million containers being started and around
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
to become simpler.

When reviewing our current state of handling docker images there were two problems
we needed to solve.  Simplifying the workflow of editing and pushing images and not
relying on an external service to host images for production jobs.

### Task Artifacts

All tasks have the ability
to upload public or private files (artifacts) that can be referenced after the task has completed.

These files are uploaded to s3 and access control is provided by our TaskCluster auth component.  Using a system like this for production tasks
has been battle tested and seemed like a great fit for storing docker images.

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
daemon or privileged capabilities within the task container itself.  These images can then be put wherever
one would like, including saving it as an artifact.

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

### Example

The workflow for on-push image building will typically be:

* edit image context
* commit changes and push to vcs
* Decision task triggers image building task if any task requires that image

Note: A task must specify the in-tree image that it uses and this image should
be treated as a task artifact.  Consult the [in-tree documentation](https://hg.mozilla.org/mozilla-central/raw-file/a8965ae93c5d098a4f91ad9da72150bb43df07a7/testing/docker/README.md)
about this feature for more information.

Looking at one of our [existing](https://hg.mozilla.org/mozilla-central/raw-file/a8965ae93c5d098a4f91ad9da72150bb43df07a7/testing/docker/builder/Dockerfile) dockerfiles for a production image, we identify
a package that needs to be installed but as a developer I would like to test this out on
try before using it in production.

Using this new workflow, I can take the dockerfile and add anywhere to it the installation
of the package like so:

```
RUN apt-get update && apt-get install -y wget
```

Once the change has been made, I commit it, push to try with flags that will trigger
the task that requires that image and watch the image being built automatically.

You can see this job on treeherder as a "taskcluster-images" job.  In the snippet below
from treeherder, you can see that a "taskcluster-images" job was scheduled ('I' symbol).  Here
the b2g desktop tasks depend on the image building task to complete and were scheduled once
that task completed.

![Alt Text](/images/dependent_tasks.png)



I hope everyone that has ever needed to build an in-tree image gets a chance to try this out.  Come
chat in #taskcluster on the Mozilla IRC network if you have questions or want to try this out.

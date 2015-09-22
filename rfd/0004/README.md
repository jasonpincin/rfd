---
authors: Todd Whiteman <todd.whiteman@joyent.com>
state: draft
---

# RFD 3 Docker Build Implementation For Triton

# Introduction

This document covers how we (Joyent) will implement docker build in Triton.
Given that docker build is mostly spec'd, this covers the high level
implementation details.

For an overview of docker build and image, see:

 * https://docs.docker.com/reference/commandline/build/
 * https://docs.docker.com/reference/builder/
 * https://github.com/docker/docker/blob/master/image/spec/v1.md

# Overview

To build up layers of 'images' using 'containers' as stepping points.

The docker client passes the build parameters and a context (tar) file to
sdc-docker. Sdc-docker then creates a container and passes-on the build context
to the Compute Node (CN) to do the building. The CN will process the Dockerfile
commands and generate the required image. After a successful build the image(s)
will be imported into imgapi - simple huh!?

# Compute Node

- The build context is sent to the CN (cn-agent) from sdc-docker - CN saves it.
- CN sets up a socket event stream back to sdc-docker
  - build output (and status) will be send back on this stream
- Extracts the Dockerfile from the build context tar file
- Parses Dockerfile and starts processing commands (from, copy, cmd, ...)
  - There are two styles of commands - a modify config or a modify container
  - Each successful command creates a new image layer - really it's either a
    modification of the config file or a modification to the container
- The first Dockerfile command is always a FROM command - which will be used to
  set the base image for the container.
  - This base image may need to be pulled into the system
    - This requires an admin endpoint on sdc-docker to do the docker pull
    - Once pulled, base images get installed locally using imgadm get
- After all commands run successfully, exports the config/container to imgapi
  - the export needs to go through sdc-docker end-point?

# Image Config

- The image config starts from a base image template (JSON)
- config gets stored as a file during build
  - /zones/$uuid/config/docker-build/$imgUuid.config
- Commands (like maintainer, label, ...) adjust the json config and save it
  - Each modification creates a new imgUuid.config file
  - After a build there can be many config files in this directory
- Modify container commands will utilize this config to know the current state
  of the container (workdir, user, env, ...)
- When the build completes successfully - all images (layers) are exported
  to imgapi
  - when all layers are uploaded successfully, they can be made active in imgapi

## Commands

These commands can occur multiple times (except MAINTAINER), so a current config
state is needed to supply to the modify container commands.

- MAINTAINER
- CMD
- LABEL
- EXPOSE
- ENV - metadata for commands and resulting image
- ENTRYPOINT
- VOLUME
- USER - metadata for container commands
- WORKDIR - metadata for container commands


# Container

- The container is initially created in a special "Created" state
  - it can be seen via 'docker ps' if the layer modification is successful
- The RUN and ONBUILD commands will require a "running" vm
  - the ADD command for a remote url may also require a running vm
- After successful modification, a new snapshot (layer) is created of the
  container, along with a matching config file.

## Commands

These commands can occur multiple times (except FROM).

- FROM
  - Requires imgapi - pull in image (and dependents - streams back results)
   - Trent tells me this pull should go through sdc-docker
  - 'FROM scratch' is a noop - an empty container - and doesn't create a new
    imgUuid.config
- RUN
  - all output from run is streamed back to the client
  - two forms, explicit executable+args or shell+args
- COPY
  - add files/directories to container - files reside in the build context
- ADD
  - like COPY, but can specify remote URL
  - remote URL will need to be downloaded (not part of the context)
- ONBUILD
  - special, only runs after all other commands complete successfuly
  - acts like a second docker file (has subcommands like RUN, COPY, ...)

# Networking

The created build container will be configured to use the users default network
and thus allow (if so configured) external network access for things like RUN
commands.

Some commands (like ADD) can reference a remote URL, which will need to be
downloaded to the container. As the CNAPI zone does not have external access,
we would need to download inside of the container or have a download service
in which CNAPI could request from.

# Dockerfile Quirks

- Docker caches results of a previous build and continues on from the
  modified point (unless 'docker build --no-cache' is used).
- By default docker build failures leave around the failed build images.
- The '--rm' and '--forcerm' build options can be used to leave/clear old
  containers - and these containers are visible throughg 'docker images -a'.


# Alternatives

Why not Use docker/docker itself to build the images?

We could re-use the existing docker code (go-lang) to build and export the
image:

  - requires an imgapi docker registry endpoint and push the built image to it
    - once pushed to imgapi, it would be available (like it had been pulled)
  - when I checked to see if we could re-use docker/docker for building
    - seems too many missing pieces (in lx zones) to support this
      - lxc, namespaces, cgroups, deep-kernel tie in
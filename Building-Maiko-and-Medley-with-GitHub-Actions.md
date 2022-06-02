_Note: 👀  still in preparation. Currently we have
[buildLoadup](https://github.com/Interlisp/medley/blob/master/.github/workflows/buildLoadup.yml) and
[buildDocker](https://github.com/Interlisp/medley/blob/master/.github/workflows/buildDocker.yml)
as manually invoked from [Interlisp/medley Actions](https://github.com/Interlisp/medley/actions)

***

##### Table of Contents
[Introduction](#introduction)  
[GitHub Build Process Overview](#github-build-process-overview)  
[GitHub Actions](#github-actions)  
[Dockerfiles](#dockerfiles)  
[Maiko](#maiko)  
[Medley](#medley)  
[Local Testing and Validation of Workflows](local-testing-and-validation-of-of-workflows)  
[References](#references)  

## Introduction

This document describes the GitHub workflow used by the Maiko and Medley repositories to build new Docker containers when commits are pushed to the master branch of either repository.

GitHub defines a workflow as:

>an automated process that you add to your repository. Workflows are made up of one or more jobs and can be scheduled or triggered by an event.  The workflow can by used to build, test, package, release, or deploy a project on GitHub.

Both repositories use the same essential workflow.

One workflow builds Maiko for the supported architectures, packages the executables into a Docker image which is then stored in DockerHub.   A second workflow constructs a Docker image that contains everything needed to run the Medley release of Interlisp and stores this image to DockerHub.

The resulting Interlisp Medley image can then be pulled from DockerHub giving the user an easy way of installing and starting to use Interlisp.

The remainder of this document describes the GitHub workflow process.  It describes the specific design of the workflow for Maiko and Medley.  The overall design of the GitHub workflow will first be described.  Next the configuration of the `yml` file used to encode the workflow is discussed.  Lastly, the `Dockerfile` that describes how to construct the Docker image that is described.

## GitHub Build Process Overview

The build process is driven by pushing updates to the master branch of the repository. This functionality can be extended to other branches, but, at present, the focus is on building Docker images for the master branch.  When a Pull Request is merged into the Master Branch, both the Maiko and Medley GitHub repositories are set up to initiate a workflow that builds a new Docker image representing the current state of the master branch.  The resulting image is stored in DockerHub and is publicly available. 

The code that defines the actions taken resides in the `build.yml` file in the `.github/actions` directory. The `yml` file specifies which branches to take actions on and the set of actions when to take when the branch is updated.

## GitHub Actions

GitHub actions are workflows that are performed when a specified event occurs.  In our case the event of interest is a commit being pushed to the master branch of either the Maiko or Medley repositories. 

A workflow consists of the following components:

- Event - the activity which triggers the workflow
- Jobs - one or more jobs perform the functionality of the workflow
- Steps - Jobs break down into one or more steps, each step representing a task within the job
- Actions - individual commands that compose steps.  The atomic elements of a workflow
- Runners - a server that executes jobs

Specifying a workflow requires identifying the trigger event and defining the jobs which compose the workflow.  This information is encoded in a `yml` file.

The Maiko `buildDocker.yml` file is presented below with commentary on the various steps that make up the job.  The Medley `yml` file is essentially the same.  

```yaml
# The name of the workflow
name:  'Build Maiko image'

# The action that initiates this workflow
on:
  push:
    branches:
      - master
# Multiple branches can be named.  A continuous build system may
# want to build every branch on a push to help ensure the master
# branch doesn't break on a merge

# define the jobs which make up the workflow
jobs:
  # docker: is the name we assigned to the job
  docker:
    # runs-on: specifies that the job use a Ubuntu Linux runner
    runs-on: ubuntu-latest
      # steps: identifies the start of the job steps
      steps:
        # A name to identify the step
        - name: Checkout
        # The action(s) used by this step.  In this case checkout V2
        # actions/checkout@v2 is a GitHub action.  This represents an
        # action written by another party that we want to use.  The
        # action is defined at:  https://github.com/actions/checkout
          uses: actions/checkout@v2

          # Setup some environment variables
          # These will be used later on to help in the workflow or
          # to help identify the resulting Docker image
        - name: Prepare
          id: prep
          run: |
            # Name of the Docker Image. 
            DOCKER_IMAGE=interlisp/${GITHUB_REPOSITORY#*/}
            VERSION=latest
            SHORTREF=${GITHUB_SHA::8}

            # Do we want to use tags and or versions 
            # If this is git tag, use the tag name as a docker tag
            if [[ $GITHUB_REF == refs/tags/* ]]; then
              VERSION=${GITHUB_REF#refs/tags/v}
            fi
            TAGS="${DOCKER_IMAGE}:${VERSION},${DOCKER_IMAGE}:${SHORTREF}"

            # If the VERSION looks like a version number, assume that
            # this is the most recent version of the image and also
            # tag it 'latest'.
            if [[ $VERSION =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
              TAGS="$TAGS,${DOCKER_IMAGE}:latest"
            fi

            # Set output parameters.
            echo ::set-output name=tags::${TAGS}
            echo ::set-output name=docker_image::${DOCKER_IMAGE}
            echo ::set-output name=build_time::$(date -u +'%Y-%m-%dT%H:%M:%SZ')

        # Setup the Docker Machine Emulation environment.  
        # This provides the infrastructure that allows us to build Docker
        # images that support multiple environments
        #  See https://github.com/docker/setup-qemu-action for more information
        - name: Set up QEMU
          uses: docker/setup-qemu-action@master
          with:
            platforms: all

        # Setup the Docker Buildx function
        # Docker Buildx is the tool that actually drives building a 
        # Docker image that supports multiple platforms
        #  See https://github.com/docker/setup-buildx-action for more information
        - name: Set up Docker Buildx
          id: buildx
          uses: docker/setup-buildx-action@master

        # Login into DockerHub - required to store the created image
        #  secrets.DOCKER_USERNAME and secrets.DOCKER_PASSWORD are 
        #  stored in the Interlisp GitHub repositories.
        - name: Login to DockerHub
          if: github.event_name != 'pull_request'
          uses: docker/login-action@v1
          with:
            username: ${{ secrets.DOCKER_USERNAME }}
            password: ${{ secrets.DOCKER_PASSWORD }}

        # Start the Docker Build using the Dockerfile in the repository we
        # checked out.  When complete push the Docker image to DockerHub
        #  See https://github.com/docker/build-push-action
        - name: Build
          uses: docker/build-push-action@v2
          with:
            builder: ${{ steps.buildx.outputs.name }}
            # Pass in the build time from the prep action above
            build-args: BUILD_DATE=${{ steps.prep.outputs.build_time }}
            context: .
            file: ./Dockerfile
            # Platforms - Sepecify the platforms to include in the build
            #  linux/amd64 -- Standard x86_64
            #  linux/arm64 -- Apple M1
            #  linux/arm/v7 -- Raspberry pi
            platforms: linux/amd64,linux/arm64,linux/arm/v7
            # Push the result to DockerHub
            push: true
            # tags to assign to the Docker image
            tags: ${{ steps.prep.outputs.tags }}
```
The notation for the GitHub action is dense.  But the essence is that it checks out the master branch when an update is pushed to it and builds a new Docker image.

The Medley and Maiko versions of the build file are essentially the same.  The differences in the build process are contained within the `Dockerfile`.  This file contains the instructions the build process uses to create the resulting Docker image.

If a workflow contains multiple jobs, the default action of the runner is to execute the jobs in parallel.

## Dockerfiles

A Dockerfile represents the set of commands that are needed to build an image.  The GitHub action described above creates the build environment required to execute the commands within the Dockerfile.

### Maiko

The primary purpose of the Maiko Dockerfile is to build the Maiko executable and provision the resulting Docker image with the tools required to run Medley.
 
The Dockerfile starts building the Maiko image from a Ubuntu image:

 ```bash
FROM ubuntu:focal
```

It then adds some labeling information to the image, information which helps identify the image and differentiate it from other Maiko image versions:

```bash
ARG BUILD_DATE
LABEL name="Maiko"
LABEL description="Virtual machine for Interlisp Medley"
LABEL url="https://github.com/Interlisp/maiko"
LABEL build-time=$BUILD_DATE
```

Next the tools needed to build Maiko are installed:

```bash
RUN apt-get update && apt-get install -y make clang libx11-dev gcc x11vnc xvfb
```

The tools required to build Maiko are:
 - Make
 - C compiler
 - libx11 libraries

*Verify which tools and libraries can be removed:*
 - *gcc*
 - *x11vnc*
 - *xvfg*

Once the image is prepared, the Maiko source files are copied in, any old executable files are removed and the Maiko engine is rebuild.

```bash
COPY --chown=nonroot:nonroot . /app/maiko
RUN rm -rf /app/maiko/linux*

WORKDIR /app/maiko/bin
RUN ./makeright x

RUN rm -rf /app/maiko/inc /app/maiko/include /app/maiko/src
```

The `makeright` script is used to build Maiko.  Once built, unneeded files are removed.

The completed Maiko image is then pushed to DockerHub.


### Medley

The Medley Dockerfile starts by verifying that the local Maiko image, if it exists, is the latest image.  If not, or if a Maiko image is not present, it pulls the latest Maiko image from DockerHub:

```bash
FROM interlisp/maiko:latest
```

Add in labeling information:

```bash
ARG BUILD_DATE
LABEL name="Medley"
LABEL description="The Medley Interlisp environment"
LABEL url="https://github.com/Interlisp/medley"
LABEL build-time=$BUILD_DATE
```

Install a VNC server:

```bash
RUN apt-get update && apt-get install -y tightvncserver
```

Expose port 5900, we use it for vnc:

```bash
EXPOSE 5900
```

Currently the Dockerfile construction takes a brute force approach to installing the needed `Medley` files, it does a bulk copy of everything into the `/app/medley` directory:

```bash
COPY . /app/medley
```

Set the working directory:

```bash
WORKDIR /app/medley
```

Create a user named __medley__ and set it to the current user.

```bash
RUN adduser --disabled-password --gecos "" medley
USER medley
```
Set up the entry point, defining the Medley image's start up actions.  In t his case we set the User to medley, startup vnc, add maiko to the path and startup medley.

```bash
ENTRYPOINT USER=medley Xvnc -geometry 1280x720 :0 & DISPLAY=:0 PATH="/app/maiko:$PATH" ./run-medley -full -g 1280x720 -sc 1280x720
```

The completed Docker image is then pushed to DockerHub and ready to be used.


## Local Testing and validation of workflows

## References

[Learn GitHub Actions](https://docs.github.com/en/actions/learn-github-actions)  
[Workflow Syntax](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions)  
[Checkout Version 2](https://github.com/actions/checkout)  
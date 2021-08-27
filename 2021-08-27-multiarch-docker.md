---
title: "Using Docker for Multiarch Images on Docker"
created_at: Fri Aug 27 2021 15:00:00 EDT
author: Montana Mendy
layout: post
permalink: 2021-08-27-multiarch-docker
category: news
excerpt_separator: <!-- more --> 
tags:
  - news
  - feature
  - infrastructure
  - community
---

![TCI-Graphics for AdsBlogs (3)](https://user-images.githubusercontent.com/20936398/131183262-6413e5cc-3616-42d2-b977-b2e3c155ad50.png)


When a client, e.g. Docker client, tries to pull an image, it must negotiate the details about what exactly to pull with the registry given the conditionals. Let's dive into multiarch builds using Docker in Travis.

<!-- more --> 

# Getting started

You can run a program compiled for Arm on 1`amd64` on a Linux machine (if specified in your `.travis.yml`), if it has [binfmt_misc](https://www.kernel.org/doc/html/v4.18/admin-guide/binfmt-misc.html) support. In simple terms, `binfmt` allows you to run a program without worrying about which architecture it was originally built for. A crucial thing you must remember is `binfmt_misc` is not a de facto dependency on Ubuntu, but don't fret, it's really simple to install it via Docker.

## Spinning up the container 

First, open up your `.travis.yml` set your `dist` to `focal` or `xenial`, there's some distros that `binfmt_misc` mount error out on. In your `.travis.yml` file let's add this line:

```yaml
before_install:
  - sudo docker run --privileged linuxkit/binfmt:v0.6
  ```

Next, we need to fetch the `buildkit`, so first we'll start the `buildkit` server as a container process, then we'll copy the `buildctl` binary which is the command line frontend for buildkit to our `/usr/bin` directory and then we'll set the `BUILDKIT_HOST` environment variable. Now head back to your `.travis.yml`, let's look at your `before_install` section again, and add the following: 

```yaml
before_install:
  - sudo docker run --privileged linuxkit/binfmt:v0.6
  - sudo docker run -d --privileged -p 1234:1234 --name buildkit moby/buildkit:latest --addr tcp://0.0.0.0:1234 --oci-worker-platform linux/amd64 --oci-worker-platform linux/armhf
  - sudo docker cp buildkit:/usr/bin/buildctl /usr/bin/
  - export BUILDKIT_HOST=tcp://0.0.0.0:1234
  ```

## Building the Docker Image 

Let's write the script to start building the images, in this example I'll be using Arm:


```Dockerfile
PLATFORM=arm 
DOCKERFILE_LOCATION="./Dockerfile.armhf"
DOCKER_USER="MontanaMendy"
DOCKER_IMAGE="MontanaMendy/server"
DOCKER_TAG="latest"

buildctl build --frontend dockerfile.v0 \
        --frontend-opt platform=linux/${PLATFORM} \
        --frontend-opt filename=./${DOCKERFILE_LOCATION} \
        --exporter image \
        --exporter-opt name=docker.io/${DOCKER_USER}/${IMAGE}:${TAG}-${PLATFORM} \
        --exporter-opt push=true \
        --local dockerfile=. \
        --local context=.
```
After writing the script (you can copy and paste the foundation I've provided) you have successfully automated the creation and push of multiarch docker images.

## Manifest

We now need to create a `manifest`, creating `manifest` is an experimental Docker CLI feature and you should update Docker. You'll need to add the following lines to your `.travis.yml`:

```yaml
addons:
  apt:
    packages:
      - docker-ce
```

With your Docker client open, also run:

```bash
export DOCKER_CLI_EXPERIMENTAL=enabled
````

<img width="890" alt="export" src="https://user-images.githubusercontent.com/20936398/131183288-ea1c4f25-b075-412f-8d0e-48e5b2b66403.png">

What we want to do first is, we'll create the `manifest`, then we annotate the `manifest` and finally then do a push using `git push`.

## More information on `manifest` 

I wrote a beautiful guide on `manifest` that can be found on my GitHub [here](https://github.com/Montana/manifest). I highly recommend referring to this if you have any questions. 

![image](https://user-images.githubusercontent.com/20936398/131183547-dc6eeab5-5fc4-4b57-b4e2-c43e674afb97.png)


## Final steps 

Under the hood, the container runs a shell script from QEMU, that registers the emulator as an executor of binaries from the external platforms. So remember `buildctl` is stating that it must use the specified Dockerfile; (the one you wrote, or grabbed), and the conditional of it must build the images for the defined platforms (I specified `linux/amd64`,`linux/arm64`,`linux/arm/v7`), create a manifests list that's tagged as the desired image, and push all the images to the registry. There you have it you just successfully used Docker for a multiarch build using Travis. 

## Conclusion 

Support for multi-platform is getting easier and things that were hard say a year ago maybe even less are just mildly complex now, and with more and more features in the Travis CI pipeline, this will be only getting easier and easier.

If you have any questions please email me at [montana@travis-ci.org](mailto:montana@travis-ci.org).

Happy building!

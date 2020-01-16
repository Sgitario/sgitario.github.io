---
layout: post
title: Login into a Docker Repository with an invalid certificate
date: 2020-01-16
tags: [ docker, podman ]
---

I wanted to write a quick tutorial about how to push a docker image into an insecure Docker repository. By insecure Docker repository, I mean a site with SSL with either an expired or invalid certificate. In summary, if you try to do the next:

```bash
docker login my-docker-repository.com
```

And it fails with:

```bash
x509: certificate signed by unknown authority
```

Then, continue reading because you will find an easy and straigh forward solution.

## Solution

Docker does not allow to login or push images into a site with invalid certificates. There are a few workarounds to create a temporal certificate in local. However, another easier solution is using [podman](https://podman.io/). 

As a very brief summary, podman is a docker client for Linux systems developed by Red Hat. Oh wait, do we need to install a tool? Next!! Wait wait! We don't need to install anything, just use Docker!

1. Run a container with Podman already installed:

```bash
docker run -it --net=host ringingmountain/podman /bin/bash
```

| We can map the volumes or create the Dockerfile directly inside the container.

2. Login using Podman

```bash
podman login --tls-verify=false my-docker-repository.com
```

The trick in podman is to use the *tls-verify* flag to not verify the certificate.

3. Create the image, push or do whatever we wanted to do at first with the Docker repository

```bash
podman build --tls-verify=false -t myimage .
podman tag myimage:latest my-docker-repository.com/layout/myimage:7.6.0-2
podman push --tls-verify=false my-docker-repository.com/layout/myimage:7.6.0-2
```

And that's all! :)
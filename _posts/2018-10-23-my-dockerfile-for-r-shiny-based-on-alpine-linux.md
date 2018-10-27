---
layout: post
title: Building a Repository of Alpine-based Docker Images for R: Base and Dev
authors: Aleksandar Ratesic
tags: [R, shiny, docker, alpine]
excerpt_separator: <!--more-->
---

![Shiny Alpine Docker]({{ site.url }}/images/shiny-alpine-docker/alpine-shiny.png){: style="display:block;margin-left:auto;margin-right:auto"}

As soon as I first found out that R became available on a stable version of Alpine Linux, back in August 2018, I thought it would be a good idea to create a Docker image based on Alpine for my Shiny apps. Two months later I no longer thing it was a good idea, but  

<!--more-->

The [Rocker Project](https://github.com/rocker-org/rocker){:target="blank"} currently maintains the Dockerfiles for R, and their images are usually based on Debian. I use their images as a base for my Shiny containers, but the virtual size of the images I build tend to fall in the range between 400 and 600 MB, so I was interested in trying to reduce the size of containers by switching to Alpine. [Alpine Linux](https://alpinelinux.org/){:target="blank"} is a lightweight Linux distribution that is popular for building Docker containers because the virtual size of an Alpine docker image starts at 5 MB.

## Building the R Base Image

The [artemklevtsov/r-alpine repository](https://github.com/artemklevtsov/r-alpine){:target="blank"} has several Dockerfiles for building images, but rather than using packages available from Alpine repositories, it builds R from source. That is a great solution if you want to have more control over which R version your containers run. However, 

is the first version I came up with: 

```docker
FROM alpine:3.8

MAINTAINER Aleksandar Ratesic "aratesic@gmail.com"

# Declare environment variables
ENV LC_ALL en_US.UTF-8
ENV LANG en_US.UTF-8

# Install R and R-dev
RUN apk upgrade --update \
    && apk add --no-cache \
    R \
    R-dev

CMD ["R", "--no-save"]
```

The resulting image has a virtual size of 136.4 MB with R 3.5.0. That's a smaller size compared to the rocker/r-base image, which is 277.4MB according to [MicroBadger](https://microbadger.com/images/rocker/r-base){:target="blank"}. However, the rocker/r-base image also includes some packages like *litter* that my Dockerfile doesn't, so it has additional features that could be of interest to R users. I didn't bother with those features for now because all I wanted was to set up a Shiny Server to host my apps.

Before continuing to build an image with Shiny Serve Installed, I noticed that the image could be smaller because the command `apk upgrade --update && apk add --no-cache R R-dev` installs a total of 77 packages. As some of those packages are build dependencies, it is possible to remove them later.

From the R's [APKBUILD file](https://git.alpinelinux.org/cgit/aports/tree/community/R?h=3.8-stable){:target="blank"} I obtained the runtime dependencies for R (*depends* variable) and R-dev (*depends_dev* variable). 

```shell
aratesic@cloudshell:~/alpineDockerImageR (repos-199514)$ docker run -it 3b82a3fa38d0

R version 3.5.0 (2018-04-23) -- "Joy in Playing"
Copyright (C) 2018 The R Foundation for Statistical Computing
Platform: x86_64-pc-linux-gnu (64-bit)
R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.

R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.

Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.

>
```

The container worked as intended, but the virtual size of the image remained 136.4 MB. Nevertheless, the benefit 


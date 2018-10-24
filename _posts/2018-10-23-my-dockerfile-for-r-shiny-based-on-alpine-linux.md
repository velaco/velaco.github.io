---
layout: post
title: Building a Docker Container for R Shiny Apps Based on Alpine Linux
authors: Aleksandar Ratesic
tags: [R, shiny, docker, alpine]
excerpt_separator: <!--more-->
---
![Shiny Alpine Docker]({{ site.url }}/images/shiny-alpine-docker/alpine-shiny.png){: style="display:block;margin-left:auto;margin-right:auto"}

As soon as I found out that R became available on a stable version of Alpine Linux, I decided to create a Docker image for Shiny apps based on it. The [Rocker Project](https://github.com/rocker-org/rocker){:target="blank"} currently maintains the Dockerfiles for R, and their containers are based on Debian, so they tend to get large. My containers for Shiny apps that are built using their solution usually fall in the range between 400 and 600 MB on . (Alpine Linux){:target="blank"} is a lightweight Linux distribution that is popular for building Docker containers because the virtual size of an Alpine docker image starts at 5 MB.

<!--more-->

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

The resulting image has a virtual size of 136.9 MB running R 3.5.0. That's a smaller size compared to the rocker/r-base image, which is 277.4MB according to [MicroBadger](https://microbadger.com/images/rocker/r-base){:target="blank"}. The rocker/r-base image also includes some packages like *litter* that my Dockerfile doesn't, so different operating systems are not the only . I

Two months after building the R-base container, I finally found the time to continue building a Shiny Docker container. Shiny Server does not have pre-compiled installers for Alpine Linux, so I had to build it from source inside the container. My solution was to:

1. Define the build dependencies that will be removed later. This includes even packages like gcc and g++, which are used to install packages from CRAN. However, the shiny package is the only one that needs to be installed system-wide to have a functional Shiny Server running. Once I set up the shiny package, I'll remove the build dependencies and use a project-specific library with [rbundler](https://cran.r-project.org/package=rbundler){:target="blank"} or [packrat](https://cran.r-project.org/package=packrat){:target="blank"} to manage dependencies.
2. I used the Dockerfile available from the [rocker/shiny repository](https://hub.docker.com/r/rocker/shiny/~/dockerfile/){:target="blank"} for reference here and some Dockerfiles from my previous projects. There is no need to install the gdebi-core package because I'll need to build the Shiny Server from scratch.
3. Run installation instructions as a single layer.
4. 


I installed the shiny package system-wide, which is a requirement to build , 

The final Dockerfile with Shiny Server looks like this:

``` docker
FROM velaco/r-base:5.1.0-r1-alpine3.8

MAINTAINER Aleksandar Ratesic "aratesic@gmail.com"

ENV SHINY_BUILD_DEPS \
    sudo \
    python2 \
    cmake \
    gcc \
    g++ \
    git
    
ENV SHINY_PERSISTENT_DEPS \
    curl-dev \
    cairo-dev \
    libxt-dev \
    libxml2-dev


```

From the Google Cloud shell, I ran the command `docker run` and here is the output on port 8080:

## Further Development

The latest stable version of Alpine Linux at the moment, v3.8, does not have pandoc in its repository. Pandoc is a system requirement for the *rmarkdown* package, so if you want to run . The [skyzyx/alpine-pandoc repository](https://github.com/skyzyx/alpine-pandoc){:target="blank"} has a Dockerfile that accomplishes that.


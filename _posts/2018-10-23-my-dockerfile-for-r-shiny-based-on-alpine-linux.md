---
layout: post
title: Building a Repository of Alpine-based Docker Images for R, Part I
authors: Aleksandar Ratesic
tags: [R, shiny, docker, alpine]
excerpt_separator: <!--more-->
---

![Shiny Alpine Docker]({{ site.url }}/images/shiny-alpine-docker/alpine-shiny.png){: style="display:block;margin-left:auto;margin-right:auto"}

As soon as I found out that R became available on a stable version of Alpine Linux, back in August 2018, I thought it would be a good idea to create a Docker image based on Alpine for my Shiny apps. Although I did build Shiny Server inside a Docker container based on Alpine Linux, my [velaco/alpine-r repository](https://github.com/velaco/alpine-r){:target="blank"} still doesn't contain a usable Dockerfile because the server won't start. In this article I'll present my current progress and .

<!--more-->

## R and Docker

The [Rocker Project](https://github.com/rocker-org/rocker){:target="blank"} currently maintains the Dockerfiles for R, and their images are usually based on Debian. I use their images as a base for my Shiny containers, but the virtual size of the images I build tend to fall in the range between 400 and 600 MB, so I was interested in trying to reduce their size by switching to Alpine. [Alpine Linux](https://alpinelinux.org/){:target="blank"} is a lightweight Linux distribution that is popular for building Docker containers because the virtual size of an Alpine Docker image starts at 5 MB.

## Building the R Base Image on Alpine

The [artemklevtsov/r-alpine repository](https://github.com/artemklevtsov/r-alpine){:target="blank"} has several Dockerfiles for building Alpine-based images, but rather than using packages available from Alpine repositories, it builds R from source. That is a great solution if you want to have more control over which R version your containers run. However, my goal was to deploy containerized Shiny apps, so I decided to install only the . Here is what my Dockerfile looks like:

```docker
FROM alpine:3.8

MAINTAINER Aleksandar Ratesic "aratesic@gmail.com"

# Declare environment variables ------------------

ENV LC_ALL en_US.UTF-8
ENV LANG en_US.UTF-8

ENV BUILD_DEPS \
    cairo-dev \ 
    libxmu-dev \
    openjdk8-jre-base \ 
    pango-dev \
    perl \
    tiff-dev \
    tk-dev

ENV PERSISTENT_DEPS \
    R-mathlib \
    gcc \
    gfortran \
    icu-dev \
    libjpeg-turbo \
    libpng-dev \
    make \
    openblas-dev \
    pcre-dev \
    readline-dev \
    xz-dev \
    zlib-dev \
    bzip2-dev \
    curl-dev

# Install R and R-dev ------------------

RUN apk upgrade --update && \
    apk add --no-cache --virtual .build-deps $BUILD_DEPS && \
    apk add --no-cache --virtual .persistent-deps $PERSISTENT_DEPS && \
    apk add --no-cache R R-dev && \
    apk del .build-deps

CMD ["R", "--no-save"]
```

The resulting image has a virtual size of 136.4 MB. That's an improvement in size compared to the rocker/r-base image, which is 277.4MB according to [MicroBadger](https://microbadger.com/images/rocker/r-base){:target="blank"}. However, the rocker/r-base image also includes some packages like *litter* that my Dockerfile doesn't, so it has additional features that could be of interest to R users. My goal was to build an image I can use to host Shiny apps, so I just needed R and R dev installed 

## Building the R Shiny Server Image

Two months after building the R-base container, I finally found the time to continue building a Shiny Docker Image. Shiny Server does not have pre-compiled installers for Alpine Linux, so I had to add instructions to the Dockerfile to build it from source. I used [Building Shiny Server from Source](https://github.com/rstudio/shiny-server/wiki/Building-Shiny-Server-from-Source){:target="blank"} as a reference to determine which dependencies I'll install and to write the installation and post-installation instructions for Docker.

In order to install the *httpuv* package, one of the dependencies of the *shiny* package, I also added automake, autoconf, m4, and file. The httpuv package required the automake version 1.15, so the first command I run as part of the installation process is `echo 'http://dl-cdn.alpinelinux.org/alpine/v3.6/main' >> /etc/apk/repositories`. This gives me the ability to use packages from Alpine 3.6, so I can install the exact version of automake I need.

### Installation

The image build failed during the installation because the command `cmake -DCMAKE_INSTALL_PREFIX=/usr/local ../` installed only pandoc, so the command `(cd .. && ./bin/npm --python="$PYTHON" install)` couldn't find NodeJS. I had to run `(cd .. && ./external/node/install-node.sh)` to retrieve node as well, but .

I ran the command `file ../ext/node/bin/node` and learned that this ELF file requested program interpreter is /lib64/ld-linux-x86-64.so.2, which is found in the *libc6-compat* package. That package didn't help, so I found the *glibc* package on GitHub ([link to repository](https://github.com/sgerrand/alpine-pkg-glibc){:target="blank"} and added the following instructions to the Dockerfile.

```shell
wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub && \
wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.28-r0/glibc-2.28-r0.apk && \
apk add glibc-2.28-r0.apk && rm glibc-2.28-r0.apk && \
```

Even though all required libraries were now included, node was still raising errors because it was looking for them in the wrong path. I installed and used patchelf to set rpath and a library that .

```shell
patchelf --set-rpath /usr/lib:/usr/glibc-compat/lib:/lib /shiny-server/ext/node/bin/node && \
patchelf --add-needed /lib/libc.musl-x86_64.so.1 /shiny-server/ext/node/bin/node && \
```

```shell
/shiny-server/tmp # (cd .. && ./bin/node ./ext/node/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js --python="/usr/bin/python" rebuild)
/shiny-server/ext/node/bin/node.bin: /usr/lib/libstdc++.so.6: no version information available (required by /shiny-server/ext/node/bin/node.bin)
/shiny-server/ext/node/bin/node.bin: /usr/lib/libstdc++.so.6: no version information available (required by /shiny-server/ext/node/bin/node.bin)
/shiny-server/ext/node/bin/node.bin: /usr/lib/libstdc++.so.6: no version information available (required by /shiny-server/ext/node/bin/node.bin)
/shiny-server/ext/node/bin/node.bin: /usr/lib/libstdc++.so.6: no version information available (required by /shiny-server/ext/node/bin/node.bin)
/shiny-server/ext/node/bin/node.bin: /usr/lib/libstdc++.so.6: no version information available (required by /shiny-server/ext/node/bin/node.bin)
/shiny-server/ext/node/bin/node.bin: /usr/lib/libstdc++.so.6: no version information available (required by /shiny-server/ext/node/bin/node.bin)
Segmentation fault (core dumped)
```

At this point I decided to stop trying to solve the pr, so I had to . Instead of downloading the . The node version from the Alpine repository was 8.11.4. The Shiny Server repository downloads version 8.11.3, so 

```shell
# Install nodejs v
apk add --no-cache nodejs npm

(cd .. && npm --python="/usr/bin/python" --unsafe-perm install)
(cd .. && node /usr/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js --python="/usr/bin/python" rebuild)
make install
```

The Docker image The final image had a virtual size of 362.3 MB, even though I didn't remove any build dependencies or the git repository I cloned from GitHub to build Shiny Server. However, the container didn't run, so it was useless.

## Next Steps

I'm going to continue working on a Docker image based on Alpine Linux for Shiny apps, and 

1. Use the *libc6-compat* package instead of *glibc*. I used *patchelf* only with the *glibc* package, maybe it would have worked better with *libc6-compat*.
2. Build R and Node from source in an image that uses Alpine Linux as a base before building Shiny Server. Because Alpine uses *musl libc* as a lightweight alternative to *glibc*, building everything from source could remove the compatibiltiy issues causing 

[velaco/alpine-r repository](https://github.com/velaco/alpine-r){:target="blank"}
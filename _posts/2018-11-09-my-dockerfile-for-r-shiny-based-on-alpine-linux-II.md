---
layout: post
title: Building a Repository of Alpine-based Docker Images for R, Part II
authors: Aleksandar Ratesic
tags: [R, shiny, docker, alpine]
excerpt_separator: <!--more-->
---

![Shiny Alpine Docker]({{ site.url }}/images/shiny-alpine-docker/alpine-shiny.png){: style="display:block;margin-left:auto;margin-right:auto"}

In the [first article of this series]({{ site.url }}/my-dockerfile-for-r-shiny-based-on-alpine-linux/){:target="blank"}, I built an Alpine-based Docker image with R base packages from Alpine's native repositories, as well as one image with R compiled from source code. The images are hosted on Docker Hub, [velaco/alpine-r](https://hub.docker.com/r/velaco/alpine-r/){:target="blank"} repository.

The next step was either to address the fatal errors I found while testing the installation of R or to proceed building an image with Shiny Server. The logical choice would have been to pass all tests with R's base packages before proceeding, but I was a bit impatient and wanted to go through the process of building a Shiny Server as soon as possible. Although I did build an image with Shiny Server based on Alpine Linux, the container can't start the server. In this article I'll go over the instructions I used to build the image and comment on possible ways to fix that problem.

<!--more-->

## Building the R Shiny Server Image

Shiny Server doesn't have precompiled installers for Alpine Linux, so I had to build it from source. I used [Building Shiny Server from Source](https://github.com/rstudio/shiny-server/wiki/Building-Shiny-Server-from-Source){:target="blank"} as a reference to determine which dependencies I'll install and to write the first version of the installation and post-installation instructions.

### Dependencies

I wasn't sure which dependencies to put under BUILD and which to put under PERSISTANT, so I added them all to a single variable for now. The packages *python2*, *gcc*, *g++*, *git*, *cmake*, and *R* (base and devel) are the main system requirements for installing Shiny Server from source. Based on my review of other Dockerfiles for Shiny, including the one maintained by Rocker, I also added *curl-dev*, *libffi*, *psmisc*, *rrdtool*, *cairo-dev*, *libxt-dev*, and *libxml2-dev* to the list of dependencies.

A system-wide installation of the *shiny* package and its dependencies is also necessary, and the first problem I encountered was the installation of the *httpuv* package. In order to install the *httpuv* package, I had to install *automake*, *autoconf*, *m4*, and *file* as well. The package asked for *automake* version 1.15 specifically, so I had to include the command `echo 'http://dl-cdn.alpinelinux.org/alpine/v3.6/main' >> /etc/apk/repositories` before adding the system requirements because that branch contains *automake* v1.15.1.

Here is the final list of dependencies I included in the Dockerfile:

```docker
FROM velaco/alpine-r:base-3.5.0-r1

MAINTAINER Aleksandar Ratesic "aratesic@gmail.com"

# TODO: Separate to build and persistent!
ENV ALL_DEPS \
    automake==1.15.1-r0 \
    autoconf \
    bash \
    m4 \
    file \
    curl-dev \
    python2 \ 
    g++ \ 
    gcc \
    cmake \
    libc6-compat \
    git \
    libffi \
    psmisc \
    rrdtool \
    cairo-dev \
    libxt-dev \
    libxml2-dev \
    npm \
    nodejs
```

I'll explain later why *npm* and *nodejs* made it to the list.

### Installation

I cloned the Shiny Server git repository to make sure I have control over the changes made to it while building the image. Apart from the installation of system dependencies and the *shiny* package, the first version of the Dockerfile was pretty much an exact copy of the instructions provided by RStudio on GitHub:

```docker
RUN echo 'http://dl-cdn.alpinelinux.org/alpine/v3.6/main' >> /etc/apk/repositories && \
    apk upgrade --update && \
    apk add --no-cache $ALL_DEPS && \
    R -e "install.packages(c('shiny'), repos='http://cran.rstudio.com/')" && \
    git clone https://github.com/velaco/shiny-server.git && \
    cd shiny-server && mkdir tmp && cd tmp && \
    DIR=`pwd` && PATH=$DIR/../bin:$PATH && \
    cmake -DCMAKE_INSTALL_PREFIX=/usr/local ../ && \
    make && mkdir ../build && \
    (cd .. && ./bin/npm --python="/usr/bin/python" install) && \
    (cd .. && ./bin/node /usr/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js --python="/usr/bin/python" rebuild) && \
    make install
```

The first problem during the installation occurred because the command `cmake -DCMAKE_INSTALL_PREFIX=/usr/local ../` installed only pandoc, so the command `(cd .. && ./bin/npm --python="$PYTHON" install)` couldn't find the NodeJS installer. I decided to build an image up to that point and run a container to interactively look for a solution from the shell.

I retrieved NodeJS with the command `(cd .. && ./external/node/install-node.sh)`, but the installation failed again. I ran the command `file ../ext/node/bin/node` and learned that the requested program interpreter is `/lib64/ld-linux-x86-64.so.2`. The *libc6-compat* package contains that file and other compatibility libraries for *glibc*, so I added it and tried to install NodeJS again, but ended up with this error message:

```shell
Error relocating ../ext/node/bin/node: __register_atfork: symbol not found
```

Alpine uses *musl lib* instead of *glibc*, which contains __register_atfork. Because I was using a precompiled binary to install NodeJS, I assumed the difference between the two libraries was causing that error. In the attempt to solve it, I found the *glibc* package for Alpine on GitHub ([link to repository](https://github.com/sgerrand/alpine-pkg-glibc){:target="blank"}) and ran the following instructions to retrieve and install the package:

```shell
wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub && \
wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.28-r0/glibc-2.28-r0.apk && \
apk add glibc-2.28-r0.apk && rm glibc-2.28-r0.apk
```

I tried to run the script again, but it was looking for the libraries in the wrong path. That problem was easy to fix with *patchelf*:

```shell
patchelf --set-rpath /usr/lib:/usr/glibc-compat/lib:/lib /shiny-server/ext/node/bin/node && \
patchelf --add-needed /lib/libc.musl-x86_64.so.1 /shiny-server/ext/node/bin/node
```

Although the installer could now find the required libraries, rebuilding the modules still failed because of *libstdc++*:

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

I ran `ldd` from inside the container and found the same error with __register_atfork as before:

```shell
/shiny-server/tmp # ldd ../ext/node/bin/node
        /lib64/ld-linux-x86-64.so.2 (0x7f6478766000)
        libdl.so.2 => /lib64/ld-linux-x86-64.so.2 (0x7f6478766000)
        librt.so.1 => /lib64/ld-linux-x86-64.so.2 (0x7f6478766000)
        libstdc++.so.6 => /usr/lib/libstdc++.so.6 (0x7f6478413000)
        libm.so.6 => /lib64/ld-linux-x86-64.so.2 (0x7f6478766000)
        libgcc_s.so.1 => /usr/lib/libgcc_s.so.1 (0x7f6478201000)
        libpthread.so.0 => /lib64/ld-linux-x86-64.so.2 (0x7f6478766000)
        libc.so.6 => /lib64/ld-linux-x86-64.so.2 (0x7f6478766000)
Error relocating ../ext/node/bin/node: __register_atfork: symbol not found
Error relocating ../ext/node/bin/node: backtrace: symbol not found
```

The *glibc* repository has an [open issue](https://github.com/sgerrand/alpine-pkg-glibc/issues/33) that reported the same problem (__register_atfork: symbol not found) in 2016, but I couldn't find the solution to it anywhere.

At this point I was out of ideas, so I decided to change my approach and attempt to install *nodejs* and *npm* packages from the native repositories. The only NodeJS version available from Alpine's 3.8 repository was 8.11.4. Shiny Server comes with a script that downloads version 8.11.3, so I hoped that the difference of one patch version wouldn't be a problem in this case.

With *npm* and *nodejs* packages installed from the native repository, I modified the instructions to rebuild the modules before installing Shiny Server, and the following instruction successfuly created a layer:

```docker
RUN echo 'http://dl-cdn.alpinelinux.org/alpine/v3.6/main' >> /etc/apk/repositories && \
    apk upgrade --update && \
    apk add --no-cache $ALL_DEPS && \
    R -e "install.packages(c('shiny'), repos='http://cran.rstudio.com/')" && \
    git clone https://github.com/velaco/shiny-server.git && \
    cd shiny-server && mkdir tmp && cd tmp && \
    cmake -DCMAKE_INSTALL_PREFIX=/usr/local ../ && \
    make && mkdir ../build && \
    (cd .. && npm --python="/usr/bin/python" --unsafe-perm install) && \
    (cd .. && node /usr/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js --python="/usr/bin/python" rebuild) && \
    make install
```

### Post-installation

The post-installation is quite straightforward. You need to create a link to the shiny-server executable in `/usr/bin`, create the shiny user, and the directories for hosting the apps, logs, configuration files, etc. I decided to leave out the logs and not to touch the permissions because the script executed on running the container does that, so this is what the post-installation part of the instructions looked like:

```shell
RUN ln -s /usr/local/shiny-server/bin/shiny-server /usr/bin/shiny-server && \
    addgroup -g 82 -S shiny; \
    adduser -u 82 -D -S -G shiny shiny && \
    mkdir -p /srv/shiny-server && \
    mkdir -p /var/lib/shiny-server && \
    mkdir -p /etc/shiny-server
    
COPY shiny-server.conf /etc/shiny-server/shiny-server.conf
COPY shiny-server.sh /usr/bin/shiny-server.sh

# Move sample apps to test installation, clean up
RUN chmod 744 /usr/bin/shiny-server.sh && \
    mv /shiny-server/samples/sample-apps /srv/shiny-server/ && \
    mv /shiny-server/samples/welcome.html /srv/shiny-server/ && \
    rm -rf /shiny-server/
    
EXPOSE 3838

CMD ["/usr/bin/shiny-server.sh"]
```

### Putting It All Together

The Dockerfile I used can be found [here](https://github.com/velaco/alpine-r/blob/master/shiny-3.5.0-r1/Dockerfile){:target="blank"}. The final image it builds has a compressed size of 381 MB. With anticipation I executed the command

`docker run --rm -it velaco/alpine-r:shiny-3.5.0-r1`

and... nothing happened. The container runs if I start an interactive shell, but as soon as I execute `exec /usr/bin/shiny-server`, it stops working without any message.

execve("/usr/local/shiny-server/ext/node/bin/shiny-server", ["/usr/local/shiny-server/ext/node"..., "/usr/local/shiny-server/lib/main"...], 0x7ffce0ae1518 /* 11 vars */) = -1 ENOENT (No such file or directory)

## Next Steps

Although my Dockerfile for Shiny Sever builds an image, it is worthless for now because the server won't start, so I have to figure out why. My goals for this project are currently as follows: 
* **Make sure the R installation passes all tests**. It doesn't have to pass every test without a single warning, but it should at least pass without fatal errors. The container runs R, so the installation is 
* **Compile NodeJS and npm from source**. This could fix the problems associated with precompiled binaries and different versions. Unfortunately, as Alpine Linux doesn't keep previous versions of packages in the repositories, I could get only version 8.11.4, whereas Shiny Server uses 8.11.3 according to the latest update to its GitHub repository. I find it hard to believe that a patch version update would cause the process to misbehave, but you never know.
* **Debug the Docker container**. I attempted to shell into the container and start the server manually with exec and OpenRC. Both attempts failed.
For now I'll take a short break from this project to clear my head and get back to it with a fresh pair of eyes. Anyone interested in contributing to the project and improving it can find the Dockerfiles in the [velaco/alpine-r repository](https://github.com/velaco/alpine-r/){:target="blank"} on GitHub. 
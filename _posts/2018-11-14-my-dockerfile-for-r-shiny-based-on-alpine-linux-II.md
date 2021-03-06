---
layout: post
title: Building a Repository of Alpine-based Docker Images for R, Part II
authors: Aleksandar Ratesic
tags: [R, shiny, docker, alpine]
excerpt_separator: <!--more-->
---

![Shiny Alpine Docker Run]({{ site.url }}/images/shiny-alpine-docker/alpine-shiny-run-success.png){: style="display:block;margin-left:auto;margin-right:auto;padding-top:25px;padding-bottom:25px"}

In the [first article of this series]({{ site.url }}/my-dockerfile-for-r-shiny-based-on-alpine-linux/){:target="blank"}, I built an Alpine-based Docker image with R base packages from Alpine's native repositories, as well as one image with R compiled from source code. The images are hosted on Docker Hub, [velaco/alpine-r](https://hub.docker.com/r/velaco/alpine-r/){:target="blank"} repository.

The next step was either to address the fatal errors I found while testing the installation of R or to proceed building an image with Shiny Server. The logical choice would have been to pass all tests with R's base packages before proceeding, but I was a bit impatient and wanted to go through the process of building a Shiny Server as soon as possible. After two weeks of trial and error, I finally have a container that can start the server and run Shiny apps.

<!--more-->

***Notes:** The Dockerfile I used is available in the [velaco/alpine-r repository](https://github.com/velaco/alpine-r/){:target="blank"} on GitHub. The latest image with Shiny Server is not available on Docker Hub because it exceeded the time limit for automated builds, so I'll have to upload it manually.*

## Building the R Shiny Server Image

Shiny Server doesn't have precompiled installers for Alpine Linux, so I had to build it from source. I used [Building Shiny Server from Source](https://github.com/rstudio/shiny-server/wiki/Building-Shiny-Server-from-Source){:target="blank"} as a reference to determine which dependencies I'll install and to write the first version of the installation and post-installation instructions.

### Dependencies

I wasn't sure which dependencies to put under BUILD and which to put under PERSISTENT, so I added them all to a single variable for now. The packages *python2*, *gcc*, *g++*, *git*, *cmake*, and *R* (base and devel) are the main system requirements for installing Shiny Server from source. Based on my review of other Dockerfiles for Shiny, including the one maintained by Rocker, I also added *curl-dev*, *libffi*, *psmisc*, *rrdtool*, *cairo-dev*, *libxt-dev*, and *libxml2-dev* to the list of dependencies.

A system-wide installation of the *shiny* package and its dependencies is also necessary, and the first problem I encountered was the installation of the *httpuv* package. In order to install the *httpuv* package, I had to install *automake*, *autoconf*, *m4*, and *file* as well. The package asked for *automake* version 1.15 specifically, so I had to include the command `echo 'http://dl-cdn.alpinelinux.org/alpine/v3.6/main' >> /etc/apk/repositories` before adding the system requirements because that branch contains *automake* v1.15.1.

The package *wget* retrieves Node.js and *linux-headers* are needed to compile it from source. (The precompiled binaries didn't work.)

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
    # Forgot to remove libc6-compat, no longer needed
    libc6-compat \
    git \
    libffi \
    psmisc \
    rrdtool \
    cairo-dev \
    libxt-dev \
    libxml2-dev \
    linux-headers \
    wget
```

### Installation

I forked the Shiny Server git repository to make sure I have control over the changes made to it while working on this image. Apart from the installation of system dependencies and the *shiny* package, the installation instructions in the first version of the Dockerfile were pretty much an exact copy of the instructions provided by RStudio on GitHub:

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

The first problem during the installation occurred because the command `cmake -DCMAKE_INSTALL_PREFIX=/usr/local ../` installed only pandoc, so the command `(cd .. && ./bin/npm --python="$PYTHON" install)` couldn't find the Node.js binary. I decided to build an image up to that point and run a container to interactively look for a solution from the shell.

I retrieved Node.js with the command `(cd .. && ./external/node/install-node.sh)`, but the installation failed again. I ran the command `file ../ext/node/bin/node` and learned that the requested program interpreter is `/lib64/ld-linux-x86-64.so.2`. The *libc6-compat* package contains that file and other compatibility libraries for *glibc*, so I added it and tried to install Node.js again. However, it didn't help. I ended up with this error message:

```shell
Error relocating ../ext/node/bin/node: __register_atfork: symbol not found
```

As an alternative to *libc6-compat*, I found the *glibc* package for Alpine on GitHub ([link to repository](https://github.com/sgerrand/alpine-pkg-glibc){:target="blank"}) and ran the following commands to retrieve and install the package:

```shell
wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub
wget https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.28-r0/glibc-2.28-r0.apk
apk add glibc-2.28-r0.apk && rm glibc-2.28-r0.apk
```

I tried to run the script again, but the binary was looking for the libraries in the wrong path. That problem was easy to fix with *patchelf*:

```shell
patchelf --set-rpath /usr/lib:/usr/glibc-compat/lib:/lib /shiny-server/ext/node/bin/node
patchelf --add-needed /lib/libc.musl-x86_64.so.1 /shiny-server/ext/node/bin/node
```

Although the location of the libraries was no longer an issue, rebuilding the modules still failed:

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

I ran `ldd` from inside the container and got the same error as before, stating that __register_atfork was not found:

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

The *glibc* repository has an [open issue](https://github.com/sgerrand/alpine-pkg-glibc/issues/33){:target="blank"} that reported the same problem (__register_atfork: symbol not found) in 2016, but I couldn't find the solution to it anywhere.

I decided to change my approach, so I tried to install *nodejs* and *npm* packages from the native repositories. The only Node.js version available from Alpine's 3.8 repository was 8.11.4. Shiny Server comes with a script that downloads version 8.11.3, so I hoped that the difference of one patch version wouldn't be a problem.

With *npm* and *nodejs* packages installed from the native repository, I modified the instructions to rebuild the modules before installing Shiny Server, and the following instruction successfully created a layer of the image:

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

The post-installation is quite straightforward. You need to create a link to the shiny-server executable from `/usr/bin`, create the shiny user, and the directories for hosting the apps, logs, configuration files, etc. I decided to leave out the logs and not to touch the permissions because the script executed on running the container does that, so this is what the post-installation part of the instructions looked like:

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

There were no problems with this part of the instructions, so the image was built successfully.

### Testing the Image

With anticipation, I executed the command

`run --rm -it velaco/alpine-r:shiny-3.5.0-r1`

and... nothing happened. It was possible to run the container and start a shell, but as soon as I executed `exec /usr/bin/shiny-server`, it stopped working without giving me any messages.

I ran the container again, this time without security restrictions, and used strace on `/usr/bin/shiny-server` to find out at which point it exits. Here's what I found:

```shell
execve("/usr/local/shiny-server/ext/node/bin/shiny-server", ["/usr/local/shiny-server/ext/node"..., "/usr/local/shiny-server/lib/main"...], 0x7ffce0ae1518 /* 11 vars */) = -1 ENOENT (No such file or directory)
```

Turns out that the script Shiny Server uses to install Node.js contains a few instructions I skipped during the installation:

```shell
cp ext/node/bin/node ext/node/bin/shiny-server
rm ext/node/bin/npm
(cd ext/node/lib/node_modules/npm && ./scripts/relocate.sh)
```

There was no way I could run those instructions with Node.js installed from native packages, so I decided to modify the installation instructions to compile it from source:

```docker
RUN echo 'http://dl-cdn.alpinelinux.org/alpine/v3.6/main' >> /etc/apk/repositories && \
    apk upgrade --update && \
    apk add --no-cache $ALL_DEPS && \
    R -e "install.packages(c('shiny'), repos='http://cran.rstudio.com/')" && \
    git clone https://github.com/velaco/shiny-server.git && \
    cd shiny-server && mkdir tmp && cd tmp && \
    DIR=`pwd` && PATH=$DIR/../bin:$PATH && \
    cmake -DCMAKE_INSTALL_PREFIX=/usr/local ../ && \
    wget https://nodejs.org/dist/v8.11.3/node-v8.11.3.tar.xz && \
    mkdir node_source && \
    tar xf node-v8.11.3.tar.xz --strip-components=1 -C "node_source" && \
    rm node-v8.11.3.tar.xz && \
    (cd node_source && ./configure --prefix=/shiny-server/ext/node) && \
    (cd node_source && make) && \
    (cd node_source && make install) && \
    cp /shiny-server/ext/node/bin/node /shiny-server/ext/node/bin/shiny-server && \
    rm /shiny-server/ext/node/bin/npm && \
    (cd ../ext/node/lib/node_modules/npm && ./scripts/relocate.sh) && \
    make && mkdir ../build && \
    (cd .. && ./bin/npm --python="/usr/bin/python" --unsafe-perm install) && \
    (cd .. && ./bin/node ./ext/node/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js --python="/usr/bin/python" rebuild) && \
    make install
```

Building the image this way took more than 2 hours, but when I started the container and got confirmation that the server was running, I started celebrating. I should have waited a bit longer to celebrate because when I checked the example app in my browser, I learned that I still had work to do:

![Shiny Alpine Docker]({{ site.url }}/images/shiny-alpine-docker/alpine-shiny-example-error.png){: style="display:block;margin-left:auto;margin-right:auto;padding-top:25px;padding-bottom:25px"}

Apparently, the command `su` in Alpine Linux doesn't understand the `--login` option. The server is configured to `run_as shiny` and the container was running as root, so I ~~guessed~~ hoped that was the only reason it had to execute `su` at some point.

When I added the `USER shiny` instruction to the Dockerfile, the `su` command was no longer necessary. However, the server still didn't work because the container executes a script to create the log directory. User `shiny` didn't have permission to create the log directory, so I created it from the Dockerfile and set `shiny:shiny` as the owner of pretty much everything. (At this point I just wanted to get it over with, so some changes were probably not necessary and there was probably a better solution.)

With a log directory and `shiny` as the default user, the server successfully started the example Shiny app:

![Shiny Alpine Docker]({{ site.url }}/images/shiny-alpine-docker/alpine-shiny-example-success.png){: style="display:block;margin-left:auto;margin-right:auto;padding-top:25px;padding-bottom:25px;"}

## Next Steps

The image I created runs Shiny Server on Alpine Linux, but I wouldn't feel comfortable using it in production yet. Here are my goals for improving the current images:
* **Make sure the R installation passes all tests**. It doesn't have to pass every test without a single warning, but it should at least pass them all without fatal errors.
* **Remove build dependencies from the image**. The virtual size of this image is currently just over 500 MB. It can be reduced by removing some dependencies that are not required for running the server and probably some other files that are not necessary for running apps (e.g., documentation files, test suites, etc.).
* **Compile Node.js from source in a base container**. Node took forever to compile and install. The image alpine-r:shiny-3.5.0-r1 can't automatically build on Docker Hub because it exceeds the time limit allowed for automated builds. Maybe it would be a better idea to compile Node.js in an separate image and use that image as a base for building one with Shiny Server.

For now I'll take a short break from this project to clear my head and get back to it with a fresh pair of eyes. Anyone interested in contributing and improving it can find the Dockerfiles in the [velaco/alpine-r repository](https://github.com/velaco/alpine-r/){:target="blank"} on GitHub.
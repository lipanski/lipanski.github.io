---
layout: default
title: "The smallest Docker image to serve static websites"
tags: devops
comments_url: https://github.com/lipanski/lipanski.github.io/discussions/6
favourite: true
cover: /assets/images/coryphaena-pentadactyla.jpg
---

# {{ page.title }}

Until recently, I used to think that serving static websites from Docker would be a waste of bandwith and storage. Bundling nginx or various other heavy runtimes inside a Docker image for the sole purpose of serving static files didn't seem like the best idea - Netlify or Github Pages can handle this much better. But my hobby server was sad and cried digital tears.

A recent HackerNews post about [redbean](https://justine.lol/redbean/index.html), a single-binary, super tiny, static file server got me thinking. So begins my journey to find the most time/storage efficient Docker image to serve a static website.

After evaluating a few static file servers with similar specs, I initially opted for [thttpd](https://www.acme.com/software/thttpd/), which comes with a similar small footprint but seems a bit more battle-tested. This got me to a whooping **186KB** image and you can read more about it in the [previous version](https://github.com/lipanski/lipanski.github.io/blob/8aa8994299d0314b4d113ea481c60561f97c2940/_posts/2021-02-28-smallest-docker-image-static-website.md) of this post.

A later comment (thanks Sergey Ponomarev) suggested the [BusyBox httpd](https://git.busybox.net/busybox/tree/networking/httpd.c) file server, which seemed fairly small and more feature-rich so I gave it a try. Let's see if it can produce an even smaller image (spoiler alert: it can).

[BusyBox](https://www.busybox.net/) is much more than just a file server - it's a set of lightweight replacements for many common UNIX utilities, like *shell*, *gzip*, or *echo*.

Running the BusyBox httpd server goes like this:

```
busybox httpd -f -v -p 3000
```

This will launch the server in foreground (`-f`), listening on port `3000` (`-p`), serving all files inside the current directory. Access logs will be printend to `STDOUT` (`-v`). It comes with a few other neat features, like serving gzipped content, custom error pages, basic auth, allow/deny rules, and reverse proxying, which can be enabled by adding a `httpd.conf` file. You can read more about it in the [source code comments](https://git.busybox.net/busybox/tree/networking/httpd.c).

My first attempt used the official `busybox` image:

```dockerfile
FROM busybox:1.35

# Create a non-root user to own the files and run our server
RUN adduser -D static
USER static
WORKDIR /home/static

# Copy the static website
# Use the .dockerignore file to control what ends up inside the image!
COPY . .

# Run BusyBox httpd
CMD ["busybox", "httpd", "-f", "-v", "-p", "3000"]
```

You can build and run the image by calling:

```sh
docker build -t static:latest .
docker run -it --rm --init -p 3000:3000 static:latest
```

...then browse to `http://localhost:3000`.

The image builds quickly and, at **1.25MB**, is fairly small:

```
> docker images | grep static
static   latest   854054cff457   1 second ago   1.25MB
```

The image is already [built](https://github.com/docker-library/busybox/tree/master/stable/musl) using [`scratch`](https://hub.docker.com/_/scratch), which is basically a *no-op* image, light as vacuum. It contains only the statically compiled BusyBox binary and nothing else. There's not much we can optimize there.

Then again BusyBox comes packaged with much more than just the static file server - it contains all these other UNIX utilities. We can create a custom build of BusyBox limiting it to only *httpd* and thus reducing its size.

We start by downloading the BusyBox source code:

```
git clone git://busybox.net/busybox.git
```

Then create a default `.config` file for the build with all features disabled:

```
make allnoconfig
```

Next we call `make menuconfig` and select the `httpd` features from within "Network Utilities". Since we don't want to depend on other OS libraries, we also need to check "Build static binary" from within "Settings". The resulting `.config` file looks like this:

```
# ...
CONFIG_STATIC=y
# ...
CONFIG_HTTPD=y
CONFIG_FEATURE_HTTPD_PORT_DEFAULT=80
CONFIG_FEATURE_HTTPD_RANGES=y
CONFIG_FEATURE_HTTPD_SETUID=y
CONFIG_FEATURE_HTTPD_BASIC_AUTH=y
CONFIG_FEATURE_HTTPD_AUTH_MD5=y
CONFIG_FEATURE_HTTPD_CGI=y
CONFIG_FEATURE_HTTPD_CONFIG_WITH_SCRIPT_INTERPR=y
CONFIG_FEATURE_HTTPD_SET_REMOTE_PORT_TO_ENV=y
CONFIG_FEATURE_HTTPD_ENCODE_URL_STR=y
CONFIG_FEATURE_HTTPD_ERROR_PAGES=y
CONFIG_FEATURE_HTTPD_PROXY=y
CONFIG_FEATURE_HTTPD_GZIP=y
CONFIG_FEATURE_HTTPD_ETAG=y
CONFIG_FEATURE_HTTPD_LAST_MODIFIED=y
CONFIG_FEATURE_HTTPD_DATE=y
CONFIG_FEATURE_HTTPD_ACL_IP=y
# ...
```

Since building the `.config` is a bit tedious, we'll save it for later use. You can find a sample on [my Github](https://github.com/lipanski/docker-static-website/blob/master/.config).

Finally, we compile the binary:

```
make
make install
```

...which will be placed at `_install/bin/busybox`.

If you've built the binary in a glibc environment (e.g. Ubuntu), it would take up around **1.5MB**, which is not that great. Actually, it's worse than the official image containing all BusyBox utilities.

Let's try and build it on **musl**, inside an Alpine container:

```dockerfile
FROM alpine:3.13.2

# Install all dependencies required for compiling busybox
RUN apk add gcc musl-dev make perl

# Download busybox sources
RUN wget https://busybox.net/downloads/busybox-1.35.0.tar.bz2 \
  && tar xf busybox-1.35.0.tar.bz2 \
  && mv /busybox-1.35.0 /busybox

WORKDIR /busybox

# Copy the busybox build config (limited to httpd)
COPY .config .

# Compile and install busybox
RUN make && make install
```

The binary size looks much better now: **177KB**!

We can improve further by dropping some unneeded *httpd* features from the `.config`:

```
CONFIG_HTTPD=y
CONFIG_FEATURE_HTTPD_PORT_DEFAULT=80
# CONFIG_FEATURE_HTTPD_RANGES is not set
# CONFIG_FEATURE_HTTPD_SETUID is not set
CONFIG_FEATURE_HTTPD_BASIC_AUTH=y
# CONFIG_FEATURE_HTTPD_AUTH_MD5 is not set
# CONFIG_FEATURE_HTTPD_CGI is not set
# CONFIG_FEATURE_HTTPD_CONFIG_WITH_SCRIPT_INTERPR is not set
# CONFIG_FEATURE_HTTPD_SET_REMOTE_PORT_TO_ENV is not set
# CONFIG_FEATURE_HTTPD_ENCODE_URL_STR is not set
CONFIG_FEATURE_HTTPD_ERROR_PAGES=y
CONFIG_FEATURE_HTTPD_PROXY=y
CONFIG_FEATURE_HTTPD_GZIP=y
CONFIG_FEATURE_HTTPD_ETAG=y
CONFIG_FEATURE_HTTPD_LAST_MODIFIED=y
CONFIG_FEATURE_HTTPD_DATE=y
CONFIG_FEATURE_HTTPD_ACL_IP=y
```

> You can disable most of these features but in my experience the biggest impact comes from dropping MD5 support for basic auth and CGI.

We've now reached **149KB**. It's time to wrap things up and copy the static BusyBox binary to a Docker [`scratch`](https://hub.docker.com/_/scratch) image. Using the `scratch` image usually requires a multi-stage approach. We start from `alpine`, download and compile *BusyBox* as a static binary, create a user, then copy these assets over to `scratch` and add our static files to the mix:


```dockerfile
FROM alpine:3.13.2 AS builder

# Install all dependencies required for compiling busybox
RUN apk add gcc musl-dev make perl

# Download busybox sources
RUN wget https://busybox.net/downloads/busybox-1.35.0.tar.bz2 \
  && tar xf busybox-1.35.0.tar.bz2 \
  && mv /busybox-1.35.0 /busybox

WORKDIR /busybox

# Copy the busybox build config (limited to httpd)
COPY .config .

# Compile and install busybox
RUN make && make install

# Create a non-root user to own the files and run our server
RUN adduser -D static

# Switch to the scratch image
FROM scratch

EXPOSE 3000

# Copy over the user
COPY --from=builder /etc/passwd /etc/passwd

# Copy the busybox static binary
COPY --from=builder /busybox/_install/bin/busybox /

# Use our non-root user
USER static
WORKDIR /home/static

# Uploads a blank default httpd.conf
# This is only needed in order to set the `-c` argument in this base file
# and save the developer the need to override the CMD line in case they ever
# want to use a httpd.conf
COPY httpd.conf .

# Copy the static website
# Use the .dockerignore file to control what ends up inside the image!
COPY . .

# Run busybox httpd
CMD ["/busybox", "httpd", "-f", "-v", "-p", "3000", "-c", "httpd.conf"]
```

Let's have another look at those numbers:

```
> docker images | grep static
static   latest   9b08b9509c32   1 second ago   154kB
```

![Excellent](/assets/images/excellent.png)

The **154KB** we're left with correspond to the size of the *BusyBox httpd* static binary and the static files that were copied over, which in my case was just one file containing the text `hello world`. Note that the `alpine` step of the multi-stage build is actually quite large in size (*~185MB*), but it can be reused across builds and doesn't get pushed to the registry. In order to skip the `alpine` step entirely, I pushed the resulting image to the Docker registry.

You can download it from [Docker Hub](https://hub.docker.com/r/lipanski/docker-static-website) and use it to serve your static websites:

```dockerfile
FROM lipanski/docker-static-website:latest

COPY . .
```

This produces a single-layer image of **154KB** + whatever the size of your static website and *nothing else*. If you need to configure *httpd* in a different way, you can just override the `CMD` line:

```dockerfile
FROM lipanski/docker-static-website:latest

COPY . .

CMD ["/busybox", "httpd", "-f", "-v", "-p", "3000", "-c", "httpd.conf"]
```

The code and an FAQ about configuring *httpd* is available at <https://github.com/lipanski/docker-static-website>.

To conclude, Docker *can* be used efficiently to package and serve static websites.

---
layout: default
title: "The smallest Docker image to serve static websites"
tags: devops
comments: true
cover: /assets/images/coryphaena-pentadactyla.jpg
---

# {{ page.title }}

Until recently, I used to think that serving static websites from Docker would be a waste of bandwith and storage. Bundling nginx or various other heavy runtimes inside a Docker image for the sole purpose of serving static files didn't seem like the best idea - Netlify or Github Pages can handle this much better. But my hobby server was sad and cried digital tears.

A recent HackerNews post about [readbean](https://justine.lol/redbean/index.html), a single-binary, super tiny, static file server got me thinking. So begins my journey to find the most time/storage efficient Docker image to serve a static website.

After evaluating a few static file servers with similar specs, I settled for [thttpd](https://www.acme.com/software/thttpd/), which comes with a similar small footprint but seems a bit more battle-tested.

Running `thttpd` goes like this:

```
thttpd -D -h 0.0.0.0 -p 3000 -d /static-website -u static-user -l - -M 60
```

This will launch the server in foreground (`-D`), listening on host `0.0.0.0` (`-h`) and port `3000` (`-p`), serving all files inside `/static-website` (`-d`) that are accessible to `static-user` (`-u`). It will print access logs to `STDOUT` (`-l -`) and set the `Cache-Control` header to `60` seconds (`-M`). There are a couple of other neat features, like basic auth, throttling and virtual hosts, which you can read about in the [documentation](https://linux.die.net/man/8/thttpd).

My first attempt uses the small `alpine` image, which already packages `thttpd`:

```dockerfile
FROM alpine:3.13.2

# Install thttpd
RUN apk add thttpd

# Create a non-root user to own the files and run our server
RUN adduser -D static
USER static
WORKDIR /home/static

# Copy the static website
# Use the .dockerignore file to control what ends up inside the image!
COPY . .

# Run thttpd
CMD ["thttpd", "-D", "-h", "0.0.0.0", "-p", "3000", "-d", "/home/static", "-u", "static", "-l", "-", "-M", "60"]
```

You can build and run the image by calling:

```sh
docker build -t static:latest .
docker run -it --rm -p 3000:3000 static:latest
```

...then browse to `http://localhost:3000`.

The image builds quickly and, at *7.77MB*, is fairly small:

```
> docker images | grep static
static              latest              cb1750e32562        About an hour ago    7.77MB
```

We can improve further by using Docker [`scratch`](https://hub.docker.com/_/scratch), which is basically a *no-op* image, light as vacuum. The problem with `scratch` is that you can't really do much inside: you can't create new users, there is no package manager or any executable for that matter - aside from the ones you copied over yourself.

Using the `scratch` image usually requires a multi-stage approach. We start from `alpine`, download and compile `thttpd` as a static binary, create a user, then copy these assets over to `scratch` and add our static files to the mix:

```dockerfile
FROM alpine:3.13.2 AS builder

ARG THTTPD_VERSION=2.29

# Install all dependencies required for compiling thttpd
RUN apk add gcc musl-dev make

# Download thttpd sources
RUN wget http://www.acme.com/software/thttpd/thttpd-${THTTPD_VERSION}.tar.gz \
  && tar xzf thttpd-${THTTPD_VERSION}.tar.gz \
  && mv /thttpd-${THTTPD_VERSION} /thttpd

# Compile thttpd to a static binary which we can copy around
RUN cd /thttpd \
  && ./configure \
  && make CCOPT='-O2 -s -static' thttpd

# Create a non-root user to own the files and run our server
RUN adduser -D static

# Switch to the scratch image
FROM scratch

EXPOSE 3000

# Copy over the user
COPY --from=builder /etc/passwd /etc/passwd

# Copy the thttpd static binary
COPY --from=builder /thttpd/thttpd /

# Use our non-root user
USER static
WORKDIR /home/static

# Copy the static website
# Use the .dockerignore file to control what ends up inside the image!
COPY . .

# Run thttpd
CMD ["/thttpd", "-D", "-h", "0.0.0.0", "-p", "3000", "-d", "/home/static", "-u", "static", "-l", "-", "-M", "60"]
```

Let's have another look at those numbers:

```
> docker images | grep static
static              latest              ab0699ed2690        About a minute ago   186kB
```

![Excellent](/assets/images/excellent.png)

The *186KB* we're left with correspond to the size of the *thttpd* static binary and the static files that were copied over, which in my case was just one file containing the text `hello world`. Note that the `alpine` step of the multi-stage build is actually quite large in size (*~130MB*), but it can be reused across builds and doesn't get pushed to the registry.

At this point, you can convert the image we built so far into a base image for all your static websites and push it to a registry, so that you can skip the `alpine` step entirely. Or you can simply use [my Docker Hub build](https://hub.docker.com/r/lipanski/docker-static-website):

```dockerfile
FROM lipanski/docker-static-website:latest

COPY . .
```

This produces a single-layer image of *186KB* + whatever the size of your static website and *nothing else*. If you need to configure *thttpd* in a different way, you can just override the `CMD` line:

```dockerfile
FROM lipanski/docker-static-website:latest

COPY . .

CMD ["/thttpd", "-D", "-h", "0.0.0.0", "-p", "3000", "-d", "/home/static", "-u", "static", "-l", "-", "-M", "60"]
```

To conclude, Docker *can* be used efficiently to package and serve static websites.

The code is available at <https://github.com/lipanski/docker-static-website>.

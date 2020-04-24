---
layout: default
title: "Speed up your Docker builds with --cache-from"
tags: devops 
comments: true
cover: /assets/images/chaetodon-guttatus.jpg
---

## {{ page.title }}

Using the Docker cache efficiently can result in significantly faster build times. Then again, there are environments where individual builds happen independent of each other and the build cache is never preserved. Every build starts from zero which can be slow and wasteful. It's the case of some CI/CD systems.

As long as you're pushing images to a remote registry, you can always use a previously built image as a cache layer for a new build. You can achieve this by setting the `--cache-from` option on the `docker build` call. For versions of Docker that don't include BuildKit, you'll have to pull the image yourself before running `docker build`. Assuming the latter, here's how things would look like:

```sh
# This is the full image name, including the registry
IMAGE="my-docker-registry.example.com/my-docker-image"

# Pull an older, existing version from the registry
docker pull ${IMAGE}:0.1.0

# Build a new version by using the older version as a cache
docker build --cache-from ${IMAGE}:0.1.0 -t ${IMAGE}:0.2.0 .

# Push the new version to the registry so that we can use it as a cache for future builds
docker push ${IMAGE}:0.2.0
```

### Maximize your chances of hitting the cache

You can pass the `--cache-from` option **several times**, to provide different images to use as a cache. Let's assume your remote registry contains version builds (`1.0.0`), which you build once a month, and branch builds, which are built whenever you push code to a branch. Ideally you'd use the branch build images, because those are fresher, but if no branch was built yet you'd like to fall back to a version build image. You can call `--cache-from` several times to fetch the most suitable image:

```sh
IMAGE="my-docker-registry.example.com/my-docker-image"

# Between `current-branch`, `master` and a tagged version `1.0.0`, we prefer current-branch
# If there was no previously built `current-branch` image, we fetch `master`
# If there's no `master` image, we fall back to the tagged version
# We don't need all 3 images, just the most suitable one, hence the `||`
docker pull ${IMAGE}:current-branch || \
  docker pull ${IMAGE}:master || \
  docker pull ${IMAGE}:1.0.0 || \
  true

# Build a new version while mentioning all possible cache sources
docker build \
  --cache-from ${IMAGE}:current-branch \
  --cache-from ${IMAGE}:master \
  --cache-from:${IMAGE}:1.0.0 \
  -t ${IMAGE_NAME}:current-branch .

# Push the new version to the registry so that we can use it as a cache for future builds
docker push ${IMAGE}:current-branch
```

At this point you'll have to do the math: depending on your build infrastructure, if the time to fetch the remote images and build with `--cache-from` is less than the time it takes to build without using the cache, then this was worth it. If you're build is fast anyway or downloading the images comes at a high cost, then it might not be something for you. 

### Multi-stage builds

With multi-stage builds things get a little more complicated. The intermediate build stages are never pushed to the remote registry so you can't use them as cache.

Consider the following *Dockerfile*:

```dockerfile
# The builder stage
FROM ruby:2.5.5-alpine AS builder

RUN apk add --update libxml2-dev

COPY Gemfile ./

RUN bundle install

# The final stage
FROM ruby:2.5.5-alpine

COPY --from=builder /usr/local/bundle/ /usr/local/bundle/

COPY app ./

CMD ["rackup"] 
```

Any change to the `Gemfile` will require a full build, including the line that installs `libxml2-dev`. Only a change restricted to the `app/` directory will be able to use the cache.

One possible solution is **storing intermediate build stages in the registry**. Your new build process could look something like this:

```sh
IMAGE=my-image

# Build the builder image
docker build --target builder -t ${IMAGE}:builder .

# Build the final image
docker build -t ${IMAGE}:final .

# Push both builder and final images to the remote registry
docker push ${IMAGE}:builder
docker push ${IMAGE}:final
```

The build itself will be just as fast as if you'd make only one call, because the second `docker build` can use the local build cache for the `builder` stage. The potential bandwidth/speed penalty comes from having to push the additional image to the registry.

The full multi-stage build including the `--cache-from` usage would end up looking something like this:

```sh
IMAGE="my-docker-registry.example.com/my-docker-image"

# Pull older versions of the builder and final images from the registry (if any)
docker pull ${IMAGE}:builder || true
docker pull ${IMAGE}:final || true

# Build the builder image by using the older builder image as a cache
docker build --cache-from ${IMAGE}:builder -t ${IMAGE}:builder .

# Build the final image by using the older final image as a cache
# ...but also the local cache from the previous builder build
docker build --cache-from ${IMAGE}:final -t ${IMAGE}:final .

# Push both images so that we can use them as a cache for future builds
docker push ${IMAGE}:builder
docker push ${IMAGE}:final
```

On top of that, as explained in the previous section you can call `--cache-from` several times, in order to identify the best image pair (builder/final) to use as cache.

This is where you have to do the math again: is the bandwidth/speed penalty incurring from pushing and pulling these intermediate images worth it? Should you optimize for build time or for deployment/scaling time? Would you trade multi-stage builds against simplifying the build process?

### Same but different: docker load/save

If your build environment has access to some shared storage (e.g. S3, EBS or just a shared directory), you can use the `docker save` and `docker load` commands to store and retrieve images. You can later reuse these images in order to enhance your local build cache. The `docker save` command saves one or more images as a tar file, which can be placed inside your shared storage. Before your next build, you can retrieve this file and unpack the images back into the local registry by calling `docker load`. During the build, point the `--cache-from` option to the loaded image. Here's how it goes:

```sh
# Before running the build, unpack and load images from my-image.tar into the local registry
docker load -i /some/shared/directory/my-image.tar || true

# Run the build with the --cache-from option pointing to the saved image
docker build --cache-from my-image:latest -t my-image:latest .

# Pack and save the freshly built image inside the shared directory 
docker save -o /some/shared/directory/my-image.tar my-image:latest
```

This approach can be a bit more flexible in environments where it's hard to access the remote registry. 

### BuildKit

If your Docker version has access to [BuildKit](https://docs.docker.com/develop/develop-images/build_enhancements/), check out the improvements around `BUILDKIT_INLINE_CACHE`, which can save you an expensive `docker pull` operation. 

### Further reading

Check out my other article on [Best practices when writing a Dockerfile]({% post_url 2019-09-20-dockerfile-ruby-best-practices %}).

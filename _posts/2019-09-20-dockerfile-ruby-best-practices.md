---
layout: default
title: "Best practices when writing a Dockerfile for a Ruby application"
favourite: true
tags: devops ruby
comments: true
cover: /assets/images/pleuronectes-limandoides.jpg
---

# {{ page.title }}
{:.no_toc}

The simplicity of the *Dockerfile* format is one of the reasons why Docker managed to become so popular in the first place. Getting something working is fairly easy. Producing a clean, small, secure image that will not leak secrets might not be as straight-forward though.

This post will try to share some best practices when writing a *Dockerfile* for a Ruby app, though most of these points should apply to any other runtime as well. Towards the end, I will provide full examples for three different use cases.

Here's a summary of what's coming:

* Table of Contents
{:toc}

The code presented here can also be found on Github: <https://github.com/lipanski/ruby-dockerfile-example>.

Let's begin:

## 1. Pin your base image version

Bad:

```dockerfile
FROM ruby

CMD ruby -e "puts 1 + 2"
```

Good:

```dockerfile
FROM ruby:2.5.5

CMD ruby -e "puts 1 + 2"
```

If you want **reproducible builds** (and trust me, you want them), make sure to pin down the version of your base image. Try to be as accurate as possible by specifying every digit, including the patch version.

If you want to update your base image, do it in a **controlled, explicit** manner which can also be reverted easily (e.g. via a pull request). It will save you a lot of debugging pain in the future.

## 2. Use only trusted or official base images

Bad:

```dockerfile
FROM random-dude-on-the-internet/ruby:2.5.5
```

Good:

```dockerfile
FROM ruby:2.5.5
```

Also good:

```dockerfile
# Assuming that is your own image registry, which you control entirely
FROM your-own-registry.com/ruby:2.5.5
```

When using images from <https://hub.docker.com>, prefer **official images** and/or try to checksum their contents. All *official* images are marked with the phrase *Docker Official Images* next to their title - like the [official Ruby image](https://hub.docker.com/_/ruby).

For anything that's not available as an official image, build and host the base image yourself - ideally by starting from a trusted/official one.

Keep in mind that Docker Hub does not prevent modifying images or tags by their authors over time, which is why you probably shouldn't trust everything in there.

## 3. Pin your application dependencies

Bad:

```dockerfile
FROM ruby:2.5.5

RUN gem install sinatra
```

Good:

```dockerfile
FROM ruby:2.5.5

RUN gem install sinatra -v 2.0.5
```

Similary to the base image version, if you don't pin your application dependencies you might be in for surprises the next time you build your image. This doesn't mean you should never update your dependencies but that you should do it in a controlled manner.

Most package managers have a way to pin dependencies, be it `Gemfile.lock`, `package-lock.json`, `yarn.lock` or `Pipfile.lock` - use it to guarantee **reproducible builds**.

## 4. Add a .dockerignore file to your repository

The Docker `COPY` instruction doesn't honour the `.gitignore` file. This means that whenever you call `COPY .` with a wildcard argument, you could be leaking unwanted files inside your Docker image.

Fortunately you can add a `.dockerignore` file to your code base, which works pretty much the same way the `.gitignore` file does. In addition to copying over the contents of your `.gitignore` file, you might want to include the `.git/` directory as well to the list of files ignored by Docker.

## 5. Group commands by how likely they are to change individually

Bad:

```dockerfile
FROM ruby:2.5.5

RUN apt update
RUN apt install -y mysql-client
RUN apt install -y postgresql-client
RUN apt install -y nginx
RUN bundle install

CMD ruby -e "puts 1 + 2"
```

Good:

```dockerfile
FROM ruby:2.5.5

# We usually only need to run this once
RUN apt update && \
  apt install -y mysql-client postgresql-client nginx

# We usually run this every time we add a new dependency
RUN bundle install

CMD ruby -e "puts 1 + 2"
```

Less build steps means less intermediary images that Docker needs to keep in storage. On the other hand, you need to be careful not to group tasks together that don't change at the same time - otherwise you might be running all tasks when only one requires changes, which results in a slower build process.

## 6. Place the least likely to change commands at the top

Bad:

```dockerfile
FROM ruby:2.5.5

# Source code
COPY my-code/ /srv/

# Application dependencies
COPY Gemfile Gemfile.lock ./
RUN bundle install

# Operating system dependencies
RUN apt update && \
  apt install -y mysql-client postgresql-client nginx
```

Good:

```dockerfile
FROM ruby:2.5.5

# Operating system dependencies
RUN apt update && \
  apt install -y mysql-client postgresql-client nginx

# Application dependencies
COPY Gemfile Gemfile.lock ./
RUN bundle install

# Source code
COPY my-source-code /srv/
```

Docker will rebuild all steps top to bottom, starting from the one where *changes* were detected. A *change* means usually that a line inside your *Dockerfile* changed with the exception of the `COPY` command, which also checks whether the files provided as argument were modified.

Placing the least likely to change commands at the top ensures an efficient usage of the Docker cache and results in shorter build times.

## 7. Avoid running your application as root

Bad:

```dockerfile
FROM ruby:2.5.5-alpine

RUN gem install sinatra -v 2.0.5

RUN echo 'require "sinatra"; run Sinatra::Application.run!' > config.ru

# By default this is run as root
CMD rackup
```

Good:

```dockerfile
FROM ruby:2.5.5-alpine

RUN gem install sinatra -v 2.0.5

# Create a dedicated user for running the application
RUN adduser -D my-sinatra-user

# Set the user for RUN, CMD or ENTRYPOINT calls from now on
# Note that this doesn't apply to COPY or ADD, which use a --chown argument instead
USER my-sinatra-user

# Set the base directory that will be used from now on
WORKDIR /home/my-sinatra-user

RUN echo 'require "sinatra"; run Sinatra::Application.run!' > config.ru

# This is run under the my-sinatra-user user
CMD rackup
```

Running your application as root introduces an **additional vector of attack**. If an attacker gains remote code execution through an application vulnerability, there are ways to escape the container environment. Much more damage can be inflicted by the root user than when running your application as an unprivileged user.

## 8. When running COPY or ADD (as a different user) use --chown

Bad:

```dockerfile
FROM ruby:2.5.5-alpine

RUN adduser -D my-sinatra-user

USER my-sinatra-user

WORKDIR /home/my-sinatra-user

# The files will be owned by the root user!
COPY Gemfile Gemfile.lock ./

RUN bundle install

CMD rackup
```

Good:

```dockerfile
FROM ruby:2.5.5-alpine

RUN adduser -D my-sinatra-user

USER my-sinatra-user

WORKDIR /home/my-sinatra-user

# The files will be owned by my-sinatra-user
COPY --chown=my-sinatra-user Gemfile Gemfile.lock ./

RUN bundle install

CMD rackup
```

The `USER` directive described in the previous step only applies to `RUN`, `CMD` or `ENTRYPOINT`. For `COPY` and `ADD` you have to use the `--chown` argument.

## 9. Avoid leaking secrets inside your image

Bad:

```dockerfile
FROM ruby:2.5.5

ENV DB_PASSWORD "secret stuff"
```

Secrets should never appear inside your *Dockerfile* in plain text. Instead, they should be injected via:

- Build-time arguments: the `ARG` command and `--build-arg` Docker argument.
- Environment variables: the `-e` or `--env-file` Docker arguments.
- Kubernetes secrets or similar methods.

**Note:** Whenever you use one of these `docker build` arguments, be it `--build-arg` or `-e`, the full command (including the secret values) will show up in your `docker history`. Depending on the environment where the build happens, you might want to avoid this. A solution to this problem is detailed in step 14.

## 10. Always clean up injected secrets within the same build step

Bad:

```dockerfile
FROM ruby:2.5.5

ARG PRIVATE_SSH_KEY

# This build step will retain the private SSH key
RUN echo "${PRIVATE_SSH_KEY}" > /root/.ssh/id_rsa

# This build step will retain the private SSH key
RUN bundle install

RUN rm /root/.ssh/id_rsa
```

Good:

```dockerfile
FROM ruby:2.5.5

ARG PRIVATE_SSH_KEY

RUN echo "${PRIVATE_SSH_KEY}" > /root/.ssh/id_rsa && \
  bundle install && \
  rm /root/.ssh/id_rsa
```

The first example produces two build steps that retain the injected secret. If anyone has access to your build history, they will be able to retrieve your secret. The suggested solution groups together the actions that inject and require the secret with the one that cleans it up. This produces a clean build history.

## 11. Fetching private dependencies via a Github token injected through the gitconfig

Quite often your application will require private dependencies, usually hosted inside private repositories, be it Ruby gems or NPM packages.

There are various ways to fetch them during the build process but as long as you are using Github, I recommend the following:

1. Set up a machine user on Github.
2. Allow your machine user read-only access to your private dependencies.
3. Generate [a personal Github access token](https://github.com/settings/tokens) for this user.
4. Use the Github token to pull dependencies during `bundle install` or `npm install`.
5. Clean up the Github token from the build.

By default, your `Gemfile` or `package.json` files probably use the SSH protocol because it's the most convenient one for development mode. If your private dependencies are referenced via `git@github.com` then this is the case.

Once you produced a working Github token, we can use a `.gitconfig` URL rewrite to tell Git to authenticate via your Github token instead of the default SSH protocol (which we still want in development). This is accomplished via the `insteadOf` Git option, which basically rewrites the repository URL to inject the token.

After we've successfully installed dependencies, it's important to remove the `.gitconfig` file *within the same step*, to avoid leaking the Github token inside the built image.

Last but not least, we'll be injecting the Github token into the build process via a build argument.

Let's put it all together:

```dockerfile
FROM ruby:2.5.5

ARG GITHUB_TOKEN

# This is a private gem that GITHUB_TOKEN has access to
RUN echo 'source "https://rubygems.org"; gem "some-private-gem", git: "git@github.com:some-user/some-private-gem"' > Gemfile

# First rewrite is for Gemfile, second is for package.json
# This should cover most other package managers as well
# Note the usage of --add (which avoids overwriting the option)
# Also note that we're cleaning up the .gitconfig file within the same build step
RUN git config --global url."https://${GITHUB_TOKEN}:x-oauth-basic@github.com/some-user".insteadOf git@github.com:some-user && \
  git config --global --add url."https://${GITHUB_TOKEN}:x-oauth-basic@github.com/some-user".insteadOf ssh://git@github && \
  bundle install && \
  rm ~/.gitconfig
```

You can build this image by calling: `docker build --build-arg GITHUB_TOKEN=xxx .`

> Another way of achieving the same is by using SSH keys. It saves you the hassle of rewriting the `Gemfile` URLs but you still need to clean up the private key at the end.

## 12. Minimize image size by opting for small base images when possible

Bad:

```dockerfile
FROM ruby:2.5.5

CMD ruby -e "puts 1 + 2"
```

Good:

```dockerfile
FROM ruby:2.5.5-alpine

CMD ruby -e "puts 1 + 2"
```

What's the difference?

```sh
> docker images -a | grep base-image
normal-base-image   869MB
small-base-image    45.3MB
```

Some base images are larger than others. A small base image produces a small final image which ultimately speeds up your deployments and saves you the additional storage costs.

[Alpine Linux](https://alpinelinux.org/) is usally a safe bet when looking for small-footprint operating systems.

When opting for a compact operating system, pay special attention to:

- The **package manager** and the available packages. Building things from source is tedious, use the package manager to your benefit.
- The **choice of shell** - some base images might not even provide a shell!
- The **security** implications and the **stability** of the underlying operating systems: avoid environments that are experimental or not battle-tested.

## 13. Use multi-stage builds to reduce the size of your image

Bad:

```dockerfile
FROM ruby:2.5.5-alpine

# Nokogiri's build dependencies
RUN apk add --update \
  build-base \
  libxml2-dev \
  libxslt-dev

# Nokogiri, yikes
RUN echo 'source "https://rubygems.org"; gem "nokogiri"' > Gemfile

RUN bundle install

CMD /bin/sh
```

Good:

```dockerfile
# The "builder" image will build nokogiri
FROM ruby:2.5.5-alpine AS builder

# Nokogiri's build dependencies
RUN apk add --update \
  build-base \
  libxml2-dev \
  libxslt-dev

# Nokogiri, yikes
RUN echo 'source "https://rubygems.org"; gem "nokogiri"' > Gemfile

RUN bundle install

# The final image: we start clean
FROM ruby:2.5.5-alpine

# We copy over the entire gems directory for our builder image, containing the already built artifact
COPY --from=builder /usr/local/bundle/ /usr/local/bundle/

CMD /bin/sh
```

Multi-stage builds are builds where the final image is composed of parts of different builds, which can potentially be based on completely different base images. You can use multi-stage builds to significantly reduce the size of your final images. Smaller artifacts ultimately means faster deployments and rollbacks.

In the current example we require the infamous *nokogiri* gem. In order to install this gem you usually need some relatively heavy OS dependencies (*libxml* and *libxslt*), though they are only useful at build time. The gem also needs to be built natively, which might produce additional build time garbage.

So what's the difference?

```sh
> docker images | grep nokogiri
nokogiri-simple   251MB
nokogiri-multi    70.1MB
```

As you can see, the difference can be quite signficant...

## 14. Use multi-stage builds to avoid leaking secrets inside your docker history

Bad:

```dockerfile
FROM ruby:2.5.5-alpine

# This is a secret
ARG PRIVATE_SSH_KEY

# Just a basic Gemfile to make bundle install happy
RUN echo 'source "https://rubygems.org"; gem "sinatra"' > Gemfile

# We require the secret for installing dependencies
RUN mkdir -p /root/.ssh/ && \
  echo "${PRIVATE_SSH_KEY}" > /root/.ssh/id_rsa && \
  bundle install && \
  rm /root/.ssh/id_rsa

CMD ruby -e "puts 1 + 2"
```

Good:

```dockerfile
FROM ruby:2.5.5-alpine AS builder

# This is a secret
ARG PRIVATE_SSH_KEY

# Just a basic Gemfile to make bundle install happy
RUN echo 'source "https://rubygems.org"; gem "sinatra"' > Gemfile

# We require the secret for installing dependencies
RUN mkdir -p /root/.ssh/ && \
  echo "${PRIVATE_SSH_KEY}" > /root/.ssh/id_rsa && \
  bundle install && \
  rm /root/.ssh/id_rsa

# The final image doesn't need the secret
FROM ruby:2.5.5-alpine

COPY --from=builder /usr/local/bundle/ /usr/local/bundle/

CMD ruby -e "puts 1 + 2"
```

You can build both examples by passing the `PRIVATE_SSH_KEY` as a build argument:

```sh
docker build -t my-fancy-image --build-arg PRIVATE_SSH_KEY=xxx .
```

As explained in step 9, you can use *Docker build arguments* to avoid leaking secrets inside your image. By default, however, this will still leak the secret inside your *Docker history*. Depending on your build environment, you might want to avoid this.

Let's see what this means for our bad example:

```sh
# Lines 3-4 contain your secret PRIVATE_SSH_KEY in clear text

> docker history my-fancy-image
IMAGE               CREATED              CREATED BY                                      SIZE                COMMENT
67e60c0853ab        19 seconds ago       /bin/sh -c #(nop)  CMD ["/bin/sh" "-c" "ruby…   0B
94dd778c4b5d        20 seconds ago       |1 PRIVATE_SSH_KEY=xxx /bin/sh -c mkdir -p /…   30.9MB
32a993af7bfb        About a minute ago   |1 PRIVATE_SSH_KEY=xxx /bin/sh -c echo 'sour…   45B
2be964ad91c7        About a minute ago   /bin/sh -c #(nop)  ARG PRIVATE_SSH_KEY          0B
44723f3ab2bd        4 months ago         /bin/sh -c #(nop)  CMD ["irb"]                  0B
<missing>           4 months ago         /bin/sh -c mkdir -p "$GEM_HOME" && chmod 777…   0B
<missing>           4 months ago         /bin/sh -c #(nop)  ENV PATH=/usr/local/bundl…   0B
<missing>           4 months ago         /bin/sh -c #(nop)  ENV BUNDLE_PATH=/usr/loca…   0B
<missing>           4 months ago         /bin/sh -c #(nop)  ENV GEM_HOME=/usr/local/b…   0B
<missing>           4 months ago         /bin/sh -c set -ex   && apk add --no-cache -…   45.5MB
<missing>           4 months ago         /bin/sh -c #(nop)  ENV RUBYGEMS_VERSION=3.0.3   0B
<missing>           4 months ago         /bin/sh -c #(nop)  ENV RUBY_DOWNLOAD_SHA256=…   0B
<missing>           4 months ago         /bin/sh -c #(nop)  ENV RUBY_VERSION=2.5.5       0B
<missing>           4 months ago         /bin/sh -c #(nop)  ENV RUBY_MAJOR=2.5           0B
<missing>           4 months ago         /bin/sh -c mkdir -p /usr/local/etc  && {   e…   45B
<missing>           4 months ago         /bin/sh -c apk add --no-cache   gmp-dev         3.4MB
<missing>           4 months ago         /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>           4 months ago         /bin/sh -c #(nop) ADD file:a86aea1f3a7d68f6a…   5.53MB
```

Now let's see what happens with the multi-stage build:

```sh
> docker history my-fancy-image
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
2706a2f47816        8 seconds ago       /bin/sh -c #(nop)  CMD ["/bin/sh" "-c" "ruby…   0B
86509dba3bd9        9 seconds ago       /bin/sh -c #(nop) COPY dir:e110956912ddb292a…   3.16MB
44723f3ab2bd        4 months ago        /bin/sh -c #(nop)  CMD ["irb"]                  0B
<missing>           4 months ago        /bin/sh -c mkdir -p "$GEM_HOME" && chmod 777…   0B
<missing>           4 months ago        /bin/sh -c #(nop)  ENV PATH=/usr/local/bundl…   0B
<missing>           4 months ago        /bin/sh -c #(nop)  ENV BUNDLE_PATH=/usr/loca…   0B
<missing>           4 months ago        /bin/sh -c #(nop)  ENV GEM_HOME=/usr/local/b…   0B
<missing>           4 months ago        /bin/sh -c set -ex   && apk add --no-cache -…   45.5MB
<missing>           4 months ago        /bin/sh -c #(nop)  ENV RUBYGEMS_VERSION=3.0.3   0B
<missing>           4 months ago        /bin/sh -c #(nop)  ENV RUBY_DOWNLOAD_SHA256=…   0B
<missing>           4 months ago        /bin/sh -c #(nop)  ENV RUBY_VERSION=2.5.5       0B
<missing>           4 months ago        /bin/sh -c #(nop)  ENV RUBY_MAJOR=2.5           0B
<missing>           4 months ago        /bin/sh -c mkdir -p /usr/local/etc  && {   e…   45B
<missing>           4 months ago        /bin/sh -c apk add --no-cache   gmp-dev         3.4MB
<missing>           4 months ago        /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>           4 months ago        /bin/sh -c #(nop) ADD file:a86aea1f3a7d68f6a…   5.53MB
```

The multi-stage build only retains the history of the final image. The builder history is ignored and thus your secret is safe.

## 15. When setting the CMD instruction, prefer the exec format over the shell format

Bad:

```dockerfile
FROM ruby:2.5.5

RUN echo 'source "https://rubygems.org"; gem "sinatra"' > Gemfile
RUN bundle install

# A simple Sinatra app which prints out HUUUUUP when the process receives the HUP signal.
RUN echo 'require "sinatra"; set bind: "0.0.0.0"; Signal.trap("HUP") { puts "HUUUUUP" }; run Sinatra::Application.run!' > config.ru

CMD bundle exec rackup
```

Good:

```dockerfile
FROM ruby:2.5.5

RUN echo 'source "https://rubygems.org"; gem "sinatra"' > Gemfile
RUN bundle install

# A simple Sinatra app which prints out HUUUUUP when the process receives the HUP signal.
RUN echo 'require "sinatra"; set bind: "0.0.0.0"; Signal.trap("HUP") { puts "HUUUUUP" }; run Sinatra::Application.run!' > config.ru

CMD ["bundle", "exec", "rackup"]
```

There are two ways of using `CMD`, as explained [here](https://docs.docker.com/engine/reference/builder/#cmd):

- The *shell format*, which invokes the command from within a shell - e.g. `CMD bundle exec rackup`
- The *exec format*, which doesn't invoke a command shell and takes the form of a JSON array - e.g. `CMD ["bundle", "exec", "rackup"]`

The recommended way of using `CMD` is in *exec format*. This ensures your process will run as PID 1 which in turn ensures that any received signals will also be handled properly.

Let's see how the process tree looks like for the container run in **shell format** (the bad example):

```sh
> docker exec $(docker ps -q) ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.7  0.0   2388   756 pts/0    Ss+  14:36   0:00 /bin/sh -c bundle exec rackup
root         6  0.8  0.2  43948 25556 pts/0    Sl+  14:36   0:00 /usr/local/bundle/bin/rackup
root        13  0.0  0.0   7640  2588 ?        Rs   14:37   0:00 ps aux
```

Our `bundle exec rackup` command is wrapped inside a `/bin/sh` call. The actual `rackup` call is not the PID 1 process. Sending a HUP signal to our container **will not get propagated** to the actual `rackup` process and will not print out `HUUUUUP`.

Now let's see how the process tree looks like for the container run in **exec format** (the good example):

```sh
> docker exec $(docker ps -q) ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1 29.0  0.2  43988 25632 pts/0    Ssl+ 14:47   0:00 /usr/local/bundle/bin/rackup
root         8  0.0  0.0   7640  2668 ?        Rs   14:47   0:00 ps aux
```

As you can see, the `rackup` process is not wrapped inside `/bin/sh` and is running as PID 1. Sending a HUP signal to our container will correctly print out `HUUUUUP`.

Why is this important? Some applications implement signals in order to exit gracefully or clean up resources. In the case of web servers, this usually means releasing connections from the database connection pool or finishing requests. If you want any of these you should care about signals.

Thanks to [Kamil Grabowski](https://twitter.com/_y3ti) for pointing this out on Twitter.

## 16. Avoid installing development or test dependencies in your production builds

Bad:

```dockerfile
FROM ruby:2.5.5

RUN echo 'source "https://rubygems.org"; gem "sinatra"' > Gemfile

RUN bundle install
```

Good:

```dockerfile
FROM ruby:2.5.5

RUN echo 'source "https://rubygems.org"; gem "sinatra"' > Gemfile

RUN bundle config set without 'development test'
RUN bundle install
```

By default, calling `bundle install` or `yarn install` will include development dependencies. Since these are usually not required in a typical production environment, excluding development and test dependencies from your production *Dockerfile* will speed up your builds and reduce the size of your images.

While we're at it, when installing gems I recommend setting the `--jobs` and `--retry` arguments, as it will speed up the process and make it more resilient to network issues:

```sh
bundle install --jobs=3 --retry=3
```

## 17. Optional: Combine production, test and development build processes into a single Dockerfile by using multi-stage builds

In many cases, your test, development and production build processes might slightly differ from each other. If you're using Docker across all these environments, a common approach is to introduce one `Dockerfile` per environment. Keeping these files in sync can gradually become redundant, tedious or just easy to forget.

A different approach to having multiple Dockerfiles is using multi-stage builds to map all your environments or build processes within the same Dockerfile. Here's an example:

```dockerfile
FROM ruby:2.5.5-alpine AS builder

# A Gemfile that contains a test dependency (minitest)
RUN echo 'source "https://rubygems.org"; gem "sinatra"; group(:test) { gem "minitest" }' > Gemfile
RUN echo 'require "sinatra"; run Sinatra::Application.run!' > config.ru

# By default we don't install development or test dependencies
RUN bundle install --without development test

# A separate build stage installs test dependencies and runs your tests
FROM builder AS test
# The test stage installs the test dependencies
RUN bundle install --with test
# Let's introduce a test that passes
RUN echo 'require "minitest/spec"; require "minitest/autorun"; class TestIndex < MiniTest::Test; def test_it_passes; assert(true); end; end' > test.rb
# The actual test run
RUN bundle exec ruby test.rb

# The production artifact doesn't contain any test dependencies
FROM ruby:2.5.5-alpine

COPY --from=builder /usr/local/bundle/ /usr/local/bundle/
COPY --from=builder /config.ru ./

CMD ["rackup"]
```

You can build the production artifact (without the test dependencies) by calling:

```sh
DOCKER_BUILDKIT=1 docker build .
```

You can run your tests by explicitly asking for the *test* stage:

```sh
DOCKER_BUILDKIT=1 docker build --target=test .
```

Notice the usage of the [BuildKit](https://docs.docker.com/develop/develop-images/build_enhancements/) feature flag. Prior to Docker 18.09 or without adding the `DOCKER_BUILDKIT=1` flag, a full build would still build all stages, including the test stage. The final artifact would still contain only the production dependencies but the build would take a little bit longer.

## 18. Bonus: Running migrations

There are various ways to run migrations in Docker, but the simplest one is by creating a *start script* for your application:

```sh
#!/bin/sh

set -e

bundle exec rake db:migrate
bundle exec rackup
```

I usually save this as `bin/start` and use it as my `CMD`:

```dockerfile
CMD ["bin/start"]
```

Note that Rails prevents running migrations in parallel from different processes. Deploying two or more containers in parallel might cause all but one of these deployments to fail. Then again, if you deploy containers in parallel, you're most likely using an automated solution (Kubernetes / Nomad / Docker Swarm), at which point your containers should get resurected and should eventually bypass the Rails migration lock.

## Putting it all together...

Enough with the theory! Let's apply these best practices. No app is the same, so I'll provide you with *Dockerfiles* for three different use cases, in order of their complexity.

We'll start with the `.dockerignore` file, which is shared by all examples. The easiest way to produce a `.dockerignore` file is by mirroring your `.gitignore` file, then adding the `.git/` directory to the list:

```sh
# Start from your .gitignore file
cp .gitignore .dockerignore

# Exclude the .git/ directory from being copied over into your images
echo ".git/" >> .dockerignore
```

## Dockerfile for a plain Ruby app or a Rails app without assets

```dockerfile
# Start from a small, trusted base image with the version pinned down
FROM ruby:2.7.1-alpine AS base

# Install system dependencies required both at runtime and build time
# The image uses Postgres but you can swap it with mariadb-dev (for MySQL) or sqlite-dev
RUN apk add --update \
  postgresql-dev \
  tzdata

# This stage will be responsible for installing gems
FROM base AS dependencies

# Install system dependencies required to build some Ruby gems (pg)
RUN apk add --update build-base

COPY Gemfile Gemfile.lock ./

# Install gems (excluding development/test dependencies)
RUN bundle config set without "development test" && \
  bundle install --jobs=3 --retry=3

# We're back at the base stage
FROM base

# Create a non-root user to run the app and own app-specific files
RUN adduser -D app

# Switch to this user
USER app

# We'll install the app in this directory
WORKDIR /home/app

# Copy over gems from the dependencies stage
COPY --from=dependencies /usr/local/bundle/ /usr/local/bundle/

# Finally, copy over the code
# This is where the .dockerignore file comes into play
# Note that we have to use `--chown` here
COPY --chown=app . ./

# Launch the server (or run some other Ruby command)
CMD ["bundle", "exec", "rackup"]
```

You can build this by calling:

```sh
docker build -t my-rails-app .
```

## Dockerfile for a Rails app with assets

```dockerfile
# Start from a small, trusted base image with the version pinned down
FROM ruby:2.7.1-alpine AS base

# Install system dependencies required both at runtime and build time
# The image uses Postgres but you can swap it with mariadb-dev (for MySQL) or sqlite-dev
RUN apk add --update \
  postgresql-dev \
  tzdata \
  nodejs \
  yarn

# This stage will be responsible for installing gems and npm packages
FROM base AS dependencies

# Install system dependencies required to build some Ruby gems (pg)
RUN apk add --update build-base

COPY Gemfile Gemfile.lock ./

# Install gems (excluding development/test dependencies)
RUN bundle config set without "development test" && \
  bundle install --jobs=3 --retry=3

COPY package.json yarn.lock ./

# Install npm packages
RUN yarn install --frozen-lockfile

# We're back at the base stage
FROM base

# Create a non-root user to run the app and own app-specific files
RUN adduser -D app

# Switch to this user
USER app

# We'll install the app in this directory
WORKDIR /home/app

# Copy over gems from the dependencies stage
COPY --from=dependencies /usr/local/bundle/ /usr/local/bundle/

# Copy over npm packages from the dependencies stage
# Note that we have to use `--chown` here
COPY --chown=app --from=dependencies /node_modules/ node_modules/

# Finally, copy over the code
# This is where the .dockerignore file comes into play
# Note that we have to use `--chown` here
COPY --chown=app . ./

# Install assets
RUN RAILS_ENV=production SECRET_KEY_BASE=assets bundle exec rake assets:precompile

# Launch the server
CMD ["bundle", "exec", "rackup"]
```

You can build this by calling:

```sh
docker build -t my-rails-app .
```

## Dockerfile for a Rails app with assets and private dependencies

```dockerfile
# Start from a small, trusted base image with the version pinned down
FROM ruby:2.7.1-alpine AS base

# Install system dependencies required both at runtime and build time
# The image uses Postgres but you can swap it with mariadb-dev (for MySQL) or sqlite-dev
RUN apk add --update \
  postgresql-dev \
  tzdata \
  nodejs \
  yarn

# This stage will be responsible for installing gems and npm packages
FROM base AS dependencies

# The argument is required later, when installing private gems or npm packages
ARG GITHUB_TOKEN

# Install system dependencies required to build some Ruby gems (pg)
RUN apk add --update build-base

COPY Gemfile Gemfile.lock ./

# Don't install development or test dependencies
RUN bundle config set without "development test"

# Install gems (including private ones)
# This uses the GITHUB_TOKEN argument, which is also cleaned up in the same step
RUN git config --global url."https://${GITHUB_TOKEN}:x-oauth-basic@github.com/some-user".insteadOf git@github.com:some-user && \
  git config --global --add url."https://${GITHUB_TOKEN}:x-oauth-basic@github.com/some-user".insteadOf ssh://git@github && \
  bundle install --jobs=3 --retry=3 && \
  rm ~/.gitconfig

COPY package.json yarn.lock ./

# Install npm packages (including private ones)
# This uses the GITHUB_TOKEN argument, which is also cleaned up in the same step
RUN git config --global url."https://${GITHUB_TOKEN}:x-oauth-basic@github.com/some-user".insteadOf git@github.com:some-user && \
  git config --global --add url."https://${GITHUB_TOKEN}:x-oauth-basic@github.com/some-user".insteadOf ssh://git@github && \
  yarn install --frozen-lockfile \
  rm ~/.gitconfig

# We're back at the base stage
FROM base

# Create a non-root user to run the app and own app-specific files
RUN adduser -D app

# Switch to this user
USER app

# We'll install the app in this directory
WORKDIR /home/app

# Copy over gems from the dependencies stage
COPY --from=dependencies /usr/local/bundle/ /usr/local/bundle/

# Copy over npm packages from the dependencies stage
# Note that we have to use `--chown` here
COPY --chown=app --from=dependencies /node_modules/ node_modules/

# Finally, copy over the code
# This is where the .dockerignore file comes into play
# Note that we have to use `--chown` here
COPY --chown=app . ./

# Install assets
RUN RAILS_ENV=production SECRET_KEY_BASE=assets bundle exec rake assets:precompile

# Launch the server
CMD ["bundle", "exec", "rackup"]
```

You can build this by calling:

```sh
docker build --build-arg GITHUB_TOKEN=xxx -t my-rails-app .
```

The code presented here can also be found on Github: <https://github.com/lipanski/ruby-dockerfile-example>. You can find the slides from my talk at the RUG:B meetup [here](/slides/dockerfile/index.html).

## Further reading
{:.no_toc}

- <https://pythonspeed.com/articles/dockerizing-python-is-hard/>
- <https://blog.docker.com/2019/07/intro-guide-to-dockerfile-best-practices/>
- <https://vsupalov.com/build-docker-image-clone-private-repo-ssh-key/>
- <https://evilmartians.com/chronicles/ruby-on-whales-docker-for-ruby-rails-development>
- <https://pythonspeed.com/articles/docker-caching-model/>
- <https://gmaslowski.com/docker-shell-vs-exec/>
- <https://medium.com/capital-one-tech/multi-stage-builds-and-dockerfile-b5866d9e2f84>

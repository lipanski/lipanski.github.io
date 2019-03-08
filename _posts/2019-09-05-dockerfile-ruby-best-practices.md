---
layout: default
title: "Best practices when writing a Dockerfile for a Ruby application"
tags: devops ruby
comments: true
---

## {{ page.title }}

The simplicity of the *Dockerfile* format is probably one of the reasons why Docker managed to become so popular in the first place. Getting something working is fairly easy. Producing a clean, small, secure image that will not leak secrets might not be as straight-forward though.

This post will try to share some best practices when writing a *Dockerfile* for a Ruby application. Most of these suggestions should be valid for any other runtime as well.

### Pin your base image version

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

If you want to update your base image, do it in a **controlled, explicit** manner which can also be reverted easily (e.g. via a pull request). It will save you a lot of unexpected errors and debugging pain in the future.

### Use only trusted or official base images

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

Use only official images from <https://hub.docker.com> and/or try to checksum the contents. All *official* images are marked with the phrase *Docker Official Images* next to their title - like the [official Ruby image](https://hub.docker.com/_/ruby).

For anything that's not available as an official image, build and host the base image yourself - ideally by starting from a trusted/official one.

Keep in mind that Docker Hub does not prevent modifying images or tags by their authors over time, which is why you probably shouldn't trust everything in there.

### Pin your application dependencies

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

Similary to the base image version, if you don't pin your application dependencies you might be in for surprises the next time you build your image. This doesn't mean you should never update your dependencies but that you should do it in a controlled manner. Most package managers have a way to pin dependencies, be it `Gemfile.lock`, `package-lock.json`, `yarn.lock` or `Pipfile.lock` - use it to guarantee **reproducible builds**.

### Add a .dockerignore file to your repository

The Docker `COPY` instruction doesn't honour the `.gitignore` file. This means that whenever you call `COPY .` with a wildcard argument, you could be leaking unwanted files inside your Docker image.

Fortunately you can add a `.dockerignore` file to your code base, which works pretty much the same way the `.gitignore` file does. In addition to copying over the contents of your `.gitignore` file, you might want to include the `.git/` directory as well to the list of files ignored by Docker.

### Group commands by how likely they are to change individually

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

### Place the least likely to change commands at the top

Bad:

```dockerfile
FROM ruby:2.5.5

# Source code
COPY my-code/ /srv/

# Application dependencies
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
RUN bundle install

# Source code
COPY my-source-code /srv/
```

Docker will rebuild all steps top to bottom, starting from the one where changes were detected. Placing the least likely to change commands at the top ensures an efficient usage of the Docker cache and results in shorter build times.

### Avoid running your application as root

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

### When running COPY or ADD (as a different user) use --chown

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

### Avoid leaking secrets inside your image

Bad:

```dockerfile
FROM ruby:2.5.5

ENV DB_PASSWORD "secret stuff"
```

Secrets should never appear inside your *Dockerfile* in plain text. Instead, they should be injected via:

- Build-time arguments: the `ARG` command and `--build-arg` Docker argument.
- Environment variables: the `-e` or `--env-file` Docker arguments.
- Kubernetes secrets or similar methods.

### Always clean up injected secrets within the same build step

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

ARG GITHUB_TOKEN

RUN echo "${PRIVATE_SSH_KEY}" > /root/.ssh/id_rsa && \
  bundle install && \
  rm /root/.ssh/id_rsa
```

The first example produces two build steps that retain the injected secret. If anyone has access to your build history, they will be able to retrieve your secret. The suggested solution groups together the actions that inject and require the secret with the one that cleans it up. This produces a clean build history.

### Fetching private dependencies via a Github token injected through the gitconfig

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

### Minimize image size by opting for small base images when possible

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

### Use multi-stage builds to reduce the size of your image

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

### Bonus: Database migrations

TODO...

### Putting it all together

### Further reading

- <https://pythonspeed.com/articles/dockerizing-python-is-hard/>
- <https://blog.docker.com/2019/07/intro-guide-to-dockerfile-best-practices/>
- <https://vsupalov.com/build-docker-image-clone-private-repo-ssh-key/>
- <https://evilmartians.com/chronicles/ruby-on-whales-docker-for-ruby-rails-development>

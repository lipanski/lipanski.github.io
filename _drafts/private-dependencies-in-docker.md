---
layout: default
title: "Fetching private Git dependencies from Docker containers - the secure way"
tags: devops
comments: true
---

## {{ page.title }}

There are several ways to fetch private Git dependencies (gems, npm packages etc.) from Docker containers. Some might pose security risks, some might be less convenient than the others. It's worth listing some of them:

- ssh key
- GITHUB_TOKEN and bundler
- GITHUB_TOKEN and npm
- git config
- ...

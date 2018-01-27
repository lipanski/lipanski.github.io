---
layout: default
title: "Upgrading Postgres and migrating your data to the new version"
tags: devops
comments: true
---

## {{ page.title }}

After upgrading your Postgres server to a newer version, your **data will not be migrated automatically** to the new installation.

In order to do so, you'll need both versions of Postgres installed on your machine. If you've uninstalled the old version, you can install it again by following [these instructions](https://www.postgresql.org/download/linux/ubuntu/).

By default, every Postgres installation creates a *main* cluster. On Ubuntu, the data of the *main* cluster is stored under `/var/lib/postgresql/VERSION/main`.

Your fresh installation already created its own *main* cluster so you'll need to drop if before migrating the old data:

```sh
sudo pg_dropcluster NEW_VERSION main
```

Afterwards, you can migrate the old cluster data by calling:

```sh
sudo pg_upgradecluster OLD_VERSION main
```

Check that your new Posgres version contains all your old data:

```sh
psql postgres
\l
```

At this point it's safe to remove the old cluster and the entire old package:

```sh
sudo pg_dropcluster OLD_VERSION main
sudo apt remove postgresql-OLD_VERSION
```

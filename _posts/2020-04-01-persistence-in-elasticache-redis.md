---
layout: default
title: "Persistence in (AWS ElastiCache) Redis"
favourite: true
tags: devops
comments: true
cover: /assets/images/trigla-pini.jpg
---

## {{ page.title }}

If you restart your Redis server and expect your data to still be there when the server comes back, you might be in for a surprise.

This article will give you a brief overview of how to ensure data persistence across restarts in Redis. Then it will focus on how to achieve the same in AWS ElastiCache Redis clusters, while discussing some of the limitations.

Before we start, it's worth stating that Redis *can* be used as a persistent data store and it *can* provide you with strong persistence guarantees, if you chose to enable them.

If you're using Redis solely as a cache and can afford losing the data between restarts or crashes, then this article might not be for you.

On the other hand, if you're using Sidekiq or other similar tools backed by Redis, this article might help protect your data on the long run.

### Persistence in Redis

There are two ways to ensure data persistence in Redis across restarts: through regular database snapshots (RDB) or by enabling the *append-only file* (AOF), a log capturing all the performed operations. Both methods, if enabled, will be replayed automatically when your Redis server starts.

**Regular snapshots (RDB)** can be enabled by configuring save points inside your `redis.conf` file:

```
# after 900 sec (15 min) if at least 1 key changed
save 900 1

# after 300 sec (5 min) if at least 10 keys changed
save 300 10

# after 60 sec if at least 10000 keys changed
save 60 10000
```

You will have to restart the server in order to apply these changes. To verify your configuration, call `info persistence` inside Redis and look for the lines starting with `rdb`:

```
rdb_changes_since_last_save:2
rdb_bgsave_in_progress:0
rdb_last_save_time:1582116473
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
```

Depending on the frequency of your configured save points, you might have to account with some data loss if your Redis server crashes before it gets to save your most recent data to disk.

**Append-only file (AOF)** persistence can be enabled from within your `redis.conf`:

```
appendonly yes
appendfsync always|everysec|no
```

The `appendfsync` option tells Redis how often it should flush the log file to disk. If you set it to `no`, it will leave this up to your operating system. If you set it to `always`, every new entry (basically every performed operation), will be flushed to disk immediately. This is definitely the safest option but it almost invalidates Redis as an in-memory data store, as it has to write to disk on every call.

A good middle ground is using the `everysec` option, which flushes the log entries to disk every second. This means that, during a crash, you can lose at most one second of data.

You will have to restart the server in order to apply this configuration and you can validate it by calling `info persistence` and looking for the lines starting with `aof`:

```
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_last_cow_size:0
aof_current_size:88
aof_base_size:88
aof_pending_rewrite:0
aof_buffer_length:0
aof_rewrite_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0
```

You can **combine the RDB and AOF methods**. The [Redis documentation](https://redis.io/topics/persistence) makes following recommendations:

> The general indication is that you should use both persistence methods if you want a degree of data safety comparable to what PostgreSQL can provide you.

> If you care a lot about your data, but still can live with a few minutes of data loss in case of disasters, you can simply use RDB alone.

### What happens when Redis runs out of memory?

When discussing persistence and data integrity, it's good to understand what happens when you're trying to push a new value to Redis and your server is out of memory.

The answer to this question resides in the value of the `maxmemory-policy` setting. When set to `noeviction`, Redis will raise an error when the memory limit was reached. This is **the safest choice** if you care about the data inside your Redis cluster.

Other options include `allkeys-lru`, `volatile-lru`, `allkeys-random`, `volatile-random` or `volatile-ttl` and they will all make room for the new key by evicting older, existing ones and thus causing data loss.

You can check your current configuration by calling `info memory` inside Redis. You can update the directive from within your `redis.conf`.

Last but not least, you can control the amount of memory Redis has access to by making use of the `maxmemory` configuration directive.

### Persistence in AWS ElastiCache Redis

Persistence in AWS ElastiCache Redis clusters is a more complicated story. They really live by that *Cache* in *ElastiCache*. For the most basic, single node deployment using the default parameter group, persistence is not guaranteed: after a restart or a crash, your data is gone.

Then again, the [AWS ElastiCache FAQ](https://aws.amazon.com/elasticache/redis/faqs/) hint at achieving persistence is not very helpful:

> Does Amazon ElastiCache for Redis support Redis persistence? Yes, you can achieve persistence by snapshotting your Redis data using the Backup and Restore feature.

Enabling **daily backups** is a good start, especially when you've got one daily snapshot per cluster free of charge. The general price is pretty low and you can go up to 35 days retention. The problem with these backups is that **you can not replay them automatically after a restart or a crash**. Furthermore you can't replay a backup manually on top of an *existing* cluster either. You may only restore a backup manually and to a new cluster.

On top of daily backups, you can enable **append-only file (AOF)** persistence but with some severe limitations:

- AOF is not supported on Redis 2.8.22 and later
- AOF is not supported for *cache.t1*, *cache.t2* or *cache.t3* instances
- AOF is not supported on Multi-AZ replication groups

...and depending on your use case, these restrictions might speak strongly against using AOF.

The last option is probably also the most attractive one: **Multi-AZ with auto-failover**. This is basically a primary/replica system across availability zones with automatic failover (replica promotion). When the primary crashes or requires an upgrade, the replica is promoted in its place. You can also decide to promote a replica manually. Multi-AZ with auto-failover has its own limitations:

- Not supported for Redis 2.8.5 or earlier
- Not supported for *cache.t1* instances
- The failover comes with a little downtime -- I assume due to DNS propagation
- When failing over, a small amount of data might be lost due to replication lag

Then again, these limitations are less of a burden than the AOF restrictions and having your Redis cluster running across different availability zone is definitely a plus.

The price for the **cheapest instance that can enable AOF** is $74.16 per month for a previous generation *cache.m3.medium* instance or $133.92 per month for a current generation *cache.m5.large* instance.

The price for the **cheapest instance that you can hook up into a Multi-AZ auto-failover system** is $13.68 per month for a *cache.t3.micro* instance, which makes the cheapest Multi-AZ auto-failover cluster cost $27.36 per month.

If you're looking for cheap/small but persistent, then Multi-AZ is the way to go.

Last but not least, the default parameter groups in ElastiCache are tailored for **volatile caches**, so `maxmemory-policy` is set to `volatile-lru`. You might want to change this to `noeviction` if you'd like to control when your data gets deleted.

In **conclusion**, the solution I'd recommend in order to keep your AWS ElastiCache Redis data as persistent as possible would be:

- Opting for a Multi-AZ auto-failover system with at least 2 nodes.
- Enabling daily backups with as much retention as needed.
- Setting the `maxmemory-policy` directive to `noeviction` inside your ElastiCache parameter group.

*The information describe above was collected around February 2020. You should consult the official AWS documentation for any changes beyond this date. All prices correspond to the Frankfurt region.*

### Reference

- <https://redis.io/topics/persistence>
- <https://redis.io/topics/lru-cache>
- <https://aws.amazon.com/elasticache/redis/faqs/>
- <https://github.com/awsdocs/amazon-elasticache-docs/blob/master/doc_source/redis/AutoFailover.md>
- <https://aws.amazon.com/premiumsupport/knowledge-center/fault-tolerance-elasticache/>
- <https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/RedisAOF.html>

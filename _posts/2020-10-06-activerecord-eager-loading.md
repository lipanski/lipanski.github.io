---
layout: default
title: "Some things you should know about eager loading in ActiveRecord"
favourite: true
tags: ruby
comments: true
cover: /assets/images/cyprinus-dobula.jpg
---

# {{ page.title }}

When squashing N+1 queries on a large code base, you might realize that tracking down *all* the associations that need to be eager loaded can be really tedious. Your code has to be *instrumented properly* and most of the times you need to reason about every single query, *one by one*. On top of that, eager loading can be fussy: calling `where`, `order` or `limit` on your associations might invalidate your eager loading efforts in some *unexpected* ways.

This article will present an [**automated way of dealing with N+1**](#automatic-eager-loading) queries and it will show you [**how to go around some of the limitations of eager loading**](#things-that-break-eager-loading-where-order-limit) in ActiveRecord. Furthermore, it will show you [**how to write tests**](#preventing-n1-regressions-with-tests) to prevent those sneaky N+1 queries from coming back.

## Automatic eager loading

[Goldiloader](https://github.com/salsify/goldiloader) is a gem that eager loads your associations *automatically* and *only when needed*.

Just add it in your `Gemfile`:

```ruby
gem "goldiloader"
```

...and watch as your associations are *magically* eager loaded:

```ruby
> users = User.limit(5).to_a
# SELECT "users".* FROM "users" LIMIT 5

> users.each { |user| user.posts.to_a }
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" IN (1,2,3,4,5)
```

Notice how there was **no need to explicitly call `includes(:posts)`** when querying users. Without Goldiloader, that second call would have triggered *five* queries instead of one. With Goldiloader, you basically don't have to think about calling `includes` any more.

**Goldiloader pairs very nicely with GraphQL APIs**. The moment your API allows for querying associations or associations of associations, you have a little *N+1 nightmare* on your hands. GraphQL APIs with various different clients are hard to optimize because they might be used in so many different ways. Integrating something like [graphql-batch](https://github.com/shopify/graphql-batch) could address the problem, but you have to apply it to every individual case and it's a more intrusive solution.

On top of that, if you're working on a large code base or if you're inexperienced about dealing with N+1 queries, Goldiloader might give you a nice performance boost at a low cost, albeit with some limitations which we will discuss in the next part.

## Things that break eager loading: `where`, `order`, `limit`

Applying `where`, `order` or `limit` clauses on your ActiveRecord associations will break eager loading, whether you're using Goldiloader or not.

> In order to make the following examples more generic, I'll be using `includes` calls instead of Goldiloader. If you're using Goldiloader, simply remove them or consider them redundant. 

Let's see what happens when we try to `order` our posts:

```ruby
> users = User.includes(:posts).limit(5).to_a
# SELECT "users".* FROM "users" LIMIT 5
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" IN (1,2,3,4,5)

> users.each { |user| user.posts.order(:created_at).to_a }
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 1 ORDER BY "posts"."created_at" ASC
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 2 ORDER BY "posts"."created_at" ASC
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 3 ORDER BY "posts"."created_at" ASC
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 4 ORDER BY "posts"."created_at" ASC
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 5 ORDER BY "posts"."created_at" ASC
```

Even though calling `includes(:posts)` produced an `IN` query which seems to cover all our posts, applying the `order` clause on our association ignored this and triggered a bunch of N+1 queries. In order for eager loading to work, **the eager loaded query should match the query required to fetch your association**. 

One way to avoid this is by moving the `order` inside a **default scope**:

```ruby
class Post
  default_scope { order(:created_at) }
end

> users = User.includes(:posts).limit(5).to_a
# SELECT "users".* FROM "users" LIMIT 5
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" IN (1,2,3,4,5) ORDER BY "posts"."created_at" ASC

> users.each { |user| user.posts.to_a }
# No queries here, they've been eager loaded already
```

...another way is by moving the `order` inside the **parent association**:

```ruby
class User
  has_many :posts, -> { order(:created_at) }
end

> users = User.includes(:posts).limit(5).to_a
# SELECT "users".* FROM "users" LIMIT 5
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" IN (1,2,3,4,5) ORDER BY "posts"."created_at" ASC

> users.each { |user| user.posts.to_a }
# No queries here, they've been eager loaded already
```

...and yet another way is by moving the `order` inside a **scoped parent association**:

```ruby
class User
  has_many :ordered_posts, -> { order(:created_at) }, class_name: "Post"
end

> users = User.includes(:ordered_posts).limit(5).to_a
# SELECT "users".* FROM "users" LIMIT 5
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" IN (1,2,3,4,5) ORDER BY "posts"."created_at" ASC

> users.each { |user| user.ordered_posts.to_a }
# No queries here, they've been eager loaded already
```

Let's see what happens when we apply a `where` condition to our posts:

```ruby
> users = User.includes(:posts).limit(5).to_a
# SELECT "users".* FROM "users" LIMIT 5
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" IN (1,2,3,4,5)

> users.each { |user| user.posts.where("created_at < ?", 1.week.ago).to_a }
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 1 AND (created_at < '2020-09-25 08:55:05.919824')
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 2 AND (created_at < '2020-09-25 08:55:05.919824')
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 3 AND (created_at < '2020-09-25 08:55:05.919824')
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 4 AND (created_at < '2020-09-25 08:55:05.919824')
# SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 5 AND (created_at < '2020-09-25 08:55:05.919824')
```

Ok, that didn't work. But we can fix it exactly the same way.

You can move the `where` condition inside a **default scope**:

```ruby
class Post
  default_scope { where("created_at < ?", 1.week.ago) } 
end

> users = User.includes(:posts).limit(5).to_a
> users.each { |user| user.posts.to_a }
```

...or inside the **parent association**:

```ruby
class User
  has_many :posts, -> { where("created_at < ?", 1.week.ago) }
end

> users = User.includes(:posts).limit(5).to_a
> users.each { |user| user.posts.to_a }
```

...or inside a **scoped parent association**:

```ruby
class User
  has_many :posts_from_one_week_ago, -> { where("created_at < ?", 1.week.ago) }
end

> users = User.includes(:posts_from_one_week_ago).limit(5).to_a
> users.each { |user| user.posts_from_one_week_ago.to_a }
```

Mastering eager loading basically means **relying more on default scopes** and **applying scopes inside your associations** rather than using those scopes directly.

What about `limit` and `offset`? Depending on your use case, you could apply the same techniques. But there's one big use case that just doesn't fit here: **pagination**. How would you supply the page number? And while we're at it, how would you deal with `where` conditions or **scopes that contain a variable**?

## Turn your queries around

Unlike scopes, **default scopes and scoped associations don't take arguments**. If we'd like to provide our associations with an outside parameter while avoiding N+1 queries, we'll have to think of something else. 

Consider the following code which produces N+1 queries:

```ruby
# An outside parameter
time = 1.week.ago

users = User.includes(:posts).to_a
users.each { |user| user.posts.where("created_at < ?", time).to_a } # N+1!
```

In such cases, you can turn your queries around and **let the association become the main subject of that query**:

```ruby
# An outside parameter
time = 1.week.ago

posts = Post.includes(:user).where("created_at < ?", time).to_a
# SELECT "posts".* FROM "posts" WHERE (created_at < '2020-09-27 10:03:23.478773')
# SELECT "users".* FROM "users" WHERE "users"."id" IN (1,2,3,4,5)

users = posts.map(&:user)
# No queries here, they've been eager loaded already
```

## Preventing N+1 regressions with tests

Given a large code base, introducing more N+1 queries is fairly easy -- as easy as writing `user.posts` inside a loop, somewhere deep inside a template, without remembering to also eager load the association in the controller. But you can write tests to prevent that...

First, let's settle on the behaviour we'd like to test: **any request should trigger at most one SQL query per table**. In order to track and count all these queries, we could use ActiveSupport instrumentation to hook up to the `sql.active_record` event, the same way ActiveRecord is [tested](https://github.com/rails/rails/blob/6-0-stable/activerecord/test/cases/test_case.rb).

I already packaged this in a tiny gem called [sql_spy](https://github.com/lipanski/sql-spy).

Just add it to your `Gemfile`:

```ruby
gem "sql_spy"
```

...and let's write our first **controller test**:

```ruby
require "sql_spy"

class PostsControllerTest < ActionDispatch::IntegrationTest
  test "GET /posts should not trigger N+1 queries" do
    # Add some realistic test data here

    queries = SqlSpy.track do
      get "/posts"
    end

    select_queries_by_model = queries.select(&:select?).group_by(&:model_name)
    # We only want the SELECT queries and we'd like them grouped by model

    assert select_queries_by_model.all? { |_, queries| queries.count <= 1 }
    # Our tolerance rate is 1 query per table
    # You can increase this value depending on your business logic
  end
end
```

Some things are worth mentioning here. **Your test is only as correct as the test data you provide it with**. Ideally try to keep a realistic set of fixtures around (3-4 examples of each core model) or create a realistic environment before the test run.

Some requests might genuinely be triggering **more than one query per table**. For example, paginated requests usually produce two queries: one to fetch the records, another one for the page count. In such cases, you can **increase the tested tolerance rate, while also increasing the number of records per table** in your test setup, to make sure you're still catching those N+1 queries.

You can introduce these N+1 regression tests to **every critical or data-intensive controller action**. Their setup is fairly cheap, while N+1 queries can come at a very high cost for your database and your response times.

## Conclusion

The best results will come from applying a combination of the solutions above. Golidloader will get rid of some of your N+1 queries, but you'll also need to start writing your associations with eager loading in mind. Testing for N+1 regressions and proper instrumentation will keep your hard-won performance improvements intact.

What do we say to the God of N+1 Queries? *Not today.*

## Links

- [Rails Guides: Eager Loading Associations](https://guides.rubyonrails.org/active_record_querying.html#eager-loading-associations)
- [Goldiloader](https://github.com/salsify/goldiloader): Automatic eager loading for ActiveRecord
- [SqlSpy](https://github.com/lipanski/sql-spy): Track SQL queries performed inside a block of code
- [Automatic eager loading for the Sequel ORM](https://sequel.jeremyevans.net/rdoc-plugins/classes/Sequel/Plugins/TacticalEagerLoading.html)
- [Bullet](https://github.com/flyerhzm/bullet): N+1 monitoring on steroids
- [rspec-sqlimit](https://github.com/nepalez/rspec-sqlimit): An RSpec matcher to check the total number of performed SQL queries
- <https://stackoverflow.com/a/46421504>
- <https://github.com/rails/rails/blob/6-0-stable/activerecord/test/cases/test_case.rb>
- <https://stackoverflow.com/a/5492207/801186>

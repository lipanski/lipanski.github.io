---
layout: default
title: "One Ruby file to rule them all: inline gems and inline ActiveRecord migrations"
favourite: true
tags: ruby
comments: true
cover: /assets/images/salmo-thymallus.jpg
---

## {{ page.title }}

Ruby is my go-to language for scripting. It's simple, concise and delivers the expected results without much hassle. Though definitely not a good idea for large applications, having everything in one file can be pretty neat when using Ruby for scripting. It makes sharing, installing and running your Ruby code easier.

This post focuses on ways to build a self-contained single-file Ruby web app that uses a database and performs migrations at run time. This is not really what you'd call *scripting*, but the techniques are pretty much the same. Some of them should be taken with a grain of salt, but they are at least interesting.

### Inline gems

A while back, Bundler [introduced](https://bundler.io/v2.0/guides/bundler_in_a_single_file_ruby_script.html) the possibility to declare gems from within Ruby files. Running such a file would automatically install any missing gems.

Here's how it works:

```ruby
#!/usr/bin/env ruby

require "bundler/inline"

gemfile do
  source "https://rubygems.org"
  gem "sinatra", "2.0.5"
end

puts "Sinatra was installed!", Sinatra::VERSION
```

You can either make this script executable or call it via `ruby my-script.rb`. In any case, it will make sure to install **and require** the listed gems (in this case `sinatra`), before running the rest of your Ruby code.

Note that there is no concept of `Gemfile.lock` in the world of inline Bundler, so a good practice is being specific about the gem versions you want installed.

### Inline ActiveRecord migrations

Let's say your script or web app requires a database. ActiveRecord migrations can also be inlined, though it's probably not very common. Here's how:

```ruby
# Define a couple of migrations as part of the same file
class CreateEventTableMigration < ActiveRecord::Migration[5.2]
  # Add the magic sauce
  def self.version
    1
  end

  def change
    create_table :events do |t|
      t.string :name
    end
  end
end

class AddEventCreatedMigration < ActiveRecord::Migration[5.2]
  def self.version
    2
  end

  def change
    change_table :events do |t|
      t.datetime :created_at
    end
  end
end

# Perform migrations
migrations = [CreateEventTableMigration, AddEventCreatedMigration]
ActiveRecord::Migrator.new(:up, migrations).migrate

# Define your model(s)
class Event < ActiveRecord::Base; end
```

The main difference from writing migrations as separate files is the need to define the `version` class method. Under normal circumstances, this would point to the file name of the migration and this is how ActiveRecord keeps track of the performed migrations, but also their designated order. The method should therefore return something unique and sortable - like a number that you increase with every new migration.

In your normal Rails or Sinatra app, you'd perform migrations by running `rake db:migrate`. For our self-contained single-file Ruby app, we will perform them automatically. We do this by calling:

```ruby
# You can maintain this list yourself or use `ActiveRecord::Migration[5.2].subclasses`
migrations = [CreateEventTableMigration, AddEventCreatedMigration]
ActiveRecord::Migrator.new(:up, migrations).migrate
```

This operation is idempotent.

Last but not least, you might have noticed there's no explicit database connection in our code, but we don't want to add a `database.yml` file, as it goes against our self-imposed single-file mantra. There's a little ActiveRecord convention that can help us out here: the `DATABASE_URL` environment variable. You can use it to specify the database of your choice:

```sh
DATABASE_URL=postgres://dbuser:dbpass@locahost:5432/dbname ./my-script.rb
```

### Inline ActiveRecord roll-backs

Rolling back the migrations can be achieved by changing the direction argument on the `ActiveRecord::Migrator` call to `:down`:

```ruby
migrations_to_roll_back = [CreateEventTableMigration, AddEventCreatedMigration]
ActiveRecord::Migrator.new(:down, migrations_to_roll_back).migrate
```

### Alternative: Inline ActiveRecord schema loading

As pointed out by Janko Marohnić in the comments, an alternative to the previously described database migration process would be to perform a schema loading, similar to what ActiveRecord does when you call `rake db:schema:load`. The result looks simpler:

```ruby
ActiveRecord::Schema.define do
  create_table :events do |t|
    t.string :name
  end

  change_table :events do |t|
    t.datetime :created_at
  end
end
```

...but there's a catch: **the code is not idempotent** and will fail when run a second time.

Fortunately ActiveRecord does provide us with [the means](https://apidock.com/rails/v4.2.7/ActiveRecord/ConnectionAdapters/SchemaStatements/column_exists%3F) to make this idempotent. You'll only need to be a bit more explicit:

```ruby
ActiveRecord::Schema.define do
  unless table_exists?(:events)
    create_table :events do |t|
      t.string :name
    end
  end

  unless column_exists?(:events, :created_at)
    change_table :events do |t|
      t.datetime :created_at
    end
  end
end
```

In this case, I'm using the `table_exists?` and `column_exists?` methods to avoid running my migrations a second time. Note that I've also preserved the incremental nature of my migrations - new migrations can be added to the `#define` block without interfering with the old ones.

### Inline everything!

Here's how the final result looks like:

```ruby
#!/usr/bin/env ruby

require "bundler/inline"

gemfile do
  source "https://rubygems.org"
  gem "sinatra", "2.0.5"
  gem "sinatra-activerecord", "2.0.13"
  gem "pg", "1.1.4"
end

class CreateEventTableMigration < ActiveRecord::Migration[5.2]
  def self.version
    1
  end

  def change
    create_table :events do |t|
      t.string :name
    end
  end
end

class AddEventCreatedMigration < ActiveRecord::Migration[5.2]
  def self.version
    2
  end

  def change
    change_table :events do |t|
      t.datetime :created_at
    end
  end
end

# Perform migrations
migrations = [CreateEventTableMigration, AddEventCreatedMigration]
ActiveRecord::Migrator.new(:up, migrations).migrate

# Define your model
class Event < ActiveRecord::Base; end

set :port, 3000

get "/events/last" do
  event = Event.last
  next "{}" unless event

  event.to_json
end

post "/events" do
  event = Event.create(name: params[:name] || "unknown", created_at: Time.now)

  event.to_json
end

Sinatra::Application.run!
```

Prerequisites:

```sh
# Create the Postgres database
createdb single-file-example

# Make the script executable
chmod +x my-script.rb
```

This is how you'd run the script:

```sh
DATABASE_URL=postgres:///single-file-example ./my-script.rb
```

When executed, the script will:

- Install any missing gems.
- Require the listed gems.
- Perform database migrations on the database specified via `DATABASE_URL`.
- Run a Sinatra application on your local port 3000 that responds to `GET /events/last` and `POST /events`.

### References

- <https://bundler.io/v2.0/guides/bundler_in_a_single_file_ruby_script.html>
- <https://github.com/bundler/bundler/blob/master/lib/bundler/inline.rb>
- <https://github.com/rails/rails/blob/master/activerecord/lib/active_record/tasks/database_tasks.rb>
- <https://apidock.com/rails/v4.2.7/ActiveRecord/ConnectionAdapters/SchemaStatements/column_exists%3F>

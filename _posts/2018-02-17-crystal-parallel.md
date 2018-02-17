---
layout: default
title: "Crystal: Raising exceptions from Fibers, the parallel macro and invalid memory access"
tags: crystal
comments: true
---

## {{ page.title }}

Concurrency can be achieved in Crystal by using *Fibers*. Communication between *Fibers* is handled via *Channels*. The [documentation](https://crystal-lang.org/docs/guides/concurrency.html) on these topics is quite comprehensive so I won't go into detail here.

This post will focus on the `parallel` macro, present one of its drawbacks when dealing with unhandled exceptions and introduce a solution: the `parallel!` macro.

### The parallel macro

One useful tool that didn't make it into the Crystal Book is the `parallel` macro. It allows firing up and waiting for several concurrent jobs in a more succinct manner:

```crystal
def say(word)
  puts word
end

parallel(
  say("a"),
  say("b"),
  say("c"),
)
```

The real beauty of this macro comes when you're interested in capturing the return values of your concurrent jobs:

```crystal
def say(word : String) : String
  word
end

a, b, c =
  parallel(
    say("a"),
    say("b"),
    say("c"),
)

puts a, b, c
```

### Raising exceptions in Fibers and usage of `uninitialized` in `parallel`

Exceptions raised from *Fibers* don't propagate to the main thread. Though there are ways to re-raise these exceptions, the `parallel` macro doesn't implement this.

For the same reason, the following code will get you in trouble:

```crystal
def say(word : String) : String
  raise Exception.new("boom")
  word
end

a, b, c =
  parallel(
    say("a"),
    say("b"),
    say("c"),
  )

puts a, b, c
```

This program will crash because of an `Invalid memory access (signal 11)`.

The problem here lies in the usage of the `a`, `b`, `c` variables on the last line, after the *Fibers* silently swallowed the exception. More specifically, the `parallel` macro implementation (which is [quite easy to read](https://github.com/crystal-lang/crystal/blob/v0.24.1/src/concurrent.cr#L131)) has to define these variables before actually evaluating them. This is achieved by initially marking them as `uninitialized`:

{% raw %}
```crystal
{% for job, i in jobs %}
  %ret{i} = uninitialized typeof({{job}})
  # ...
{% end %}
```
{% endraw %}

Yes - Crystal [allows declaring uninitialized variables](https://crystal-lang.org/docs/syntax_and_semantics/declare_var.html), and no - it's probably not the best idea to use this [unless you know what you're doing](https://github.com/crystal-lang/crystal/issues/4544#issuecomment-307612363). This is *unsafe* code.

Also note that placing checks before using the variable:

```crystal
if a
  puts a
end
```

...will not solve the issue and [there's no way](https://github.com/crystal-lang/crystal/issues/4544#issuecomment-307635912) to tell if a variable is initialized or not.

As long as your *Fibers* don't raise unhandled exceptions, you can safely use the `parallel` macro. The moment you start raising undhandled exceptions, you'll want to replace the `parallel` macro with some more explicit code:

```crystal
def say(word : String) : String
  raise Exception.new("boom")
  word
end

channel = Channel(Nil).new

a : String? = nil
b : String? = nil
c : String? = nil

spawn do
  a = say("a")
ensure
  channel.send(nil)
end

spawn do
  b = say("b")
ensure
  channel.send(nil)
end

spawn do
  c = say("c")
ensure
  channel.send(nil)
end

3.times { channel.receive }

# In this case a nil-check is not needed, but for other method calls you might need it.
puts a, b, c
```

Notice how we had to compromise on the type of our variables here. Also, this solution is pretty verbose.

### Re-raising exceptions and the `parallel!` macro

I'm not a big fan of verbose (which is why I really enjoy Crystal and Ruby), so it was time for a macro to hide away all this code. Because I didn't want to compromise on the variable type, I decided to re-raise the exceptions to the main thread.

Make way for the `parallel!` macro:

{% raw %}
```crystal
macro parallel!(*jobs)
  %channel = Channel(Exception | Nil).new

  {% for job, i in jobs %}
    %ret{i} = uninitialized typeof({{job}})
    spawn do
      begin
        %ret{i} = {{job}}
      rescue e : Exception
        %channel.send e
      else
        %channel.send nil
      end
    end
  {% end %}

  {{ jobs.size }}.times do
    %value = %channel.receive
    if %value.is_a?(Exception)
      raise %value
    end
  end

  {
    {% for job, i in jobs %}
      %ret{i},
    {% end %}
  }
end
```
{% endraw %}

> Note that prepending variable names with `%` inside macros will generate unique names for them, so that they won't collide with your other variable names outside the macro implementation.

The implementation is mostly based on the original `parallel` implementation, the difference being that exceptions from *Fibers* will be re-raised in the main thread.

The following code will raise `Exception.new("boom")` even before the last line, instead of that nasty `Invalid memory access`:

```crystal
def say(word : String) : String
  raise Exception.new("boom")
  word
end

a, b, c =
  parallel!(
    say("a"),
    say("b"),
    say("c"),
  )

puts a, b, c
```

### Links

* [Concurrency in Crystal](https://crystal-lang.org/docs/guides/concurrency.html)
* [The parallel macro implementation](https://github.com/crystal-lang/crystal/blob/v0.24.1/src/concurrent.cr#L131)
* [Uninitialized variables in Crystal](https://crystal-lang.org/docs/syntax_and_semantics/declare_var.html)
* [Issue explaining invalid memory access](https://github.com/crystal-lang/crystal/issues/4544)
* [Crystal 0.7.0 release notes, which introduced the %var macro notation](https://github.com/crystal-lang/crystal/releases/tag/0.7.0)

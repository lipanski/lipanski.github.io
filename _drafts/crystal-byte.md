alias MyType = String | Time | Nil

Use sentry.

---

Use the shard environment variable for shards.

---

Compare slices:

```crystal
other = Bytes[0x03, 0xd9, 0xa2, 0x9a, 0x67, 0xfb, 0x4b, 0xb5]
data = Bytes.new(8)
io.read_fully(data) # or io.read(data)
data == other
```

Read type:

```crystal
data = Bytes.new(2)
io.read_fully(data)
converted_data = IO::ByteFormat::LittleEndian.decode(UInt16, data)
```

Shorter way to read type:

```crystal
converted_data = io.read_bytes(UInt16, IO::ByteFormat::LittleEndian)
```

Try:

```crystal
something.try(&.first_level).try { |obj| obj.second_level }.try(&.third_level)
```

not_nil!:

```crystal
something : String | Nil = "test"
puts something.not_nil!.size
```

Access exception object:

```crystal
def something
  raise "boom"
rescue e : Exception
  puts e.message
end
```

Unique variables in macros - see https://github.com/crystal-lang/crystal/releases/tag/0.7.0

```crysal
%var = 1
```

---

case/when with instance method calls:

```crystal
case kind
when .scalar?
  read_anchor @event.data.scalar.anchor
when .sequence_start?
  read_anchor @event.data.sequence_start.anchor
when .mapping_start?
  read_anchor @event.data.mapping_start.anchor
when .alias?
  read_anchor @event.data.alias.anchor
else
  nil
end
```

---

Class as a restriction

- see https://crystal-lang.org/docs/syntax_and_semantics/type_restrictions.html

```crystal
def something : Array(MyClass.class)
end
```

---

Private shards in shards.yml:

```
git: git@github.com:user/repo.git
```

...instead of `github: user/repo` - otherwise you get `https://`.

---

No garbage collection:

```crystal
require "lib_c"
require "lib_c/i686-linux-gnu/c/stdlib"
require "lib_c/i686-linux-gnu/c/stdio"

def free(object)
  LibC.free(pointerof(object))
end

class String
  def to_unsafe
    pointerof(@c)
  end
end

class Foo
  def bar
    LibC.printf "Hello, World!\n"
  end
end

f = Foo.new
f.bar
free(f)
```

Compile with:

```sh
crystal build app.cr --prelude="empty" -p --release --no-debug
```

Source: <https://perens.com/2018/07/06/tiny-crystal-language-programs/>

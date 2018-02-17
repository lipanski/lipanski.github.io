alias MyType = String | Time | Nil

Use sentry.

---

Use the shard environment variable.

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

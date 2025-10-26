---
layout: default
title: "Linux: How to change the file encoding from the command line"
tags: linux
---

# {{ page.title }}

Let's say you have a file encoded as ISO-8859-16 and would like to turn it into UTF-8.

You can use `iconv` to accomplish that:

```sh
iconv -f ISO_8859-16 -t UTF-8 -o output.txt input.txt
```

In case you'd like to convert to or from a different encoding, you can list all encodings known to *iconv* by calling:

```sh
iconv -l
```

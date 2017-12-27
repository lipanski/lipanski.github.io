---
layout: default
title: "Linux: How to make your own keyboard layout"
tags: linux
---

## {{ page.title }}

### History time

Apart from the Latin alphabet, the Romanian language uses five special letters: *ă*, *â*, *î*, *ș*, *ț*.

Due to the lack of Romanian hardware keyboards back in the day (which hasn't changed much), one of the most popular Romanian keyboard mapping is fully compatible with US keyboards. The special letters are produced by pressing `AltGr` and the corresponding Latin letter, with the exception of "ă", which is attached to `AltGr + Q`. This is called the *Romanian (Programmers)* layout and is the de facto Romanian layout in the Linux world. Pairing the extra glyphs to the Latin keys is quite convenient, especially for programmers and people generally accustomed with the US layout.

On the other hand, the German layouts expect a German hardware keyboard. When using a US keyboard, the default layout maps `'`, `;`, `[` to produce *ä*, *ö*, *ü*. This is definitely not something that plays well with US keyboards or with my brain.

In my ideal world, I would trigger the German *umlauts* with the `AltGr` key on top of the US layout and have the Romanian characters work side by side, the same way. There's definitely no layout for that. Fortunately, creating your own custom layout is quite easy in Ubuntu.

### How to make your own custom layout

Layouts are contained within the `/usr/share/X11/xkb/symbols` directory.

Let's start by copying the Romanian layout (feel free to choose any other layout as your starting point):

```sh
sudo su
cd /usr/share/X11/xkb/symbols
cp ro fl
```

I'm calling my layout `fl`. If you open this file, you'll see lots of lines that look like this:

```
key <AD01> { [ q, Q, acircumflex, Acircumflex ] };
```

These lines are the key bindings - they connect a particular key on your keyboard to the value that will be printed out.

The first part `key <AD01>` represents **the key** that we want to map. The code `<AD01>` has the following meaning:

- The first letter, `A`, points to the *alphanumeric* key block. Other options include `KP` for the keypad and `FN` for the function keys.
- The second letter, `D`, points to the row. The count starts with the row containing the space bar, which would be row `A`.
- The last two digits represent the column of the key, starting with `01`, going from left to right and ignoring special keys (like `TAB` or `CapsLock`).

So `key <AD01>` would point to the *first key* (ignoring the `TAB`) of the *forth row* of the *alphanumeric block*, which on most keyboards is the letter `Q`.

The second part, `[ q, Q, acircumflex, Acircumflex ]`, points to the character that will be printed out, for the various combinations of `Shift` and `AltGr`:

1. No combination: q
2. `Shift`: Q
3. `AltGr`: â (or *a circumflex*)
4. `Shift + AltGr`: Â (or *A circumflex*)

If don't intend to use the `Shift` and/or `AltGr` combinations, you can simply leave those positions out.

Note that for representing the printed value you can either use the entity name - like `q` for the letter *q* and `acircumflex` for the letter *â* - or the [Unicode control codes](https://en.wikipedia.org/wiki/List_of_Unicode_characters#Control_codes) - like `U00E5` for the letter *å*.

Based on these conventions, it's quite easy to either add new bindings or modify existing ones.

In my case, I've added the following lines to the default block, in order to extend the Romanian layout with the German special letters:

```
// Print ö when pressing o + AltGr
key <AD09> { [ o, O, odiaeresis, Odiaeresis ] };

// Print ü when pressing u + AltGr
key <AD07> { [ u, U, udiaeresis, Udiaeresis ] };

// Print ß when pressing z + AltGr
// Because there's no capital ß in the German language, I've left out the Shift + AltGr combination
key <AB01> { [ z, Z, ssharp ] };
```

I've also edited the existing binding for the letter *w*:

```
// Print ä when pressing w + AltGr
key <AD02> { [ w, W,  adiaeresis, Adiaeresis ] };
```

Once you're done, you can switch to your new layout by calling:

```sh
setxkbmap fl
```

### Links:

- [The keyboard layout file from my dotfiles](https://github.com/lipanski/dotfiles/blob/master/usr/share/X11/xkb/symbols/fl)
- <https://askubuntu.com/questions/510024/what-are-the-steps-needed-to-create-new-keyboard-layout-on-ubuntu>

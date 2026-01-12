---
layout: post
title:  "Lua Quines"
date:   2023-02-25
tags:
- lua
- quine
- fun
aliases:
- "../2023/02/25/lua-quines.html"
---

## Cheating Quines

### The Trivial Quine

The empty string `` technically is a quine in most scripting languages since it outputs itself - nothing, the empty string - when run.

### A Debug Quine

Reads its own source. May not work if the source file has been moved.

Also works if the quine is not loaded from a file
(i.e. loaded from a string), as long as no "name" has been passed
(doesn't work in the Lua REPL, since there the source will be `=stdin`).

```lua
local source = debug.getinfo(1).source
print(source:match"^@" and io.open(source:sub(2)):read"*a" or source)
```

It could however be combined with `load`:

## A `load` Quine

Works on Lua 5.1 and later (`loadstring or load` rather than just `load`).

```lua
s="print(('s=%qt=%q%s'):format(s,t,t))"t="(loadstring or load)(s)()"(loadstring or load)(s)()
```

## A Proper "Constructive" Quine

Simply uses a format string to construct itself.

```lua
s="s=%q;print(s:format(s))";print(s:format(s))
```

---

Thanks to [luk3yx](https://github.com/luk3yx) for pointing out broken quines & suggesting fixes.

---
layout: post
title:  "Lua Quines"
date:   2022-07-31 22:35:00 +0100
tags:
  - lua
  - quine
---

## Cheating Quines

### The Trivial Quine

The empty string `` technically is a quine in most scripting languages since it outputs itself - nothing, the empty string - when run.

### A Debug Quine

Reads its own source. May not work if the source file has been moved.

Also works if the quine is not loaded from a file.

```lua
local source = debug.getinfo(1).source
print(source:match"^@" and io.open(info:sub(2)):read"*a") or source)
```

## A `load`(string) Quine

Works in Lua 5.1 and later.

```lua
function f(s) print(s) (loadstring or load)(s) print(("f%q"):format(s)) end
f"function f(s) print(s) (loadstring or load)(s) print((\"f%q\"):format(s)) end"
```

## A Proper "Constructive" Quine

Simply uses a format string to construct itself.

```lua
s="s=%q;print(s:format(s))";print(s:format(s))
```

---
layout: post
title: "Syntactic Sugar in Lua"
date: 2023-07-26
tags: lua, syntactic sugar, programming
---

Lua is a very minimalistic language.
This doesn't mean that it lacks syntactic sugar though;
plenty of constructs native to other languages are "emulated"
using more powerful constructs in Lua and thus need sugar
for Lua to look & feel nice - Lua has more syntactic sugar than you may realize!

## Table Indexing

Lua's one-in-all datastructure is the *table*, which is array and hash map in one.

Since tables are used to implement data structures including "instances" of "classes",
accessing "fields" must be convenient. Lua thus provides the `.` indexing sugar:

```lua
field = table.name
```

is equivalent to

```lua
field = table["name"]
```

where `name` is a valid Lua identifier; similarly,

```lua
table = {name = value}
```

is equivalent to

```lua
table = {["name"] = value}
```

### Environmental Variables

All variables that are not declared local are "environmental" (typically global) by default.

This means `name_a = name_b` is just syntactic sugar for `_ENV.name_a = _ENV.name_b`.

## Functions

### Declarations

The typical "named" function declaration

```lua
function name(...) end
```

is just shorthand for the assignment

```lua
name = function(...) end
```

this works for locally declared variables just as well:

```lua
local name
function name(...) end
```

is still equivalent to

```lua
local name
name = function(...) end
```

or in short (assuming the function doesn't recurse):

```lua
local name = function(...) end
```

More sugar is provided to allow local functions to recurse. Consider

```lua
local f = function() return f() end
```

as the right-hand side is evaluated first, `f` inside the function body refers to the environmental variable `_ENV.f` rather than the local variable `f`.

The solution is to split declaration & assignment such that `f` becomes an upvalue of the function that is later assigned to it:

```lua
local f
f = function() return f() end
```

since recursive local functions are frequently required, Lua provides `local function` sugar for this:

```lua
local function f() return f() end
```

Functions being table fields (where the table possibly is the environment `_ENV`) don't require such a sugar
\- they only need access to the table containing them, which is usually provided as an upvalue (often `_ENV`).

All functions in Lua are anonymous; Lua actually figures out the function name to show in error messages
by looking at from where (local / global variable / table field) the function was accessed.

Additionally, names support the dot indexing sugar:

```lua
function x.y.z(...) end
```

is shorthand for

```lua
x.y.z = function(...) end
```

arbitrary expressions (including `[x]` indexing) are however not allowed;
finally, Lua supports adding a trailing `:method`

```lua
function x.y.z:method(...) end
```

which is equivalent to `.method` plus an implicit `self` parameter:

```lua
function x.y.z.method(self, ...) end
```

to easen metatable-based "OOP" for instance methods, which can then be registered as `function class:method(...) end`.

### Calls

#### Implicit Self ("OOP")

Consistent with the `:` sugar for adding an implicit `self` parameter when declaring a function,
Lua also offers the `:` sugar to add an implicit `self` argument when calling a function obtained through indexing:

```lua
prefixexp:name(...)
```

is roughly equivalent to

```lua
(function(self) return self.name(self, ...) end)(prefixexp)
```

a particularly cursed example of this (which helps illustrate that this really just syntactic sugar)
is `_G:pairs()`, which will loop over the global variables.

#### Single String or Table Argument

If the only argument is a string or table, Lua allows omitting the call parentheses; parentheses-less calls can be chained:

```lua
f"double-quoted string"
f'single-quoted string'
f[[long string]]
f{table = true}

f{"chaining"}"works""just"'fine'[[yay]]
```

equivalent to

```lua
f("double-quoted string")
f('single-quoted string')
f([[long string]])
f({table = true})

f({"chaining"})("works")("just")('fine')([[yay]])
```

The omission of parentheses around "table calls" in particular makes Lua very suitable for implementing DSLs.

Side note: Chaining may be (ab?)used to implement ropes without requiring a separator (like `,` or `;` in tables).
This enables a neat hack to implement indented multiline strings:

```lua
local lines = {__call = function(self, str)
	if not str then return table.concat(self, "\n") end
	table.insert(self, str)
	return self
end}
local function L(str)
	return setmetatable({}, lines)
end
```

Usage is straightforward:

```lua
local multiline = L
	"a"
	"couple"
	"lines"
()
```

## Metatables

Metatables allow you to change how Lua's built-in operators work,
effectively making operators syntactic sugar for function calls.

This is very flexible; besides the use of the `__index` metamethod to emulate OOP,
metatables are probably most commonly used to implement the arithmetic operators
e.g. for vectors or custom number types (fractions, big integers),
Lua itself sets a metatable using `__index` on strings to enable the OOP-style `str:method(...)` syntactic sugar.

Even more, using `debug.setmetatable` you can largely alter operations on builtin Lua types
such as `nil`, booleans, functions, threads, so long as they aren't Lua-defined already.

## Conclusion

Lua has just the right syntactic sugar it needs to give you the most bang for the buck,
enabling it to be very minimalistic while remaining very readable and rather easy to parse.

---
date: 2026-02-18
title: "On string concatenation"
description: "A frequent culprit for bad performance and how to avoid it"
---

When building strings, perhaps the first operation programmers learn,
one of the few operators which tend to be introduced early on, is *string concatenation*.
A simple, straightforward way to build strings.

However, in modern programming languages, strings are typically *immutable*, to avoid many of the surprises that
may occur with mutable strings (and mutability in general).
An advantage of this is that it enables string interning, which saves memory if many strings are duplicated
and makes string comparisons constant time.

This is especially important in languages like Lua which use strings *a lot* as hash map (table) keys:
Indexing tables *needs* to be effectively constant time to make using tables as "structs" viable. [^lua]

[^lua]: This has changed a bit: Newer Lua versions only intern short strings.

Given immutable strings and a simple array-backed string implementation,
string concatenation always has to copy the two strings to be concatenated fully, making it a linear time operation
that takes time proportional to the length of the result.
(With string interning, this is also rather hard to avoid.)

Among the standard operations, this makes string concatenation the "odd one out":
In Lua for example, arithmetic, logical, bitwise and comparison operators typically all take constant time [^strs].
Indexing a table is expected constant time.
The length operator can take logarithmic time on tables, which already gets plenty of criticism.

[^strs]: There is another exception, also related to strings: Lexicographic comparison of strings, e.g. via `<`, takes linear time again.

Programs often process containers (e.g. lists or strings) linearly, building up some string as they go.
It is common to, say, process some string character-by-character,
producing some other string, e.g. escaping or unescaping a string.
As such, you inevitably get some kind of loop which builds a string.
In Lua, this might look as follows (where `...` is some iterator):

```lua
local s = ""
for x in ... do
	s = s .. x
end
```

The issue with this is that it takes quadratic runtime, which is often unacceptable:
If there are many `x`s to concatenate, this will become very slow.

This performance issue is frequently found in the wild - for example, [in Go code](https://github.com/charmbracelet/glow/pull/505).
It's one of the easiest ways to ruin linear-ish time complexity of serialization.

## String builders

The solution is to use an appropriate data structure to buffer the string being built, fittingly called a "string builder".

This is effectively just an "array list" (in Java terms) of characters which is grown using powers of two,
making the "append character" operation amortized constant time (vs. linear time for string concatenation).

In Lua and many other scripting languages, a list (table) of strings to be concatenated serves as a string builder:

```
local t = {}
for x in ... do
	table.insert(t, x)
end
local s = table.concat(t)
```

Languages with mutable strings can take another approach.
Ruby for example has both a `+` operator which does not modify the destination, and a `<<` operator which does. [^cpp]
In such languages, strings are effectively dynamic character arrays ("array lists"), making them suitable for use as string builders.

[^cpp]: In some other languages, this operator is `+=`.
C++ has both "ostringstreams", which are effectively string builders,
and mutable strings that themselves have a capacity and can be appended to.

## Why do people not use string builders?

It turns out that JavaScript engines typically optimize this blunder away by deferring string concatenation.
The downside of such an optimization is that it may result in surprising performance characteristics.

For example, you can cheat at benchmarks by comparing a solution which eagerly pays the cost of string concatenation against one which doesn't (but which will most likely have to pay them sometime later).

Let's look at a concrete example: Consider this piece of code

```javascript
let s = "";
for (let i = 0; i < 1e7; ++i)
	s += "a";
console.log(s.length);
```

This runs efficiently in linear time on any serious JS engine,
because string concatenation is deferred, as you would like for it to be.
Under the hood, some kind of data structure that is essentially a linked list of string fragments is being built.
Only when we ask for the length of the complete string is this data structure traversed and turned into a string.

Now let's see what happens when we add a harmless sanity check.
String indexing surely is a cheap constant time operation, so this should not alter the runtime of our program, right?

```javascript
let s = "";
for (let i = 0; i < 1e7; ++i) {
	s += "a";
	if (s[i] != "a") throw "what";
}
console.log(s.length);
```

On my machine this takes something like 8 hours in Node.js (V8).
That's because we're suddenly up to *quadratic time*:
The string indexing forces the JS engine to actually carry out the deferred string concatenation in every iteration.
So every iteration produces a string in linear time, just to take a look at the last character.

Thus this optimization is very brittle. It is not something programmers should start relying on.
All it does is build bad habits. Damn you, JavaScript!

## Use string builders

Not using a string builder in a loop that has no good upper bound on the number of iterations is rarely justified.
If done right, using a string builder should both be more efficient and more readable.

Often code can be elegantly rewritten to functional-style using widely available standard library helpers
that concatenate a list / iterator of strings (e.g. `table.concat`, `str.join`, and so on).

Beware also of using string builders too narrowly.
It's good to recognize and optimize the simple "string concatenation in a loop" pattern,
but you need to look beyond this and think about the concatenation operations your program might be doing at runtime.

For example, recursive serialization can still result in quadratic time complexity in the depth of the objects to be serialized.
If you write recursive serialization code, you should usually pass around a string builder rather than returning strings.

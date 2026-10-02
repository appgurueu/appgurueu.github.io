---
layout: post
title: "Variadic functions in Lua"
date: 2026-04-21
tags:
- lua
- programming
- performance
---


- Command-line arguments

### Implementation details

#### Alternatives

### Perfect forwarding

### Iterating varargs

When iterating varargs, there are three concerns:

* Order;
* Handling of `nil` arguments;
* Performance

Consider the following example:
We want to implement a function `sum` that computes the sum of its arguments.
`sum()` should be `0`. `nil` arguments, e.g. `sum(1, nil, 3)`, but also `sum(1, nil)`,
should preferably produce an "attempt to perform arithmetic on a nil value" error.
Order of summation is usually irrelevant, though we would prefer consistent left-to-right.

A quite natural way to iterate varargs is using a recursive function,
especially if you are familiar with functional programming languages:

```lua
local function sum(x, ...)
    if x == nil then
        return 0
    end
    return x + sum(...)
end
```

This aborts the summation at the first missing, or `nil`, argument.
`sum(1, nil, 3)` would silently produce `1`, which is not ideal.

We can repair this using the built-in function `select`, which lets us check if the argument list is empty:
`select("#", ...)` counts the number of arguments passed in `...`.
Furthermore, we can use `select(2, ...)` to get the vararg starting at the second argument,
that is, the "tail" of the vararg without the first element, which we get implicitly by truncation
when we use `...` in an expression (as left operand of `+` in this example). [^select2]

[^select2]: Note that we could easily implement `select(i, ...)` ourselves,
whereas we could not reliably implement `select("#", ...)` if there was no built-in function for it
(excepting `table.pack(...).n`).

```lua
local function sum(...)
    if select("#", ...) == 0 then
        return 0
    end
    return ... + sum(select(2, ...))
end
```

Semantically, this is pretty much exactly what we want.
It sums up the arguments left-to-right and errors on `nil`.

Unfortunately, it still has terrible performance characteristics.
As we've seen earlier, there is no "magic" behind typical vararg implementations (e.g. in PUC Lua or LuaJIT).
Arguments to a function are passed as a contiguous slice of values on the stack.
Functions are told how many arguments are on the stack when they are called.

This means that every time we pass `...` to `select`, we are copying the entire vararg `...` on the stack!
`select("#", ...)` copies the entire vararg, just to throw it away and return its count.
When running `select(2, ...)`, `...` is copied to the stack, and `select` simply reports that it wants
to return the topmost $n-1$ elements, corresponding to the last $n-1$ arguments,
which are then copied to call `sum`.

That is, in every iteration, both variants perform linear time operations in the size of the vararg;
*and* the entire vararg is left on the stack every time!
Both memory usage and runtime are thus quadratic, which is terrible.

Knowing `select`, we can get rid of the recursion. The following solution has only linear memory usage at least:

```lua
local function sum(...)
    local s = 0
    for i = 1, select("#", ...) do
        s = s + select(i, ...)
    end
    return s
end
```

Still, the vararg `...` may be copied on every invocation of `select`, making this quadratic time.
However, there is reason to be optimistic: `select(i, ...)` is immediately truncated to the
first return value, meaning it effectively is just a way of getting the `i`-th argument.
An optimizing Lua compiler could see this and avoid copying `...` in its entirety, just to get at the `i`-th argument.
And indeed, LuaJIT does optimize this to linear time!
Which is why, for a long time, I used to believe that this `select`-loop was the "blessed" way of vararg iteration;
and at least on LuaJIT, it clearly is (as we will see in some benchmarks later).

But for PUC Lua, we need to find a different solution. On all versions up to 5.5,
we effectively do not get around copying our vararg to a table which we can then iterate efficiently.

A naive approach looks something like

```lua
function sum(...)
    local s = 0
    for _, v in ipairs{...} do
        s = s + v
    end
    return s
end
```

But this stops at the first `nil` value, which is not ideal.
`#t` may stop at *any* `nil`, introducing potential for nondeterminism.
If we use `pairs`, we simply ignore `nil` values instead, but we also lose any order guarantee.

A proper approach queries the length of the vararg using `select`:

```lua
function sum(...)
    local t = {...}
    local s = 0
    for i = 1, select("#", ...) do
        s = s + t[i]
    end
    return s
end
```

But we still copy `...` twice, when we would like to copy and count it at once.
Since Lua 5.2, `table.pack` lets us do that:

```lua
function sum(...)
    local t = table.pack(...)
    local s = 0
    for i = 1, t.n do
        s = s + t[i]
    end
    return s
end
```

Finally, as of Lua 5.5, there even is a way to directly name the vararg,
which can subsequently be worked with like a packed table:

```lua
function sum(...t)
    local s = 0
	for i = 1, t.n do
        s = s + t[i]
	end
    return s
end
```

If you only index the "vararg table" by `t.n` and `t[i]`, these operations are
[guaranteed to be optimized such that no unnecessary table is created](https://www.lua.org/manual/5.5/manual.html#3.4.11).
(Effectively they just correspond to indexing the stack and getting the number of arguments.)

On the one hand, we now finally have a canonical way to iterate varargs efficiently.
On the other hand, it is clearly largely redundant with `table.pack` and `select`.
I might have preferred a reliable peephole optimization of `select`, similar to what LuaJIT turns out to do:
The construct was already there, it just needed to be optimized properly.
New syntax is not really necessary.

### Constructing varargs

There are essentially two ways to construct a vararg: By `unpack`ing a table, or by a recursive function.

A recursive function is not a good option except for very small varargs: It again runs in quadratic time.
`return v, ...`, as limited as it is, still typically copies the entirety of `...`, even though it is not obvious
why it would need to (`v` could be allocated on top of the stack and arguments could be numbered from top to bottom, no?).

So really the only reliable way is to go through a table again.

```lua
function countdown(n)
	if n == 0 then
		return
	end
	return n, countdown(n - 1)
end

function countdown2(n)
	local t = {}
	for i = 1, n do
		t[i] = n - i + 1
	end
	return table.unpack(t, 1, n)
end

countdown(1e5)
```

### Mapping varargs

Mapping is now just a combination of iteration and construction; since those are best done using a table, so is mapping.

### Packing and unpacking varargs

There is a canonical packing for a vararg, which is a table of the form `t = {n = select("#", ...), ...}`. [^nit]
On Lua 5.2+, `table.pack(...)` does this.

[^nit]: Careful: `{..., n = select("#", ...)}` will not work, `...` will be truncated to a single element.

This can be turned into a vararg again via `unpack(t, 1, t.n)`
(called `table.unpack` in 5.2+, even though it does not default to `t.n` for the length).

There are some alternatives, like storing a vararg on the stack of a coroutine,
but none of them are competitive; they have much more overhead while offering a much more awkward and limited interface.

## Conclusion

If you want to work with varargs, you often end up packing them up as tables at one point or another.

You should strongly consider tables.

Optimizations like [allocation sinking] make tables less expensive than they might appear to be.
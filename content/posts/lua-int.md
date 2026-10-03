---
date: 2026-10-02
title: "How Lua got integers wrong"
---

Lua 5.3 introduced a 64-bit integer type to a language that had previously only known a single number type, 64-bit floats.
This resulted in language design decisions which subvert Lua's minimalist philosophy, introduce new pitfalls for Lua programmers, subtly harm backwards compatibility, and make Lua harder to implement efficiently.

## Why?

Why have 64-bit integers at all?

Lua is a simple language. It should stay that way.
Any new feature should be sufficiently important, sufficiently necessary, to justify the additional complexity.
Before the introduction of "integers", Lua had just a single "number" type, backed by 64-bit floats.
This works surprisingly well. A large range of integers can be represented exactly as floats.
There is no need to insert casts or distinguish between `1` and `1.0`.
If you need specific, exact number types like fractions, big integers or big decimals,
you have the metamethods to implement them yourself. [^warts]

[^warts]:
    There are a few minor warts if you are trying to write number-type agnostic algorithms, mostly surrounding equality testing.
    For example, the simple check `x == 0` will always fail if `x` is not the number `0`,
    because Lua's `==` operator requires that both arguments have the same metatable,
    otherwise the `__eq` metamethod will not be called.
    Comparisons via `<`, `>`, `<=` and `>=` however will call `__lt` / `__le` metamethods
    even if the types do not match, meaning you could work around this problem
    by writing the (likely less efficient) `x <= 0 and x >= 0` or similar.

Are 64-bit integers important enough to justify complicating numbers? I would say no, not really.
An idiomatic, pure Lua program can generally do *very well* without 64-bit integers.

Because IEEE754 [^nitpicks] 64-bit floats have 53 mantissa bits (one of which is implicit), they can represent all integers
from \(-2^{53}\) to \(2^{53}\) precisely.
You can go up to \(9007199254740992\). That is on the order of fifteen zeros.
Unlike for two's complement integers, this range is even symmetric, because the representation uses a sign bit.
64-bit floats can represent all the values, and do all the arithmetic you can do, on a fictive "54-bit signed integer" type.
If you incremented a counter *every microsecond*, it would take you about *300 years* to get to the upper limit.

Going from 54 to 64 bits gives you a measly 10 extra bits, about three additional decimal digits. Is it worth it?
Which enormous -- but not quite too enormous -- quantity are you storing such that 54 bits are not enough, but 64 bits are?
<!-- The weight of your mother?
There's no way that fits in 64 bits either. -->

But okay, okay, I get it, we don't live in our perfectly simple pure Lua world.
We live in a world where 64-bit integers are a thing.
64 is a nice power of two and made a lot of sense as the next step up from 32-bit hardware
(though do note that addresses are usually at most 48 bit).
Along with the hardware, many systems programming languages use 64-bit integers,
and so there is some need to support them well for decent interfacing.

We might be given a 64-bit ID, or a 64-bit seed for a random number generator, or a 64-bit checksum in some file we're trying to read.
While in many cases we can sidestep the problem (e.g. by storing IDs in strings instead),
it *is* useful to be able to work with 64-bit integers as a number type.
But which operations are typically done on this type?

[^nitpicks]: Yes, technically Lua could be compiled with other number types, but this is the default on all reasonable configurations.
Lua with a non-IEEE754-64-bit-float number type is not "standard Lua" as far as I'm concerned.

## Bit operations

Bit operations are most naturally defined on arrays of bits,
and those map straightforwardly to (unsigned) integers.
So that's what we need 64-bit integers for!

Or is it?

Recall that a large number of integers can be embedded just fine in 64-bit floats.
So why not just do that?

We can cast a float to an integer, say 32-bit, maybe throw an error if out of range,
perform a bit operation, then pack up the result in a 64-bit float again.
This is perfectly well defined, perfectly easy to understand, and perfectly useful.
JavaScript did it and got away with it.
Why can't we do the same and live happily ever after with a single number type?

Turns out we can. In Lua 5.1 land, we have been doing just this.
Simply grab Mike Pall's delightfully simple [LuaBitOp](https://bitop.luajit.org/) library,
which provides a bunch of useful and efficient `bit.*` functions.
These functions are already included in LuaJIT, where they benefit from additional optimizations.
What more do you need?

Bit twiddling is not needed terribly often. It is employed primarily as an optimization
(e.g., bitsets instead of much more expensive table-based sets; checksum or compression algorithms),
often in scenarios where you would prefer for the problem in question to be solved
by an implementation in a hosting systems language (e.g. C) anyways.
Thus I do not think that additional operators are warranted; `bit.*` is all you need [^perforomance].
If you need large arrays of bits, throw in a dedicated "bitset" data structure implemented
efficiently in C as userdata or via FFI in LuaJIT.

[^perforomance]: If standard library function calls are too expensive, that is an implementation problem. Go use LuaJIT.

And mind you: Despite the introduction of 64-bit integers and bitops,
Lua neglected to expose the really cool stuff used for advanced bit twiddling.
Not even as standard library functions!
There is no "count leading zeros" (CLZ [^clz]).
No "population count" (number of set bits).
No looking at the bit representation of a float [^sort].
The only one we got is unsigned-less-than (`math.ult`).
You need to implement them yourself (in C) if you really want them.
(Which to be fair does align with Lua's philosophy.)

[^clz]: aka "bit width", a rounded base 2 logarithm; useful to tell you the highest bit by which two integers differ in combination with XOR, which in turn has applications in specialized algorithms and data structures, e.g. for compressed quadtrees.

[^sort]: This is useful e.g. for radix sort and variants.

## Arithmetic safety

We need not talk about basic float-float operations; those are essentially all well defined by IEEE-754.
(For how `%` and `^` are defined, check out the Lua reference manual.)

Let's start with the most basic thing: Arithmetic operations on just integers.

Say, addition and subtraction. Okay, so we do integer operations.
But what if the result doesn't fit?
We could throw an error. We could convert to float, maybe use plus/minus infinity to represent a result that doesn't fit?
Okay, that's a bit strange, let's not do that. Let's rewind.
Let's *simply make arithmetic be modulo \(2^{64}\).*

Yes, you read that right. Silent two's complement wrapping integer arithmetic *in a scripting language*, by default.
Because of course, the average programmer who adds two large numbers wants the sum to be an even smaller number. Sometimes.

This is fucked up. Much more fucked up than getting some rounded, but still at least *not smaller* floating point result.
Even modern systems programming languages that implement similar behavior for performance reasons (e.g. Zig, Rust)
usually *at least offer the option to trap it in debug mode*.
This behavior is clearly considered extremely undesirable.
Virtually no popular scripting language other than Lua does this.

Consider, for example, Dijkstra's shortest path algorithm.
With float arithmetic, the algorithm degrades somewhat gracefully, because the sum between the distance of a point to the source
and the cost of an edge, \(d(v) + c(v, w)\), is still at least as large as \(d(v)\).
Maybe you get some slightly inaccurate shortest paths, but they won't be that far off.
With wraparound, funny shit happens.
<!-- TODO maybe a bad example -->

It's hopefully not hard to accept that silent wraparound is not good for software stability *at all*,
especially as we must consider the possibility of adversaries throwing some funny numbers at us
to provoke an unanticipated wraparound.

Can wrapped arithmetic be useful? Sure, sometimes, usually because someone else thought it would be useful,
or as an optimization trick in a low-level implementation of some low-level function (like a checksum).
But then provide *explicit* ways to do that. Those can be separate operators (like in Zig),
though a scripting language doesn't really deserve this complexity, especially not for such a rarely needed feature.
With the appropriate optimizations, `math.*` functions would do the trick just fine.

## Compatibility

The way Lua implements arithmetic operations differently for integers and floats means that a mostly hidden number type now matters.

This makes it cumbersome to ensure that a Lua 5.2 or older program that is being upgraded *still does basic arithmetic operations safely*.

All operations could now very well do something slightly different, depending on the (implicit) number type that flows through them.

There are some funny consequences, mostly related to the fact that there are now *redundant*, but still *subtly different* representations
*for all integers from \(-2^{53}\) to \(2^{53}\)*: While they do compare as equal, `42` is not the same as `42.0`, `42/6` is not the same as `7`, and it may very well matter which one you write; `tostring` now adds a trailing `.0` to floats as a subtle type indicator.

One example:
In Lua 5.3, for some numbers `a` and `b`, you can have `a == b`, but `2 * a ~= 2 * b`,
e.g. `(1 << 62) == 2^61`, but `2 * (1 << 62) ~= 2 * 2^61`.

Exponentiation (`^`) always casts to float, so `x^2` is always a float, and *not equivalent* to `x * x`.

We also get a new `//` operator for integer division. Curiously, the result need not be an integer;
if one of the two operands is a float, the result is again a float.
So unless you need to compute a large (but not too large) integer quotient,
`x // y` is just a way of writing `math.floor(x / y)`.

This sucks, because we really want to use floats for integers, and not pretend they're some different thing;
they are an extension of integers after all.

Here is an especially funny consequence. You see, tables need to use keys by identity.
But we also have optimizations such as the array part under the hood.
Clearly [^why-though], `t[1.0]` should be the same as `t[1]`, and typically will be stored in the first slot of the array part.
But oops! We've just thrown away our knowledge about the key type.

[^why-though]: `t[1]` should be the same as `t[1.0]` since `rawequal(1, 1.0)` is `true`; we want `t[k]` to be consistent with raw equality.

To deal with this, Lua simply stipulates the following:

> To avoid ambiguities, any float used as a key that is equal to an integer is converted to that integer.
> For instance, if you write `a[2.0] = true`, the actual key inserted into the table will be the integer `2`.

So get this:

```lua
local x = 42.0
local t = {[x] = 1}
local k = next(t)
assert(k == 42) -- works as expected
print(k, x) -- 42 42.0
assert(k + (1 << 63) == x + (1 << 63)) -- fails
```

Oh how lovely! You can put a value in a table as a key and get the same value, but behaving slightly differently in arithmetic, out.

Don't put implicit wraparound arithmetic into a scripting language. It's just wrong.

## Implementation details

Under the hood, when you implement a scripting language, you need to represent values somehow.

Conceptually, values are a *tagged union*: You have a type *tag*, and some contents inside, the interpretation of which depends on the type tag.
In Lua, the type will be store one of the eight types: nil, boolean, number, string, table, userdata, function, or thread (coroutine).
Corresponding to these are no value, a single bit that indicates "true" or "false", a 64-bit floating point number, or a pointer,
which is what all garbage-collected "object" types (string, table, userdata, function, thread) use.

A naive implementation would use a struct consisting of a one-byte type tag, plus an 8-byte union for the actual contents.
(Note that all of the possible contents mentioned above fit in 64 bits, so that's what the size of the union will be.)

If you do this, your representation of a value will probably be 16 bytes large on a 64-bit machine.
A whopping 7 bytes are wasted just on padding of the type field.

You can then compensate by optimizing your table memory layout to pack the type tags separately and more tightly,
which is what newer Lua versions do, but it adds work.

And for a just-in-time compiler, you really want a single value to fit in a single register.
So what do you do instead?

On most 64-bit systems, a pointer is not actually 64 bits.
That would be absurd: No one will ever have that much addressable memory,
and virtual memory means we don't need tricks to wastefully allocate the space either.
So practically, pointers only have up to 48 significant bits, which means quite a few bits at our disposal!
More than enough to store a type in, yay!

We could now "box" everything, meaning allocating everything on the heap, implementing primitive types as reference types.
We can store the type either in the object on the heap, or in the pointer, depending on how wasteful we feel.
CPython for example boxes everything. It makes things simple in the implementation.
But it's also one major contributing factor for why it is so slow.
You really want basic arithmetic to just operate on values directly, and not to incur memory access costs, costs of cache misses, costs of heap allocations.
So let's not do that.

If we look at the lists carefully, we can see that floats are effectively the only problem; everything else is small enough.
So we're going to pretend *everything* is a float. That leaves pointers to be encoded and distinguished somehow.
And that's where a trick famously known as *NaN boxing*, standard among JavaScript JIT compilers and employed by LuaJIT, comes in.
You use the fact that the IEEE-754 standard designates all floats with a certain exponent as "Not-a-Number" (NaN).
In such NaN floats, you can store whatever you want in the mantissa.
We take one bit to tell us whether we have a "real" float NaN at hand.
Add some more bits for the type tag, and we still have plenty left for a pointer.

NaN boxing is a lovely hack that lets numbers be cheap in scripting languages, without wasting space.

But once you introduce 64-bit integers, this hack stops working:
There is simply no way to fit all the 64 bits of a 64-bit integer in the 52 mantissa bits.

So if you really want 64-bit integers, you'll have to put them on the heap. [^🤓]
Which sucks, because performance reasons such as efficient bitops are a big part of why we care about 64-bit integers to begin with.
You don't want the "supposedly efficient" number type to actually be the inefficient one.
And if they incur such additional costs, should they not be explicit? [^nit]

[^🤓]: Oops, you caught me! I made a silly off-by-one error with the mantissa bits! Except I didn't.
I'm deliberately not counting the implicit leading one mantissa bit here because we can't use it, as the exponent is fixed.
We really only have 52 mantissa bits at our disposal to pack information into.

[^nit]: There are probably ways to make this work, e.g. to NaN-box small integers (say, up to 48 bits or something),
and to only put the (rare) larger ones on the heap, but it won't be pretty, and it would likely result in funny performance characteristics.

## A better way

LuaJIT shows a better way. It implements explicit opt-in boxed 64-bit integers with wraparound arithmetic
as a FFI "cdata" type. A simple `LL` suffix makes an integer a 64-bit FFI integer.
All the usual arithmetic operations work on these. `+`, `-` and `*` are the usual integer operations
you would expect, with two's complement wrapping. Division and modulo are integer too;
division by zero produces funny numbers, as does modulo zero. (Perhaps an error would be preferable here.)
Integer exponentiation (`^`) is also implemented.
Coercion is always from numbers to 64-bit integers.

## Big integers?

Lua could go even further. Like Python, or Haskell, or a few other languages,
it could go all the way and include first-class support for big integers.
This would make for a better case of orthogonality between arbitrary-size integers and limited-size floats.

It would feel nice in that many integer computations would "just work",
no worry about overflow, no worry about inexact results, numbers just grow large, operations get slow.
But that is also exactly the downside.
For instance, there is no longer a guarantee that a simple arithmetic operation is constant time;
you need to be careful to cover your ass against various [DoS attacks](https://lwn.net/Articles/907572/).

Again this would go against Lua's goal of minimalism.
Big integers can and should be provided by libraries, backed by userdata or tables.
They do not need hardcoded language support. They are too special-purpose to deserve that.
For purposes which aren't some funny online puzzles, i.e. cryptography,
you often need additional special properties, e.g. constant time operations, anyways.

## Conclusion

When it comes to integers, bitops ~~and all the rest~~ in Lua,
I think Mike Pall made the right call.
The introduction of integers in such a hazardous manner
is one of the reasons I prefer older Lua versions.

I don't blame the Lua authors for choosing the current direction.
They saw a need for 64-bit integers and addressed it by means of a language feature,
which is somewhat natural for language designers.
However, using the existing userdata mechanism with operator overloading (as LuaJIT mostly does)
would have been sufficient.

The addition of 64-bit integers has made Lua slightly more useful,
at the regrettable cost of a bit of elegance, a bit of simplicity, and a bit of reliability.

---
date: 2026-05-03
title: "In defense of the table"
hidden: true
---

Two data structures are essential to programming:

*Lists* of items allow efficient iteration, as well as insertion and removal of items at one end.
The most commonly used *array lists* also allow efficient "random access"
(retrieval or replacement of an item at a given position).

*Maps* associate arbitrary keys with values. Usually they are implemented as *hash maps*.
Maps also offer asymptotically efficient insertion, removal, random access, and iteration.

Some scripting languages like Lua unify array lists and hash maps in a single data structure,
the *table*, also known under the name of "associative array" in other languages.

This is often mistaken for some kind of historical "quirk" to put up with,
when in fact it is a very deliberate design decision with considerable upsides.
So let's look at why tables are a worthwhile abstraction:
What motivates the concept of a table,
which tricks go into an efficient implementation,
and some concrete code examples where tables shine.

## Lists are redundant

Say we start with a hash map.
We can simply use keys \(1, ..., n\) and assign to them some values.
Such a hash map can be used very similarly to an array with \(n\) items.
We still have constant time insertion, deletion, and random access. [^exp]
Hash map access will have significant overhead,
but this is the kind of performance concern scripting languages often tolerate:
It's "just" a "modest" constant factor.

[^exp]: We omit the term "amortized" here.
The constant time access of a hash map is technically an expectation of a random variable
depending on the set of keys and the order of insertion;
in theory, a single access might take up to linear time if enough keys collide.
But practically, this is almost never relevant.
The adversarial scenarios where it might be are relatively easy to mitigate with high probability.
And in any case, a half decent hash function should do well on the range \(1, ..., n\).

Hash map iteration lets you visit all the items in an unspecified order;
ordered iteration can be done simply by iterating keys \(1, 2, ...\)
until you get a key without an associated value.
The latter is exactly what `ipairs` does in Lua.

A bonus is that you can construct the hash map in any order.
You don't need to bother with only adding items at the end, or preallocating an array to the right size.

The following just works and does exactly what you think it would:

```lua
local function iota(n)
	local t = {}
	for i = 1, n do
		t[i] = i
	end
	return t
end
```

This is part of the reason Lua corresponds so naturally to most pseudocode.

To fully obsolete array lists, we still need to do some more work.

## The length operator

There is a problem if you try to implement array lists like this:
You do not know the length.
A priori, it is not easy to efficiently add elements at the end without an additional counter.

It turns out that there is a neat algorithmic trick that can be employed to efficiently query the length
for a contiguous sequence of keys \(1, ..., n\) given just a map that tells you whether there is a value associated with a key:
You simply define the "length" as any index \(i\) such that there is a value associated with the key \(i\),
but where the following index, \(i+1\), is not associated with any value.
Such an index \(i\) is called a *boundary*.
If no value with index \(1\) exists, the sequence is considered to be empty; the length is defined to be \(0\).

This can now be computed efficiently with a binary search, exploiting the fast random access:
Given an upper bound \(u\) and a lower bound \(l\), we look at the midpoint \(m = \left \lceil \frac{u+l}{2} \right \rceil\).
If no value exists at that position, we know that there exists a boundary \(i\) with \(i < u\);
Otherwise, we know that there exists a boundary \(i\) with \(i \geq u\).
We apply this repeatedly, halving the size of the search interval until we have a boundary.

Under the hood, the hash table implementation will be keeping track of the capacity,
which for the most part linearly corresponds to the actual size
and gives you a decent upper bound \(u\). [^nit]

Thus we can compute length in *logarithmic* time.

We would however like constant time, so we do normally *cache* this length;
this cache can be eagerly updated when an item is added or removed at the end,
or lazily when the length is queried.

This is effectively what the length operator `#t` does in Lua.

[^nit]: Except if you delete many key-value pairs,
because Lua does not shrink tables if you only delete entries.
Technically you do not even need this upper bound, as you can do a so-called "exponential search"
where you keep doubling \(u\) until you get a key for which no corresponding value exists,
but that gives you much worse bounds for pathological examples;
in the end this is only upper bounded by the absolute maximum table size,
which will be on the order of \(2^{30}\) or more, meaning up to about \(30\)
steps for a single length query.

## Fixing performance

We still have a problem: Hash maps are not nearly as efficient as array lists.
Besides all the extra logic adding overhead -- having to hash keys, needing to search for free slots, to compare keys once found, etc. --
hash maps result in memory access patterns. The larger they get, the worse cache locality becomes.
A simple "linear" traversal of the keys \(1, ..., n\) -- as well as the linear construction in that order -- jumps all over memory.
For large enough hash maps, you get orders of magnitude more cache misses and massive slowdowns just from this.

As array lists, and linear operations on them, are *extremely* common in code,
this translates to significant performance penalties across the board --
from the hashing overhead in the case of small tables,
and additionally from the cache misses in the case of big tables.

Thus, while asymptotically there is essentially no difference --
both array and hash map access are "constant time" --
it would not be acceptable to leave this much performance on the table (hah!),
especially as array lists are straightforward to implement.

### Counting keys

Thus, Lua employs another trick:
We can try to see if a hash map is being used in an "array-like" manner and if so,
optimize it to be stored much more efficiently by using an array under the hood.

This is done by keeping both an "array" part and a "hash" part.
Both have a capacity, which usually is a power of two and starts out at zero.
The array part only stores the values (some of which may be `nil`) corresponding to the keys \(1, ..., a\),
up to its capacity \(a\); all other key-value pairs are stored in the hash map.

When a new key-value pair is to be inserted into the hash part,
it will eventually happen that it doesn't fit because the capacity is exhausted.
At that point, we need a *rehash*: We need to rebuild the hash table, doubling its capacity and spreading out the items.
But if we visit all key-value pairs anyways, we might as well look at the keys.
We can count how many keys in a certain range we have in buckets of the form
\([1, 1], [2, 3], [4, 7], [8, 15], ...\) -- that is, \([2^l, 2^{l+1} - 1]\) for \(l = 0, 1, ...\). [^clz]

[^clz]: This can be done by looking at the bit pattern of \(k - 1\) and determining the highest set bit,
which is a standard bit operation (known as "count leading zeros" (CLZ) in x86-64,
abstracted as `std::bit_width` in the C++ standard library).

This gives us a small histogram of logarithmically many counts.
We can then compute all the prefix sums in this array, which gives us the counts for the ranges \([1, 2^k - 1]\) for \(k = 0, 1, ...\);
we then look for the largest bucket that is at least half full (or however dense we want our arrays to be at minimum).
This gives us a suitable array capacity; we set up an array and migrate all corresponding key-value pairs there.
Then we proceed with the rehash.

Let that sink in:
The choice of whether to use a hash map or an array becomes a mere *implementation detail*;
the best of both worlds is chosen for you *at runtime*, based on the density of the data.

Say you have some IDs. You can map them to some objects simply via a table.
If your IDs turn out to be dense enough, Lua will notice, and will use an array for most (or all) of them.
If things change and your IDs are suddenly sparse, no problem, Lua will use a hash map.

## The good

We can get rid of the need of a dedicated "array list" in a scripting language as far as the user is concerned.
But why would we do this?

Ultimately, because the "associative array" is a good abstraction of what we, mathematically speaking, would call a "finite map":
We just want to map some keys to some values; whether we use an array or a hash map for that does not really matter all that much.
We can write the same code, and depending on what our keys are, a suitable data structure can be used under the hood.
Indices and strings are both just labels. Why should we not be able to use both as we please?

The basic interface is the same: We can

* Set a key-value pair (`t[k] = v`)
* Get a value corresponding to a key (`t[k]`)
* Iterate all key-value pairs (`for k, v in pairs(t) do ... end`)

A table used as a list can be iterated in order using `for i, v in ipairs(t) do ... end`;
the length can be determined using `#t`.

We also consolidate our treatment of absent values.
In Lua, `t[k]` is simply `nil` ("nothing") if there is no such key-value pair;
`t[k] = nil` can be used to erase a key-value pair.

The fact that `t[k]` defaults to `nil` turns out to be surprisingly neat.
Often, you want to have some fallback, for which you can simply do `t[k] or fallback`;
if on the contrary you do require `k` to be a valid key,
simply do `assert(t[k])` (both assuming `t[k]` can not be `false`).

Some neat consequences of this consolidation of interfaces are best seen through examples.
We will see how some simple operations on maps generalize nicely
if we abstract finite maps as a single data structure.

### Copy

Consider a shallow copy:

```lua
local function copy(t)
	local t2 = {}
	for k, v in pairs(t) do
		t2[k] = v
	end
	return t2
end
```

This copies all kinds of tables, just like you would expect. It works for lists and maps alike.
Why would it not?

In fact, if `t` is stored in the array part, `pairs` will effectively behave like `ipairs`,
so `t2` will be constructed in order and thus live in the array part as well.

### Map inversion

Or for another example: Inverting a map.

```lua
local function invert(t)
	local inv = {}
	for k, v in pairs(t) do
		inv[v] = k
	end
	return inv
end
```

Again this is beautifully polymorphic. It just operates on the right level of abstraction.

You can use this to invert a permutation:

```lua
print(table.concat(invert({2, 3, 1}), " ")) -- 3 1 2
```

but you could use it just as well on any other invertible map:

```lua
local wife = {tim = "anne", tom = "hanne"}
local husband = invert(wife)
print(husband.anne, husband.hanne) -- tim tom
```

you can easily build a map from indices this way:

```lua
local rank = invert({"tim", "tom", "tam"})
print(rank.tam) -- 3
```

and conversely, given a ranking, you can reconstruct a list:

```lua
local ranking = invert({tim = 1, tom = 2, tam = 3})
print(table.concat(ranking, " ")) -- tim tom tam
```

all in a single function! If you strictly separated lists and maps,
you would normally need something like 4 different functions for all of this
to cover all possible combinations of in and out types.

### Map composition

Another good example is *composition*: Let `t3 = compose(t2, t1)`.
Then `t3[k]` should be `t2[t1[k]]` for every key `k` in `t1`.
I'll leave this as an exercise to you:

How would you implement this?
What is it useful for?
Why would a separation of lists and maps be annoying here?

<details>
<summary>Possible implementation</summary>
```lua
local function compose(t2, t1)
	local t3 = {}
	for k, v in pairs(t1) do
		t3[k] = t2[v]
	end
	return t3
end
```
</details>

Now let's look at some more interesting examples.

### Segment tree upsertion

Assignment to a table field (`t[i] = ...`) covering both updates and insertions (in short: "upsertions")
facilitates the implementation of upsertions in derived data structures.
The below helper function does upsertion in a segment tree storing sums of ranges:

```lua
local function upsert(layers, i, w)
	layers[1][i] = w
	for li = 2, #layers do
		i = math.ceil(i / 2)
		layers[li][i] = layers[li-1][i*2 - 1] + (layers[li-1][i*2] or 0)
	end
end
```

If adding new elements to an array (vs. updating existing elements)
required use of a special "append" method, the "update" and "append" case
would need to be separated, and this would become much less neat to write.
(Note also how out-of-bounds accesses producing `nil` is neat here;
a simple `or 0` takes care of the lack of a right sibling in a layer.)

### 2d map transposition

By nesting tables, you obtain a natural representation of 2-dimensional finite maps;
\(f(x, y)\) translates to `f[x][y]` (or `(f[x] or {})[y]` for a sparse map).
Such a map is often thought of as a *matrix*, with \(x\) and \(y\) the headers of rows and columns respectively.
A frequently useful operation is *transposition*: We want to construct a function \(g(y, x) := f(x, y)\)
which simply swaps the arguments before applying \(f\).
In matrix terminology, we think of it as swapping the roles of rows and columns, reflecting the matrix at its diagonal.

Implemented using tables in Lua, that looks as follows:

```lua
local function transpose(mat)
	local trans = {}
	for row_key, row in pairs(mat) do
		for col_key, val in pairs(row) do
			trans[col_key] = trans[col_key] or {}
			trans[col_key][row_key] = val
		end
	end
	return trans
end
```

For a silly example, suppose you ran a deeply unserious business.
You have a series of datapoints of months and money burnt in that month.
A reasonable way to store this is as a list of pairs:

```lua
local points = {
	{month = 1, money_burnt = 1e11},
	{month = 2, money_burnt = 2e12},
	{month = 3, money_burnt = 3e13},
	-- stonks!
}
```

... but perhaps the tool you use for plotting demands two lists of coordinates,
one for \(x\) and another for \(y\) coordinates. 
`transpose` lets you conveniently convert between the two representations:

```lua
local series = transpose(points) -- {month = {1, 2, 3}, money_burnt = {1e11, 2e12, 3e13}}
plot({x = series.month, y = series.money_burnt})
```

### General matrix multiplication

As we've just seen, a matrix is effectively nothing but a two-dimensional finite map,
which can be represented naturally by nesting finite maps.

Now consider the following code for a general matrix multiplication:

```lua
local function matmul(A, B)
	local C = {}
	for i, row in pairs(A) do
		for k, va in pairs(row) do
			for j, vb in pairs(B[k] or {}) do
				C[i] = C[i] or {}
				C[i][j] = (C[i][j] or 0) + va * vb
			end
		end
	end
	return C
end
```

This computes the [matrix product](https://en.wikipedia.org/wiki/Matrix_multiplication) \(C = AB\).

We can do a simple dense matrix multiplication this way, for example to repeatedly apply a matrix:

```lua
local f = {
	{1, 1},
	{1, 0},
}
fib = f
for i = 1, 9 do
	fib = matmul(fib, f)
end
print(fib[2][1]) -- 55
```

Whoa, it's the [10th Fibonacci number](https://en.wikipedia.org/wiki/Fibonacci_sequence)!
Coincidence? I think not!

But it is much more flexible. We also get sparse matrix multiplication, for example scaling:

```lua
-- scale the i-th coordinate by i
local diag = {
	{[1] = 1},
	{[2] = 2},
	{[3] = 3},
}
local B = {
	{1, 2, 3},
	{4, 5, 6},
	{7, 8, 9},
}
local C = matmul(diag, B)
for _, row in ipairs(C) do
	print(table.concat(row, " "))
end
-- 1 2 3
-- 8 10 12
-- 21 24 27
```

We don't need to use indices either. We can use names just as well:

```lua
local stats = {
	apple = {cost = 1, sugar = 2},
	banana = {cost = 3, sugar = 4},
	elderberry = {cost = 5, sugar = 6},
}
local diets = {
	arthur = {apple = 1, banana = 2, elderberry = 3},
	tim = {apple = 4, banana = 5, elderberry = 6},
	tom = {apple = 7, banana = 8, elderberry = 9},
}
local diet_stats = matmul(diets, stats)
for name, diet_stat in pairs(diet_stats) do
	io.write(name .. ":")
	for stat, v in pairs(diet_stat) do
		io.write(" " .. stat .. "=" .. v)
	end
	print()
end
-- arthur: cost=22 sugar=28
-- tom: cost=76 sugar=100
-- tim: cost=49 sugar=64
```

This does what you would expect it to: It computes the dietary stats for Arthur, Tim and Tom in terms of cost and sugar.
Matrices are just two-dimensional linear maps; matrix multiplication is just the composition of these maps.
There is no principal need for the indices to be numeric.

### Graphs

In a similar vein, nested tables make for a good graph representation. You simply store the (sparse) adjacency matrix.
This gives you the best of both worlds: You can iterate the neighbors of a node in linear time,
you can update and query the attribute(s) of an edge in constant time (including removal or addition of edges),
and if your graph is actually dense, you automatically get dense storage under the hood.
It's very convenient to program with.

### Refactoring

A general consequence is this: Tables can be treated as just mapping some labels, some keys, to some values.
Your design might start out using indices as IDs, but later on you find that some string identifier,
or perhaps even some kind of object (by reference) turns out to be more suitable.
No problem, you can simply replace the labels, and everything still works all the same.

## The bad

You need to accept some performance losses.
But with the described optimizations, this tends to be very much tolerable for a scripting language.
You have some complexity in rehashing, in the length operator.
You need fatter tables, which practically results in considerably higher memory usage if many tables are small (e.g. pairs),
though this could in principle also be optimized.

## The ugly

Not separating lists and maps as abstract data types creates much (theoretical) potential for error.

### Nonsense indices

Does asking for the `3.14`th element of a list make sense? Not really.
But, with maps, it won't throw an error.
Since we do accept out of bounds indices by default, such otherwise clear mistakes can easily be hidden.
(That said, Lua lets you set a metatable to make this error.)

### Holes

Holes create all kinds of potential problems.
`ipairs` stops at the first hole. Thus lists with holes would appear to be truncated in iteration.
You need a value other than `nil` (e.g. `false`) to represent absent values in a list,
or you need to explicitly store the list length, both of which feel a bit hacky.
The length is no longer unique, multiple valid boundaries exists. This invites all kinds of mess.
For example, in a table with multiple holes, `table.insert` may fill any of them; `table.remove` may remove any boundary.

### Missing order guarantees

It's easy to accidentally do an unordered iteration (`pairs`) when you need an ordered iteration (`ipairs`),
which creates the potential for some really nasty bugs, especially as `pairs` will most of the time behave like `ipairs`,
*depending on how the table was constructed*.
Code that relies on this can often be snuffed out by randomizing `pairs` to sometimes start at the second key,
visiting the first key-value pair last.

### The rest of the world

The vast majority of programming languages unfortunately do not unify lists and maps in this manner.
This leads to some nasty interoperability problems,
e.g. when converting between Lua data structures and C++ data structures,
when working with serialization formats like JSON, type systems, and so on.
Often these problems are not solved properly which creates issues further down the line
(e.g. serializers that force a mixed Lua table into either an array or a hash map representation, losing data,
or the inability to properly type an API that expects a mixed table when using TypeScript to Lua).

## Even more consolidation

Going one step further, we can say: Tables are functions. Functions are tables.
Why limit yourself? At their core, both do the same thing: They *map* keys to values.
They are *maps*. There are some nuances like the possibility of side effects,
but let us consider effectively side-effect-free ("pure") functions for now.

Lua lets you specify the `__call` metamethod, should a table need to act like a function;
conversely, the `__index` metamethod lets you run a function in place of a table access.

This means, for example, that you don't need to defensively litter your code with getters and setters,
as the Java people like to do. [^java]

You can hook getting and setting table fields later, if it becomes necessary.
(Of course this can result in some surprises, as usually indexing is not expected to be overridden,
except for installing a "prototype" when implementing OOP.)

[^java]: Another reason Java encourages this are the limitations of interfaces:
You can demand that a class "should implement methods `X getX()` and `void setX(X x)`",
you can't demand that it "should have a readable and writable field `x` of type `X`".

## So what?

At the end of the day, if you're a pragmatist, there is nothing wrong with preferring a separation between lists and maps.

But you should cherish some of the benefits a unification can bring; it is far from pointless.
It is ultimately another tradeoff.
A unified map data structure is a very convenient way for thinking about programs without substantially changing the (asymptotic) runtime characteristics.
Whether to specialize these maps as arrays, array lists, hash maps, tree-based maps or something else entirely is then just an implementation detail.

The more specific your data structures, the more errors can be eliminated, the more optimizations can be implemented --
but also the more minutiae you need to keep track of, the more work you need to do every time you use it.

If you do separate lists and maps, **maximize the common interface**.
For example, (unordered) key-value iteration as done by `pairs`
should be possible for lists and maps alike, with the same operation. [^cpp]

[^cpp]: It still baffles me how awkward C++ has made the simple "iterate over index, value pairs in an array" operation,
and how it is usually inconsistent with the patterns for iterating key-value pairs of a hash map.
Unfortunately, C++ is not the exception here. Many languages slap some kind of general "iterator" interface on it
and then pretend you only really care about the keys, or values, of a container, often applied inconsistently
(e.g. in Python, hash map iteration iterates keys, whereas array iteration iterates values). Argh!

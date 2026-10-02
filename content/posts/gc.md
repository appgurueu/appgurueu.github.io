---
date: 2026-02-08
title: "On garbage collection"
---

Garbage collection does not absolve you of the responsibility to know what you do
and do not need. You must make sure to only reference objects that are still useful to you.

It just absolves you of the responsibility to inform the computer of things it can already infer.

However, it can be unclear at times what that does or does not include.

This essay does not strive to explain in great detail the implementation of efficient garbage collection.
Entire books can be, and have been, devoted to the matter.
It strives to give you a simple, pragmatic mental model of how a GC roughly operates
and what it roughly guarantees.

## The ideal

The ideal, intuitive model of garbage collection (GC) guarantees that:

1. Only up to a constant fraction of time is spent in garbage collection.
2. Only up to a constant fraction of memory is wasted on garbage awaiting collection.

This is how I intuitively understand GC.
It means that GC does not change the total asymptotic runtime, nor the memory usage, of a program.
It means that we generally do not need to think too hard about memory management when designing algorithms.

We must however accept possible *GC pauses* that might reduce responsiveness.

## Practical performance

According to folklore, this makes GC unsuitable for "real-time" scenarios.
But this is not really up to date anymore.
Modern GCs are able to guarantee extremely small pause times,
and are often able to largely work *concurrently*.
See for example Java's ZGC or the Go GC:

> ZGC performs all expensive work concurrently, without stopping the execution of application threads for more than a few milliseconds.
> It is suitable for applications which require low latency. Pause times are independent of heap size that is being used.
> \- [ZGC tuning docs](https://docs.oracle.com/en/java/javase/18/gctuning/z-garbage-collector.html#GUID-8637B158-4F35-4E2D-8E7B-9DAEF15BB3CD)

Even if these defaults fail you, there's still a lot of room for GC tuning,
or changing the way your application allocates objects.

There are still scenarios, e.g. in embedded development, where memory is at a premium,
or even a couple milliseconds of delay are unacceptable, but those are not at all common.
For your average application, garbage collection is most probably fine.
The time overhead is also limited. I would roughly estimate it to be somewhere within a factor of 1.5-2x.

There is a space-time-tradeoff in garbage collection:
The more memory you can afford to waste on garbage,
the less frequently you need to collect, and thus the running time decreases.

## Implementations

There are a few very simple ways to achieve this ideal.

We can think of the set of objects as nodes in a graph,
with edges from a node (object) A to a node (object) B
if A holds a reference to B.

We take some nodes to be the "root" nodes, e.g. global variables or variables that appear in the program stack.
Garbage collection then solves the problem of not letting the unreachable parts of this graph get too large.
(Where large is usually defined in terms of memory usage in bytes.)

### Reference counting

In reference counting (RC), each object keeps a counter of how many (live) objects reference it.
When an object drops a reference to another object (e.g. because it dies),
it decreases the reference counters of all the objects it references.
If an object's reference counter drops to zero, it is killed.
When an object acquires a reference to another object B, the reference counter of B is increased.

RC has some nice properties.
It is very simple to implement. It obviously only adds constant time overhead to dropping or grabbing references. [^ref]
Garbage is collected immediately. There need not be any pauses (there could theoretically be pauses if a
large number of garbage objects is collected,
but it is trivial to defer collection if necessary).

[^ref]: What's not so nice for cache locality is that we have to follow pointers when we move them around, but eh.

The major caveat is that RC does not support *cycles*: If A references B, and B references C, and C references D,
and so on, until eventually A is referenced again, all objects in this cycle will have a reference count of at least 1,
and will therefore not be collected.

Thus RC only works in *acyclic* reference graphs.
This is however not as terrible a limitation as it might seem.
(In fact, in comparison to the "single ownership" patterns found in semi-manual memory management, it is quite a relief.)

All kinds of tree structures work just fine. "Sharing" subtrees is fine;
"forward" references that let you get to a descendant more quickly are fine.
The only problem are really just cycles. Often this can be mitigated by employing *weak* references in one direction to break cycles.
(A weak reference does not increase the reference counter of the referenced object.)
For this reason, RC is often not considered to be "real" GC, as it fails to collect cyclic garbage.

Cyclic references can be excluded at the type system level:
If you only have *immutable* containers, meaning you need to pass a set
of referenced objects when creating an object, and this set can not change,
it is impossible to create a cycle.

All *pure functional* languages *force* you to work with this kind of data structure.
A peculiar example are Lisp-like languages, where everything may be constructed from small, constant-size structs,
specifically the `cons` *pair* of two existing objects (or nothing).
(The downside with this is of course that implementing a simple array introduces logarithmic factor overhead in time complexity.)

### Tracing GC

Tracing GCs work by doing a graph traversal starting from the root set
to find the set of live objects in time proportional to the number of live objects.

This requires the ability to *mark* objects as live and to distinguish them from *dead* objects --
objects not reachable from the root.

One variant is the *mark-and-sweep* collector:

Each object has a bit that tells whether it is live, which is unset at initialization.
In the *mark* phase, the set (subgraph) of live objects is traversed,
and all live objects get their bit *marked*.
In the *sweep* phase, the set of *all* objects is iterated
(all objects are kept in e.g. a doubly linked list),
and all objects that don't have their "live" bit set are killed;
the objects that do have it set have it unset in preparation of the next GC cycle.

In this variant, objects keep their memory addresses.
This is called a *non-moving* GC.

Non-moving garbage collectors have some limitations.
If your allocations somehow end up with many small objects that are evenly spread out,
this may make it impossible for large allocations to use the remaining gaps.
Assuming a contiguous slab of memory of size \(n\), \(\sqrt{n}\) small objects
are enough to reduce the maximum size of a contiguous allocation to \(\sqrt{n}\).
This kind of problem is known as "heap fragmentation".
It can be alleviated to some extent via paging
(pages that live entirely within gaps can be returned to the OS).

A moving GC can operate differently. Instead of marking objects to be killed and sweeping in a second phase,
we create a new arena when GC is due, and copy all live objects to be saved there.
Then, we drop the arena containing all old objects as a whole.
(Alternatively, this can also be done in-place by moving stuff around.)
This means we change pointers, so the old pointers of objects now need to record their new pointers somewhere.

A moving GC can ameliorate heap fragmentation, because it will reorder the live objects to fit tightly.
Additionally, if implemented using some kind of depth-first search, it will put objects next to other objects that reference them
(the best case being a linked list, which will be laid out sequentially in memory).

<!-- improve -->

### Combination

Some implementations (e.g. CPython's GC) combine reference counting with a tracing GC.
The tracing GC is used for containers in order to be able to collect reference cycles.
Non-container objects (e.g. primitives like big integers [^cpython]) are cleaned up as quickly as possible.
This is desirable especially for resource handles, e.g. file descriptor wrappers.

[^cpython]: In fact, in CPython, *everything* is put on the heap, including types like floats and booleans.

Now that we have a rough idea of how GC might operate and what it guarantees,
let's look at some related details, some edge cases that might seem surprising.

## Implementation details

### The meaning of life

What does it mean for an object to be "live", to be "referenced"?
What is the *root set* in our reference graph?

#### Local variables

```lua
do
	local large_obj = ... -- some large object
	...
end
-- large obj is out of scope here. can it be collected?
```

It depends. From the perspective of the programmer, `large_obj` is both syntactically and semantically clearly out of scope. [^py]
But under the hood, there's a good chance that local variables are all allocated in the stack frame when the function is called,
and that local variables are not "unset" or "freed" the moment their scope ends.
We can make it a bit more interesting even:

```lua
local large_obj = ... -- some large object
...
-- suppose large object is not used past this point. can it be collected?
```

Now the variable is syntactically still in scope, but the compiler knows that semantically,
there will not be any more references to it, so in principle the GC is free to collect it.
But is it?

<!-- Closures keep all their upvalues alive. But that's not really surprising, is it? -->

[^py]: Not missing this opportunity to dunk on Python. If your variables naturally leak into function scope,
obviously things are likely to be kept alive longer than they need to be, by default.
See also the bloarg post.

#### Views

Many languages will have *views* into containers,
for example *string views*, views into n-d-arrays, etc.

Often, these views, no matter how small the portion they reference,
can keep the entire container they reference alive.

This can also be an optimization (substring in V8).

### Containers

Strictly speaking, containers need not be considered a subject of garbage collection,
but they are closely related. The question we need to ask ourselves is:
Does it shrink?

When I create a list, and I add many items, and remove all but a few,
can I assume that the space allocated for the many items is (eventually) reclaimed?

What about other data structures, like a hash map?

It turns out the answers vary surprisingly much.

Let's look at the Lua reference implementation, PUC Lua, for example. Here, tables *generally don't shrink*.
The only case in which they do is if keys are migrated from the hash part of the table to the array part.
A table used simply as a list, inserting items at and removing items from the end, will never reduce its underlying capacity.
A table used simply as a map (that is, e.g., only with string keys), will never reduce its capacity from its peak either.

Unfortunately, there is no simple rule such as "containers never shrink on their own".
Depending on the language and implementation, some of them do, some of them don't;
you just need to know.
(Developers coming from systems programming languages might not find this surprising.)

Scripting languages
    | List | (Hash) map
Lua |
JavaScript |
Python |

Compiled languages
Java
Golang

C#?

(Not including systems programming languages with semi-automatic memory management, like C++ or Rust.)


### Stacks

Containers need not be explicit.
Your program has a *stack* on which typically local variables and return addresses reside.
It turns out that in many language implementations, this stack may grow but never shrink;
it always stays at its peak memory usage.

Consider the following stupid recursive gaussian sum:

```lua
local function sum(n)
	if n == 0 then
		return 0
	end
	return n + sum(n - 1)
end
```

Now say you run `sum(1e5)` or similar. Watch your memory usage go up, never to go down again.

But hey, this is at least just a single stack, and possible a quite limited one at that (e.g., even on modern Lua versions, limited to a million slots, which is on the order of a couplemegabytes; a different issue deserving of a different essay).

Or is it? Recall that we can have multiple stacks. We can have *coroutines*.

Not only do coroutines start out with a bunch of stuff, including a stack of some minimum size, to begin with
(which makes them an order of magnitude fatter than say, empty tables).

Coroutine stacks *also* have the same property of never shrinking.
You can have many coroutines, whose stacks temporarily grow,
and then occupy memory forever as the coroutines are kept around.

<!-- TODO example -->

### Weak references

Sometimes, you want a unidirectional keep-alive relationship:
B should be kept alive by A, but not the other way around: A should not be kept alive by B.

This is very useful for all kinds of caches and external properties attached from outside.
If an object dies, you want its caches and properties to die with it.
You don't want a cache to keep the entity alive.

This is the problem *weak references* solve. You can tell your programming language that an entry in a
cache should not keep the object alive, and can instead be collecting along with it.

In Lua, that would look something like this:

```lua
local cache = {} -- [obj] = cached property
setmetatable(cache, {__mode = "k"}) -- keys are weak references: can be collected early
```

### Immediate release



---

Consider the following implementation to "chain" two iterators:

```lua
function chain(its)
	-- Note: Keeps its alive unnecessarily long
	local i = 1
	local function iter_next(...)
		if ... ~= nil then
			return ...
		end
		i = i + 1
		local it = its[i]
		if it ~= nil then
			return iter_next(it())
		end
	end
	return function()
		return iter_next(its[i]())
	end
end
```

Do you see the problem?

(Maybe a bad example)

TODO construct example

Remedy: Copy and set to nil, or use linked list type structure and always cut off head.

(Bonus: Does it die early enough? Resources, refcounting)

Finalizers and all the mess they make!

### Weak refs

## External resources

Scripts often interface with host applications that may allocate more memory,
which the garbage collected scripting language needs to be made aware of.

If there is no good API for the host application to inform the scripting language's GC
of these (from its perspective) "hidden allocations", it's easy to get leaks:
It may look like there is not enough garbage to warrant a collection,
when in reality the memory usage incurred is much higher.

We need to be able to inform the GC when we allocate or deallocate memory
based on the creation or destruction of objects managed by the GC
so that the GC's contract of ensuring limited waste also applies to these indirect allocations.

Unfortunately, there need not be a good API for this.
As a workaround, you can resort to manual memory management,
letting the user explicitly and immediately free indirect allocations
(and then leaving the wrapper object in an empty state).

<!-- VoxelManip -->

PUC Lua 5.4 solves this using `lua_gc(L, LUA_GCSTEP, int kb)`, which tells the GC that you have allocated `kb` Kbytes.
(Earlier Lua versions already had `LUA_GCSTEP`, but it was not clear what the `stepsize` really corresponded to.)


Userdata problem

(Does it collect often / early enough?)

## All the rest

### Array allocation

So, you're writing 

Manual memory management (also what you can get with all kinds of IDs)

`defer`, arenas, etc. (Zig) => less granularity, but also potential performance boons.
arena kind of allocation of e.g. trees can also done using arrays in any kind
of programming language including scripting languages (but often sucks a bit),
goes well with transpositions (AoS -> SoA).

Semi-automatic memory management à la C++

Essentially automatic memory management as in Rust, with reference counting and IDs as escape hatches

## So what?

Garbage collection is great. It is a solid way for programmers to let the computer
solve some of the more trivial problems, at limited and usually acceptable runtime cost.
It is no surprise that garbage collected languages, be they dynamically or statically typed, are so popular.

There are, however, a few implementation details to be aware of.
The naive idea that memory usage is proportional to the observable size of your data structures is an illusion;
it does not hold up over scrutiny.
However, it is a very useful illusion, if you are aware of the (in practice relatively rare) edge cases that break it.
Pure functional languages are at an advantage here. [^lisp]

[^lisp]: This may help explain why Lisp was pioneering garbage collection.
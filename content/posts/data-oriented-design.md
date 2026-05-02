---
date: 2026-05-02
title: "A tour of data-oriented design"
---

Waiting for main memory access is on the order of a thousand times slower
than doing a simple computation.
The main way of hiding this are ever growing CPU *caches* that make a small portion of memory
accessible to the CPU at much higher speeds.
It is not surprising then that most applications today are *memory bound*:
The limiting factor for performance is speed of memory access, not speed of computation.
To write efficient code, we need to write *cache-friendly* code.

There are two dimensions to this: Time and space.
When we have loaded some data into the cache recently,
we want to make the most out of it; we want to reuse it as often as possible in our computations.
When we have loaded some data, data in close spatial proximity will be loaded along with it.
Our data layout should be such that this data is useful to us.

This requires thinking about the memory access patterns a program exhibits,
and the data layout that best accomodates them.

This post presents some rules (of thumb) for how to optimize certain patterns at a low level.
That is, this post is not concerned with optimizing asymptotic runtime,
but rather with optimizing the real-world constant factors arising from the necessity of cache hierarchies,
which is typically suppressed in the simplified [random-access machine model](https://en.wikipedia.org/wiki/Random-access_machine).
Naturally, none of these rules is "absolute"; you still need to measure for yourself.

## Suitable programming languages

Many of the concepts discussed here do also apply in high-level scripting languages like Python,
but tend to be much more awkward to (try to) apply in there;
the language is effectively working against you at every step of the way.
The examples here will use mostly C++ as one of the most widespread systems programming languages.

## Array-backed data structures

Let's start with the most basic container type: Lists.

How do you arrange a list of items in memory,
if you want to be able to add and remove items at the end efficiently?

The most simple solution is the *linked list*.
Adding and removing items just requires a single allocation
and setting some pointers, which is all possible in constant time.
Traversal also "only" requires dereferencing a pointer in each iteration, which is still constant time.

What this simplistic view neglects are the massive constant factors involved here.
Heap allocations are not cheap. Pointer dereference is not cheap if the address is not in cache.
In the worst case, where items are spread out all over memory, you will more or less be waiting on memory every single time,
making the traversal some hundreds to thousands times slower.
In reality, this worst case typically doesn't happen, e.g. because many of the linked list node allocations do luckily happen
in proximity and thus do end up close in memory. But linked lists will still be significantly slower than *dynamic arrays*.

Dynamic arrays can use a very simple trick: They simply grow their capacity in powers of two. [^growth]
It can be shown that this results in (amortized) constant time per append operation.

[^growth]: Other exponential growth factors can be used to balance memory usage vs. cost of appending.

**Rule**: Prefer dynamic arrays (`std::vector`) over linked lists (`std::list`).

The fact that data is then stored contiguously opens up a number of optimization opportunities related to processing data "in bulk".
For example, for primitive types, a copy is then as simple as a `memcpy`; arithmetic operations can be vectorized more easily.

This has the added benefit of giving us a contiguous enumeration of items by their indices.
For example, to pick an element uniformly at random, you can simply pick a random valid index and then get the corresponding element.
Furthermore, arrays admit efficient *slicing*.

There are still use cases where linked lists have some benefits.
For example if you need to remove items at various positions, while keeping the order of the remaining items.
(If you do not need to keep the order, you can simply swap with the last element of a dynamic array, and then remove that.)

A special case that comes up often is the *double-ended queue* (deque),
where you want to be able to add or remove items efficiently at *both ends*.
But even here, a simple linked list is typically not preferred. Many implementations opt for a hybrid data structure:
A linked list of small, fixed size arrays. If these arrays are large enough, you still get good spatial locality;
when you need to pop or insert a node at the front, you may need to shift around the elements in the array,
but if the array is small enough, that is not too expensive: There is a decent sweet spot.

Alternatively, deques can be implemented as a kind of dynamic array with wraparound.
This is more to my taste, since it preserves many of the nice array properties such as a mostly contiguous memory layout.
It is what Zig does, for example.

**Rule**: Prefer proper deques (`std::deque`) over linked lists (`std::list`).

In my experience, the cases where a program truly *needs* a linked list are rather rare,
and even then some kind of hybrid data structure is often preferable.
If you find yourself using a linked list, ask yourself whether there is a more appropriate,
more cache-friendly data structure you could use instead. Usually the answer will be yes.

### Maps and sets

Do not use sorted sets or maps *unless you have to*.
These are implemented as balanced search trees (e.g. red-black trees), meaning:

* All operations incur logarithmic factors;
* Worse, logarithmically many (dependent) pointers are being chased;
* Each element gets an allocation.

In contrast, `std::unordered_set` and `std::unordered_map` are backed by hash tables, meaning expected constant time operations.

Using an unordered set / map is also preferable for readability:
If you use an ordered set, I will assume that you are doing so because the order is important.

**Rule**: Prefer hash maps (`std::unordered_set` and `std::unordered_map`) over sorted maps (`std::set` and `std::map`).

Red-black trees are generally not ideal for minimizing the number of allocations, the memory overhead per item, or the number of cache misses.
For these purposes, it is once again typically preferable to consider a "hybrid" data structure where each tree node stores multiple items contiguously, namely a [B-tree](https://en.wikipedia.org/wiki/B-tree), which is the standard sorted set / map data structure e.g. in Rust.

Unfortunately, the C++ STL has specification requirements that strongly suggest an "open hashing" implementation, meaning there typically still is an allocation per item.
For proper "closed hashing" that is really just backed by a single array, [consider alternative hash map implementations](https://martin.ankerl.com/2022/08/27/hashmap-bench-01/).

**Rule**: Use more efficient alternatives to C++ STL data structures.

### Allocate to the right size

A vector will grow as necessary, but if you know the right size from the beginning,
`reserve()` from the get go to save allocations and space,
*especially* if your vectors are very small. [^log]

[^log]: For small vectors, the overhead of logarithmically many reallocations matters much more.

Whether this is a good idea is not so clear when (1) you only know a rough upper bound
or (2) your vector changes size frequently.
`reserve()` typically does an allocation to the exact size and copy,
so you can easily get quadratic time if you call `reserve()` just to append a few elements in a loop.
(For this, use `std::insert`.)

Often functional patterns are the way to go (e.g. `map`);
unfortunately they tend to be cumbersome to express in C++, especially in older versions.

### Higher-dimensional arrays

The "obvious" way to do a 2d array might seem to be the nesting of 1d arrays. Say:

```c++
using Row = std::vector<Cell>;
std::vector<Row> grid; // grid[y][x] is a cell
```

If your rows are of sufficient length, this might be okay, even though each row gets its own allocation
(it might even have some advantages, like letting you swap, drop or add rows efficiently).

But it doesn't scale well to higher dimensions, and it has considerable overhead if rows are short.

Consider this 3d variant:

```c++
using Row = std::vector<Cell>;
using Grid2d = std::vector<Row>;
std::vector<Grid2d> grid3d;
```

For a \(16^3\) grid, these will be 256 rows scattered around memory. Ugh.

For this reason, especially if the grid dimensions do not change dynamically,
a so-called *ndarray* representation is preferred, which lays out the n-dimensional grid sequentially in a single array.

A 2d grid, stored in row-major order, might look as follows:

```c++
template<class T>
struct Grid2d {
	size_t width, height;
	std::unique_ptr<T[]> data;

	Grid2d(size_t width, height) : width(width), height(height) {
		data = std::make_unique<T[]>(width * height); // note: initialized
	}

	T &at(size_t x, size_t y) {
		// omitting bounds checking for simplicity
		return data[y * width + x];
	}
};
```

Note how, when `x` is increased by one, we move to the next position in the array;
whereas when `y` is increased by one, we jump a whole `width` positions.

Now consider the following loop order:

```c++
Grid2d<Cell> grid(42, 42);
for (size_t x = 0; x < grid.width; ++x) {
	for (size_t y = 0; y < grid.height; ++y) {
		// do something with grid.at(x, y)
	}
}
```

Is this good for cache locality? Absolutely not: The inner loop iterates in strides of `width`.
The proper loop order to use here is Y-X, consistent with the index calculation.
This iterates the array linearly.

```c++
for (size_t y = 0; y < grid.height; ++y) {
	for (size_t x = 0; x < grid.width; ++x) {
		// do something with grid.at(x, y)
	}
}
```

In fact, we might even save ourselves the nested loops (unless we need the coordinates):
If we just need to look at every cell, we can iterate the underlying `data` array directly.

**Rule**: Iterate higher-dimensional arrays in the right index order.

For a slightly more elaborate example, consider matrix multiplication.

```c++
size_t n = 42;
Grid2d<float> A(n, n), B(n, n);
Grid2d<float> C(n, n); // note: zero-initialized
// populate A and B...
for (size_t i = 0; i < n; ++i)
	for (size_t j = 0; j < n; ++j)
		for (size_t k = 0; k < n; ++k)
			C.at(i, j) += A.at(i, k) * B.at(k, j);
```

This iterates `A` in the row-major order we want, but `B` is iterated in column-major order.
Again a simple loop interchange does the trick:

```c++
for (size_t i = 0; i < n; ++i)
	for (size_t k = 0; k < n; ++k) // swapped
		for (size_t j = 0; j < n; ++j) // swapped
			C.at(i, j) += A.at(i, k) * B.at(k, j);
```

(In fact, for every possible row-major / column-major combination of `A` and `B`,
there is a suitable ordering of the loops that processes both `A` and `B` linearly!)

## Arena allocation

C and C++ have popularized the use of a single, global allocator, implemented by the C standard library functions `malloc` and `free`
(hidden behind operators `new` and `delete` in C++, often entirely invisible when smart pointers and STL containers are being used).

This is of course very convenient. You need not worry about telling functions where and how to allocate, they simply know to use the one and only global allocator.

But it also comes with a number of limitations as everything is swimming in the same big pool of memory soup.
For one, this is terrible for safety and security; small memory safety mistakes in one part of a program can end up
affecting data populated and used in an entirely different part.
For two, this means that whenever some function constructs and tears down some intricate linked data structure (say, a parse tree),
you have to pull all the chunks of memory you used back out of the global soup and return them to the allocator: Deletion requires traversal.
For three, the properties `malloc` and `free` need to provide (e.g. thread safety, limited fragmentation, efficient memory use, memory *reuse*) come at significant cost.
One important consequence is that, the longer your program runs, the less likely it becomes that the allocator is able to satisfy subsequent memory allocations
with neighboring chunks of memory, so memory is spread all over the heap; spatial locality suffers.
An allocator that doesn't provide most of these is still very useful and will allow for much cheaper allocations.
In many situations it is admissible to temporarily waste memory
(say, a function using a dozen times as much memory as it would need to carry out its task)
if inputs are guaranteed to be sufficiently small.

The simplest form of this is the so-called *bump allocator*. You take [^what] a sufficiently large chunk of memory,
and every time you allocate, you just bump the offset into this chunk (possibly plus some alignment).
After you're done with all the objects, you may throw the chunk away [^what]; or reset the offset to reuse it.
Of course, this very simple technique requires an acceptable upper bound on the size of your allocations.
To alleviate this, we simply switch from a single chunk to a *linked list of large chunks*.
This is typically called an *arena allocator* or *region*.
When there's not enough space left in the current chunk, we fetch [^what] a new one, maybe trim the current one
(as to not waste the residual space), and then serve the allocation from there.
To deallocate everything, we walk the list of chunks,
but this is very cheap if chunks are large enough -- much cheaper than walking objects all over the heap to free them individually.

In C++, a simple arena allocator is implemented by [`std::pmr::monotonic_buffer_resource`](https://en.cppreference.com/cpp/memory/monotonic_buffer_resource),
which can be wrapped by a [`std::pmr::polymorphic_allocator`](https://en.cppreference.com/cpp/memory/polymorphic_allocator)
and then used in containers like `std::pmr::vector`.
(In short: `std::pmr` is the thing to look for.)

[^what]: I'm deliberately using somewhat vague terms like "throw away" or "fetch" here; while this can be just a call to `malloc`,
it need not be, and often is not. The chunk of memory for a bump allocator may live in the static data segment (if the size is statically bounded)
for example; the chunks for an arena allocator may be obtained directly from syscalls like `mmap`.
Allocators can be written on top of other allocators, but they need not be.

### Stack allocation

In fact, you've used a kind of bump allocator already, all the time: Stack allocation.
Every time you declare some local variables in a function, space for them is allocated on the stack by bumping the stack pointer under the hood.
(Usually an entire *stack frame* is allocated for all local variables when a function is called.)
To deallocate everything as the function returns, the stack pointer is simply reset to what it previously was.

In many cases, you have a good bound on the stack depth and the size of some smaller temporary data structures,
in which case you can and should just put them on the stack.
This of course requires that objects do not care where they are allocated.
For example, you should not make your objects inherit from a base class
that expects to manage objects on the heap for reference counting; don't try to make C++ into Java.
For C++ containers, you can use `std::pmr::*`; or you can directly provide custom allocators
as template parameters, but this taints the resulting types.

### Array "allocation"

In many cases, you can get away without any fancy allocator machinery at all. You just use an array (`std::vector`).
Pointers become indices into this array, which has an added benefit of saving bits
(pointers may be 64-bit, for indices you can often get away with 32 or even 16 bits).
This also limits the accessible memory range if you leave bounds checks in.
The array can be cheaply thrown away in whole after you are done with it,
and all the items in the array are stored contiguously in memory in their order of allocation.

**Rule**: Use an appropriate form of allocation.

## Inline objects

Heap allocations are expensive and add layers of indirection.

Whenever you see a `std::unique_ptr` or similar, ask yourself:
Might it be more efficient to just directly store the object rather than a pointer to it?
The smaller the object, the cheaper it is to copy.
Very small objects, say, an array of 2, 3 or 4 floats, are hardly ever worth putting directly on the heap.
This is also one of the main reasons non-systems languages tend to make achieving good performance hard:
They don't give you control over allocations. Typically, they put more or less *everything* on the heap by default. [py]
This is great for ease of programming and language implementation (objects have uniform size and can be copied cheaply),
but terrible for performance.

[^py]: CPython put *everything* on the heap. (Yes, that includes "primitives" like booleans or floats.)
This makes it possible that "everything is an object" (a pointer) in the implementation.
Java has a notion of primitive types, but boxed types (like `Integer`)
which you need to use for generics (like `ArrayList<Integer>`) are still put on the heap,
as are all objects, no matter how small.

A special case of this is that of a *variant*: Maybe a field can store an object of class `A` or `B`.
In classic OOP design, you would introduce an interface `I` which both `A` and `B` implement
(in C++: an abstract class which both `A` and `B` derive from).
Now `I` of course has no real "size" -- it doesn't know which classes might implement it -- so you can't just have a field of type `I`.
The straightforward solution is to just put the field on the heap. Then you get `std::unique_ptr<I>`, which is completely fine;
the object on the heap being pointed to can either be an `A` or `B` object of the appropriate size,
the appropriate destructor will be found via the vtable, and the allocator will know the size of the object it needs to free anyways.

In many situations, an old reliable *tagged union* will be a better alternative. In modern C++, this would be `std::variant<A, B>`.
The object is stored directly in the field, along with a tag that tells us the type of object.
Beware however if one of the alternatives is much larger then the others: Then a variant may be rather wasteful in terms of space.

An interesting example of this is inlining data of small containers, for example the famous "short string optimization" of `std::string`:
Under the hood, you can imagine that a `std::string` is roughly implemented as follows:

```c++
struct String {
	char *chars;
	size_t cap;
	size_t size;
};
```

The string contents (`chars`) are always allocated on the heap.
This is rather wasteful for very short strings of just a couple characters, which appear frequently.
Thus, C++ STL implementations usually prefer to store short strings directly in the string type,
using the space for the `chars` and `cap` fields to *directly store the characters*,
and deciding (say, based on whether the highest byte of `cap` is `0`) whether to interpret the string as one or the other.

```c++
struct String {
	union {
		std::array<char, 16> inline_chars;
		struct {
			char *ptr;
			size_t cap;
		} heap_chars;
	};
	size_t size;
};
```

The end effect is that typically strings of length 15 (or 11 on a 32-bit system) or so can be stored inline,
avoiding the cost of an additional heap allocation and deallocation, and avoiding the indirection of having to follow another pointer
to somewhere on the heap to find the actual string contents.
If, for example, I have a struct along the lines of

```c++
struct Entry {
	std::string key;
	int value;
};
```

and I now iterate a `std::vector<Entry>` with short `key`s, I have good spatial locality: The actual bytes live directly in my vector,
possibly with some empty space and some `size` fields in-between. This is much better than the bytes being spread over the heap though.

This kind of optimization can be applied to containers more generally. Consider what should happen if you have very few, small objects.
Often this is worth a special case.

**Rule**: Try to eliminate unnecessary indirection; avoid putting small objects on the heap.

## Reuse allocations

Often you have code along the lines of:

```c++
for (...) {
	std::vector<X> xs;
	// populate xs...
	// do something with xs...
}
```

which is equivalent to

```c++
std::vector<X> xs;
for (...) {
	xs.clear();
	// populate xs...
	// do something with xs...
}
```

except the latter potentially saves many allocations as `clear()` keeps the capacity of the underlying vector.

**Rule**: Make the most out of already allocated memory.

## Transposing matrices

One important data layout optimization that you absolutely **must** know is, essentially, to simply *transpose a matrix*.

Whenever you have some kind of "Array of Structs" (AoS), you can think of this as a 2-dimensional map that could be written down as a table.
Say, in the context of a game, you have some "entities". Naturally, [assume entities are perfect spheres](https://en.wikipedia.org/wiki/Spherical_cow).
These have some center, some radius, and a whole lot of game data (whatever that may be).
You can think of this as a table (or "matrix" in math lingo):

| `center`   | `radius` | `game_state` |
|------------|----------|--------------|
| \((0, 0)\) | \(2\)    | ...          |
| \((0, 2)\) | \(3\)    | ...          |
| \((3, 0)\) | \(1\)    | ...          |
| \((4, 4)\) | \(2\)    | ...          |

Now, there are two ways to store this matrix.
Perhaps the most intuitive is *row-major order*:
You simply concatenate all the entity "rows" to fit them into memory.

This is what you typically get with an "Array of Structs":

```c++
struct Entity {
	Vector2f center;
	f32 radius;
	GameState game_state; // large
};
...
std::vector<Entity> entities;
```

Now consider the access pattern. Let's say we have \(n\) entities.

We will probably have some kind of simulation "step" where we touch all our entities.
This probably needs to look at most entity fields, including most of the game state.
For this kind of access pattern, the current layout is reasonable; it offers good spatial locality.
We load all of the data, we use all the data.

But now consider a more specialized *physics* simulation step.
Such a step may need to run at a higher frequency, and it does not care about the fat game state.
It only cares about positions and radii. And it will be looking at those a lot.
A straightforward implementation has to consider all \(O(n^2)\) pairwise possibilities for collisions. [^spatial]
This means we want to be able to fetch positions and radii very efficiently.
And for this to have good performance, we need them to be tightly packed in memory.
If not, the larger the game state, the worse iteration performance gets, until most of a cache block is effectively garbage to us.
(And cache blocks are pretty large these days. Say, 32 bytes.)

For this, it would be preferable to look only at the *projection* of entities to only their "physical" properties, specifically `center` and `radius`.
It might be beneficial, depending on the specifics, to create a new array of "physical entities" (colliders) which consists only of the physical properties:

```c++
struct Collider {
	Vector2f center;
	f32 radius;
};
std::vector<Collider> colliders;
colliders.reserve(entities.size());
for (const auto &entity : entities)
	colliders.emplace_back({entity.center, entity.radius});
```

A potential `physicsStep` function would then take this vector of `colliders` and operate on it,
which will be much more cache friendly now that the data sits tightly in memory.
Note that we have also decoupled matters: `physicsStep` does not need to be aware of `Entity` anymore, its input consists only of what is relevant to it.
We might refactor further by directly giving entities a `Collider` field.

[^spatial]: Generally, there are various spatial indexing data structures you would want to leverage here, but we ignore them for the sake of the example.

In this example, the projection is done on-demand, requiring a wasteful iteration over the entities.
In general, that may be undesirable, or not so easy to do.
We want to store the data in the right format *from the get go*. Instead of storing the matrix in *row-major* order, we store it in *column-major* order:
As a *Struct of Arrays*, where each array stores densely the properties of a certain "component", say, the physical component.

```c++
struct Entities {
	// technically, these are separate allocations that may be arbitrarily far apart in memory.
	// we might want a single allocation sometimes, but for the considerations here the "gap"
	// between the columns is not really important;
	// only for very small numbers of entities might it be.
    std::vector<Collider> colliders;
    std::vector<GameData> game_datas;	
};
Entities entities;
```

Now, projection to certain properties is wonderfully simple: We just pass `entities.colliders` to `physicsStep`. Done.
Game logic works on `entities.game_datas`. Done.

Some people are somewhat worried about hurting performance of a loop that actually looks at *both* collider and game data.
The two arrays surely cost us something, for less suitable access patterns?
Well, not really. (As we will see a bit later, quite the opposite in some cases, actually.)

Sure, there are now two arrays on which we occur cache misses, but misses will occur less frequently for both, because the items are smaller.
The data is still at least as densely packed (in fact, we'll see that it might be more densely packed even),
the spatial locality is still optimal. Asymptotically, if there are enough entities, we are not losing anything. [^hwpref]

[^hwpref]: Hardware prefetchers are also smart enough to recognize when you're iterating two arrays linearly.

Only in extreme edge cases may this become relevant, e.g. if you only have 1, 2, 3 or so entities:
Because then the additional cache miss that you may get *on the first access* matters more.

Put simply: The mind-boggling thing is that, even when switching to a column-major layout,
we don't really lose on an access pattern where we look at rows,
*if we iterate the two arrays at the same time*.

There *is* an access pattern that greatly benefits from the AoS layout: Random access.
Assume that \(n\) is large, such that cache misses are highly likely.
If I want to pick a random entity (random row), and look at some of its collider properties, as well as some of its game data properties,
AoS means that loading a game data property into cache has a good chance of loading the collider property and vice versa.
Assuming optimistically that an entity fits in a cache block, we might expect double the cache misses with a SoA:
Each lookup is (likely) a miss both in the `colliders` and in the `game_datas` array, not just a single miss in the `entities` array.

In a real scenario, we would have a bunch more properties, and a bunch more subsystems (e.g. a rendering subsystem).
Managing entity properties in separate arrays (sometimes other containers) is also known as an *Entity Component System* (ECS).

Whichever subset of the fields an access pattern that does a full traversal demands,
we can simply traverse the corresponding arrays with the same index,
and data will be optimally dense in memory.

**Rule**: See whether an Array of Structs or a Struct of Arrays is more appropriate for your access patterns.

### No more padding

Say you have a simple

```c++
struct Value {
	enum class Type : u8 {
		...	
	};
	Type tag;
	uint64_t value; // for simplicity. usually this is a 64-bit wide union.
};
std::unique_ptr<Value[]> values;
```

What will the size of `Value` be; how will the size of `values` grow in the number of items?

On a 64-bit machine, the `Value` struct will likely be padded to 128-bit.
That is, 7 bytes are wasted to pad the `tag` to take up 64 bit.
Almost half the memory is wasted!
And as we only have two fields, the usual trick of shuffling fields around a bit to reduce padding is not applicable.

If instead we stored separately:

```c++
struct Values {
	std::unique_ptr<Type[]> tag;
	std::unique_ptr<uint64_t[]> value;	
};
```

This grows much nicer: The tags are now a densely packed byte array. For each value, we need around 9 bytes.
We've cut the memory usage roughly in half. This will generally also reduce cache misses.

An even more extreme example can be made for booleans, which can be bitpacked, making them 64x denser.
When you then do vectorized operations on bitsets is when things start to get really fast.

You might think I just made this example up. But I did not.
This is, in essence, pretty much what Lua up to version 5.5 suffered from,
as they opted for a "portable" tagged union value representation rather than "NaN boxing",
further complicated by the introduction of 64-bit integers in Lua 5.3.

In the case of Lua, separate arrays were an option they considered,
but ultimately they did want one allocation at least for rather small arrays
(due to the aformentioned reasons of initial cache misses, and heap allocation overhead).
So you're packing up tags and values in the same array, which there are a bunch of ways to do.
You could just place one after the other.
Or you could group together values in groups of 8, with 8 tags, so no space is wasted on padding.

[After a decent amount of benchmarking](https://sol.sbc.org.br/index.php/sblp/article/view/30252/30059),
Lua 5.5 addressed this with a special "reflected array" implementation,
the layout of which in memory is tag \(n\), ..., tag \(1\), value \(1\), ..., value \(n\).
This has all the nice properties of a typical SoA optimization, and some more even: Notice how the first few tags and the corresponding values
are close in memory! So you can likely even avoid the otherwise typical "double cache miss" when starting iteration.
A neat side effect of which is that tags are now densely packed; operations that only need to check tags can be sped up by orders of magnitude,
for example [checking that a table has no holes can be done roughly a hundred times faster](https://groups.google.com/g/lua-l/c/Daw6UZwHXvk/m/dSyevNKRAwAJ).

**Rule**: Move fields "out-of-band" if a significant amount of memory is wasted on padding.

### Omit unused fields in bulk

In graphics programming, each vertex of a mesh is assigned some attributes,
such as a position in 3d space, a 3d normal vector, a color, and so on.
A number of vertices is stored in a "vertex buffer", which is sent to the GPU.
For a well optimized rendering pipeline, the CPU-GPU memory transfer bottleneck often limits how much you can render.

There are a bunch of "advanced" attributes which should be optional.
In an AoS design, this is awkward to express. You might start out with something like:

```c++
struct Vertex {
	Vector3f pos;
	Vector3f normal;
	ARGB8 color; // 32-bit
	std::optional<std::array<Vector3f, 2>> tangents;
	std::optional<Vector2f> texcoord2;
};

struct VertexBuffer {
	std::vector<Vertex> vertices;
	...
};
```

but this is bad: a large amount of space is wasted on optional attributes that will usually be absent.

For performance reasons, you might then start introducing different `struct` types, say, `BasicVertex`, `VertexWithTangents`, etc.
The result will basically be a huge violation of DRY unless you create some kind of macro hell.

If however we store vertex buffers as a SoA...

```c++
struct VertexBuffer {
	// could also use std::unique_ptr's here to not store size + cap redundantly,
	// but this is usually not worth it.
	std::vector<Vector3f> positions;
	std::vector<std::array<Vector3f, 2>> normals;
	std::vector<ARGB8> colors;
	std::vector<std::array<Vector3f, 2>> tangents;
	std::vector<std::array<Vector3f, 2>> texcoord2;
	...
}
```

... then unused fields simply remain `nullptr`s or empty vectors.
They cost a fixed, low amount per vertex buffer, *not per vertex*.
You can mix and match freely, you don't need to define structures for all possible combinations.
You will find that this does wonders for your code quality.

**Rule**: Save space on often empty fields. Converting an AoS to a SoA is one way to do this.

### Tradeoff

SoA vs AoS is often presented as a "binary" kind of optimization: You do it or you don't, it helps or it doesn't.
But somewhat surprisingly, there actually is a spectrum here if you want to "balance" the performance of different access patterns.
Arguably it seems to me that most patterns fall on either end of the spectrum. Still, it might be valuable to see the spectrum for what it is.

A tradeoff is achieved by *tiling*, or *chunking*, if you will. Say we have our AoS.
We batch together entities in some suitably chosen "group" size.
([This was one of the optimizations explored for arrays in Lua](https://sol.sbc.org.br/index.php/sblp/article/view/30252/30059).
It performed well, but ended up being rejected for its complexity; the "reflected array" was already considered good enough.)
In each group, we store a few colliders, and the game data of their corresponding entities, *contiguously*.

It looks something like this:

```c++
struct Entities {
	struct Chunk {
		constexpr static size_t SIZE = 8;
		std::array<Collider, SIZE> colliders;
		std::array<GameData, SIZE> game_datas;
	};
	std::vector<Chunk> chunks;
};
```

This is often done to increase data density.

**Rule**: Mix AoS and SoA to suit your access patterns.

## Blocking

So far, we've mostly looked at optimizations that are aimed at *spatial* locality:
Placing data that is used together closely in memory.
This misses the *temporal* dimension of keeping values in the cache that we will be reusing frequently later.

It is generally not trivial to find simple examples for this because many programs have a low "operational intensity":
They load some data, then they do a few computations with it, and then they're done with it.

You can of course construct somewhat contrived "naive" code, e.g. simple examples that obviously benefit from a loop fusion
(and likely would have been written that way in the first place, because no programmer likes repeating themselves).

For a good example where temporal locality is really important, we need to look at algorithms that reuse values many times.
The above "all-pairs collision detection" example will do. In this simple example (the generalization of which is the "outer product"),
each value is reused on the order of \(n\) times.

A naive implementation would be a nested loop, something like

```c++
for (size_t i = 0; i < n; ++i) {
	for (size_t j = i + 1; j < n; ++j) {
		// do something with colliders[i] and colliders[j].
		// assume that most of the time, there is no collision.
		// so no need to populate a dense matrix or anything; this is all sparse.
	}
}
```

Spatial locality is excellent with this. It looks like this is the best we can do. Or is it?

Observe that if \(n\) is large enough, the inner "for" loop basically becomes a "sliding window".
Earlier `colliders[j]` are evicted to make space for newer ones.
In the next iteration, we start at the beginning again. We're evicting and refetching data all the time!

What we can do instead is to grab us a working set of a modest size, among which we do the computation for all pairs.
We compute a \(B \times B\) "block" of the resulting pairs, if you will.
If both blocks fit into cache, we can be sure that each value gets reused \(B\) times, whereas before, values would be evicted on the order of \(n\) times.
And if \(B\) is large enough, we will still be using cache lines optimally, so this factor cancels out.
This means that for suitably chosen \(B\), by also exploring temporal locality, we do get a speedup of roughly \(B\) in memory access!

```c++
for (size_t ib = 0; i < n; ib += B) {
	for (size_t jb = ib; j < n; jb += B) {
		for (size_t i = ib; i < std::min(ib + B, n); ++i) {
			for (size_t j = jb; j < std::min(jb + B, n); ++j) {
				// do something if there is a collision between colliders[i] and colliders[j]
			}
		}
	}
}
```

This strategy is not just beneficial for caching: It also lends itself naturally to multithreading or even multiprocessing.

**Rule**: Process cartesian products blockwise.

It makes sense for our blocks to be contiguous in memory. In this simple example, that is not a problem.
But in applications like matrix multiplication, where the inputs are already 2d rather than 1d arrays, blocks are actual blocks.
At that point, you might also want to store them that way.

There are also other instances in which a "blocked" storage is preferable.
Imagine you have a 3d grid of voxels in a game.
You could try to represent that as one huge 3d array.
Then, along one axis, you have excellent spatial locality. A neighboring node is also neighboring in memory.
Along another axis, you have very large gaps, growing linearly with the map dimensions, which is effectively no spatial locality;
and along the last axis, you have even larger gaps, growing quadratically with the map dimensions.
This means that even simple neighborhood iteration will exhibit bad spatial locality for most accesses.
To better preserve spatial locality, you divide the map into "mapblocks" of a fixed, relatively small size.
The gaps between grid cells in the same mapblock in memory now depend on the mapblock dimensions rather than the map dimensions,
which is much better for cache locality if you operations tend to operate "locally" on areas of the map, as they usually do in games.
For example, once a mapblock has been loaded entirely into cache,
visiting neighbors is cheap for most of the contained nodes -- all but those on the boundary.
(There are also a number additional concerns a blocked representation greatly helps with;
among others, it facilitates an efficient representation of a map that is sparsely populated on demand,
for example if only mapblocks that are in proximity to players are to be generated and kept in memory.
In fact, this is another angle you can derive it from: You want some sort of sparse data structure for voxels,
say like a hash map, but that would completely wreck performance.
So you want to mix this with an array as we discussed earlier,
and then using (3d) arrays to store mapblocks is the natural choice.)

**Rule**: Use "blocked" storage to achieve spatial locality in higher dimensions.

## Summary

Writing programs in a cache-friendly manner is crucial for good performance;
and if we're lucky, we might even get cleaner code out of it!
I've presented a few "rules of thumb" to give you some ideas:

* Prefer dynamic arrays (`std::vector`) or proper deques (`std::deque`) over linked lists (`std::list`).
* Prefer hash maps (`std::unordered_set` and `std::unordered_map`) over sorted maps (`std::set` and `std::map`).
* Use more efficient alternatives to C++ STL data structures.
* Iterate higher-dimensional arrays in the right index order.
* Use an appropriate form of allocation.
* Try to eliminate unnecessary indirection; avoid putting small objects on the heap.
* Make the most out of already allocated memory.
* See whether an Array of Structs or a Struct of Arrays is more appropriate for your access patterns.
* Move fields "out-of-band" if a significant amount of memory is wasted on padding.
* Save space on often empty fields. Converting an AoS to a SoA is one way to do this.
* Mix AoS and SoA to suit your access patterns.
* Process cartesian products blockwise.
* Use blocked storage to achieve spatial locality in higher dimensions.

More generally, I hope to have given you some intuition for how to reason about how
different choices of data layout and corresponding access patterns may have an impact on program performance.

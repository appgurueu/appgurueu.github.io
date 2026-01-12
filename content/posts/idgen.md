---
date: '2026-01-12'
title: 'On ID generation'
---

It happens very often in software that we have some kind of object
and we want to give it some kind of unique identifier, called "ID" for short.
It's important that you get this right, or at least not horribly wrong.
So here's what you *can* do.

## Use the object itself as ID

Why redo what the memory allocator already did: Allocate, for each unique object, a unique memory address. [^malloc0]
[^malloc0]: Indeed `malloc` often even has the nice feat of giving you unique addresses for zero-sized allocations!

Most languages let you use "objects" (tables), by reference (memory address), as keys in a (hash) table.
You can even mark the keys as "weak" references to make sure that the
lifetime of your key objects is not extended due to them being referenced by these tables (e.g. for caching).

(An alternative solution is to just bloat the object itself with fields for everything you may wish to attach to it,
but that often leads to some ugly coupling.)

## Generate random IDs

Random is a powerful tool. Programmers should be less afraid to leverage it.
The problem is that we just don't have an intuition for very small quantities (or extremely large quantities, for that matter).
We think "so there *is* a chance it doesn't work" for a randomized algorithm (emphasizing a negligible risk),
whereas for a deterministic algorithm we think "this will *always* work" (sweeping under the rug all the other points of failure).

Randomized algorithms can often be significantly simpler,
yet perform better (in expectation) than deterministic algorithms. [^expectation]

[^expected]: Better in terms of *expected* ("average") runtime. This may seem like a weak statement,
but if the variance of your distribution is well bounded (which it probably is),
the chance that you have extreme outliers eventually becomes negligible
(see e.g. Chebyshev's inequality for a formal statement).
This is why it's generally fine to say that hash table lookups are constant time, for example.
There is a remaining concern with intelligent adversaries that try to exploit edge cases.
But if you do not give them control over the random bits upon which the runtime depends, they are effectively powerless.
Hence many programming languages today randomly choose a seed at startup to incorporate into their hash functions.

Random is pretty much a perfect fit for ID generation: Bits are cheap.
And if you throw enough independent bits at it, the chance of a collision
is so close to zero that the chance of a technical defect - or, you know, a bug! -
bricking your application is orders of magnitude higher. [^bday]

[^bday]: You have to be a bit careful with the math here: Due to the birthday paradox,
the chance of a collision is not about one in \(2^n\); rather it's about one in \(2^(n/2)\). This is why 64 bits are generally not enough.
You also have to ensure a good entropy source, but computers have had that for a long while for cryptography reasons alone.

This is basically what a "Universally Unique Identifier" (UUID) v4, with almost 128 random bits, is.
With that many bits, the chances of a collision between any two independently generated IDs
in this universe are [vanishingly](https://jhall.io/archive/2021/05/19/what-are-the-odds/) low.

This brings us to the next point: You want to avoid ID reuse and ID similarity.
This makes bugs, as well as user error, more likely to surface.
It also reduces your attack surface: Attackers need to know the ID, or try gargillions of IDs.
With random IDs, we get this for free.

They're easy to generate, depend on nothing but the universe's randomness, and two random IDs will on average differ a whole lot.

(Sometimes you want short, "memorable" IDs. But that is a bit of a different discussion.)

## Autoincrement

Autoincrement suffers from a related misconception: It *feels* like it will hit a limit at some point.
It feels like we're not cleaning up after ourselves.
Surely, at some point, we've incremented too many times, and we overflow, and bad stuff happens!

But again this is not true and it *just works*. Again we struggle to estimate magnitudes.
Again bits are by far too damn cheap.
It is impossible for whichever computer runs your software to keep incrementing IDs long enough to hit the limit of a 64-bit integer [^float].

[^float]: Or even a 53-bit mantissa 64-bit floating point number as found in scripting languages.

Autoincrement also works, autoincrement is also nice and simple and, if we generate few IDs, results in nice, small IDs which we might like.

But there are drawbacks versus random IDs. As said before: More similar IDs, higher risk of mistakes.
Attackers can easily guess valid IDs by incrementing.

Also: Contention. You can't parallelize this well, because everything needs to mutex the ID counter.
There are solutions to this (like allocating ID ranges), but they make things more complex again.
Random is more simple.

With persistence, you also run into some more risks.
Ideally everything would be consistent. Often it's not.
If you use random IDs, inconsistent states will be detected.
If you use autoincremented IDs, and somehow an ID gets persisted, but the global counter does not, you get collisions.
Horrible things ensue.

Now again, you can do things. You can mix timestamps in there. You can mix random in there.
You can mix your grandmother's maiden name in there. But why not just use random at that point?

## Do not reuse IDs

If you reuse IDs, I will come for you.

You don't have to do this. [^ex]

[^ex]: Of course it depends. But too often programmers are free to choose a reasonably sized ID type,
to just never reuse IDs, and still they choose some small-sized bullshit.
It's premature optimization all the way down.

There are enough IDs for everyone.

I do not want to see yet another unnecessary, potentially inefficient algorithm
that searches for a free 32-bit ID. [^sample]

[^sample]: If you do find yourself needing that kind of algorithm,
simple rejection sampling goes a long way: Just try a random ID in the range of valid IDs
until you find a free one. The expected runtime for this is constant if there are linearly many free IDs.

Programmers often solve the wrong problem when they generate IDs.
They reuse IDs when there is no need to.
They optimize the size of the ID by a negligible constant factor when there is no need to.
They tend to do this at the cost of complexity, of robustness, of security, possibly of performance [^perf].

[^perf]: Of ID generation performance specifically: Atrocities such as linear searches for free IDs.

Don't do this.

Rule of thumb: Use **autoincrement** or **random** with **enough bits** and you will be golden.
Especially for *universally unique* identifiers, random is *the* solution.

It can be this easy. Why make it harder?

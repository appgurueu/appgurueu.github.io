---
layout: post
title: "Sticks and stones won't break my bones"
date: 2024-02-20
tags: luanti, irrlicht, gamedev, foss, math
---

... but [Irrlicht](https://irrlicht.sourceforge.net/) will.

This software horror tale has all you expect:
There's an old legacy C++ codebase, a familiar culprit and ultimately,
a bugfix followed by workarounds to put the bug back in to some extent.

## The beginning

Many years ago [Luanti](https://luanti.org) (then called Minetest) suffered from a major bug:
Skeletal keyframe-based animation and bone overrides could not be used together.

As soon as you overrode *any* bone, the entire model would cease animating.

This was of course very nasty for mods that wanted to animate the player's head or right arm
via bone overrides to signal what the player was looking at or interacting with.

However, there was an ugly hack, to my knowledge first used by a mod called [`playeranim`](https://github.com/minetest-mods/playeranim/)
You could animate all bones serverside, in pure Lua, in your mod,
and then set them all via bone overrides.

The way `playeranim` did this, all animations were hardcoded, which in general sucks badly,
but was about good enough for players specifically.

The animation would look much jankier, but at least the player would animate at all,
and you would get an overridable head and arm. [^1]

[^1]: And, in fact, you could - at least in theory - implement further skeletal animation features on top, such as the ability to "disable" bone overrides,
which Minetest lacked for a long while, or the ability to blend and compose animations.

Now, I was not very fond of how this hack was implemented:
Since the animations were hardcoded, they were less nice. Some slight nudges were omitted, for example.
It also wouldn't be able to support another player model with different animations easily.

Since I had written a [b3d loader](https://github.com/appgurueu/modlib/blob/f6de802d6f50cab6eee1f0d760a7f2f737d81a42/b3d.lua),
I wanted to write my own mod to do this, but it would properly read the animations from a file.
(Frankly, this was a common theme throughout my mods: Reimplementing engine functionality in pure Lua for one reason or another.)

So that's what I did. And it didn't work. And it didn't make a lick of sense.
No matter which rotation order I assumed, no matter how I permuted the axes, no matter which combination of signs I picked,
no matter whether clockwise or counterclockwise - nothing worked. Something was always broken.

After long trial and error, I found something which *somehow* made sense - it seemed like the animated rotations
needed to be relative to the *default* (static pose) rotations of the bones.
Since I wanted to move on from a hacky workaround for character animations at some point,
I just implemented that (despite it not being how skeletal animations work:
the rotation should simply override the default rotation), and called it a day.

(In hindsight, why this worked for the most part makes perfect sense, but we'll get to it later.)

## The heisenbug

There was a bug where sometimes, animated models would briefly appear to "flicker" for a frame.
If you recorded this, you could see that what was actually happening during that frame was that
some bones were rotated wrongly by a half-turn (180°).

An ugly workaround was put into Luanti where it would simply compare absolute rotation around the X axis with expected rotation and,
if they didn't match, recalculate the absolute transform.

## The red herring

For a while it seemed to me (and others, judging by the comments on the function)
like something was wrong with getting rotations back out of matrices.
Maybe the implementation was unstable and exhibited some funny behavior at extremal points?

At least it seemed that, if the matrix decomposition, including getting a rotation back out,
was gotten rid of entirely, the bug was gone, and the workaround obsolete.

But this was a red herring: rotations were just one side of the coin;
the other, broken side was the less suspicious (because seemingly much simpler) function to get scales back out.
Getting rid of `getRotationDegrees()` simply correlated with getting rid of the broken `getScale()`.

`getRotationDegrees()` ultimately proved algebraically correct and persevered when subjected to decent unit test coverage,
at which point I was certain that "Chev" wasn't to blame. Sorry Chev!

## The explanation

Let's take a step back.

First, some linear algebra. Let's talk matrices. Skeletal animation is done using *TRS transforms*:

4x4-matrices which are the product of a translation ($T$), rotation ($R$), and scale ($S$) matrix, in that order.
Note that function application is right-to-left, so this means a TRS transform first scales, then rotates, and finally translates vectors.

We won't be looking at the translation here; that's hard to mess up and irrelevant to the bug. We'll be focusing on the 3x3-submatrix that does the rotating and scaling.

This is the product of a rotation matrix and a scale matrix. We'll be using "column conventions" here:
The columns of a matrix are the images of the canonical base vectors  
$x = ((1), (0), (0)), y = ((0), (1), (0)), z = ((0), (0), (1))$.

A scaling matrix $S$ with component-wise scale factors $sigma_x, sigma_y, sigma_z$ is just a diagonal matrix:

$S = (
	(sigma_x, 0, 0),
	(0, sigma_y, 0),
	(0, 0, sigma_z)
)$

For a rotation matrix, all we need to know here is that it preserves lengths, thus it has the form

$R = (
	(r_x | r_y | r_z)
)$ where $r_x, r_y, r_z$ are 3-dimensional column vectors of length 1:  
$||r_x||_2 = ||r_y||_2 = ||r_z||_2 = 1$, where  
$||((v_x), (v_y), (v_z))||_2 = sqrt(v_x^2 + v_y^2 + v_z^2)$ denotes euclidean length.

This means that the matrix product $RS$ is pretty simple to express column-wise:

$RS = (
	(sigma_x r_x | sigma_y r_y | sigma_z r_z)
)$

The images of $x, y, z$ are simply the scaled and rotated images respectively.

Thus, to decompose a product $RS$ into rotation and scale, all we need to do is find the length of each column vector of $RS$.

So now, take a look at this innocuous Irrlicht function to get the scale of a matrix:

```c++
template <class T>
inline vector3d<T> CMatrix4<T>::getScale() const
{
	// See http://www.robertblum.com/articles/2005/02/14/decomposing-matrices
	// Deal with the 0 rotation case first
	// Prior to Irrlicht 1.6, we always returned this value.
	if (core::iszero(M[1]) && core::iszero(M[2]) &&
			core::iszero(M[4]) && core::iszero(M[6]) &&
			core::iszero(M[8]) && core::iszero(M[9]))
		return vector3d<T>(M[0], M[5], M[10]);
	// We have to do the full calculation.
	return vector3d<T>(sqrtf(M[0] * M[0] + M[1] * M[1] + M[2] * M[2]),
			sqrtf(M[4] * M[4] + M[5] * M[5] + M[6] * M[6]),
			sqrtf(M[8] * M[8] + M[9] * M[9] + M[10] * M[10]));
}
```

Don't mind the cryptic indices here - that's just another classic case of poor Irrlicht code quality. [^irrsucks]

[^irrsucks]: Mind you, they do have operator `()` overloaded to allow 2d indexing -
I can only assume that the author was terribly worried that using this with constant indices
might cost them another CPU cycle in a debug build.
Or that it simply didn't occur to anyone to use this simple abstraction.

Note the masterful optimization: If all non-diagonal entries are close to zero,
we can save ourselves the computation and read the scale straight from the diagonal!
Billions of otherwise wasted CPU cycles have been saved! [^backcompat]

[^backcompat]: As another factor, whoever updated this function to properly calculate the column lengths on the slow path
might've been just a little *too* worried about perfect backwards compatibility for diagonal matrices and due to that created a nonsensical,
non-continuous function. It would probably have been wiser to split this up into a pure "get diagonal" and "get column scales" functions,
or to get rid of the former entirely and only have a plain and simple "get scale" function.
Negative scaling isn't properly supported presently;
the only real application is mirroring by inverting an axis (because, as we established, flipping two signs can be done using rotations instead),
and that creates various problems and makes the decomposition into rotation and scale not unique,
hence it never worked to begin with.

There's just one tiny problem: Something is missing. Take a closer look at the `return vector3d<T>(M[0], M[5], M[10]);`.

What about the signs? What if there are negative diagonal entries? You get *negative scale*.
Correct would be taking the absolute value of each diagonal element: `return vector3d<T>(std::abs(M[0]), std::abs(M[5]), std::abs(M[10]));`.

Well, you might be thinking, that ain't so bad. Maybe it even makes sense: What if someone did in fact set a negative scale?

There's just one major problem: Many rotations are nothing but negatively scaling two axes.

The assumption that "no off-diagonal entries, therefore no rotation" is *plain wrong*.

Consider a rotation of 180°, a "half-turn". Do it with your fingers or on a piece of paper, or just imagine it in your head:
What it does is precisely flip the signs of both components.

Thus in 3d, any rotation of 180° around one of the three cardinal axes, does just that: It flips two signs.

And any composition of such "perfect" rotations will do the same thing. For example a 180° rotation around the X axis will invert Y and Z.
A 180° rotation around the Z axis will invert Y and X. If you compose the two, you end up with X and Z inverted -
equivalent to a 180° rotation around the Y axis! [^quat]

[^quat]: In quaternions: $ik ~= +-j, ij ~= +-k, jk ~= +-i$. Note that the sign doesn't matter for half-turns $i, j, k$;
these are just diagonal matrices, they commute.
The order of operations doesn't matter when only half-turns are involved.

From the fact that rotations must have determinant $1$, and since each column has length 1,
we can deduce that there can either be one negative sign or two negative ones on the diagonal.

So these are the three possibilities for two negative ones:

* X and Y are inverted (half-turn around Z)
* X and Z are inverted (half-turn around Y)
* Y and Z are inverted (half-turn around X)

Note that it is *not possible* for a rotation to invert just a single axis: That would have determinant -1.

And in fact, in a different place (`CMatrix4<T>::getRotationDegrees()`), someone did notice this:

```c++
// We assume the matrix uses rotations instead of negative scaling 2 axes.
// Otherwise it fails even for some simple cases, like rotating around
// 2 axes by 180° which getScale thinks is a negative scaling.
if (scale.Y < 0 && scale.Z < 0) {
	scale.Y = -scale.Y;
	scale.Z = -scale.Z;
} else if (scale.X < 0 && scale.Z < 0) {
	scale.X = -scale.X;
	scale.Z = -scale.Z;
} else if (scale.X < 0 && scale.Y < 0) {
	scale.X = -scale.X;
	scale.Y = -scale.Y;
}
```

(You don't even need to rotate around two axes by 180°; that's just a more contrived way of rotating by 180° around the third axis.)

But whoever noticed this clearly didn't bother to fix it at the root.

So what happened? Irrlicht was (mostly unnecessarily) decomposing matrices corresponding to TRS transforms.

When it did the RS decomposition, it shoved perfect 180° rotations into negative scale.

If something - like an animation, or a bone override - now overrides the rotation, it's *wrong*:
It will be relative to the negative scale. Both the default rotation and then, relative to it,
the overriding rotation are applied - when only the latter should be applied.

In an animation, the half-turn will be applied *twice*,
once by the scale, and once by the rotation.

This is not terribly bad since typically, the "perfect" rotation will only exist very briefly for a single frame,
so you get the described "flickering", of which there are two instances:
Once [for attachments](https://github.com/luanti-org/luanti/issues/11991) [^morebugs],
and once for the models themselves, as you've already seen above.

[^morebugs]: Note that these are [desynced by about a frame due to yet another bug](https://github.com/luanti-org/luanti/issues/14818),
enabling attachments to still be wrong even though the workaround "fixed" animations to some extent.

For bones, it's much worse, because the negative scale will exist *permanently* (unless you override that too).

This is why, in my testing of `character_anim`, it seemed as if animated rotation was relative to default rotations:
Because Irrlicht shoveled the rotations into the scale, which is precomposed with the rotation!

At the time, I did also observe [similar flickering](https://www.youtube.com/watch?v=K4xtXjgrAL0) when overriding the "Body" bone,
which in hindsight was yet another instance of this bug.

## The Minetest workaround

When someone fixed the long-standing bug of bone overrides disabling skeletal animation completely,
they also put a workaround for this bug in: Look for seemingly wrongly rotated bones
(with somewhat dubious euler angle logic), and if the rotation you got from the matrix seems off,
put it back in.

You can see the historical details [in this PR](https://github.com/luanti-org/luanti/pull/9577/).
I conjecture that the workaround worked due to incurring numerical errors typically large enough
to create off-diagonal elements that prevented the wrong path in `getScale()` from being taken.

Note however that the workaround *explicitly excluded* manually overridden bones,
so these were certainly still left in a nonsense state where you needed to supply bogus rotations.

## The Minetest workaround: Electric Boogaloo 2

Based on this PR and my bad experience trying to override bones,
I [tried to extend the workaround](https://github.com/luanti-org/luanti/pull/10405).

In hindsight I must say: My hunch that this workaround ought to apply not just to X,
but to all three axes, was correct, as was the notion that overridden bones were affected as well!

Unfortunately, I later ended up [reverting it](https://github.com/luanti-org/luanti/pull/10534),
on the grounds that `playeranim` and `character_anim` were broken -
I did not consider at the time (but really should have) that instead these mods were simply setting nonsensical rotations
to work around an ancient engine bug they had had to get used to for them to work at all.

Ultimately trial and error wasn't enough to properly address the issue and just ended up with going in circles
and wasting development time.

## The fix

At long last, after long sessions of working on the skeletal animation code
(which culminated in a bunch of refactorings and most importantly
[a PR to fix most of the outstanding jankiness](https://github.com/luanti-org/luanti/pull/15722)),
I had understood the problem and found a fix. [^dbg]

[^dbg]: Debugging this was nasty, because being a fleeting visual issue, it wasn't trivial to inspect.
I temporarily recorded videos of the screen along with debug output side-by-side,
but ultimately stepping through frames with a debugger at the suggestion of [a friend](https://lizzy.rs) I had vented my frustrations to
(after fixing animation steps to 60 Hz with some hacky edits) was much more helpful.

The resulting [changes](https://github.com/luanti-org/luanti/commit/8719a816e723f48e78480107e4d8e9056f45da8a) were rather neat.
I opted to get rid of the premature optimization entirely (rather than fixing it) for a couple reasons:

* No performance need has been demonstrated. Why overcomplicate the code?
* I will get rid of the unnecessary matrix decompositions in performance-critical code later anyways;
  I've looked at all usages and I don't expect there to ever be a serious performance need.
* It is unclear whether it is an optimization at all; it depends on the matrices
  and processor specifics. It saves some square roots, but incurs a bunch of branching.
  I'm too lazy to benchmark, but it certainly isn't *obvious* that bailing out earlier *is even faster* to begin with;
  it certainly isn't for matrices where you don't get to bail out early and only pay for the check.

Of course, now that `getScale()` was fixed, I also simplified `getRotationDegrees()`.

## The aftermath

Unfortunately, mods setting bogus rotations to work around the ancient engine bug... are still doing that.

Few people really look at the numbers and notice that they don't match what's in the models.
And if they did, they'd just realize it makes little sense and end up with bogus logic like `character_anim` (well, `modlib` to be precise).
Most mod developers probably ended up just trying rotations until things looked right if they wanted to use bone overrides at all.

So now, we find ourselves in need of a workaround which works on clients prior to 5.11,
which will still be running the faulty decomposition,
just as well as 5.11+ clients.

I [filed an issue](https://github.com/luanti-org/luanti/issues/15692) for the not-a-bug regression in behavior,
where I explored such workarounds and tried to estimate who would be affected.

But, talking to I realized, that I did a really crappy job at explaining things, which is what led me to write this post.

### Identifying affected mods

We can largely ignore animations, because for them we can almost always expect the bugfix to really be a bugfix,
getting rid of the "flicker frame".

This leaves only bone overrides. Mods are affected if they override a bone with a perfect 180° default rotation.

Looking for such perfect default rotations, and suggesting equivalent absolute negative scales for a workaround to be revealed shortly,
[can be automated using `modlib`](https://gist.github.com/appgurueu/e8c4affd1b121b50decb98a40aaf34a6).

Looking for bone overrides can be done using a simple `grep -r set_bone_`, which will match both `set_bone_position`
and `set_bone_override` (and often also utility wrappers around the aforementioned methods).

Primarily and most noticeably, player-animating mods such as
`playeranim`, `character_anim` and their equivalents built into VoxeLibre and Mineclonia are affected.

These mods typically end up overriding many if not all player bones, doing animation completely serverside.
Almost all player model bones have a perfect rotation. Thus players end up looking quite funny -
in the words of a VoxeLibre bug report, ["put ya hands up in the air if you just don't care!"](https://git.minetest.land/VoxeLibre/VoxeLibre/issues/4930)

![A VoxeLibre player is being held at gunpoint](breaking-bones/hands-up.png)

Luckily [`headanim`](https://content.luanti.org/packages/Lone_Wolf/headanim/) is not affected,
because the head bone has zero default rotation in typical player models.

### The engine

I ultimately decided against putting this bug back into the engine.
There were multiple contributing factors to this decision:

* It is a bug, plain and simple. I do not want bone overrides to be buggy by default.
* The current skeletal animation code is already in a really bad and fragile state.
  I do not want to make this any worse than necessary.
* It is possible for modders to work around this issue such that both older and newer clients are happy.
  Even more, modders could tweak their models a bit and then use the correct override rotations.
* The extent of breakage is bad, but not critical: It's purely a visual issue
  and far from everything is affected. Primarily, more or less hacky player-(re)animating mods suffer.

I'm still not certain I made the right decision, but I don't feel that it is entirely and obviously wrong either.

## The workarounds

I've come up with three different workarounds for this issue.
All of them do to some extent make it work on both older and newer clients.
They correspond to three different strategies:

* Avoid triggering the buggy logic by slight perturbation of model rotations.
* Retroactively "fix" the bug on older clients by forcing the correct scale.
* "Put the bug back in" for newer clients by forcing the wrong, negative scale.

### Wiggling bone rotations

This workaround is purely *model-side* and *could be done manually in a model editor*.

You simply look for bones with such perfect rotations and wiggle them a tiny bit such that they cease being perfect;
the off-diagonal elements of the rotation matrix calculated by Irrlicht exceed Irrlicht's threshold.

If these rotations are always overridden, this workaround should be *completely unnoticeable*.

With the right infrastructure around b3d loading, writing, quaternions, and media paths,
you can even automate this entirely in not terribly many lines of code,
which is what I ended up doing for `character_anim`:

For each model with flawed rotations, `character_anim` generates a static model
(because it animates every bone manually anyways) with slightly wiggled rotations.

Hooking `set_properties` and `get_properties`, `character_anim` then completely transparently pretends to still be using the old model.

The rotations `character_anim` used had to be fixed, which was effectively just a
["one-liner" bugfix change in `modlib`](https://github.com/appgurueu/modlib/commit/bb83bb5848c5aeb2c5feaa49c3e9b4a0f2ea5773):
It had to stop precomposing with the default node rotation.

If you were to fix animations manually, fixing rotations is nastier:

Let $R$ be the current rotation.
We know that $R$ effectively ended up as $R R_D$, where $R_D$ is the default rotation, applied via negative scale.
Hence, to achieve the same effect now that negative scale is gone, we simply need to "precompose" $R$ with $R_D$:
First apply $R_D$ (a 180° rotation), then apply $R$.
With euler angles, that's a bit nasty. With quaternions it's rather simple however.

### Forcing the correct decomposition

Since 5.9+, Minetest gained the ability to override bone *scale*.
This can be used to retroactively fix the bug on older clients using server-side logic:
When setting a bone override, you simply also override the scale to be correct.
Typically, this will be a scale of $((1), (1), (1))$,
but it might also be different: Check your model to be sure.

Of course, now the rotation is no longer relative to the perfect default rotation, so it needs to be fixed too -
same as when wiggling default rotations!

### Forcing the wrong decomposition

This is exactly the complement of the previous trick:
Instead of fixing the bug for older clients, we can reintroduce it for newer (5.9+) clients.

We simply look for bones with perfect rotations equivalent to negatively scaling two axes,
and we stuff those two signs into the scale.

For example, if a bone has a default rotation of 180° around the Y axis and a scale of $((1), (2), (3))$,
we would now set a scale of $((-1), (2), (-3))$.

(This is just an example, typically you will only have $+-1$ scale components.)

### Comparison

#### Wiggling bone rotations

This works for all clients and all server versions; it really avoids the bug.
Overall I feel this workaround might be slightly overengineered, but it should work well once it's in place.

#### Forcing the correct decomposition

A major limitation of this is that it requires both client and server to use at least version 5.9.

An older client with on a newer server won't work (the client won't support overriding scale).
A newer client on an older server won't work either (the server won't support overriding scale).

Furthermore, it requires all rotations to be fixed, which can be nasty to do quickly
in larger codebases which use lots of procedural animation and are presently using euler angles
(though this could be done properly via quaternions).

Arguably leaving the wrong rotations in the code isn't great long term,
but is more convenient short term.

Thus this probably won't be a very popular workaround, but it is worth mentioning nevertheless.

#### Forcing the wrong decomposition

The only situation where this workaround fails to work is with a 5.9 or newer client connected to a 5.8 or older server,
because the newer client won't get the scale. Thus this is technically slightly inferior to wiggling rotations.

Otherwise it is quite a decent workaround, avoids having to fix rotations, and avoids having to edit models (manual or automated).

This makes it very suitable for a [general workaround mod](https://codeberg.org/appgurueu/bone_workaround):
You simply use a b3d reader to anticipate the bug
and then hook `set_bone_(override|position)` to put the wrong scale right back in. Voilà!

### Status

A quick summary of the status of some popular mods and games regarding this problem:

* [x] [`headanim`](https://content.luanti.org/packages/Lone_Wolf/headanim/): Unaffected. KISS paid off. Chapeau.
* [x] [`character_anim`](https://content.luanti.org/packages/LMD/character_anim/): Affected, [fixed using the rotation wiggling (model editing) workaround](https://github.com/appgurueu/character_anim/commit/61b62ae2c71518bf6363736a63f54010f969c78a).
  * Note that this also requires [a `modlib` fix](https://github.com/appgurueu/modlib/commit/bb83bb5848c5aeb2c5feaa49c3e9b4a0f2ea5773).
  * As said: Possibly overengineered workaround. But: This means it works in a couple more constellations, and we can use the correct rotations right away.
* [ ] [`playeranim`](https://content.luanti.org/packages/Rui/playeranim/): Affected, [PR using the negative scale workaround pending](https://github.com/minetest-mods/playeranim/pull/6).
* [ ] [VoxeLibre](https://content.luanti.org/packages/Wuzzy/mineclone2/): Affected, [PR using the negative scale workaround pending](https://git.minetest.land/VoxeLibre/VoxeLibre/pulls/4932).
* [x] [Mineclonia](https://content.luanti.org/packages/ryvnf/mineclonia/): Affected, [fixed using the negative scale workaround](https://codeberg.org/mineclonia/mineclonia/pulls/2918).

## Afterthoughts

I hope you enjoyed the story as little as I did. Now time for some morals of the story!

Throw out your premature optimizations - they will cost you.

Have some good unit test coverage for your code, especially if you think it's tricky.

Dive deeper: Don't always fix (or rather, work around) bugs narrowly at too high a level.
Similarly, ideally, modders like me would have figured out and reported that bone override rotations are bogus.

Don't just throw code at the problem and see if it sticks: You should have a clear idea of
*why* the code you wrote works. Otherwise you *will* rely on unspecified, possibly buggy behavior,
and the bugs might very well never be reported and fixed (and neither will the documentation be improved),
so everyone is forced to fling the same shit (or worse, inherit flung shit) until it sticks.

<!--
Use proper abstractions for rotations. Euler angles suck.
(Ideally the engine should provide these, but until it does, it isn't that hard to implement quaternions yourself.)
-->

Of course you'll have to compromise on these morals.
But you should at least try to a reasonable extent.

And finally:
Never underestimate how much work is required for very foundational parts of a game engine -
in this case, a small fraction of skeletal animation.
A few missing calls to `std::abs` were enough to generate substantial amounts
of work down the line - for everyone, mod and engine developers alike, over a span of years.

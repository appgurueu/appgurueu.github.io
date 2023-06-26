---
layout: post
title: "Approximating π"
date: 2023-06-26
tags: math, computer science, pi
---

<!-- TODO find a cleaner solution than putting this here -->
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
<script>
MathJax = {tex: {inlineMath: [['$', '$']]}};
</script>

Toying with a few intuitive (and less intuitive but more efficient) ways to approximate $\pi$.

## Throwing darts

This is perhaps the simplest method I've encountered yet -
simple enough that you could teach it to absolute beginners
once you've introduced `math.random`.

Based on the formula $A = \pi r^2$ for the area A of a circle,
$\pi$ can be defined as the ratio of a circle with radius $r$
to a square with both sides of length $r$.

In particular, for $r = 1$, we obtain that $\pi$ is
the ratio of the area of the unit circle to the unit square,
that is, $\pi$ is the area of the unit circle.

If you now throw darts at the unit square, the probability that you hit the unit circle
is the area of the quadrant of the unit circle, divided by the area of the unit square,
which is exactly one fourth of $\pi$.

Thus after throwing $n$ darts, the expected number of hits is $\mu = \frac{\pi}{4} n$.
If we take the number of hits times 4 (since there are 4 quadrants of the unit circle)
and divide by $n$, we obtain an expected result of exactly $\pi$.

![Visualization](/assets/approximating-pi/darts.svg)

In code, this is very straightforward:

```lua
local n = 1e8 -- many, many darts
local hits = 0
for _ = 1, n do
	local x, y = math.random(), math.random()
	if x^2 + y^2 < 1 then
		hits = hits + 1
	end
end
local pi = 4 * hits / n
print(pi) -- something like 3.14174612
```

(this can be optimized slightly on LuaJIT by counting misses using
`misses = misses + math.floor(x^2 + y^2)` instead, which makes the loop branchless)

Unfortunately, this simplicity comes at a great cost;
not only is this method very inefficient, converging very slowly,
it also depends greatly on the uniformity of the used random
and thus on the "pseudorandom magic" behind the scenes.

Mathematically, each dart has the same chance constant chance of $p = \frac{\pi}{4}$ to hit, yielding a binomial distribution.

The standard deviation for the binomial distribution is given by $\sigma = \sqrt{p(1-p)n}$.
For $\sigma > 3$, we can use a normal distribution to approximate the binomial distribution,
giving us confidence intervals for how probable it is that we get within a certain range of $\pi$:
99% of values are guaranteed to be in the range of $\mu - 2.58\sigma$ to $\mu + 2.58\sigma$.
The absolute deviation from $\frac{\pi}{4}$ will be $\pm 2.58\sigma$, giving a deviation from $\pi$ of $\pm 4 \cdot 2.58 \sigma$.
For the $10^8$ darts used above, this yields a deviation of $\approx 0.000424$.

We're interested in how the deviation, which is linear in $\sigma$,
decreases as $n$ increases: $\frac{\sigma}{n} = \frac{\sqrt{p(1-p)n}}{n} = \sqrt{p(1-p)} \frac{1}{\sqrt{n}}$ which is proportional to $\frac{1}{\sqrt{n}}$.

Note that if we multiply $n$ by a constant factor, say $10$, we get $\frac{1}{\sqrt{10n}} = \frac{1}{\sqrt{10}} \cdot \frac{1}{\sqrt{n}}$
\- that is, our deviation only improves by a constant factor of $\frac{1}{\sqrt{10}}$ too.

This means that to get another digit of precision, we need to multiply the number of throws with a constant factor;
the number of throws grows *exponentially* in the precision we want to reach with a certain fixed probability (99% here).

## Rasterization

Sticking with the ratio-of-areas definition of $\pi$, we can devise another method to approximate it:
Through rasterization. Rather than throwing darts, we simply rasterize a quadrant of the unit circle
in order to approximate the ratio of the area of the circle to the area of the square.

### Simple iterative variant

![Visualization](/assets/approximating-pi/pixels.svg)

Again this translates straightforwardly into code:

```lua
local m = 1e4
local misses = 0
for x = 0, m - 1 do
	for y = 0, m - 1 do
		-- Check whether the lower left corner is inside the circle
		misses = misses + math.floor((x^2 + y^2) / m^2)
	end
end
local pi = 4*(1 - misses / m^2)
print(pi) -- 3.14199016
```

The runtime obviously grows quadratically in the resolution $m$ here.

### Recursive variant

A picture says more than a thousand words:

![Visualization](/assets/approximating-pi/pixels_recursive.svg)

The idea of the recursive approach is simple:
If we know that an entire square is entirely inside or outside of the circle,
we can account for its area without recursing further;
otherwise, we subdivide the square into four equally sized squares and recurse;
if the recursion depth is reached, we consider the squares inside of the circle, overestimating $\pi$.

This effectively approximates only the *border*, thus being much more efficient
(intuitively, it is easy to imagine that the number of squares at the maximum recursion depth
intersecting the border of the circle grows only linearly in the resolution).
The resolution is in term exponential in terms of the maximum recursion depth.

Since the function only does constant time checks and recurses four times,
the run time of the recursion is bounded by the amount of leaves of the recursion tree,
which is again bounded by the count of leaves on the lowest level.

```lua
local min_size = 2^-24 -- minimum size of a pixel
local function approx_outside_area(x, y, size)
	if size <= min_size then
		return size^2 -- overestimate pi
	end
	-- upper-right corner is inside circle => square is fully contained in the circle
	if (x + size)^2 + (y + size)^2 <= 1 then
		return size^2
	end
	-- lower-left corner is outside of circle => square is not contained in the circle
	if x^2 + y^2 > 1 then
		return 0
	end
	-- otherwise, split four ways
	size = size/2
	return approx_outside_area(x, y, size)
		+ approx_outside_area(x + size, y, size)
		+ approx_outside_area(x, y + size, size)
		+ approx_outside_area(x + size, y + size, size)
end
local pi = 4 * approx_outside_area(0, 0, 1)
print(pi) -- 3.1415931302873
```

## Analytical variants

### Finding a root of `sin`

Analytically, $\pi$ is often defined as the root of $sin$ between $3$ and $4$:
Since $sin(3)$ is positive, $sin(4)$ is negative and `sin` is continuous,
at least one root must exist.
It can also be shown that this is the only root by showing that $sin$
is strictly decreasing in the given interval.

Constructively, since $sin$ is strictly decreasing in the interval,
the root can be found using a simple binary search on `math.sin`.
Based on the needed precision, the search is terminated after a fixed number of steps.

```lua
local max_depth = 50
local function binary_search_root(func, from, to, depth)
	local mid = (from + to) / 2
	if depth == max_depth then
		return mid
	end
	if func(mid) < 0 then
		return binary_search_root(func, from, mid, depth + 1)
	end
	return binary_search_root(func, mid, to, depth + 1)
end
print(binary_search_root(math.sin, 3, 4, 1)) -- 3.1415926535898
```

For comparison, `print(math.pi)` yields the exact same result.
Note however that coercion to string rounds floats in Lua, losing precision.

The nice property of this algorithm is that we linearly
gain precision with the depth of the binary search:
Every iteration of the binary search gives us another bit of precision.

Considering that the mantissa of 64-bit floats has 52 bytes,
a depth of around 50 already yields very good results close to `math.pi`.

However, we cheated a bit by using `math.sin` here.

#### Approximating `sin`

`sin` can be approximated using a power series:

```lua
local sin
do
	local n = 42
	function sin(x)
		local res = x
		local x_sq = x^2
		local sign, pow, fac = 1, x, 1
		local j = 2
		for _ = 2, n do
			sign, pow, fac, j = -sign, pow * x_sq, fac * j * (j + 1), j + 2
			res = res + sign * pow / fac
		end
		return res
	end
end
```

This yields similarly good results:

```lua
print(binary_search_root(sin, 3, 4, 1)) -- 3.1415926535898
```

### Madhava-Leibniz-Series

Not all series are created equal. As noted on Wikipedia, ["this one converges very slowly."](https://en.wikipedia.org/wiki/Approximations_of_%CF%80#Middle_Ages)

```lua
local n = 1e9
local sign = 1
local sum = 0
for i = 1, n, 2 do
	sum = sum + sign / i
	sign = -sign
end
local pi = 4 * sum
print(pi) -- 3.1415926515893
```

### The other Madhava-Series

This one converges much faster.

```lua
local n = 50
local sum = 0
local pow = 1
for i = 0, n do
	sum = sum + pow / (2*i + 1)
	pow = pow / -3
end
local pi = math.sqrt(12) * sum
print(pi) -- 3.1415926535898
```

If you look carefully, you notice that we're cheating again by using `math.sqrt(12)`;
the square root of 12 would have to be approximated
using binary search or Newton's Method otherwise.

### The real one

In our analysis, we didn't bother with number precision or big decimals;
we simply used Lua's builtin number type, which uses 64-bit floats under the hood,
and could thus assume constant time operations.

For world record approximations of $\pi$, you'll need much, much more precision;
arithmetic operations on numbers can't be assumed to be constant time anymore.
The most recent world record of [approximating $\pi$ to 100 trillion digits](https://cloud.google.com/blog/products/compute/calculating-100-trillion-digits-of-pi-on-google-cloud)
was set using the [Chudnovsky algorithm](https://en.wikipedia.org/wiki/Chudnovsky_algorithm).

## Summary

There are a couple very simple, visually intuitive methods to approximate $\pi$
such as a simple stochastic approach ("throwing darts") or rasterization based on the area definition of $\pi$.

Much better results are however obtained using less intuitive analytical approaches,
such as approximating a root of $sin$ or evaluating series which converge against $\pi$.

For serious number crunching, dedicated algorithms such as the Chudnovsky algorithm exist.

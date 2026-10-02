---
date: 2026-04-12
title: "Choosing the right mean"
---

You have a system consisting of many components.
Say you are comparing runtimes of a few programming language implementation benchmarks.
So, you've done some benchmarks, measured some runtimes.
Typically, you've optimized some components and now want to demonstrate how great your implementation is.
So you compute the ratios of your optimized implementation vs. the reference implementation, the so-called "speedups".
You get a bunch of speedups $s_1, ..., s_n$ on different benchmarks.
If you've not done a terribly job, nothing should be substantially slower
(all speedups should be roughly at least 1, there shouldn't be large slowdowns).
Some speedups might be considerable, perhaps improvements by orders of magnitude.
But now you've got a bunch of numbers. You want one number. You want a mean.
You want to say, "I'm on average this much faster on this set of benchmarks".

But what is a "good", or at least not awful, notion of "average" here?

We make the simplifying assumption that all benchmarks are equally important;
if these were different components of a system, that they are used in roughly equal parts.

The first thing that comes to mind is an arithmetic mean:
Sum up all the speedups, divide them by the number of benchmarks.
As a formula, $\frac{1}{n} \cdot \sum_{i=1}^n s_i$.

If we were looking at *runtimes* instead of *speedups*, this would be perfectly appropriate.
But we aren't. And thus a simple arithmetic mean is not it, since it blatantly violates our assumption.
This leads to a violation of Amdahl's law.
Suppose you have two components, one of which you sped up 999x,
and a second one which remains at 1x. Does that give you a speedup of \(1000/2 = 500\)x
for the total time it takes to run the benchmarks? No, not at all.
Instead, you expect roughly a ~2x speedup as the second component becomes the bottleneck.

Do note however that this is an excellent way to inflate your speedups:
Pretend that some very specific order of magnitude improvement somehow
translates to an improvement "across the board" when it really doesn't.

Now, people see this and they go "oh, wasn't there a multiplicative mean?",
and so they use the geometric mean: $(\prod_{i=1}^n s_i)^\frac{1}{n}$.
And that's.. not as bad. For the above example, we would get $\sqrt{1 \cdot 999}$,
which is roughly ~30x. It's less wrong, but still wrong. It still grossly violates Amdahl's law.

A proper formula is quite easy to derive. Suppose each component, initially, had a runtime of $1$ by our assumption of equal importance.
Then, the runtime after the speedup is $t_i := \frac{1}{s_i}$.
Thus the runtime of the whole system is the sum of these runtimes, $t := \sum_{i=1}^n t_i$.
Then the expected total speedup of the whole system is $\frac{n}{t}$.
This is the so-called [harmonic mean](https://en.wikipedia.org/wiki/Harmonic_mean).
If we define $t_i := \frac{w_i}{s_i}$ instead, we can also assign weights to the different components depending on their assumed importance.

### Frame challenge

Why try to reconstruct a runtime expectation from the parts when you can measure the system as a whole instead?
Sometimes there are reasons for this, e.g. a speedup can neatly be bounded,
and you want to take an average of these bounds to see how close you are to an upper bound.
But often it's probably a good idea to try to measure the performance of larger systems.

Is the geometric mean useless? No, not at all, but it is a red herring in this situation.
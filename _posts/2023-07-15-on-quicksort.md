---
layout: post
title: "On Quicksort"
date: 2023-07-15
tags: computer science, algorithms, quicksort, sorting
---

<!-- TODO find a cleaner solution than putting this here -->
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
<script>
MathJax = {tex: {inlineMath: [['$', '$']]}};
</script>

Quicksort is a rather popular sorting algorithm implemented by most standard libraries.
Yet it has its shortcomings and special care needs to be taken to implement it properly.
It is not uncommon for standard libraries to use a mix of sorting algorithms for the best performance,
often using asymptotically suboptimal but simple algorithms like insertion sort once the problem has been sufficiently reduced
and falling back to asymptotically reliable algorithms like heapsort if quicksort doesn't progress fast enough
(see for example Go's [mix of sorting algorithms under the hood](https://cs.opensource.google/go/go/+/master:src/sort/zsortfunc.go)).

## Quicksort vs. other sorting algorithms

*Mergesort* achieves an upper bound of $O(n log(n))$,
which is as good as an optimal Quicksort;
it usually will require $O(n)$ auxiliary space however;
in-place implementations of Mergesort are nontrivial.

*Heapsort* achieves $O(n log(n))$ with a simple implementation
and with constant auxiliary space; it could thus be considered better.
However, in practice, Quicksort is usually faster.

## Implementation details to get wrong

### Pivot Picking

First of all, you have to intelligently pick a pivot.
Naively, you might pick the first, last or middle value of the slice.
For all slices, there will be cases where this results quadratic runtime
in the worst case - most notably sorted lists if picking the first or last value.

The optimal pivot is the median; any number that's guaranteed
to have a certain number of smaller or larger numbers suffices
for asymptotically logarithmic runtime however.
Using Median-of-Medians to find such an approximate median to use as
a pivot in linear time, Quicksort becomes asymptotically optimal,
achieving a hard upper bound of $O(n log(n))$.

The caveat is that the overhead is rather significant
and the implementation rather complex,
so this is not done in practice.

What's instead usually done is picking a *random* pivot.
It can be shown that the *expected* time complexity will
then be in $O(n log(n))$, no matter the given input array.

For numbers, there are heuristics considering e.g. the mean
or the mean/median of three randomly selected values.

### Partitioning Scheme

Another issue requiring extensive care is the *partitioning scheme*,
especially with regards to *duplicates*:
Consider an array consisting only of duplicates.
`Partition` has nothing to do; it should ideally return an index
close to the middle of the array to allow reducing the array to a constant fraction of its size.
Even better, `Partition` should return the range of values equal to the pivot,
such that the subsequent recursive calls to Quicksort don't have to consider them.
The same applies for Quickselect.

A naive *Lomuto partitioning* will usually move the pivot index all the way through
or not at all in such a case, resulting in catastrophic $O(n^2)$ runtime.

A good partitioning scheme is *three-way partitioning* ("dutch flag sorting"),
which will partition the array into values smaller than, equal to, and greater than the pivot;
you will only have to call recursively quicksort on the smaller and greater partitions.
For the extreme case of an array consisting only of duplicates, this will run in linear time;
in general, the more duplicates there are, the more the three-way partitioning pays off.

### Space Usage

Finally, assuming you are using a suboptimal partitioning scheme or pivot selection strategy,
you have to carefully arrange your recursive calls to keep the auxiliary
memory usage logarithmic in the size of the array:
First sort the smaller half, then sort the larger half *using a tail call*
(alternatively, you may iteratively push sorting "jobs" on a stack *in this order*).
Otherwise, you may get worst case linear memory usage.

---

Here is my [Lua implementation of Quicksort](https://github.com/TheAlgorithms/Lua/blob/849c46a4ba93a6d41ec77e117c2449a1afd4a46f/src/sorting/quicksort.lua) which strives to get the implementation details right; there also is an implementation of [Quickselect](https://github.com/TheAlgorithms/Lua/blob/849c46a4ba93a6d41ec77e117c2449a1afd4a46f/src/sorting/quickselect.lua) and [Quickselect using the "median of medians" pivot selection strategy](https://github.com/TheAlgorithms/Lua/blob/main/src/sorting/quickselect_median_of_medians.lua).

---
date: 2026-10-24
title: "On shuffling"
---

One of the most fundamental operations of randomized algorithms is *shuffling*:
Given an array of \(n\) items, randomly permute their order, such that every possible order (every permutation) is equally likely.
The classic algorithm for this is due to [Fisher and Yates](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle).
This post is about shuffling and a neat related puzzle which occurred to me about a year ago.

## Fisher-Yates shuffling

Suppose you had a "repository" of remaining items to be shuffled.
In each step, you pick one of these items uniformly at random, removing it from the repository,
and adding it to a list of shuffled items.

The natural data structure for such a repository is an array (or more precisely, a "slice"):
By choosing an index, you can efficiently choose an item uniformly at random.

Because we do not care about the order of items in the repository,
we can efficiently remove an item from the repository by swapping it with the last item,
and then "removing" that item by restricting the slice to the remaining range of slots.

```lua
function shuffled(items)
	local n = #items
	local repo = {}
	for i = 1, n do
		repo[i] = items[i] -- copy the list
	end
	local shuf = {}
	for i = 1, n do
		local j = math.random(1, #repo)
		shuf[i] = repo[j]
		repo[j] = repo[#repo]
		repo[#repo] = nil -- shrink repo
	end
	return shuf
end
```

(The examples here use Lua, but translate straightforwardly to other programming languages.)

This is however a bit more complicated than necessary. It is also wasteful, constructing two additional tables.
More efficiently, we can keep the shuffled result and the repository of remaining items *in the same array* as we go:
After the i-th loop iteration, the slice `items[1]`, ..., `items[i]` is a random, shuffled subset of the items;
the slice `items[i+1]`, ..., `items[n]` is the repository of remaining items.
We maintain this invariant by picking a random item from the repository to swap to the i-th place in the i-th iteration:

```lua
function shuffle(items)
	local n = #items
	for i = 1, n - 1 do -- last iteration changes nothing, can be skipped
		local j = math.random(i, n) -- choose from repository
		-- Swap chosen item from repository to shuffled slice.
		items[i], items[j] = items[j], items[i]
	end
end
```

This is the classic in-place Fisher-Yates shuffle.
(To get a shuffled copy instead, you would simply copy the list of items first.)

Fun fact: It seems to happen [not too rarely](https://blog.codinghorror.com/the-danger-of-naivete/)
that programmers implement "shuffling by random swaps",
but don't restrict the swaps properly (e.g., they write a loop similar to the above, but pick `j = math.random(1, n)`),
which leads to a very skewed distribution where some permutations are much more likely than others
(you can either try this out by writing [some tests](https://github.com/TheAlgorithms/Lua/blob/main/.spec/random/shuffle_spec.lua),
or by computing the probabilities for some small examples by hand).

<details>
<summary>Bonus: Functional shuffling</summary>

Shuffling is a stress test for functional programming languages:
In addition to needing a way to represent a "random source", an efficient implementation relies fundamentally on random access.

In languages based on constant-size immutable data structures, e.g. `cons` pairs in many Lisps, random access takes at least logarithmic time.
Thus the above shuffle implementations would all take at least \(\Omega(n \log n)\) time.
If you allow immutable arrays that can be constructed from index-value pairs in linear time (as Haskell does), there are tricks that let you sort in linear time.

If you are fine with \(O(n \log n)\) time, there is a simple way to implement shuffling in a purely functional programming language (glossing over the random source),
using only immutable singly linked lists:

If you have a list of a single element, or an empty list, it is already shuffled.

If you have a list with more than one element, you flip a coin for each element to determine whether it should go right or left.
This gives you two sublists which you can then shuffle recursively;
you then return the concatenation of the two shuffled lists.

(Equivalently, you can assign a large enough random "weight" to each item for collisions to be improbable
and then sort by these weights using any purely functional sorting algorithm.
The above algorithm is essentially what quicksort on random weights boils down to.)
</details>

Clearly, at least linearly many swaps are necessary for every permutation to be possible at all.
And because at least half of all permutations require at least linearly many swaps,
linearly many swaps are even necessary in expectation.
In this sense, the linear runtime of Fisher-Yates is optimal;
we can not hope to find an algorithm with sublinear expected runtime here.

However, we often only wish to sample a subset of a list, the size of which may not be known in advance.
Can this be done more efficiently than shuffling the entire list?
The answer is yes.

## The ballot box

Say you want to draw *some* distinct items from a set, with each subset (and each order) being equally likely --
but there are potentially many candidates and you do not know how many items you will want to draw ahead of time,
so just shuffling all items might end up being rather wasteful if you only end up drawing a few.

A "ballot box" only supports a single operation: extracting an item chosen uniformly at random ("drawing" an item).
We can treat a ballot box as a function (a closure) which upon each call returns the drawn item:
`draw = ballot_box(n)` gives you a ballot box containing the indices \(1, 2, ..., n\),
which upon each call `index = draw()` returns and removes one of the remaining indices in the box chosen uniformly at random.

To shuffle a list, you could simply call `draw()` repeatedly, recording all the picks in an array:

```lua
function shuffled(list)
	local draw = ballot_box(#list)
	local shuf = {}
	for i = 1, #list do
		shuf[i] = list[draw()]
	end
	return shuf
end
```

We make a first attempt at implementing such a "ballot box":

### Rejection sampling

We can record previously chosen indices in a hash set; we can then repeatedly choose a random index from \(1\) to \(n\),
rejecting any indices that were recorded previously, until we get a fresh index:

```lua
function ballot_box(n)
	local chosen = {}
	return function()
		local i
		repeat
			i = math.random(1, n)
		until not chosen[i]
		chosen[i] = true
		return i
	end
end
```

Strictly speaking, this algorithm is not guaranteed to terminate at all once the first choice has been made:
We could get perpetually unlucky, such that the `repeat` loop would never be left.
But the chance for this is virtually zero as we are about to see.
Instead of a "true" worst-case runtime, we thus speak of an *expected* runtime,
treating the runtime as a random variable depending on the random numbers `math.random` gives us. [^randsrc]

[^randsrc]: We normally assume independent, uniformly distributed random number generation (RNG) here.
Practically, pseudo-random number generation (PRNG) is often used. While PRNGs do exhibit certain patterns,
these are usually extremely unlikely to coincide with the patterns in the input data in an unfortunate way.
Special care must sometimes be taken to make it harder for an intelligent adversary to predict such patterns,
but that is not very difficult either.
As a last result, you can ultimately always use a hardware RNG to reseed a PRNG,
thus limiting the amount of damage any kind of correlation, intentional or not, can do.

What is the expected asymptotic runtime if we call `draw` \(n\) times?
Perhaps surprisingly, the answer is \(O(n \log n)\).

To see why, we look at the expected runtime of a single draw.

We assume that each line in the pseudocode shown above runs in expected amortized constant time.
In particular, we assume that the hash map access is expected constant time.
This is reasonable as keys are drawn uniformly at random from the integers \(1, ..., n\).

It is also possible to draw uniformly distributed random integers in a certain range in expected constant time. [^rejection]

[^rejection]: This again uses rejection sampling! A PRNG or hardware RNG will usually give you too many bits, say, 16, 32 or 64 bits. You first throw away as many bits as possible so you get the smallest possible range that includes your range.
It is still possible to draw an out-of-range value. However, the probability of this is at most \(\frac{1}{2}\).
If you reject out-of-range values, you will in expectation need to draw two values until you obtain a value in the range,
again the expectation of a geometric random variable.

Thus it suffices to count the number of steps until we obtain a fresh index.
This is a random variable following a [geometric distribution](https://en.wikipedia.org/wiki/Geometric_distribution).

Let \(m\) be the number of indices that have already been drawn.
Let \(p = \frac{m}{n}\) be the probability that a random choice is a "miss",
meaning we draw an index we have already seen previously.
The probability that we need exactly \(k\) steps is \((1-p) p^k\),
decreasing exponentially in \(k\);
the expected number of steps is \(\frac{1}{1 - p} = \frac{n}{n - m}\).

<!--
could do the simple derivation for the expectation of a geom RV here,
but that would derail too much...?
-->

This will be very efficient in the beginning, when few items have been chosen;
it will get less and less efficient the more often `draw()` is called,
the more likely it becomes that you have to reject an index.
This gets worst for the last draw, where you only have a \(\frac{1}{n}\) chance of hitting the single remaining index,
and thus can expect linearly many tries.

However, for a sequence of \(k\) draws starting from a fresh ballot box,
we have \(k \cdot \sum_{m=0}^{k-1} \frac{1}{n - m} = k \cdot \sum_{i=1}^k \frac{1}{i} = k \cdot H_k = O(k \log k)\) steps in expectation, where we use the fact that
\(H_k = O(\log k)\) [^hn]. If we blame each of the \(j\) operations for an equal part of this runtime,
we see that this cute approach is not too bad: in expectation,
each `draw()` operation has amortized logarithmic runtime.

[^hn]: The reason that \(H_n = O(\log n)\) is that \(\frac{1}{x}\) is the derivative of the logarithm.
Therefore summing over \(\frac{1}{i}\) is effectively a crude approximation of that integral.
This is also why the analysis is tight.

This kind of problem is also known as the [Coupon collector's problem](https://en.wikipedia.org/wiki/Coupon_collector%27s_problem):
There are \(n\) different coupons. You want to collect them all.
Every time you go buy something, you get a coupon, chosen uniformly at random.
How long does it take until you have all \(n\) different coupons?
By an analogeous argument, the answer is \(O(n \log n)\).

But we want to do better. We want asymptotically optimal amortized, expected *constant time* complexity per `draw()`.
*How can you achieve this?*

You might want to think about it yourself for a bit.

<details>
<summary>Hint</summary>
Combine an asymptotically optimal approach for choosing less than \(n / 2\) elements
with an asymptotically optimal approach for choosing more than \(n / 2\) elements.
</details>

<details>
<summary>Straightforward solution</summary>

The idea is to use rejection sampling for choosing (up to) the first \(n/2\) items,
and then shuffling the remaining \(n/2\) items in linear time,
which the first \(n/2\) `draw()` operations have paid for.

We keep counters around for how many items there were initially (\(n\)), and how many items are left in the set.

The idea is to first remember items that we already chose in a hash set.
We pick a random integer from \(1\) to \(n\); if it was chosen earlier, we pick again, until we have a new choice.
Assuming at least half of the items have not been chosen yet,
this runs in expected constant time (the expected number of iterations is a geometric series).
This is standard rejection sampling.

Once at least half of the items have been chosen, we construct an array of the remaining items in \(O(n)\) time
and shuffle it (e.g. with Fisher-Yates, still in \(O(n)\) time; the shuffling can also be done lazily).
We add the items to a stack, which now holds remaining items in random order.
Each operation simply pops from this stack in \(O(1)\) time from this point onwards.

```lua
function ballot_box(n)
	local n_chosen = 0
	local chosen = {}
	local stack

	return function()
		-- Second phase: Take from stack of shuffled items.
		if stack then
			return table.remove(stack)
		end

		-- First phase: Use rejection sampling.
		-- This is expected constant time because there's a probability of at least 1/2
		-- of finding an index that has not been chosen before.
		local i
		repeat
			i = math.random(1, n)
		until not chosen[i]
		chosen[i] = true

		-- Transition to second phase once some fixed fraction of items have been chosen.
		-- The linearly many choices we have done so far pay for this.
		n_chosen = n_chosen + 1
		if n_chosen > n/2 then
			stack = {}
			for i = 1, n do
				if not chosen[i] then
					table.insert(stack, i)
				end
			end
			shuffle(stack)
		end

		return i
	end
end
```

This is a good example of the generally useful "combine the best of both worlds" trick:
The runtime of rejection sampling suffers the fewer items are left,
constructing an array is too expensive if too few choices are drawn from the set.

We can tune the proportion of indices where we switch from a hash set of chosen indices to a list of remaining items
to make an optimal tradeoff between the constant factors of the hash set and the stack of remaining items.
</details>


<details>
<summary>Cute solution</summary>
Using a hash map, we can get away with just a "lazy" Fisher-Yates:

We maintain a permutation \(f\) as a hash map, where no entry for \(i\) existing means \(i\) maps to itself.
We also remember the number of remaining items, \(r\), in the set.

We draw an integer \(k\) from \(1\) to \(r\).
We remember \(i = f(r)\) as the item to return.
We set \(f(k) \larr f(r)\). (In Fisher-Yates, we would swap \(f(r)\) and \(f(k)\).
But because we immediately extract \(i = f(k)\), there is no point in this.)
We then return \(i\). (We may also delete \(f(r)\) if we like, but this is not necessary).

The runtime of a draw is thus just the runtime expected, amortized \(O(1)\) of hash map access.

```lua
function ballot_box(n)
	local perm = {}
	local i = n
	return function()
		local j = math.random(i)
		local index = perm[j] or j
		perm[j] = perm[i] or i
		perm[i] = nil
		i = i - 1
		return index
	end
end
```

<!--
there's a very subtle thing here that perm[i]=nil does not actually free up space, though it does free up a slot.
the end result is probably fine. either way it's trivial to address.
-->

</details>

<details>
<summary>Yet another solution</summary>

If you examine the first solution closely, you might observe that we do not need Fisher-Yates at all!
Rejection sampling alone suffices.
The trick is to reduce the chances of a miss after half of the remaining indices have been chosen.
As before, we remember which indices have been chosen previously.
We can afford to go through all indices, keeping only the ones that have not been chosen yet, stuffing them in an array
of remaining choices.
Then, we draw from this array, marking drawn items as chosen.
Once at least half the items in the array have been marked as chosen, we remove all the chosen ones,
downsizing to an array half the size.

We need to be a bit careful with initialization: Setting up an array of choices `1, 2, ..., n` would already be too expensive.
Thus we employ the same trick as in the elegant solution, where we take an absent entry in `choices[i]` to mean `i`.
(Note that after the first shrinking, no absent entries exist anymore.)

```lua
function ballot_box(n)
	local n_chosen = 0
	local n_choices = n
	local choices = {}

	local function shrink()
		local new_choices = {}
		for i = 1, n_choices do
			if choices[i] ~= false then
				table.insert(new_choices, choices[i] or i)
			end
		end
		n_choices = n_choices - n_chosen
		n_chosen = 0
		choices = new_choices
	end
	
	return function()
		local i
		repeat
			i = math.random(1, n_choices)
		until choices[i] ~= false
		local choice = choices[i] or i
		choices[i] = false
		n_chosen = n_chosen + 1
		if n_chosen >= n_choices/2 then
			shrink()
		end
		return choice
	end
end
```

So really it turns out that rejection sampling, or Fisher-Yates, or a combination of both all lead to a solution if applied properly!
</details>

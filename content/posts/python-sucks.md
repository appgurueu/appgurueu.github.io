# Why Python sucks

## Coercion

Not too dissimilar from JS, Python has gross type coercion rules,
but somehow we all let it get away with it.

The truthiness is not very useful, because too many things are falsey,
among them empty strings or the number 0.

We need to write explicit "is not none" checks.

You can just add `0 + True` and it's `1`.

But guess where Python *did* decide to have explicit conversions?
When concatenating things with strings.
Because nobody wants to write the `"x = " + x`, everyone wants to write `"x = " + str(x)`.
This is so that there can be another syntax for you to remember, the "f-string":
The pythonic way is `f"x = {x}"`.

## Nonsensical "natural" language constructs

Programming languages have to decide on some conventions:
Do they place function names before or after the arguments?

This influences reading direction. In Python, it's done badly and inconsistently.

### Ternaries

The prime example of this is the awful ternary syntax: `a if cond else b`.

Any sane language has something like `(if cond a b)`, or `if cond then a else b`, or `if cond { a } else { b }`, or `cond ? a : b`,
or `cond and a or b` [^lua], or dammit, something like `a b cond ?` in a stacklang.

[^lua]: Not exactly a ternary, but for most purposes equivalent

You know, something that remotely makes sense and fits in with the imperative `if`-`else`:
First you evaluate the condition, then you evaluate one of the branches depending on that.
This is readable. This makes sense. But not in Python, no!
You start parsing an expression, only to find out midway through that this was the left arm of a ternary all along.

The way to read this code is to start at the *middle*, the condition, which is evaluated first,
and to then look at the two branches left and right of it. Yay.

### List comprehensions

List comprehensions are also a hot mess, and inconsistent with how the imperative generators would be written and read.

To map over a collection `xs`, you'd do something like this:

```python
ys = []
for x in xs:
	ys.append(x ** 2)
```

Now with a lovely list comprehension, you need to reverse the order - the things come first.
This also fucks with any sensible notion of lexical scoping. Variables are used before they are defined:

```python
ys = [x ** 2 for x in xs]
```

#### Nested list comprehensions

To iterate all pairs of elements from two iterators, I can do the following:

```python
pairs = []
for i in range(3):
	for j in range(2):
		pairs.append((i, j))
```

where `b` is the inner loop.

Pop quiz! In a list comprehension, would that be `for i in range(3) for j in range(2)`, or `for j in range(2) for i in range(3)` -- if we want the same order of iteration?

Who knows! It would make sense for it to be the former, right, if it tries to be consistent with how it would work if written in imperative style.

But it would also make sense for it to be the latter if we think about it as two nested list comprehensions to be flattened.

If you guessed the former, you'd be right. Python chooses to invert the order of things, but not for the purposes of nesting!

#### Filtering

Similarly, say you add a condition:

```python
# stupid example, i know, who cares. real code is often stupid too.
s = 0
for i in range(10):
	if i != 0:
		s += 1 / i
```

That would be written `sum(1 / i for i in range(10) if i != 0)`.

How damn awful is that! Only by the time I've read the entire thing do I see that the `1 / i` is actually guarded by an `i != 0` check.

So what will any experienced Python programmer do?
Their eyes will have to jump around to compensate.
Spotting a list comprehension? Jump straight to what's being iterated (the `for`), read that (including the `if` guard),
then go back to reading the expression. Only then will the expression make sense.

And now imagine how utterly comical this could get if you start nesting these things.

Now that is what we call pythonic.

(In all fairness, I believe there is a sort of fundamental problem in our prefix convention for function application.
This effectively requires you to read back-to-front to trace the control flow.
This makes it somewhat natural to want `map(xs, f)` to be `map(f, xs)`, and then if list comprehensions are consistent with that you get this mess,
so you get chains that need to be read back-to-front in an otherwise front-to-back programming language.)

# Why Python sucks for teaching

## Overly complex data model

There are simply too damn many data types.

## Overly complex number model


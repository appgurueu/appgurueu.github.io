---
date: 2026-02-08
title: "Python's variable scoping sucks"
---

> "[...] could you explain Python's variable scoping rules?"
>
> "Well, it's very easy. They are local to arbitrarily defined block[s] unless they are not.
> In such case they are available only in otherwise arbitrarily defined blocks."  
> -- Aaa, on the Lua discord

How precisely it sucks is best demonstrated by example.
I hope to convince you that maybe there are some aspects to Python
that don't exactly make it the friendliest language for a beginner to get into.
A seasoned programmer will be able to cope with this, but might still be surprised about an edge case or two.

Let's get into it. Say we have a "global" variable `x` at the top-level scope.

```python
x = 1
print(f"x = {x}") # 1
x = 2 # change ("mutate") the variable
print(f"x = {x}") # 2, as expected
```

So far so good. We create a new variable `x`, assigning `1` to it.
We can read the value, and we can change it, and read the updated value.

We can also move to mutation to its own block, like this:

```python
x = 1
print(f"x = {x}") # 1
if x == 1:
    x = 2
print(f"x = {x}") # 2, as expected
```

Now say we introduce a function. We can read `x` from a function just fine.

```python
x = 1

def read_x():
    print(f"x = {x}. works!")

read_x() # 1
x = 2
read_x() # 2, no surprise.
```

Generally, variables are read-write in Python.
So if we can read, surely we can also change it from a function?
Let's make a first attempt:

```python
x = 1

def write_x():
    x = 2 # this should work, right?
    print(f"x inside write_x: {x}") # 2, as expected

write_x()

print(f"x outside write_x: {x}. oops!") # still 1!?
```

This does not work. It looks reasonable: We just moved the assignment `x = 2` into the function, didn't we?
But as it turns out, assignment is not just assignment. It is assignment and variable *declaration* in one.
So there is an ambiguity as to what `x = 2` should do:
Should it change the existing variable, or should it declare a new one?
We just saw that inside a "normal" block, like an `if` statement, it reuses the existing variable.
But inside a *function* body the rules are different: Here we get a new scope, so `x = 2` becomes a *declaration*.
Oops!

We learned that we can read the variable, but when we try to change it,
we somehow introduce a new variable that "hides" ("shadows") the real one.
Still, if code is just executed top to bottom, "line by line", the following should work:

```python
x = 1

def rw_x():
    # first read the outer variable. should be 1.
    print(f"x outside rw_x: {x}")
    # declare new x, shadowing the outer x, and set to 2
    x = 2
    # now read the inner variable. should be 2.
    print(f"x inside rw_x: {x}")

try:
    rw_x()
except UnboundLocalError as e:
    print(f"Error running rw_x: {e}")
```

Wrong again! What we really have to keep in mind is some ridiculous tomfoolery:
When you set `x = 2`, you're not actually just declaring and assigning to a variable once that statement runs.
Oh no, that would be too simple. You're first declaring a local variable *at the top of the function*.
But this variable only becomes usable *once you assign to it*;
trying to use it earlier is an error, which the first statement in `rw_x()` runs into. [^hoisting]

[^hoisting]:
    Moving declarations to the top of a scope is referred to as *hoisting*.
    It has the benefit of making functions that refer to each other "just work".
    Languages with proper lexical scoping, like Lua, instead require you to explicitly declare variables
    your functions depend on, which you may change later.
    Alternatively, you can topologically sort your functions,
    or explicitly use a table that is in scope for all functions as a layer of indirection.

So yeah, no strictly lexical scoping: Instead, funny hoisting rules!
This also facilitates the problem of functions that depend on each other being in no reasonable order,
because any function defined at the top level is visible to any other function.
There need not be any order. When reading the code, you may need to jump around wildly in the file.

Speaking of lexical scoping: List comprehensions, anyone?

```python
print([x ** 2 for x in range(5)]) # [0, 1, 4, 9, 16]
```

List comprehensions let you declare a variable well after it is used. Isn't that lovely?
So intuitive. We should call this genius invention "anti-lexical scoping".
Programming learners must love this.
Imagine how *readable* and *pythonic* this can get if we nest a few list comprehensions!
(I hope you don't mind if I spare you the sight.)

Okay, okay, let's get back on track. How do we actually change `x` from inside this function now?
How do I tell Python that I want to change the variable `x` that's right there
instead of declaring a new variable shadowing it?

You probably look this up sooner or later (probably after getting the above error message)
and find out you're supposed to use the `global` keyword
(and that you should try to minimize your use of global variables).

The `global` keyword is not really a "declaration", where we "create" a variable
and make it available from some point onwards;
it is sort of the opposite: We must declare that from some point onwards,
an *existing* variable is to be brought into scope.

```python
x = 1

def set_x_real():
    global x
    x = 2 # now we are changing the "global" variable x!

set_x_real()
print(f"x after global: {x}") # 2
```

Great. Phew. We finally figured out variables in Python, huh?
Or did we?

Consider the following:

```python
if True:
    y = 1
# is y a thing here? it depends.
# this time the condition was true,
# so the variable happily leaks into the parent scope.
print(f"y = {y}") # 1
```

But not if the path was not taken:

```python
if False:
    z = 1

try:
    print(f"z = {z}")
except NameError as e:
    print(f"Error trying to read z: {e}") # name 'z' is not defined
```

So yes, variables just happily leak into their parent scopes,
but only if respective paths that populate them are taken.

Another example. Surely we're done with a loop variable after, uh, the loop is done iterating?
Otherwise, what should the value even be? The value in the last iteration?
The value upon termination (so the value one after the last, which a traditional "while" loop would give you)?

```python
for i in range(42):
    pass

print(f"i = {i}") # 41
```

Now of course these can nest. Variables living happily ever after.
The following example searches for small Pythagorean triplets, leaking 3 variables in the process.

```python
print("Pythagorean triplets:")
for a in range(1, 10):
    for b in range(a, 10):
        c_sq = a**2 + b**2
        if round(c_sq**0.5)**2 == c_sq:
            print(f"{a}^2 + {b}^2 = {c_sq}")

print(f"a = {a}, b = {b}, c_sq = {c_sq}")
```

Talk about variable pollution and risk of mistakes creeping in.
Anti-lexical scoping strikes again.

Okay, okay, you get it. variables are not constrained by any block.
They only care about *function bodies*, because uh, yes.

But that's it? Right? No more funny business?
I'm afraid I have bad news for you.
Enter closures. Here's how we would like to be able to write Graham's litmus test in Python:

```python
def accumulator():
    state = 0
    def add(increment):
        state += increment
        return state
    return add

acc = accumulator()
try:
    print(acc(42))
except UnboundLocalError as e:
    print(f"Error trying to increment accumulator: {e}")
    # error!
    # cannot access local variable 'state'
    # where it is not associated with a value
```

Similar to global variables, it's the same nonsense all over again. Just reading is fine:

```python
def outer():
    v = 1
    def inner():
        print(f"v = {v}") # 1
    inner()
outer()
```

... but updating (assignment) implies a declaration and thus is not fine.
Again we need to do something funny, but this time it can't be called "global"
because the variable we want to tell Python to reuse is not global.

```python
def accumulator2():
    state = 0
    def add(increment):
        # reuse the variable `state` from the parent scope,
        # do not make introduce a new local here
        nonlocal state
        state += increment
        return state
    return add

acc2 = accumulator2()
print(acc2(42)) # 42, finally
```

This also gives the funny option of skipping a scope:

```python
g = 1
def foo():
    g = 2
    def bar():
        global g # straight to global scope
        print(f"g = {g}") # 1
    bar()
foo()
```

Oh by the way: Unlike variables declared in `if` blocks, loops, etc.,
errors captured by `except` do not leak!
Generally variables introduced using `as` do not leak from their blocks;
the same goes for `with`.
Naturally. Yay, more rules to remember!

```python
try:
    try:
        raise Exception("oops")
    except Exception as e:
        pass
    print(f"Can we get e again? e = {e}")
except NameError as e:
    print(f"Error trying to get e: {e}") # name 'e' is not defined
```

And then there's also the walrus operator, and it doesn't work like `as`,
even though that would make a lot of sense.
But at least you can sort of think of it like a normal assignment, just that it is an expression.

```python
if w := 1:
    pass
print(f"w = {w}") # 1
```

Finally, a classic which even languages that get scoping right tend to get wrong.
Of course, Python also gets it wrong. How could it not?

```python
fs = []
for i in range(3):
    fs.append(lambda: i)

print(f"{fs[0]()} {fs[1]()} {fs[2]()}") # 2 2 2
```

What's going on here? The three lambdas in the list `fs`
all share the same variable `i` that was mutated,
because that variable was hoisted from the loop to the parent scope.
After the loop has ended, we have `i = 2`.

How would we work around this? The pythonic workaround is to introduce a default parameter that is stored with the lambda:
`lambda: i` is replaced with `lambda i=i: i`. Ugh. [^iife]

[^iife]: Alternatively and more verbosely, we could use an IIFE to get a new scope with an `i` variable in every iteration.

Now where do all these "globals" go? It turns out they land in a so called "namespace" for a "module" (file).
You can import that module, and then you can access these variables as `module.var`.
How do you hide a variable? You don't. You just prefix it with an underscore (e.g. `_pls_dont_touch = 1`)
and by convention, hope that nobody touches it from outside the module.

---

A general theme is that Python suffers from lack of explicit declaration of local variables;
declaration and assignment are conflated.
It then introduces, instead of "normal" variable declarations, the "using" declarations
`global` and `nonlocal` to bring variables into scope.
There are a bunch of different syntactic constructs that make variables available,
all with slightly different rules.
Variable scopes are somewhat arbitrarily centered around functions.
Hoisting makes variables leak everywhere.

It doesn't have to be such a pythonic mess.
Languages other than Python have long implemented proper *lexical scoping*
(e.g. `local` variables in Lua).

There are still quirks, but much fewer.

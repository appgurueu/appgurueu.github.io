---
date: 2026-02-08
title: "Python's variable scoping sucks"
---

> [...] could you explain Python's variable scoping rules?
> "Well, it's very easy. They are local to arbitrarily defined block[s] unless they are not.
> In such case they are available only in otherwise arbitrarily defined blocks."
> \- Aaa, on the Lua discord

How precisely it sucks is best demonstrated by one lengthy example,
by the end of which I hope to have convinced you that maybe there are some aspects to Python
that don't exactly make it the friendliest language for a beginner to get into.

```python
x = 1
print(f"x = {x}") # 1. so far so good.

# we can read x from a function just fine.
def read_x():
    print(f"x = {x}. works!")

print("reading x from a function:")
x = 2
read_x() # 2, no surprise.

# generally, variables are read-write in python.
# so if we can read, surely we can also change it from a function!
print("attempt #1 at mutating x from a function:")
def write_x():
    x = 3 # what does this do?
    print(f"x inside write_x: {x}")

write_x()
print(f"x outside write_x: {x}. oops!") # still 2. guess we somehow declared a new variable above. oops!

# now okay. we learned that we can get the variable, but when we try to assign to it,
# we somehow introduce a new variable "shadowing" the real one.
# still, if code is just executed "line by line", intuitively speaking, the following should work, no?
def rw_x():
    print(f"x outside rw_x: {x}")
    x = 3 # "declare new x, shadowing the outer x, and set it to 3"?
    print(f"x inside rw_x: {x}")

try:
    rw_x()
except UnboundLocalError as e:
    print(f"Error running rw_x: {e}")

# wrong again! what we really have to keep in mind is some tomfoolery:
# when you set x = 3, you're not actually just declaring and assigning to a variable once that statement runs.
# oh no. you're first declaring a local variable *at the top of the function*.
# but it only becomes usable *once you assign to it*;
# trying to use it earlier is an error, which the first statement in rw_x() runs into.

# so yeah, no strictly lexical scoping: instead, funny hoisting rules!
# speaking of lexical scoping: list comprehensions, anyone?
print([x ** 2 for x in range(5)]) # [0, 1, 4, 9, 16]
# ^ a variable can, syntactically, be declared well after it is used. isn't that lovely?
# so intuitive. we should call this genius invention "anti-lexical scoping".

# okay, but how do we actually change x from inside this function now?
# how do i tell python that i want to change the variable x that's actually already there
# instead of declaring a new variable shadowing it?
# you probably look this up sooner or later and find out you're supposed to use the "global" keyword
# (and to try to minimize your usage of global variables).
# this is not really a "declaration", where we declare that from some point onwards, a variable is available;
# it is sort of the opposite: we must declare that from some point onwards,
# an existing variable is to be brought into scope.

def set_x_real():
    global x
    x = 4 # now we are changing the "global" variable x!

set_x_real()
print(f"x after global: {x}") # 4

# great. phew. we finally figured out variables in python, huh?
# or did we?

# consider the following:
if True:
    y = 1
# is y a thing here? it depends. this time the condition was true.
# so the variable happily leaks into the parent scope.
print(f"y = {y}") # 1

# but not if the path was not taken:

if False:
    z = 1

try:
    print(f"z = {z}")
except NameError as e:
    print(f"Error trying to read z: {e}") # name 'z' is not defined

# so yes, variables just happily leak into their parent scopes,
# but only if respective paths that populate them are taken.

# another example. surely we're done with a loop variable after, uh, the loop is done iterating?
# otherwise, what should the value even be? the value in the last iteration?
# the value upon termination (so the value one after the last, which a traditional "while" loop would give you)?

for i in range(42):
    pass

print(f"i = {i}") # 41

# now of course these can nest. variables happily living ever after.
# this example searches for small pythagorean triplets, leaking 3 variables in the process.
print("Pythagorean triplets:")
for a in range(1, 10):
    for b in range(a, 10):
        c_sq = a**2 + b**2
        if round(c_sq**0.5)**2 == c_sq:
            print(f"{a}^2 + {b}^2 = {c_sq}")

print(f"a = {a}, b = {b}, c_sq = {c_sq}")

# talk about variable pollution and risk of mistakes creeping in.
# anti-lexical scoping strikes again.

# okay, okay, you get it. variables are not constrained by any block.
# they only care about *function bodies*.

# but that's it? right? no more funny business?
# i'm afraid i have bad news for you.
# enter closures. here's how we would like to be able to write Graham's litmus test in python:

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
    print(f"Error trying to increment accumulator: {e}") # error:
    # cannot access local variable 'state' where it is not associated with a value

# similar to global variables, it's the same nonsense all over again. just reading is fine: 
def outer():
    v = 1
    def inner():
        print(f"v = {v}") # 1
    inner()
outer()

# ... but updating (assignment) implies a declaration and thus is not fine.
# again we need to do something funny, but this time it can't be global
# because the variable we want to tell python to reuse is not global.

def accumulator2():
    state = 0 # this could also have been a parameter
    def add(increment):
        nonlocal state # take state from the parent scope, do not make it a local here
        state += increment
        return state
    return add

acc2 = accumulator2()
print(acc2(42)) # 42, finally

# this also gives the funny option of skipping a scope:
g = 1
def foo():
    g = 2
    def bar():
        global g # straight to global scope
        print(f"g = {g}") # 1
    bar()
foo()

# oh by the way: errors captured by "except" do not leak!
# (generally variables introduced using "as" do not leak from their blocks; also goes for "with").
# naturally. yay, more rules to remember!
try:
    try:
        raise Exception("oops")
    except Exception as e:
        pass
    print(f"Can we get e again? e = {e}")
except NameError as e:
    print(f"Error trying to get e: {e}") # name 'e' is not defined

# and then there's also the walrus operator, and it doesn't work like "as",
# even though that would make a lot of sense.
# but at least i guess you can sort of think of it like a normal assignment, just that it is an expression.
if w := 1:
    pass
print(f"w = {w}") # 1

# finally, a classic which even languages that get scoping right tend to get wrong.
# of course, python also gets it wrong. how could it not?

fs = []
for i in range(3):
    fs.append(lambda: i)

print(f"{fs[0]()} {fs[1]()} {fs[2]()}") # 2 2 2
# why? because they all share the same variable i that was mutated,
# because that variable was hoisted from the loop to the parent scope.

# now where do all these "globals" go? it turns out they land in a so called "namespace" for a "module" (this file).
# you can import that module, and then you can access these variables as `module.var`.
# how do you hide a variable? you don't. you just prefix it with an underscore and by convention,
# hope that nobody touches it outside this module.
_pls_dont_touch = 1
```

A general theme is that Python suffers from lack of explicit declaration of local variables;
declaration and assignment are conflated.
It then introduces, instead of "normal" variable declarations, the "using" declarations
`global` and `nonlocal` to bring variables into scope.
There are a bunch of different syntactic constructs that make variables available,
all with slightly different rules. Hoisting makes variables leak everywhere.

It doesn't have to be such a pythonic mess.
*Lexical scoping* has long come to the rescue.
Languages like Lua have proper `local` variables.

There are still quirks, but much fewer.

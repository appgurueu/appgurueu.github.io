---
layout: post
title: "The case for closures (in Lua)"
date: 2023-09-16
tags: lua, closures, functional programming, performance
---

I have recently been criticized for my use of "nested" functions in Lua;
the criticism revolved mostly about presumed performance problems (due to the cost of closure creation)
and the claim that the usage of nested functions was "unnecessary" from a code quality standpoint.
Here is my attempt at a thorough analysis, making a case for closures and more specifically the usage of "nested" functions in Lua.

## Functional Programming

(Lua) closures alone suffice as Lambda Calculus - that is, closures paired with function application are Turing-complete.
With higher-order functions, functional programming - fundamentally based on Lambda Calculus - is fully possible in Lua.
Even just some very basic higher-order functions like `map`, `filter` or `fold` can already be very useful abstractions for working with iterators -
most often implemented as closures, unless it is possible to efficiently wrap up their state in a single control variable + invariant state (as with `ipairs` and `pairs`) -
but of course aren't inexpensive in Lua;
in much less dynamic languages like Rust, closures can often even be used as zero-cost abstractions.

Still, performance is usually not a top concern - especially if Lua is used rather than one of the classic fast compiled languages -
and functional code can always be optimized to more imperative code if necessary.

## "Nested" Functions

Creating closures inside other functions, even if not strictly necessary, offers various advantages:

* Code quality:
	* Namespacing: Closures can be localized to the function, rather than the parent namespace of the function;
	  there is no need for an additional wrapping `do`-`end` block
	  (or even worse, naming conventions like prefixes or suffixes) to achieve the same effect.
	* Upvalues: This is the main reason for using them: Closures *close* over their lexical context;
	  all locals in the lexical scope of the closure can be used as upvalues within the closure.
	  This means you don't have to pass them as parameters, and if you want to mutate them,
	  you don't have to take the detour over using a reference type
	  (like a table - which doesn't exactly help performance) or return values.
* Performance:
	* Using upvalues can be marginally faster than passing all parameters on the stack,
	  esp. if there are multiple of them.
	* Using upvalues saves stack space: You don't redundantly store the same variables in the stack.
	* Closure creation is expensive since it involves memory allocations. However, this is often not a concern:
		* If the closure is called often, the cost amortizes away over the calls.
		* If the closure is called rarely, there may still be other bottlenecks, such as I/O, poor pattern matching, etc.
		  This can be expected for example in readers, like my [modlib's Blitz3D model reader](https://github.com/appgurueu/modlib/blob/5f0dea2780b88d44d85b9704e0e81348c459404d/b3d.lua#L19-L318),
		  which makes heavy use of closures. A few closure instantiations at "startup" are not an issue at all when expecting to parse a multiple kilobyte large model;
		  if I were to optimize this (for which I've had no need yet), I'd look at much more critical areas like e.g. the many heap allocations for single vertices
		  (which is rather memory- and time-inefficient, but much better for code quality; the alternative is to shove their values into large separate arrays).
		  Rarely will the creation of closures be the bottleneck unless done very excessively and on very hot paths.
		* Most often: The code isn't performance-critical anyways and doesn't need to be maximally efficient.
		  If it is optimized after having been identified as a hotspot in profiling,
		  extracting closures and converting them to local functions, passing the variables, can be tested.
		  As always, the rules of optimization apply: Code should not be optimized prematurely.
		  Before doing constant-factor micro-optimizations,
		  it's often more worthwhile to do algorithmic optimizations that improve the asymptotic time complexities.
	* Closures have been extensively studied e.g. by Lispers; they *do not need to be inefficient per se*.
	  Very often closures are used in a "macro"-like fashion:
	  A sufficiently intelligent compiler or preprocessor could trivially inline each call.
	  (Note however that current Lua interpreters / JIT compilers do not seem to do this.
	  I am considering improving upon this, but writing a preprocessor or own interpreter / compiler
	  is significant work, esp. as care needs to be taken to take some nuances into account.)
	  In many cases, the costly closure creation could be avoided,
	  since the "first-class property" of being able to pass around the closure like any other value is not used extensively;
	  very often closures are only bound and accessed through local variables.

However, these advantages also come at a cost:

* Code quality:
	* Using closures may leave you with deeply indented functions;
	  at some point flattening by moving out closures into local functions in parent scopes
	  and explicitly passing parameters / receiving return values should be considered.
	* Mutating closures may sometimes be less readable than explicitly returning changed values;
	  using them may lead to bugs (through unexpected side effects).
* Performance:
	* Closure creation is expensive; closures put strain on the GC.
	  The hotter the path, the smaller the function and the fewer the times it is called during its lifetime,
	  the stronger it should be considered to convert a closure into a normal function, passing parameters manually.
	* If you have many closures that *may* be needed, but few of which are actually needed most of the time,
	  creating all the closures upfront comes at a significant cost and may be very wasteful.
	  At this point the shared state should be refactored -
	  into a state table if there are multiple variables and/or mutation is involved -
	  and an OOP-like metatable-based approach is worth pursuing (which comes at its own costs however).
	  An example of this is [modlib's texture modifier reader](https://github.com/appgurueu/modlib/blob/5f0dea2780b88d44d85b9704e0e81348c459404d/minetest/texmod/read.lua):
	  The parser functions for individual texture modifiers all operate on a reader "object" (table with metatable) as shared state.
	  (They could be made "methods" of the object, but it is more useful to place them in lookup tables.)
	* Usage of closures may hamper further optimizations done by a compiler.
	* Closure calls *may* be more expensive than local function calls (direct vs. indirect jumps) given a compiler.
	* Upvalue access is slightly more expensive than local variable access (one more layer of indirection; but in turn the cost of parameter passing is saved).
	* Memory-wise, creating many closures which share the same set of upvalues duplicates the cost of the upvalue slots;
	  returning a table of closures is not advisable if (memory)-efficient OOP is desired:
	  It requires closing all "methods" for each instantiation,
	  and will incur memory costs for all methods and all fields for all methods that use them.

## Conclusion

As is usually the case in programming, 
tradeoffs with regards to both code quality and performance are involved
when it comes to the decision of whether to use closures.
Closures are essential to unlock the power of functional programming.
Code quality trumps until optimization is done on the basis of profiling;
closures - even if "nested functions" that could rather easily be refucktored away -
should not be dismissed as "bad practice" on the basis of premature optimization
(but of course - for the sake of completeness - neither are closures universally "good practice" for code quality).

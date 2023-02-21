---
layout: post
title:  "How RegEx implementations suck"
date:   2023-02-21 14:49:42 +0100
tags:
  - regular languages
  - regex
  - parsing
  - lua
---

Regular languages are a powerful and beautiful formal tool to validate or parse strings.

Yet "regular" expression ("RegEx[p]") implementations are often despised by programmers for a couple (good) reasons:

## More than just RegEx

Part of what keeps regular languages and the theory surrounding them simple
is their limitations: Regular languages can't count infinitely;
they can't match brackets or keep track of indentation.

This led to hacky extensions to RegEx to add such "features" like backreferences
(which is even more powerful than counting) or bracket matching, extending the expressiveness of RegEx
beyond that of regular grammars, making regular expressions nonregular and often matching context-free grammars (CFGs) or - even worse - context-sensitive grammars,
which in turn 99% of the time is backed by a horribly inefficient implementation.

Use a proper CFG to do such heavy lifting. Possibly implement it using a recursive descent parser.

## Lookaround & the evil of captures

It can be shown that (implicit) concatenation (`ab`), union (`a|b`), the Kleene star (`a*`) and parenthesis / "grouping" (`(a)`) are sufficient
to give regular expressions the full expressive power of regular languages. Often `a+` is added as syntactical sugar for `aa*`, which is fine.

Now lookaround enters the scene, mainly due to captures: Perhaps you want your capture to be followed or preceded by something without capturing that.
Lookaround often adds unnecessary complexity to RegEx implementations.

It's a shame that the evil of captures has led to regular parenthesis being tainted; simple grouping requires *noncapturing* parenthesis: `(?:<expr>)`. These should have been the default.

## Substring matching

Regular languages are *the* tool for tokenization.
It is a shame any time a language comes around not allowing RegEx matching to be applied to a substring since this effectively makes RegEx useless for recursive descent parsing purposes
and also pretty useless for a reasonable tokenization loop. JavaScript is to be named as a bad example here: It provides powerful RegEx, yet doesn't allow applying them to substrings.

The ugly "substring, then RegEx" workaround is not a solution: In languages with immutable strings and only copy substrings (no slices) this ends up with quadratic performance, way below
the linear baseline of RegEx.

Props to Lua for `string.find`.

## Patterns: Botched RegEx

Patterns are a "linearized" form of RegEx which mostly gets rid of unions and grouping, allowing unions only at a character level through charsets
and disallowing grouping entirely - quantifiers like `*` can only be applied to "items" - charsets or other special items.

While this greatly simplifies the implementation, it also greatly restricts the expressive power.
This would be fine if it was left at this, since most of the time patterns suffice to get the job done (although they may invite ugly, overly permissive workarounds).

However patterns soon start adding other "useful" "more than just RegEx" features, running into the same issues. The odd thing 

## Poor syntax & Language support

RegExes are usually "just strings". Some languages at least add special RegEx (or raw string literals) syntax to not make a mess out of escaping RegEx.

Ideally RegExes should be part of the language grammar though, validated at compile-time if possible. Editors should then be able to provide syntax highlighting for operators etc.

[Comments and whitespace need to be possible](https://blog.codinghorror.com/regular-expressions-now-you-have-two-problems/) for complex RegEx without resorting to hacks like string concatenation to break up the RegEx.

The current syntax (`(?#comment)`) & flag to allow whitespace are rather messy & not supported everywhere.

Flags are a mess (especially ones like `/g` - this ought to be part of the func API). Also the entire line-orientation.

Building RegExes through string building is also messy. RegEx building - ideally through built-in language operators, as possible in Lua - should be implemented.

RegEx syntax should possibly match that of EBNF (even though RegExes are usually more character oriented): `[x]` instead of `x?`; `{x}` instead of `x*`.

At some point people might then stop using webapps to highlight & explain their RegExes to them as their editors are capable of the same, and in a distant future people might even understand the RegExes they are writing.

### Conciseness

RegExes ought to be fused with regular grammars as an extension (like EBNF to BNF) to allow formulating certain regular grammars more concise than is possible using RegEx;
currently RegEx is horribly inefficient to write in some cases.

In particular this would allow naming charsets
(rather than having to memorize a bunch of predefined charsets or repeating charsets, or even worse using string building), f.E. `alpha = A-Z | a-z | _`.

## Horrible Performance

RegEx matching can be performed in linear time, reading each character exactly once.
Unfortunately this is not what the majority of engines does since it requires some work
(first parse the RegEx, then build an NFA from the AST (Thompson's algorithm),
then turn that into a DFA using a powerset construction, finally possibly minify the DFA;
also annotate everything with capture begin & start markers etc.)

Instead, they go the simpler route of a shotgun recursive descent parser. This is particularly popular when using simple patterns.
The implementation is straightforward: The RegEx represents a NFA; the state is stored as the current index where matching is tried
(this is how programmers like to think about RegEx and thus intuitively implemented). If the match succeeded, the state is advanced
and continuing matching at the next item is attempted. This quickly blows up if something can match the empty string, such as `a*`:
The pattern matching engine now has to try both paths in a DFS-recursive-manner: Both matching an `a` or matching the empty string,
since it doesn't take into account what comes after the `a*` as a powerset construction would.

### (Seemingly) Linear Time

Despite the poor implementation, such pattern matching often runs in seemingly linear time in the average (!) case, introducing subtle DoS vulns where a simple pattern matching
operation - as is often used for input validation - takes ages to complete. Programmers using patterns usually don't take the time to think through whether a polynomial or exponential blowup
is a possibility; in fact I would assume that quadratic or sometimes even cubic blowups might go unnoticed in practice as long as the input strings are kept short, hideously slowing down the program.

### Quadratic Time

Consider the following pattern matching in Lua:

```lua
assert(not ("a"):rep(1e6):match(("a"):rep(1e6) .. "b"))
```

if Lua implemented RegEx the right way - building a DFA - it could parse the string of length `1e6` in linear time; it would be done in the blink of an eye
(since compiling the RegEx into a DFA can also be done in linear time). Lua however uses the naive implementation described above and as such
will take quadratic time as it tries matching at every position from `1` to `1e6`, succeeding & matching `1e6` to `1` `a`s on average, but then failing on the `b` (or end-of-string).

This means that rather than taking `O(n + m)` time for a string search with haystack of length `n` and needle of length `m`,
Lua takes `O(nm)` time in the worst case, as it implements a naive string search.

### Exponential Time

Consider a slight modification:

```lua
assert(not ("a"):rep(100):match(("a*"):rep(10) .. "b"))
```

note how both string and pattern have been shortened significantly. A decent RegEx engine would again manage matching in linear time.

But as described above, pattern matching sees an `a*` and now has two options: Match `a` zero times, advance the pattern but not the position in the search string
or match `a` once, advancing the position both in pattern & search string (the latter option is tried first, since `*` is greedy), rinse and repeat, for every single `a*`,
until the `b` lets the match fail (the `b` is required as otherwise we'd run into the best case where the first `a*` matches the entire string and everything early returns since a match has been found).

## List of Shame

By no means exhaustive.

* Lua (bonus: will happily scream "pattern too complex" at you when it should rather error with "implementation too simple")
* JavaScript (too powerful, the substring issue)
* Python (too powerful)

## Hall of Fame

(obviously all of these tend to the historical conventions surrounding RegEx to some extent)

* Go
* Rust

~~probably exhaustive~~

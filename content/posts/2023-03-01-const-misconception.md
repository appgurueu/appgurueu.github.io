---
layout: post
title:  "On the misconception that const is recursive"
date:   2023-03-01
tags:
- const
- immutable
- javascript
- lua
aliases:
- "../2023/03/01/const-misconception.html"
---

The `const` keyword applies only to variables - not to the contents of variables.
The following is valid JavaScript (Java's `final` or Lua's `local <const>` are analogeous):

```javascript
const p = {x: 1, y: 2};
p.x = 3;
```

since `p` is not assigned to.

```javascript
const p = {x: 1, y: 2};
p = {x: 42, y: 33};
```

would be illegal however.

JavaScript provides `Object.freeze` to make an object immutable (which again only is shallow though!):

```javascript
const p = {x: 1, y: 2};
Object.freeze(p);
p.x = 3;	// illegal
```

however, once again, this is not recursive ("deep"); sub-objects won't be frozen:

```javascript
const p = {x: 1, y: 2, obj: {z: 3}};
Object.freeze(p);
p.obj.z = 4;	// legal
```

if you want a deep freeze, you have to write it yourself:

```javascript
function deepFreeze(obj) {
	if (typeof obj !== "object") return;
	Object.freeze(obj);
	for (const prop in obj) {
		deepFreeze(prop);
		deepFreeze(obj[prop]);
	}
}
```

if you want it to handle cycles:

```javascript
function deepFreeze(obj, frozen) {
	if (typeof obj !== "object") return;
	frozen |= new Set();
	if (frozen.has(obj)) return;
	Object.freeze(obj);
	frozen.add(obj);
	for (const prop in obj) {
		deepFreeze(prop, frozen);
		deepFreeze(obj[prop], frozen);
	}
}
```

(and if you want objects to be immutable by default, switch to a functional language)

---

Update: It occurred to me that a likely reason for this misconception might be coming from statically typed languages,
where `const` is a property of a *type* that may propagate recursively rather than of a *variable*.
The interface of a `const` container type typically differs from that of the mutable container type -
the `const` type only allows you to use methods marked `const` -
and may thus only give you `const` element (reference) types.

For example, say you have the following C++:

```c++
const std::vector<Foo> foos;
```

Then of course `foos` is `const` - you can't add elements to it, remove elements, etc.
But its interiors - the `Foo`s - are also `const` because you're only given `const` references -
say, when calling `foos.back()`.

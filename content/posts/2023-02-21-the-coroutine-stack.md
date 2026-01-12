---
layout: post
title:  "The coroutine stack"
date:   2023-02-21
tags:
- lua
- coroutines
- stack
aliases:
- "../2023/02/21/the-coroutine-stack.html"
---

A coroutine + callstack-based stack of varargs in pure Lua.
Inspired by [this tweet](https://twitter.com/ImogenBits/status/1325424621286518784).

Note: Currently, for the sake of simplicity, only a single stack is implemented.
By parameterizing each function in terms of the `stack`, multiple stacks
could be implemented, but this is not for practical purposes
(in practice, you should usually use tables paired with `table.insert`
and `table.remove` instead, unless your problem lends itself well
to a recursive solution; keep stack overflows in mind though).

```lua
local function new_entry(...)
	local cmd_
	local function process_cmd(cmd, ...)
		cmd_ = cmd
		if cmd == "push" then
			return process_cmd(new_entry(...))
		end
	end
	repeat
		process_cmd(coroutine.yield())
	until cmd_ == "pop"
	return coroutine.yield(...)
end

local stack = coroutine.create(function(cmd, ...)
	while true do
		assert(cmd == "push", "empty stack")
		new_entry(...)
	end
end)

local function push(...)
	assert(coroutine.resume(stack, "push", ...))
end

local function retvals(status, ...)
	local potential_error = ...
	assert(status, potential_error)
	return ...
end

local function pop()
	return retvals(coroutine.resume(stack, "pop"))
end

-- Tests

do
	push(1, 2, 3)
	push(4, 5, 6)
	local a, b, c = pop()
	assert(a == 4 and b == 5 and c == 6)
	a, b, c = pop()
	assert(a == 1 and b == 2 and c == 3)
	for i = 1, 1000 do
		push(i)
	end
	for i = 1000, 1, -1 do
		assert(pop() == i)
	end
end
```

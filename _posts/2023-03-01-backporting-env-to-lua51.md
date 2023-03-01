---
layout: post
title:  "Backporting _ENV to Lua 5.1"
date:   2023-03-01 21:49:00 +0100
tags:
  - lua
  - environment
  - hax
---

Lua 5.2 has replaced `setfenv(stacklevel_or_func, env)` and `getfenv(stacklevel_or_func)` with assignments to / accesses of the implicit `_ENV` upvalue of each function:

The statement `_ENV = env` in a function is equivalent to `setfenv(1, env)`, changing the current function environment to `env`, and the expression `_ENV` is equivalent to `getfenv(1)`, getting the current function environment.

By using `setfenv` and `debug.sethook`, this "feature" can be (superficially) added to all Lua functions: Whenever a function is called, the environment is hooked using a proxy table to provide special treatment for assigning to (`__newindex`) or getting (`__index`) `_ENV`.

Additionally, `setfenv` and `getfenv` must be overidden to not get rid of the environment, but instead provide a metatable for it (alternatively, just don't use them if you want to use `_ENV`).

Putting it all together:

```lua
local function envify(_ENV)
    rawset(_ENV, "_ENV", _ENV)
    return setmetatable({}, {
        __index = function(_, name)
            return _ENV[name]
        end,
        __newindex = function(_, name, value)
            if name == "_ENV" then
                -- ensure self reference
                setfenv(2, envify(value))
                return
            end
            _ENV[name] = value
        end
    })
end

local function add_env(stacklevel)
    stacklevel = (stacklevel or 1) + 1
    local _ENV = getfenv(stacklevel)
    -- Set the environment of the calling function
    setfenv(stacklevel, envify(_ENV))
end

local function try_add_env()
    if not getfenv(4)._ENV then
        add_env(4)
    end
end

local function sethook()
    debug.sethook(function()
    	-- Catch & silence errors when attempting to change the environment of Lua builtins
        pcall(try_add_env)
    end, "c")
end
sethook()

function test()
    _ENV = {_G = _G, assert = assert}
    assert(not math)
    assert(_ENV.assert)
    _ENV = {assert = assert}
    assert(_ENV and not _G)
end

test()
```

See also the other way around: [Implementing `setfenv` in Lua 5.2 and above](https://leafo.net/guides/setfenv-in-lua52-and-above.html) by Leafo.

## Limitations

It is not possible to "create" an `_ENV` upvalue for a given function after it has been loaded.
Thus code working with the `_ENV` upvalue through the debug library
such as Leafo's port of `setfenv` and `getfenv` to Lua 5.2+ won't work.

This is nothing but a demonstration, and not in the slightest suitable for serious programming;
it is likely to introduce confusion since it messes with environments,
and it severely reduces performance, hooking every (env.) variable access and function call.

# lua-resty-mlcache

[![Build Status][badge-travis-image]][badge-travis-url]

Fast and automated multi-level cache for OpenResty.

This library can be can be manipulated as a key/value store caching scalar Lua
types and tables, but is built on top of [lua_shared_dict] and
[lua-resty-lrucache]. This combination allows for extremely performant and
flexible caching.

Features:

- Caching and negative caching with TTLs.
- Built-in mutex via [lua-resty-lock] to prevent dog-pile effects to your
  database/backend on cache misses.
- Built-in inter-workers communication to propagate cache evictions
  and allow workers to update their L1 (lua-resty-lru) caches upon changes
  (`set()`, `delete()`).
- Multiple isolated instances can be created to hold various types of data
  while relying on the *same* lua_shared_dict L2 cache.

Illustration of the various caching levels built into this library:

```
┌─────────────────────────────────────────────────┐
│ Nginx                                           │
│       ┌───────────┐ ┌───────────┐ ┌───────────┐ │
│       │worker     │ │worker     │ │worker     │ │
│ L1    │           │ │           │ │           │ │
│       │ Lua cache │ │ Lua cache │ │ Lua cache │ │
│       └───────────┘ └───────────┘ └───────────┘ │
│             │             │             │       │
│             ▼             ▼             ▼       │
│       ┌───────────────────────────────────────┐ │
│       │                                       │ │
│ L2    │           lua_shared_dict             │ │
│       │                                       │ │
│       └───────────────────────────────────────┘ │
│                           │                     │
│                           ▼                     │
│                  ┌──────────────────┐           │
│                  │     callback     │           │
│                  └────────┬─────────┘           │
└───────────────────────────┼─────────────────────┘
                            │
  L3                        │   I/O fetch
                            ▼

                   Database, API, I/O...
```

The cache level hierarchy is:
- **L1**: Least-Recently-Used Lua-land cache using [lua-resty-lrucache].
   Provides the fastest lookup if populated, and avoids exhausting the workers'
   Lua VM memory.
- **L2**: `lua_shared_dict` memory zone shared by all workers. This level
   is only accessed if L1 was a miss, and prevents workers from requesting the
   L3 cache.
- **L3**: a custom function that will only be run by a single worker
   to avoid the dog-pile effect on your database/backend
   (via [lua-resty-lock]). Values fetched via L3 will be set to the L2 cache
   for other workers to access.

## Table of Contents

- [Synopsis](#synopsis)
- [Installation](#installation)
- [Methods](#methods)
    - [new](#new)
    - [get](#get)
    - [probe](#probe)
    - [set](#set)
    - [delete](#delete)
    - [update](#update)
- [License](#license)

## Synopsis

```
# nginx.conf

http {
    # you do not need to configure the following line when you
    # use LuaRocks or opm.
    lua_package_path "/path/to/lua-resty-mlcache/lib/?.lua;;";

    lua_shared_dict cache 1m;

    init_by_lua_block {
        local mlcache = require "resty.mlcache"

        local cache, err = mlcache.new("cache", {
            lru_size = 500,    -- size of the L1 (Lua-land LRU) cache
            ttl      = 3600,   -- 1h ttl for hits
            neg_ttl  = 30,     -- 30s ttl for misses
        })
        if err then

        end

        -- we put our instance in the global table for brivety in
        -- this example, but prefer an upvalue to one of your modules
        -- as recommended by ngx_lua
        _G.cache = cache
    }

    server {
        listen 8080;

        location / {
            content_by_lua_block {
                local function callback(username)
                    -- this only runs *once* until the key expires, so
                    -- do expansive operations like connecting to a remote
                    -- backend here. i.e: call a MySQL server in this callback
                    return db:get_user(username) -- { name = "John Doe", email = "john@example.com" }
                end

                -- this call will respectively hit L1 and L2 before running the
                -- callback (L3). The returned value will then be stored in L2 and
                -- L1 for the next request.
                local user, err = cache:get("my_key", nil, callback, "John Doe")
                if err then

                end

                ngx.say(user.username) -- "John Doe"
            }
        }
    }
}
```

[Back to TOC](#table-of-contents)

## Installation

TODO

[Back to TOC](#table-of-contents)

## Methods

new
---
**syntax:** `cache, err = mlcache.new(shm, opts?)`

Creates a new mlcache instance. If failed, returns `nil` and a string
describing the error.

The first argument `shm` is the name of the `lua_shared_dict` shared memory
zone. Several instances of mlcache can use the same shm (values will be
namespaced).

The second argument `opts` is optional. If provided, it must be a table
holding the desired options for this instance. The possible options are:

- `lru_size`: a number defining the size of the underlying L1 cache
  (lua-resty-lru instance). This size is the maximal number of items
  that the L1 LRU cache can hold.
  **Default:** `100`.
- `ttl`: a number specifying the expiration time period of the cached
  values. The unit is seconds, but accepts fractional number parts, like
  `0.3`. A `ttl` of `0` means the cached values will never expire.
  **Default:** `30`.
- `neg_ttl`: a number specifying the expiration time period of the cached
  misses (when the L3 callback returns `nil`). The unit is seconds, but
  accepts fractional number parts, like `0.3`. A `neg_ttl` of `0` means the
  cached misses will never expire.
  **Default:** `5`.
- `lru`: a lua-resty-lru instance of your choice. If specified, mlcache
  will not instanciate an LRU. One can use this value to use the
  `resty.lrucache.pureffi` implementation of lua-resty-lru if desired.
- `resty_lock_opts`: options for [lua-resty-lock] instances. When mlcache
  runs the L3 callback, it uses lua-resty-lock to ensure that a single
  worker runs the provided callback.
- `ipc_shm`: if you wish to use [set()](#set) or [delete()](#delete), you
  must specify a second `lua_shared_dict` shared memory zone. This shm will
  be used as a pub/sub backend for eviction events propagation across
  workers. Several mlcache instances can use the same `ipc_shm`.

**Note:** Instances of mlcache can be instantiated without an `ipc_shm` shared
memory zone. However, such instances will not be able to use the [set()](#set)
and [delete()](#delete) methods. This is quite acceptable, as the primary
use-case for this library is to only use [get()](#get), which upon a miss, will
run the L3 callback function and cache the returned value (the callback can
run a database query, for example). The cached value will then expire according
to its `ttl` (or `neg_ttl` if the callback returned a miss).

Example:

```lua
local mlcache = require "resty.mlcache"

local cache, err = mlcache.new("mlcache_shm", {
    lru_size = 1000, -- hold up to 1000 items in the L1 cache (Lua VM)
    ttl      = 3600, -- caches scalar types and tables for 1h
    neg_ttl  = 60    -- caches nil values for 60s,
})
if not cache then
    error("could not create mlcache: " .. err)
end
```

[Back to TOC](#table-of-contents)

get
---
**syntax:** `value, err, hit_level = cache:get(key, opts?, callback, ...)`

Performs a cache lookup. This is the primary and most efficient method of this
module. A typical pattern is to *not* call [set()](#set), and let [get()](#get)
perform all the work.

The first argument `key` is a string. Each value must be stored under a unique
key.

The second argument `opts` is optional. If provided, it must be a table holding
the desired options for this key. These options will supersede the instance's
options:

- `ttl`: a number specifying the expiration time period of the cached
  values. The unit is seconds, but accepts fractional number parts, like
  `0.3`. A `ttl` of `0` means the cached values will never expire.
  **Default:** inherited from the instance.
- `neg_ttl`: a number specifying the expiration time period of the cached
  misses (when the L3 callback returns `nil`). The unit is seconds, but
  accepts fractional number parts, like `0.3`. A `neg_ttl` of `0` means the
  cached misses will never expire.
  **Default:** inherited from the instance.

The third argument `callback` **must** be a function. This function **must**
return a single value (Lua scalar or table). It **can** throw Lua errors as it
runs in protected mode. It can also accept arguments (see below example).

If an error is encountered, this method returns `nil` plus a string describing
the error.

When it succeeds, it returns `value` and no error. **Because `nil` misses from
the L3 callback are cached, `value` can be nil, hence one must rely on the
second return value `err` to determine if this method succeeded or not**.

The third return value is a number which is set if no error was encountered.
It indicated the level at which the value was fetched: `1` for L1, `2` for L2,
and `3` for L3.

When called, `get()` follows those steps:

1. query the L1 cache (lua-resty-lru instance). This cache lives in the
   Lua-land, and as such, it is the most efficient to query.
    1. if the L1 cache has the value, it returns the value.
    2. if the L1 cache does not have the value (L1 miss), it continues.
2. query the L2 cache (lua_shared_dict shared memory zone). This cache is
   shared by all workers, and is less efficient than the L1 cache. It also
   involves serialization for Lua tables.
    1. if the L2 cache has the value, it sets the value in the L1 cache,
       and returns it.
    2. if the L2 cache does not have the value (L2 miss), it continues.
3. creates a [lua-resty-lock], and ensures that a single worker will run
   the callback (other workers trying to access the same value will wait).
4. a single worker runs the L3 callback.
5. the callback returns (ex: it performed a database query), and the worker
   sets the value in the L2 and L1 caches, and returns it.
6. other workers that were trying to access the same value but were waiting
   fetch the value from the L2 cache (they do not run the L3 callback) and
   return it.

Example:

```lua
local mlcache = require "mlcache"

local cache, err = mlcache.new("mlcache_shm", {
    lru_size = 1000
})
if not cache then
    -- ...
end

local function fetch_user(db, user_id)
    local user, err = db:query_user(user_id)
    if err then
        error(err) -- safe to throw
    end

    return user -- table or nil
end

local db      = my_db_connection -- lua-resty-mysql instance
local user_id = 3

local user, err = cache:get("users:" .. user_id, { ttl = 3600 }, fetch_user, db, user_id)
if err then
    ngx.log(ngx.ERR, "could not retrieve user: ", err)
    return
end

-- `user` can be a table, but cal also be `nil` (does not exist)
-- regardless, it will be cached and subsequent calls to get() will
-- return the cached value, for up to `ttl` or `neg_ttl`.
if user then
    ngx.say("user exists: ", user.name)
else
    ngx.say("user does not exists")
end
```

You can create several mlcache instances relying on the same underlying
lua_shared_dict shared memory zone:

```lua
local mlcache = require "mlcache"

local cache_1 = mlcache.new("mlcache_shm", { lru_size = 100 })
local cache_2 = mlcache.new("mlcache_shm", { lru_size = 1e5 })
```

In the above example, `cache_1` is ideal for holding a few, very large values.
`cache_2` can be used to hold a large number of small values. Both instances
will rely on the same shm: `lua_shared_dict mlcache_shm 2048m;`.

[Back to TOC](#table-of-contents)

probe
-----
**syntax:** `ttl, err, value = cache:probe(key)`

Probes the L2 (lua_shared_dict) cache. This method is useful if you want to
know whether a value is cached or not. The source of truth is the L2 cache,
because the L1 cache is more volatile, and the L2 cache is still several
orders of magnitude faster than the L3 (callback) lookup.

As its only intent is to "probe" the cache to determine its warmth for a given
value, it does not count as a query like [get()](#get), and does not set the
probed value in the L1 cache.

The first and only argument `key` is a string, and it is the key to lookup.

This method returns `nil` and a string describing the error upon failure.

Upon success, but if there is no such value for the queried `key`, it returns
`nil` as its first argument, and no error.

Upon success, and if there is such a value for the queried `key`, it returns a
number indicating the remaining TTL of the cached value. The third returned
value in that case will be the cached value itself, for convenience.

Example:

```lua
local mlcache = require "mlcache"

local cache = mlcache.new("mlcache_shm")

local ttl, err, value = cache:probe("key")
if err then
    ngx.log(ngx.ERR, "could not probe cache: ", err)
    return
end

ngx.say(ttl)   -- nil because `key` has no value yet
ngx.say(value) -- nil

-- cache the value

cache:get("key", { ttl = 5 }, function() return "some value" end)

-- wait 2 seconds

ngx.sleep(2)

local ttl, err, value = cache:probe("key")
if err then
    ngx.log(ngx.ERR, "could not probe cache: ", err)
    return
end

ngx.say(ttl)   -- 3
ngx.say(value) -- "some value"
```

[Back to TOC](#table-of-contents)

set
---
**syntax:** `ok, err = cache:set(key, opts?, value)`

Unconditionally sets a value in the L2 cache.

The first argument `key` is a string, and is the key under which to store the
value.

The second argument `opts` is optional, and if provided, is identical to the
one of [get()](#get).

The third argument `value` is the value to cache, similar to the return value
of the L3 callback. Just like the callback's return value, it must be a Lua
scalar, a table, or `nil`.

On failure, this method returns `nil` and a string describing the error.

On success, the first return value will be `true`.

**Note:** methods such as [set()](#set) and [delete()](#delete) require that
other instances of mlcache (from other workers) evict the value from their
L1 (LRU) cache. Since OpenResty currently has no built-in mechanism for
inter-worker communications, this module relies on a polling mechanism via
a `lua_shared_dict` shared memory zone to propagate inter-worker events. If
`set()` or `delete()` are called from a single worker, other workers must call
[update()](#update) before their cache is requested, to make sure they evicted
their L1 value, and that the L2 (fresh value) will be returned.

It is generally considered inefficient to call `set()` on a hot code path
(such as in a request being proxied). Instead, one should rely on [get()](#get)
and its built-in mutex in the L3 callback.

`set()` is better suited when called occasionally from a single worker, upon a
particular event that triggers a cached value to be updated, for example. Once
`set()` updated the L2 cache with the fresh value, other workers will rely on
[update()](#update) to poll eviction events. Calling `get()` on those other
workers thus triggers an L1 miss, but the L2 access will hit the fresh value.

```lua
local mlcache = require "resty.mlcache"

local cache, err = mlcache.new("mlcache_shm", {
    ipc_shm = "mlcache_ipc_shm" -- lua_shared_dict dedicated to inter-worker events _only_
})
if not cache then
    error("could not create mlcache: " .. err)
end

---------------------------
-- in access_by_lua handler
---------------------------
local ok, err = cache:update()
if not ok then
    ngx.log(ngx.ERR, "failed to poll eviction events: ", err)
    return
end

local user, err = cache:get("user" .. user_id, nil, fetch_user_from_db, user_id)
if err then
    ngx.log(ngx.ERR, "failed to get user from cache: ", err)
    return
end

ngx.say("email: ", user.email)

-------------------------------------------------------
-- in response to an invalidation event (e.g. REST API)
-------------------------------------------------------
local ok, err = cache:update("user" .. user_id, nil, new_user)
if not ok then
    ngx.log(ngx.ERR, "failed to update user in L2 cache: ", err)
    return
end
```

[Back to TOC](#table-of-contents)

delete
------

[Back to TOC](#table-of-contents)

update
------

[Back to TOC](#table-of-contents)

## TODO

- `set()` should set in L1 cache as will for the caller worker
- Lack of ipc shm should throw an error (or be allowed under the caveat that
  other workers do not see the change in their L1)
- Rename "invalidations" to "evictions"
- Document the necessity for an shm polling mechanism and future compatibility
  plans.

## License

Work licensed under the MIT License.

[Back to TOC](#table-of-contents)


[lua-resty-lock]: https://github.com/openresty/lua-resty-lock
[lua-resty-lrucache]: https://github.com/openresty/lua-resty-lrucache
[lua_shared_dict]: https://github.com/openresty/lua-nginx-module#lua_shared_dict

[badge-travis-url]: https://travis-ci.org/thibaultcha/lua-resty-mlcache
[badge-travis-image]: https://travis-ci.org/thibaultcha/lua-resty-mlcache.svg?branch=master

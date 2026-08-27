# 06 · Performance at Scale

Module 6 of Level 3 covered *measuring* performance. This module covers
the most common *fix* once you've measured and found a bottleneck:
caching — storing the result of expensive work so you don't repeat it.
The core idea is always the same trade: spend memory (or a cache
server's memory) to save CPU time or a round-trip.

## Memoization — caching within one object's lifetime

The simplest cache is just a hash, keyed by input, checked before doing
the real work:

```ruby
def fib_uncached(n)
  n < 2 ? n : fib_uncached(n - 1) + fib_uncached(n - 2)
end

class MemoFib
  def initialize
    @cache = {}
  end

  def call(n)
    @cache[n] ||= n < 2 ? n : call(n - 1) + call(n - 2)
  end
end
```

```ruby
require 'benchmark'

t1 = Benchmark.realtime { fib_uncached(28) }
puts "Uncached fib(28): #{(t1 * 1000).round(2)}ms"

memo = MemoFib.new
t2 = Benchmark.realtime { memo.call(28) }
puts "Memoized fib(28) first call: #{(t2 * 1000).round(2)}ms"

t3 = Benchmark.realtime { memo.call(28) }
puts "Memoized fib(28) second call: #{(t3 * 1000).round(4)}ms"
```

Captured output:

```text
Uncached fib(28): 28.01ms
Memoized fib(28) first call: 0.01ms
Memoized fib(28) second call: 0.001ms
```

The **naive recursive** version recomputes the same sub-values
exponentially many times (`fib(26)` gets computed separately inside both
`fib(27)` and `fib(28)`'s call trees). The **memoized** version's `@cache`
means each `n` is computed exactly once ever for that object's lifetime
— nearly 3,000x faster on this input, and the second call is
effectively free since everything needed is already cached.

`||=` is the idiom doing the work: `@cache[n] ||= expensive_thing` only
evaluates `expensive_thing` if `@cache[n]` is currently `nil`/`false`,
otherwise it short-circuits and returns the cached value directly.

## TTL caching — for values that go stale

Memoization assumes a value never changes once computed. Data that
changes over time (a weather API result, an exchange rate) instead needs
a **time-to-live (TTL)**: cache it, but only for a bounded window, then
recompute:

```ruby
class TTLCache
  Entry = Struct.new(:value, :expires_at)

  def initialize
    @store = {}
  end

  def fetch(key, ttl: 60)
    entry = @store[key]
    return entry.value if entry && entry.expires_at > Time.now

    value = yield
    @store[key] = Entry.new(value, Time.now + ttl)
    value
  end
end

cache = TTLCache.new
calls = 0
3.times do
  result = cache.fetch("weather", ttl: 100) { calls += 1; "sunny" }
  puts result
end
puts "Block executed #{calls} time(s)"
```

Captured output:

```text
sunny
sunny
sunny
Block executed 1 time(s)
```

All three `.fetch` calls return `"sunny"`, but the block (standing in
for an actual API call) only ran **once** — the other two calls were
served from the still-valid cache entry within the 100-second TTL. In
production, this pattern is exactly what a Redis-backed cache
(`Rails.cache.fetch("weather", expires_in: 100) { ... }`) does, just
with the cache store living outside the process so multiple app servers
share one cache instead of each having its own.

## Where to cache — three common layers

- **In-process memoization**: fastest, but private to one process — a
  second web worker process has its own separate cache, so it doesn't
  help under load-balanced multi-process deployment.
- **Shared cache (Redis/Memcached)**: one cache all app processes/servers
  read from, essential once you run more than one process — the
  `TTLCache` above, but backed by a server instead of an in-process
  hash.
- **HTTP/CDN caching**: caching a whole response (`Cache-Control`
  headers) so a repeat request never even reaches your app process —
  the cheapest possible cache hit, appropriate for content that's
  identical for every user (a public blog post, not a personalized
  dashboard).

## Cache invalidation — the hard part

Caching is easy; knowing *when* a cached value is wrong is the hard part
("there are only two hard things in computer science: cache
invalidation and naming things"). Two common strategies:

- **Time-based (TTL)**: shown above — simple, always eventually correct,
  but can serve stale data for up to the TTL window.
- **Event-based invalidation**: explicitly delete/update the cache entry
  the moment the underlying data changes (e.g. `cache.delete("user:#{id}")`
  right after `user.save`) — always fresh, but requires finding *every*
  place the underlying data can change and remembering to invalidate
  there too, which is easy to miss in a large codebase.

## Performance-at-scale-specific traps

- **Caching a value that depends on hidden context.** Caching the result
  of `current_user.dashboard_data` keyed only by `"dashboard"` (forgetting
  to include the user's id in the key) serves one user's data to every
  other user — always include every input the result actually depends on
  in the cache key.
- **A cache that never expires and is never invalidated** slowly
  accumulates stale data as the underlying source changes — always have
  *some* eviction strategy (TTL, explicit invalidation, or a bounded
  size with LRU eviction), even a generous one.
- **Caching exceptions/errors accidentally.** A memoization pattern like
  `@cache[key] ||= fetch_from_api` will re-run `fetch_from_api` on every
  call if it raises (good — errors shouldn't be cached), but a variant
  that catches the error and caches `nil` as "the result" silently keeps
  reporting failure as if it were a valid empty result.
- **Unbounded in-process memoization on a long-lived process** (a
  background worker running for days) can leak memory if the cache key
  space is effectively infinite (e.g. keyed by a unique request ID) —
  memoization needs bounded key space or an LRU cap, not just "cache
  everything forever."
- **Assuming a cache hit is always fast.** A remote cache (Redis over the
  network) still has latency — for extremely hot, small, read-heavy data,
  an in-process cache in front of the remote cache (a two-tier cache) is
  sometimes worth the added invalidation complexity.

## Cheat sheet

| Pattern | Use when | Ruby idiom |
|---|---|---|
| Memoization | Same input always produces the same output, for this process's lifetime | `@cache[key] ||= compute` |
| TTL cache | Value can go stale, tolerable staleness window | `store expires_at`, check before reuse |
| Shared cache (Redis) | Multiple processes need the same cache | `Rails.cache.fetch(key, expires_in: n) { ... }` |
| Event-based invalidation | Freshness matters more than simplicity | `cache.delete(key)` at the point of mutation |
| HTTP caching | Same response for everyone, cacheable at the edge | `Cache-Control` response header |

## Exercise

1. Add a `max_size` option to `TTLCache` that evicts the oldest entry
   (by insertion order) once the store exceeds `max_size` keys —
   demonstrate inserting 4 keys into a cache with `max_size: 3` and show
   the first key is gone.
2. Write a `memoized_expensive_query(user_id)` method wrapping a
   simulated slow ActiveRecord call (`sleep(0.05)` standing in for
   the query), and benchmark calling it 5 times with the same
   `user_id` versus 5 times with 5 different `user_id`s — explain in a
   comment why the timings differ the way they do.
3. Implement event-based invalidation: a `UserCache` with `fetch(id)`
   and `invalidate(id)`, and a fake `update_user(id, attrs)` function
   that calls `invalidate(id)` right after making a change — show a
   `fetch` after `update_user` recomputing instead of returning stale
   data.

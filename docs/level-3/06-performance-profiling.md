# 06 · Performance & Profiling

"It's slow" is not a diagnosis. Before optimizing anything, you need
numbers: how long does this actually take, how does it scale, and where
is the time going. Ruby's standard library ships `benchmark` for timing
and the `benchmark-ips` gem for throughput comparisons; `GC.stat` gives
you visibility into allocation pressure without any extra gem at all.

## Benchmark.bm — comparing wall-clock time

```ruby
require 'benchmark'

N = 100_000

Benchmark.bm(20) do |x|
  x.report("string +=") do
    s = +""
    N.times { s += "a" }
  end
  x.report("string <<") do
    s = +""
    N.times { s << "a" }
  end
end
```

Captured output:

```text
                           user     system      total        real
string +=              0.323240   0.084546   0.407786 (  0.432337)
string <<              0.007203   0.000169   0.007372 (  0.007466)
```

`+=` on a string allocates a brand-new string object every single
iteration and copies the old contents into it — O(n²) total work across
the loop. `<<` mutates the existing string buffer in place — this run
shows it roughly **58x faster** for 100,000 iterations. `bm(20)` is just
the label column width; `user`/`system`/`total`/`real` are the same four
timing columns you'd get from the Unix `time` command.

## benchmark-ips — throughput with statistical confidence

Wall-clock timing on a single run is noisy — background processes, GC
pauses, and CPU frequency scaling all jitter the result. `benchmark-ips`
runs your code repeatedly, reports iterations-per-second with an error
margin, and can directly compare two approaches:

```ruby
require 'benchmark/ips'

Benchmark.ips do |x|
  x.report("map+sum") { (1..1000).map { |n| n * 2 }.sum }
  x.report("inject")  { (1..1000).inject(0) { |acc, n| acc + n * 2 } }
  x.compare!
end
```

Captured output:

```text
Warming up --------------------------------------
             map+sum   451.000 i/100ms
              inject   335.000 i/100ms
Calculating -------------------------------------
             map+sum     17.816k (±25.7%) i/s   (56.13 μs/i) -     89.298k in   5.012308s
              inject     22.177k (± 9.1%) i/s   (45.09 μs/i) -    111.220k in   5.015069s

Comparison:
 inject:    22177.2 i/s
map+sum:    17815.7 i/s - same-ish: difference falls within error
```

Note the **±25.7%** error margin on `map+sum` in this run — `compare!`
correctly reports "same-ish" rather than declaring a winner, because the
confidence intervals overlap. Never trust a benchmark comparison without
checking the error bars; a single number without a margin is not enough
to declare one approach faster.

## Watching allocations with GC.stat

You don't need a gem to see how many objects your code allocates —
`GC.stat` exposes counters straight from Ruby's garbage collector:

```ruby
before = GC.stat[:total_allocated_objects]
arr = Array.new(100_000) { |i| i.to_s }
after = GC.stat[:total_allocated_objects]
puts "Objects allocated: #{after - before}"
```

```text
Objects allocated: 100006
```

100,000 `to_s` calls each allocate one new String object, plus a handful
of overhead objects from the `Array.new` block machinery itself. Watching
this number before/after a change is a cheap way to catch an
accidentally-quadratic allocation pattern before it shows up as a
production memory problem.

## Common Ruby hotspots

- **String concatenation in a loop** (`+=`) — use `<<` or build an array
  and `.join` at the end.
- **Repeated regex compilation** — a regex literal written inline inside
  a hot loop is recompiled on every pass in older Rubies; hoist it to a
  constant (`REGEX = /.../ `) outside the loop.
- **N+1 database queries** — covered in the ActiveRecord module; the
  single most common real-world performance bug in Ruby web apps.
- **Unnecessary intermediate arrays** — `arr.map { ... }.select { ... }`
  builds two full arrays; `each_with_object` or lazy enumerators
  (`arr.lazy.map { ... }.select { ... }.first(10)`) can avoid materializing
  intermediate results you'll immediately filter down.
- **Symbol vs. String hash keys at scale** — symbols are interned (one
  object reused everywhere); using them as hash keys avoids the
  allocation cost of a fresh string every time you build a similar hash.

## Reading a profiler's output (conceptually)

Gems like `stackprof` sample the call stack thousands of times per
second and report which method was executing at each sample — the
method with the most samples is where time is actually going. The
report format you'll typically see ranks methods by "self time"
(time in that method, excluding calls it made) versus "total time"
(including everything it called). Optimize self-time hotspots first —
a method with high total time but low self time is usually just a thin
wrapper around the real bottleneck further down the stack.

## Performance-specific traps

- **Micro-benchmarking the wrong thing.** A tight loop benchmark can show
  a 10x difference that's completely invisible in a real request, because
  the real bottleneck is a network call or database query that dwarfs
  both options. Profile the real workload before chasing a micro-benchmark
  win.
- **Ignoring the error margin.** `benchmark-ips`'s `±` percentage is not
  decoration — a "2x faster" result with ±40% error bars on both sides is
  not a reliable finding, especially on a busy or thermally-throttled
  machine.
- **Optimizing before measuring.** Rewriting "obviously slow" code that
  turns out to run once per request, when the actual bottleneck is
  elsewhere, wastes time and adds complexity for zero real-world gain.
- **Forgetting warm-up.** The JIT/inline caches and object allocation
  pools behave differently on a cold first call versus a warmed-up loop —
  `benchmark-ips`'s "Warming up" phase exists specifically to avoid
  measuring cold-start noise as if it were steady-state performance.
- **GC pauses skewing wall-clock time.** A benchmark that happens to
  trigger a major GC mid-run will look artificially slow for reasons
  unrelated to your code change — running enough iterations (which
  `benchmark-ips` does by default) smooths this out; a single
  `Benchmark.bm` pass does not.

## Cheat sheet

| Task | Code |
|---|---|
| Time a block once | `Benchmark.measure { ... }` |
| Compare a few blocks, one run each | `Benchmark.bm(20) { \|x\| x.report("a") { ... } }` |
| Compare with statistical confidence | `Benchmark.ips { \|x\| x.report("a") { ... }; x.compare! }` |
| Count allocated objects | `GC.stat[:total_allocated_objects]` |
| Force a GC run (rarely needed) | `GC.start` |
| Check current heap size | `GC.stat[:heap_live_slots]` |
| Time a whole script from the shell | `time ruby script.rb` |

## Exercise

1. Benchmark three ways of building a 10,000-element array of squares:
   a `for` loop pushing into an array, `Array.new(10_000) { \|i\| i**2 }`,
   and `(0...10_000).map { \|i\| i**2 }`. Use `Benchmark.bm` and report
   the real-time column for each.
2. Use `benchmark-ips` with `.compare!` to settle which of `Hash#each`
   and `Hash#each_pair` is faster on a 1,000-entry hash (hint: they
   should come out "same-ish" — explain in a comment why that makes
   sense given what each method actually does).
3. Use `GC.stat[:total_allocated_objects]` to measure and compare the
   allocation cost of `arr.select { }.map { }` versus a single
   `arr.each_with_object([]) { }` pass that does both the filter and the
   transform in one loop.

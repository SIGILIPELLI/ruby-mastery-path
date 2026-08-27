# 07 · Concurrency in Ruby

Ruby has had real OS-level threads since forever, but MRI (the reference
Ruby implementation you're almost certainly running) has a **Global VM
Lock (GVL)** — only one thread executes Ruby bytecode at a time, even on
a multi-core machine. That sounds like it defeats the purpose of threads
entirely, but it doesn't: threads still let you overlap I/O waits (network
calls, file reads, sleeping) with other work, because the GVL is released
during blocking I/O. What threads in MRI do *not* give you is true
parallel CPU-bound computation — for that you'd reach for separate
processes or JRuby/TruffleRuby, which don't have a GVL.

## Basic threads

```ruby
threads = 3.times.map do |i|
  Thread.new do
    sleep(0.1)
    puts "Thread #{i} done"
  end
end
threads.each(&:join)
puts "All threads finished"
```

Captured output (order of the first three lines varies run to run):

```text
Thread 2 done
Thread 0 done
Thread 1 done
All threads finished
```

`Thread.new` starts a thread immediately; the block runs concurrently
with the main thread. `join` blocks the caller until that thread
finishes — without it, `puts "All threads finished"` could print before
any worker thread does, since the main thread would race ahead.

## Why the GVL still lets I/O-bound work speed up

Three `sleep(0.1)` calls above finished in roughly 0.1 seconds total, not
0.3 — because `sleep` releases the GVL while waiting, letting the other
threads run during that idle time. The same is true for network requests
and file I/O. This is why threads are genuinely useful in Ruby web
servers: while one request is waiting on a slow database query, the GVL
is free for another thread to make progress on a different request.

## Race conditions — the lost update problem

Concurrent access to shared, mutable state is unsafe even with a GVL,
because the GVL can switch between threads in the middle of a
multi-step operation:

```ruby
balance = 100

def withdraw(current)
  sleep(0.0001)   # simulates work between reading and writing
  current - 10
end

threads = 3.times.map do
  Thread.new { balance = withdraw(balance) }
end
threads.each(&:join)

puts "Balance: #{balance} (expected 70 if no lost updates)"
```

Captured output:

```text
Balance: 90 (expected 70 if no lost updates)
```

All three threads read `balance == 100` before any of them wrote back a
new value, so two of the three withdrawals were silently lost — the
final result reflects only one successful `-10`. This is the "lost
update" race condition: reading, computing, and writing are three
separate steps, and the GVL only guarantees each individual Ruby
bytecode instruction is atomic, not a whole `read-modify-write` sequence.

## Mutex — protecting a critical section

`Mutex` (mutual exclusion lock) ensures only one thread executes a given
block at a time, closing the gap that caused the race above:

```ruby
mutex = Mutex.new
counter = 0

workers = 10.times.map do
  Thread.new do
    1000.times { mutex.synchronize { counter += 1 } }
  end
end
workers.each(&:join)

puts "Counter: #{counter}"
```

```text
Counter: 10000
```

`synchronize` acquires the lock, runs the block, and releases the lock
even if the block raises — every increment happens as an indivisible
unit, so all 10,000 increments across 10 threads land correctly.

## Fibers — cooperative, single-threaded pausing

A `Fiber` is lighter-weight than a `Thread`: it never runs concurrently
with anything, it only pauses and resumes exactly when you tell it to,
via `Fiber.yield` and `.resume`:

```ruby
fiber = Fiber.new do
  puts "Fiber step 1"
  Fiber.yield
  puts "Fiber step 2"
  Fiber.yield
  puts "Fiber step 3"
end

fiber.resume
puts "back in main"
fiber.resume
fiber.resume
```

```text
Fiber step 1
back in main
Fiber step 2
Fiber step 3
```

Each `.resume` runs the fiber until the next `Fiber.yield` (or until it
finishes), then hands control back to whoever called `resume`. Ruby's
`Enumerator` and lazy enumerators are built on Fibers internally — this
is how `each` on an infinite lazy sequence can produce one value at a
time without materializing the whole sequence.

## Threads vs. Fibers vs. Processes

- **Thread**: OS-scheduled, preemptive, good for overlapping I/O waits;
  needs `Mutex` for shared mutable state; still limited by the GVL for
  CPU-bound Ruby code.
- **Fiber**: cooperative, single-threaded, no locking needed because only
  one runs at a time; used to build custom iteration/coroutine patterns,
  not for parallelism.
- **Process** (`fork`, or gems like `parallel`): true OS-level
  parallelism, bypassing the GVL entirely, at the cost of separate
  memory spaces (no shared mutable state without explicit IPC) and
  higher startup overhead.

## Concurrency-specific traps

- **Assuming the GVL makes all Ruby code thread-safe.** It only
  guarantees individual bytecode instructions are atomic — a
  read-then-write sequence like `balance = balance - 10` is several
  bytecode instructions, not one, and can be interleaved exactly as
  shown above.
- **Holding a Mutex across a blocking I/O call.** If a thread calls
  `mutex.synchronize { http_request }`, every other thread waiting on
  that mutex is now blocked for the entire network round-trip —
  defeating the concurrency benefit threads were supposed to provide.
  Keep critical sections as short as possible.
- **Deadlock from lock ordering.** Two threads each holding one mutex
  and waiting on the other's mutex will wait forever. Always acquire
  multiple locks in a consistent, agreed-upon order across your codebase.
- **Forgetting `.join`.** A Ruby process exits once the main thread
  finishes, even if background threads are still running — code relying
  on a spawned thread's side effects needs an explicit `join` (or a
  thread pool with proper shutdown) or it may never complete.
- **Fibers are not thread-safe replacements.** A `Fiber` never runs in
  parallel with anything — using one to try to "parallelize" CPU work
  does nothing for throughput; its value is structuring pausable,
  resumable control flow, not concurrency.

## Cheat sheet

| Task | Code |
|---|---|
| Start a thread | `Thread.new { ... }` |
| Wait for a thread to finish | `thread.join` |
| Get a thread's return value | `thread.value` |
| Protect shared state | `mutex = Mutex.new; mutex.synchronize { ... }` |
| Create a fiber | `Fiber.new { ... }` |
| Pause a fiber | `Fiber.yield` |
| Resume a fiber | `fiber.resume` |
| Fork a real OS process | `Process.fork { ... }` |
| Wait for a forked process | `Process.wait` |

## Exercise

1. Write a script that spawns 5 threads, each fetching a (simulated)
   "URL" by sleeping a different duration (`0.05 * i` seconds) and
   pushing a result string into a shared array — protect the array push
   with a `Mutex` and confirm all 5 results land in the array regardless
   of completion order.
2. Reproduce the lost-update race condition from this module but with 5
   threads doing 5 concurrent `+10` deposits instead of `-10`
   withdrawals; show the buggy total, then fix it with a `Mutex` and show
   the correct total.
3. Write a `Fiber`-based generator that lazily yields the first 5 Fibonacci
   numbers one `.resume` call at a time, printing each as it's produced.

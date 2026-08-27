# 02 · Background Jobs (Sidekiq)

Some work shouldn't happen inside a web request: sending an email,
resizing an image, generating a report. Making the user wait on that
work (or worse, losing it if the request times out) is bad UX. Sidekiq
is the most widely used Ruby background job library — it pushes a job
description onto a Redis-backed queue, and a separate worker process
pulls jobs off that queue and runs them, entirely decoupled from the
web request that enqueued them.

This module doesn't require Redis to be running — the examples use a
tiny in-process stand-in with the identical method shape (`perform_async`
/ `perform`) so you can see the pattern and run the code, without a
Redis dependency. The real Sidekiq API is called out at each step.

## The job class shape

A real Sidekiq job looks like this:

```ruby
class SendWelcomeEmailJob
  include Sidekiq::Job

  def perform(user_email)
    UserMailer.welcome(user_email).deliver_now
  end
end

# Enqueue it from anywhere in your app — returns immediately:
SendWelcomeEmailJob.perform_async("alice@example.com")
```

`perform_async` serializes the arguments (they must be simple JSON-safe
types: strings, numbers, arrays, hashes — not full Ruby objects) into
Redis and returns instantly. A separate Sidekiq worker process, running
independently of your web server, picks the job up and calls `perform`
on a fresh instance of the job class.

## Simulating it without Redis

```ruby
# A tiny in-process stand-in for Sidekiq's queue, used here so the
# example runs without Redis. Real Sidekiq exposes the identical
# perform_async / perform method shape via `include Sidekiq::Job`.
class InlineQueue
  def self.jobs
    @jobs ||= []
  end

  def self.enqueue(klass, *args)
    jobs << [klass, args]
  end

  def self.drain
    jobs.each { |klass, args| Object.const_get(klass).new.perform(*args) }
    jobs.clear
  end
end

class SendWelcomeEmailJob
  # In real Sidekiq: include Sidekiq::Job
  def self.perform_async(*args)
    InlineQueue.enqueue(name, *args)
  end

  def perform(user_email)
    puts "Sending welcome email to #{user_email}"
  end
end

SendWelcomeEmailJob.perform_async("alice@example.com")
puts "Queued #{InlineQueue.jobs.size} job(s), nothing sent yet"
InlineQueue.drain
```

Captured output:

```text
Queued 1 job(s), nothing sent yet
Sending welcome email to alice@example.com
```

`perform_async` only *enqueues* — the "Sending welcome email" line
doesn't print until `InlineQueue.drain` actually runs the job, which
mirrors real Sidekiq: the web request that called `perform_async`
returns to the user immediately, while the email actually sends whenever
a worker process next picks it up (usually milliseconds later, but
decoupled).

## Retries — jobs are expected to fail sometimes

External calls (payment gateways, third-party APIs) fail transiently.
Sidekiq retries a failed job automatically, with exponential backoff, up
to a configurable number of times before giving up and moving it to a
"dead" queue for manual inspection. The idea, simulated here manually:

```ruby
class FlakyJob
  MAX_RETRIES = 3

  def perform(attempt = 1)
    puts "Attempt #{attempt}"
    raise "simulated network error" if attempt < 3
    puts "Succeeded on attempt #{attempt}"
  rescue => e
    if attempt < MAX_RETRIES
      puts "Failed: #{e.message}, retrying..."
      perform(attempt + 1)
    else
      puts "Giving up after #{attempt} attempts"
    end
  end
end

FlakyJob.new.perform
```

Captured output:

```text
Attempt 1
Failed: simulated network error, retrying...
Attempt 2
Failed: simulated network error, retrying...
Attempt 3
Succeeded on attempt 3
```

In real Sidekiq, you don't write this retry loop by hand — a job that
raises is automatically re-enqueued by Sidekiq's middleware with
increasing delay (`sidekiq_options retry: 5` configures the count). The
manual version here exists purely to make the *concept* concrete before
you rely on the framework to do it invisibly.

## Idempotency — the property retries demand

Because a job might run more than once (a retry after a partial failure,
or Sidekiq's at-least-once delivery guarantee under rare failure
conditions), a well-written job must be **idempotent**: running it twice
produces the same end state as running it once.

```ruby
# NOT idempotent — running twice charges the customer twice
def perform(order_id)
  Order.find(order_id).charge_card!
end

# Idempotent — a repeat run is a no-op
def perform(order_id)
  order = Order.find(order_id)
  return if order.charged?
  order.charge_card!
end
```

The guard clause (`return if order.charged?`) is the whole difference —
cheap to write, and the reason production Sidekiq jobs almost always
check "has this already happened?" before doing anything with a
real-world side effect.

## Queues and priority

Sidekiq supports multiple named queues (`default`, `critical`, `low`),
processed with configurable priority:

```ruby
class SendWelcomeEmailJob
  include Sidekiq::Job
  sidekiq_options queue: "low"
end

class ChargeCardJob
  include Sidekiq::Job
  sidekiq_options queue: "critical"
end
```

A worker pool configured to check `critical` before `low` ensures
time-sensitive jobs (charging a card) don't sit behind a backlog of
low-priority ones (sending a marketing email) during a traffic spike.

## Background-job-specific traps

- **Passing whole ActiveRecord objects as arguments instead of IDs.**
  Sidekiq serializes arguments to JSON, so an ActiveRecord object gets
  mangled or fails to serialize; the correct pattern is always
  `SomeJob.perform_async(user.id)`, then `User.find(user_id)` inside
  `perform` — the record might have changed (or been deleted) between
  enqueue and execution, and re-fetching guarantees you see current data.
- **Assuming a job runs exactly once.** Sidekiq's default is
  **at-least-once** delivery — a crash between finishing work and
  acknowledging the job can cause a legitimate re-run. Non-idempotent
  jobs (charging twice, sending duplicate emails) are a very common
  production bug class traced back to this.
- **Long-running jobs blocking a worker slot.** A job that takes 10
  minutes ties up one of a fixed number of worker threads/processes for
  that whole time — break large batch work into smaller enqueued chunks
  instead of one giant job.
- **Forgetting jobs need their own error visibility.** A raised exception
  in a background job doesn't show up anywhere a user or developer would
  normally look (no HTTP response, no browser console) unless you wire up
  error tracking (Sidekiq's built-in retry/dead-queue UI, or a service
  like Sentry) — silent job failures are a classic "why didn't the email
  ever send" investigation.
- **Testing jobs by actually running Sidekiq/Redis in CI** when
  `Sidekiq::Testing.fake!` (or an equivalent inline stub) lets you assert
  a job was enqueued with the right arguments without any Redis
  dependency at all — much faster and more deterministic.

## Cheat sheet

| Task | Sidekiq API |
|---|---|
| Make a class a job | `include Sidekiq::Job` |
| Enqueue a job | `SomeJob.perform_async(arg1, arg2)` |
| Enqueue for later | `SomeJob.perform_in(5.minutes, arg1)` |
| Define the work | `def perform(arg1, arg2); ...; end` |
| Set retry count | `sidekiq_options retry: 5` |
| Assign a queue | `sidekiq_options queue: "critical"` |
| Test without Redis | `Sidekiq::Testing.fake!` |
| Assert enqueued in a test | `expect(SomeJob.jobs.size).to eq(1)` |

## Exercise

1. Extend the `InlineQueue` simulation with a `perform_in(delay, klass,
   *args)` method that stores a "run at" timestamp alongside the job,
   and a `drain(now: Time.now)` that only runs jobs whose scheduled time
   has passed — demonstrate a job scheduled 10 seconds out not running
   when drained immediately.
2. Write an idempotent `ChargeOrderJob` (using a plain in-memory `Order`
   struct with a `charged` boolean) and prove that calling `perform`
   twice only charges once, printing a message either way so you can see
   the guard clause firing on the second call.
3. Simulate the retry-then-give-up path: a job that always fails, retried
   3 times, printing each attempt and a final "moved to dead queue"
   message instead of raising all the way up to crash the process.

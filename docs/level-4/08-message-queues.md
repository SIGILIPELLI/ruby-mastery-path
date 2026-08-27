# 08 · Message Queues

Background jobs (module 2) decouple *when* work happens from the
request that triggered it, but still assume one producer and one
well-known consumer (a job class). Message queues generalize this: a
producer publishes a message without knowing or caring who (if anyone)
consumes it, which is the foundation of loosely-coupled, event-driven
systems (think RabbitMQ, Kafka, AWS SQS/SNS in production — the concepts
below apply to all of them, demonstrated here with Ruby's built-in
`Queue` and a hand-rolled pub/sub, so nothing external needs to run).

## Point-to-point — one message, one consumer

```ruby
require 'thread'

class InMemoryQueue
  def initialize
    @queue = Queue.new
  end

  def publish(message)
    @queue << message
  end

  def subscribe
    loop do
      msg = @queue.pop
      break if msg == :stop
      yield msg
    end
  end

  def size = @queue.size
end

q = InMemoryQueue.new
q.publish({ event: "order_placed", order_id: 1 })
q.publish({ event: "order_placed", order_id: 2 })
puts "Queue size before consuming: #{q.size}"

consumed = []
q.publish(:stop)
q.subscribe { |msg| consumed << msg }
puts "Consumed #{consumed.size} messages"
consumed.each { |m| puts m.inspect }
```

Captured output:

```text
Queue size before consuming: 2
Consumed 2 messages
{event: "order_placed", order_id: 1}
{event: "order_placed", order_id: 2}
```

Ruby's `Queue` (from the standard library, thread-safe by design) is
literally a FIFO buffer — `<<`/`push` adds, `pop` blocks until something
is available. Real message brokers add persistence (messages survive a
crash), delivery guarantees, and network transport between separate
processes/machines, but the mental model — a durable buffer between
producer and consumer — is identical.

## Publish/subscribe — one message, many consumers (fan-out)

A queue delivers each message to exactly one consumer. **Pub/sub**
broadcasts each message to every subscriber — useful when multiple,
independent parts of a system all care about the same event without
knowing about each other:

```ruby
class Topic
  def initialize
    @subscribers = []
  end

  def subscribe(&handler)
    @subscribers << handler
  end

  def publish(message)
    @subscribers.each { |handler| handler.call(message) }
  end
end

topic = Topic.new
topic.subscribe { |msg| puts "EmailService got: #{msg}" }
topic.subscribe { |msg| puts "AnalyticsService got: #{msg}" }
topic.publish("user_signed_up")
```

Captured output:

```text
EmailService got: user_signed_up
AnalyticsService got: user_signed_up
```

One `publish("user_signed_up")` reaches both subscribers. Neither
subscriber knows the other exists — adding a third subscriber (say, a
fraud-detection service) later requires zero changes to the code that
publishes the event. This is the architectural payoff: producers and
consumers are decoupled, so the system can grow new consumers of an
existing event stream without touching the producer at all.

## Real-world queue/broker vocabulary

- **Producer**: the code that publishes a message.
- **Consumer**: the code that receives and processes a message.
- **Queue**: a point-to-point buffer — each message goes to one
  consumer (even with multiple consumers competing for messages, for
  load-balancing).
- **Topic/exchange**: a pub/sub channel — each message goes to every
  subscriber.
- **Acknowledgment (ack)**: a consumer tells the broker "I successfully
  processed this" — an unacknowledged message gets redelivered,
  which is why consumers need to be idempotent (same principle as
  background jobs).
- **Dead-letter queue**: where messages that fail processing repeatedly
  end up, for manual inspection, instead of being silently dropped or
  retried forever.

## When to reach for a message queue over a background job

Background jobs (Sidekiq) are usually enough when you have one
well-known job class doing one thing, triggered from your own app.
Reach for a real message broker (RabbitMQ, Kafka, SQS/SNS) when:

- **Multiple independent services** (possibly in different languages,
  different codebases, different teams) need to react to the same
  event.
- **You need durability across service restarts** at a scale beyond what
  a single Redis-backed job queue comfortably handles.
- **You need strict ordering guarantees or exactly-once semantics** —
  Kafka in particular is built around ordered, replayable event logs,
  which Sidekiq's job queue is not designed to be.

## Message-queue-specific traps

- **Assuming pub/sub delivery is synchronous and error-visible.** If a
  subscriber's handler raises, in the naive `Topic` implementation
  above, it stops *all* remaining subscribers from being notified for
  that message — real brokers isolate subscriber failures from each
  other, which a hand-rolled version needs explicit `rescue` per handler
  to replicate.
- **Treating "published" as "processed."** A producer that publishes a
  message and considers its job done has no guarantee the message was
  ever successfully consumed — that's specifically what acknowledgments
  and dead-letter queues exist to make visible.
- **Message schema drift with no versioning.** If a producer starts
  publishing a message shape a consumer doesn't expect (a renamed or
  removed field), consumers written against the old shape start failing
  silently or throwing — the same discipline from the API-versioning
  module applies to message payloads.
- **Non-idempotent consumers**, exactly as in the background-jobs
  module — at-least-once delivery is the common default for most
  brokers, so a consumer that isn't safe to run twice on the same
  message will eventually double-process something.
- **Unbounded queue growth when consumers fall behind producers.** If
  publishing is much faster than consuming, an unbounded queue grows
  without limit — real systems set backpressure (rate-limiting
  producers) or bounded queue sizes rather than letting memory grow
  indefinitely.

## Cheat sheet

| Concept | Ruby stand-in used here | Real-world equivalent |
|---|---|---|
| Point-to-point queue | `Queue.new`, `push`/`pop` | SQS, a Sidekiq/Redis queue |
| Pub/sub fan-out | `Topic#subscribe`/`#publish` | RabbitMQ exchange, SNS topic |
| Acknowledgment | (not modeled — `pop` is destructive) | Broker-level `ack`/`nack` |
| Dead-letter queue | (not modeled) | Broker's DLQ feature |
| Idempotent consumer | Guard clause before side effects | Same principle, any broker |

## Exercise

1. Extend `Topic` with an `unsubscribe(handler)` method, and demonstrate
   a subscriber that unsubscribes itself after receiving one message
   (a "notify me once" pattern).
2. Add error isolation to `Topic#publish`: wrap each subscriber's call in
   a `begin/rescue`, print `"Subscriber failed: #{e.message}"` for a
   raising subscriber, and prove the *other* subscribers still get
   called even when one fails.
3. Build a small "order processing" pub/sub scenario: one `Topic`, three
   subscribers (`InventoryService`, `EmailService`, `AnalyticsService`),
   each printing a different message when an `"order_placed"` event
   fires — then simulate a schema change (the event payload gaining a
   new required field) and show which subscriber(s) break versus which
   ones tolerate the addition gracefully.

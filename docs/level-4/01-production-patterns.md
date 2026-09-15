---
description: "Rails/Production Patterns — Rails controllers and ActiveRecord models are convenient defaults, but a growing app quickly accumulates business logic that…"
---

# 01 · Rails/Production Patterns

Rails controllers and ActiveRecord models are convenient defaults, but a
growing app quickly accumulates business logic that doesn't belong in
either — a controller stuffed with 40 lines of branching logic, or a
model with a dozen unrelated callbacks. **Service objects** are the most
common pattern production Rails/Sinatra apps reach for: a plain Ruby
object with one job, one public entry point, and an explicit result.

## The problem service objects solve

Imagine a money transfer between two accounts. It's tempting to put this
directly in a controller action or as an `Account` instance method, but
it genuinely involves *two* accounts, needs to be atomic, and can fail
for multiple distinct reasons — none of which is naturally "one
account's" responsibility alone.

## A service object

```ruby
class TransferMoney
  Result = Struct.new(:success?, :error)

  def initialize(from:, to:, amount:)
    @from = from
    @to = to
    @amount = amount
  end

  def call
    return Result.new(false, "amount must be positive") unless @amount.positive?
    return Result.new(false, "insufficient funds") if @from.balance < @amount

    ActiveRecord::Base.transaction do
      @from.decrement!(:balance, @amount)
      @to.increment!(:balance, @amount)
    end

    Result.new(true, nil)
  end
end
```

```ruby
a = Account.create!(name: "Alice", balance: 100)
b = Account.create!(name: "Bob", balance: 20)

result = TransferMoney.new(from: a, to: b, amount: 30).call
puts "Success: #{result.success?}"
puts "Alice: #{a.reload.balance}, Bob: #{b.reload.balance}"

bad = TransferMoney.new(from: a, to: b, amount: 1000).call
puts "Success: #{bad.success?}, error: #{bad.error}"
```

Captured output:

```text
Success: true
Alice: 70, Bob: 50
Success: false, error: insufficient funds
```

The controller (or Sinatra route) calling this ends up as just:

```ruby
result = TransferMoney.new(from: source, to: dest, amount: params[:amount].to_i).call
if result.success?
  # render success
else
  # render result.error
end
```

No branching business logic lives in the route at all — it just reports
whatever the service decided.

## Anatomy of a good service object

- **One public method**, conventionally `call` — the class name itself
  is the verb (`TransferMoney`, not `TransferMoneyService`'s vague
  `execute`), so `TransferMoney.new(...).call` reads like an instruction.
- **Explicit inputs via keyword arguments** in `initialize` — no reaching
  into global state or ambient params hashes inside the service.
- **An explicit result object**, not a raw boolean or a raised exception
  for expected failure cases. Reserve real exceptions for genuinely
  unexpected failures (a database connection drop), not for "insufficient
  funds," which is a normal, anticipated outcome the caller needs to
  branch on.
- **A database transaction wrapping any multi-step write** — `decrement!`
  and `increment!` are two separate `UPDATE` statements; without
  `ActiveRecord::Base.transaction`, a crash between them leaves the two
  accounts' balances permanently inconsistent.

## `.call` as a class method convenience

A common shorthand adds a class method so callers don't need to chain
`.new(...).call`:

```ruby
class TransferMoney
  def self.call(...) = new(...).call
  # ...
end

TransferMoney.call(from: a, to: b, amount: 30)
```

`...` (the "argument forwarding" operator) passes whatever arguments the
class method received straight through to `new`, without having to
re-list keyword names.

## Other production-shaped patterns worth knowing

- **Form objects**: like a service object, but modeling a form's
  multi-model input/validation (e.g. signing up creates both a `User`
  and a `Profile` from one submitted form) as a single object with its
  own `valid?`/`errors`, so the controller doesn't have to coordinate
  two models' validations by hand.
- **Query objects**: a class wrapping one complex, reused ActiveRecord
  query (`OverdueInvoices.new.call` returning a scope) instead of
  copy-pasting a multi-line `where`/`joins` chain across several
  controllers.
- **Presenter/decorator objects**: wrapping a model to add
  view-specific formatting methods (`invoice.formatted_total`) without
  polluting the model itself with display logic it shouldn't know
  about.

All of these share the same shape: pull one specific, reusable
responsibility out of a controller or a bloated model into its own small,
testable class.

## Production-pattern-specific traps

- **Turning every method into a service object** is over-engineering — a
  one-line lookup doesn't need a class wrapper. Reach for a service
  object when logic branches, touches multiple models, or needs to be
  reused from more than one entry point (a controller and a background
  job, for instance).
- **Silently swallowing the transaction's rollback.** If code inside
  `ActiveRecord::Base.transaction` raises, the whole block rolls back —
  but only if the exception propagates out of the block. A `rescue`
  inside the transaction block that swallows the error without
  re-raising commits a **partial** transaction as if nothing failed.
- **Result objects that are just "success or raise"** in disguise don't
  actually give the caller a real branch point — if `call` still raises
  for validation failures instead of returning a `Result`, you haven't
  actually gained anything over a raw method call.
- **Passing whole `params` hashes into a service** re-creates the mass
  assignment problem from the ActiveRecord module — services should take
  named, already-validated arguments, not an unfiltered request payload.
- **Testing services by mocking ActiveRecord itself** rather than using a
  real (in-memory) test database makes tests brittle to any internal
  refactor of the model layer; prefer testing against a real, disposable
  database as shown in this module's example.

## How It Actually Works

A "production pattern" like a connection pool or circuit breaker is
ultimately just careful management of two things MRI gives you directly:
threads/fibers for concurrency and objects for state. A connection pool
(e.g., `ConnectionPool.new`) wraps a `Queue` (a thread-safe FIFO built on a
`Mutex` + `ConditionVariable`) holding pre-created connection objects;
checking a connection out pops from the queue (blocking the calling thread
via the condition variable if empty, which releases the GVL exactly like
any blocking wait), and checking it back in pushes it back and wakes one
waiter. A circuit breaker is just a small state machine (closed/open/
half-open) stored in an instance variable, guarded by a `Mutex` so
concurrent threads don't race on the state transition — the "breaking"
itself is nothing more than an `if` that raises early instead of attempting
the wrapped call, informed by counters incremented on each success/failure.
None of this requires anything beyond primitives already covered in
Level 3's concurrency module — production resilience patterns are
compositions of `Mutex`, `Queue`, and plain object state, not a separate
runtime feature.

## Cheat sheet

| Pattern | Responsibility | Typical entry point |
|---|---|---|
| Service object | One business operation, possibly multi-model | `SomeAction.new(...).call` |
| Form object | Multi-model input + validation for one form | `form.valid?`, `form.save` |
| Query object | One reusable, complex query | `SomeQuery.new.call` |
| Presenter/decorator | View-specific formatting over a model | `Presenter.new(model).formatted_x` |
| Result struct | Explicit success/failure + payload/error | `Struct.new(:success?, :error, :data)` |

## Exercise

1. Build a `RegisterUser` service that takes `email:` and `password:`,
   validates the email contains `@` and the password is at least 8
   characters (return a `Result` with an error message otherwise), and
   on success creates a `User` ActiveRecord record and returns
   `Result.new(true, nil, data: user)`.
2. Wrap the `User.create!` call in `ActiveRecord::Base.transaction` even
   though it's a single write, and explain in a code comment why it
   still matters once you add a second write (e.g. creating an
   associated `Profile` record) in a later change.
3. Write three test calls — one missing `@`, one with a short password,
   one fully valid — and print each `Result`'s `success?`/`error`/`data`.

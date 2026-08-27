# 05 · Design Patterns in Ruby

Classic design patterns come from the Gang of Four book, written with
statically-typed languages like C++ and Java in mind. Ruby's dynamic
typing, blocks, mixins, and open classes make several of them nearly free
— sometimes a whole pattern collapses into a single line of idiomatic
Ruby. This module covers five patterns you'll actually see in Ruby
codebases, in their idiomatic Ruby form.

## Singleton — via the standard library

You rarely need to hand-roll the classic "private constructor + class
variable" singleton dance. Ruby ships `singleton` in the standard
library:

```ruby
require 'singleton'

class AppConfig
  include Singleton
  attr_accessor :env
end

AppConfig.instance.env = "production"
puts AppConfig.instance.env
```

```text
production
```

`include Singleton` makes `.new` private and gives you `.instance`,
which lazily creates and memoizes the one instance. Reach for it only
when you truly need process-wide shared state (a config object, a
connection pool) — it's easy to overuse and turns into hidden global
state that complicates testing.

## Observer — via the standard library

```ruby
require 'observer'

class Ticker
  include Observable

  def tick(price)
    changed
    notify_observers(price)
  end
end

class PriceLogger
  def update(price)
    puts "Price is now #{price}"
  end
end

t = Ticker.new
t.add_observer(PriceLogger.new)
t.tick(101.5)
```

```text
Price is now 101.5
```

`changed` marks state as dirty; `notify_observers` then calls `update` on
every registered observer, but only if `changed` was called first — a
deliberate two-step so you can batch multiple internal changes into one
notification.

## Factory — routing construction through one method

```ruby
class ShapeFactory
  def self.create(type, *args)
    case type
    when :circle then Circle.new(*args)
    when :square then Square.new(*args)
    else raise ArgumentError, "unknown shape #{type}"
    end
  end
end

Circle = Struct.new(:radius) { def area = (Math::PI * radius**2).round(2) }
Square = Struct.new(:side)   { def area = side**2 }

puts ShapeFactory.create(:circle, 3).area
puts ShapeFactory.create(:square, 4).area
```

```text
28.27
16
```

The caller never mentions `Circle` or `Square` directly — it asks the
factory for "a circle" and gets back the right object. This pays off
when construction logic grows complicated (picking a class based on
config, feature flags, or environment) and you don't want that logic
duplicated at every call site.

## Decorator — via module prepend

Ruby's `prepend` inserts a module *above* a class in the method lookup
chain, so the module's method runs first and can call `super` to reach
the original — a clean, native way to wrap behavior without editing the
original class:

```ruby
module Loud
  def speak
    super.upcase
  end
end

class Greeter
  prepend Loud
  def speak = "hello"
end

puts Greeter.new.speak
```

```text
HELLO
```

Compare this to the classic Decorator pattern (a wrapper object holding
a reference to the wrapped object and forwarding calls) — `prepend` gets
you the same layering effect with less boilerplate, at the cost of
modifying the method resolution order of the class itself rather than
creating a separate wrapper object.

## Strategy — via a block or lambda

Where Java's Strategy pattern needs an interface and a family of classes
implementing it, Ruby's first-class blocks *are* the strategy:

```ruby
class Payment
  def initialize(&strategy)
    @strategy = strategy
  end

  def pay(amount)
    @strategy.call(amount)
  end
end

credit_card = Payment.new { |amt| "Charged $#{amt} to credit card" }
puts credit_card.pay(50)
```

```text
Charged $50 to credit card
```

Swapping strategies means passing a different block — no class hierarchy
required. Use a real class-based Strategy instead when the "strategy"
needs multiple methods or its own state beyond a single callable.

## When to reach for a pattern (and when not to)

Every pattern above solves a real problem, but Ruby's flexibility makes
it tempting to over-engineer a 20-line script into a "proper" object
model with five collaborating patterns. Rule of thumb: reach for
Factory when construction logic is genuinely complex or varies by
config; reach for Strategy/Decorator when you have a concrete need to
vary or layer behavior; skip Singleton unless you specifically need
enforced global uniqueness (it usually isn't the answer — plain
dependency injection with a regular object is easier to test).

## Design-pattern-specific traps

- **Singleton makes tests fight over shared state.** Because
  `AppConfig.instance` is one object for the whole process, one test
  that sets `.env = "test"` can leak into the next test unless you
  explicitly reset it — a common source of order-dependent test
  failures.
- **`Observable#changed` is easy to forget.** Skipping it means
  `notify_observers` silently does nothing — no error, just no
  observers called, which looks like a broken observer instead of a
  missed one-line call.
- **`prepend` order matters when you prepend multiple modules** — the
  most recently prepended module runs first, which can surprise you if
  you assumed source-order-independent behavior.
- **Overusing Factory for simple `.new` calls** adds an indirection layer
  with no payoff — if there's only ever one class being constructed, a
  factory method is pure ceremony.
- **A Strategy block capturing outer local variables** creates an
  implicit dependency on the enclosing scope that isn't visible at the
  `Payment.new { ... }` call site — fine for a script, worth promoting to
  a real object with explicit dependencies once the block grows past a
  couple of lines.

## Cheat sheet

| Pattern | Ruby idiom |
|---|---|
| Singleton | `include Singleton`, then `Klass.instance` |
| Observer | `include Observable`, `changed` + `notify_observers` |
| Factory | Class method returning different classes by argument |
| Decorator | `prepend SomeModule` + `super` |
| Strategy | Pass a block/lambda instead of a class hierarchy |
| Adapter (bonus) | Wrap an incompatible object, forward calls via `method_missing` or explicit delegation |
| Template method (bonus) | Base class defines the skeleton, calls `NotImplementedError`-raising hook methods subclasses override |

## Exercise

Build a small notification system combining three patterns:

1. A `NotificationCenter` singleton (`include Singleton`) holding a list
   of subscriber callables.
2. A `NotifierFactory.build(:email)` / `build(:sms)` that returns objects
   responding to `.send_message(text)`, each formatting the text
   differently (e.g. `"[EMAIL] text"` vs `"[SMS] text"`).
3. A Strategy-style `retry_with(strategy:)` method that takes a block and
   retries it up to 3 times using the given backoff strategy (a lambda
   mapping attempt number to a sleep duration you print instead of
   actually sleeping).

Wire them together: fetch the singleton, register both notifier types
built through the factory, and broadcast one message through
`retry_with`, printing every step.
